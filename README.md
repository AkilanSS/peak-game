peak readme
https://github.com/copilot/share/ca2703a4-0ac0-8084-b853-a404e446011c

def combine_solutions(sub_solutions, boundaries, original_flows):
    """
    Combines solutions from sub-grids into a complete grid.
    
    Args:
        sub_solutions: List of solved sub-grid solutions
        boundaries: List of boundary coordinates between sub-grids
        original_flows: Original flow pairs from the full grid
    
    Returns:
        Combined full grid solution, or None if combining fails
    """
    
    # Step 1: Initialize the full grid by merging sub-grids
    full_grid = merge_sub_grids(sub_solutions, boundaries)
    
    # Step 2: Validate boundary connections
    if not validate_boundaries(full_grid, boundaries, original_flows):
        return None
    
    # Step 3: Identify incomplete flows that span sub-grids
    incomplete_flows = find_incomplete_flows(full_grid, original_flows)
    
    # Step 4: Repair/connect incomplete flows using backtracking
    if not repair_flows(full_grid, incomplete_flows):
        return None
    
    # Step 5: Final validation
    if not validate_complete_solution(full_grid, original_flows):
        return None
    
    return full_grid


def merge_sub_grids(sub_solutions, boundaries):
    """
    Physically merge sub-grid solutions into one grid.
    
    Args:
        sub_solutions: List of 2D grids
        boundaries: List of tuples indicating where each sub-grid sits
                   e.g., [(0, 0, 3, 3), (0, 3, 3, 6)] for top-left and top-right
    
    Returns:
        Full merged grid
    """
    # Determine full grid size
    max_row = max(b[2] for b in boundaries)
    max_col = max(b[3] for b in boundaries)
    full_grid = [[None for _ in range(max_col)] for _ in range(max_row)]
    
    # Place each sub-grid into the full grid
    for (r_start, c_start, r_end, c_end), sub_grid in zip(boundaries, sub_solutions):
        for i, row in enumerate(sub_grid):
            for j, cell in enumerate(row):
                full_grid[r_start + i][c_start + j] = cell
    
    return full_grid


def validate_boundaries(full_grid, boundaries, original_flows):
    """
    Checks that flows are not broken at sub-grid boundaries.
    
    Args:
        full_grid: Merged grid
        boundaries: Sub-grid boundary information
        original_flows: Original flow pairs
    
    Returns:
        True if all boundaries are valid, False otherwise
    """
    
    # For each pair of adjacent sub-grids, check boundary consistency
    for i in range(len(boundaries) - 1):
        boundary1 = boundaries[i]
        boundary2 = boundaries[i + 1]
        
        # Determine if they're horizontally or vertically adjacent
        if boundary1[3] == boundary2[1]:  # Horizontally adjacent (boundary1 on left)
            # Check vertical edge between them
            for row in range(boundary1[0], boundary1[2]):
                cell1 = full_grid[row][boundary1[3] - 1]
                cell2 = full_grid[row][boundary2[1]]
                
                # If one is filled, the other should be the same color or empty
                if cell1 is not None and cell2 is not None:
                    if cell1 != cell2 and cell1 != "EMPTY" and cell2 != "EMPTY":
                        return False  # Color mismatch at boundary
        
        elif boundary1[2] == boundary2[0]:  # Vertically adjacent (boundary1 on top)
            # Check horizontal edge between them
            for col in range(boundary1[1], boundary1[3]):
                cell1 = full_grid[boundary1[2] - 1][col]
                cell2 = full_grid[boundary2[0]][col]
                
                if cell1 is not None and cell2 is not None:
                    if cell1 != cell2 and cell1 != "EMPTY" and cell2 != "EMPTY":
                        return False
    
    return True


def find_incomplete_flows(full_grid, original_flows):
    """
    Identifies flows that are broken or incomplete (span multiple sub-grids).
    
    Args:
        full_grid: Merged grid
        original_flows: Original flow pairs [(start1, end1), (start2, end2), ...]
    
    Returns:
        List of incomplete flows: [(color, start, end), ...]
    """
    incomplete = []
    
    for color, (start, end) in enumerate(original_flows):
        # Check if start and end are connected in the full grid
        if not is_connected(full_grid, start, end, color):
            incomplete.append((color, start, end))
    
    return incomplete


def is_connected(grid, start, end, color):
    """
    Uses BFS to check if start and end are connected with the given color.
    """
    from collections import deque
    
    visited = set()
    queue = deque([start])
    visited.add(start)
    
    while queue:
        row, col = queue.popleft()
        
        if (row, col) == end:
            return True
        
        # Check all 4 neighbors
        for dr, dc in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
            nr, nc = row + dr, col + dc
            if 0 <= nr < len(grid) and 0 <= nc < len(grid[0]):
                if (nr, nc) not in visited:
                    cell = grid[nr][nc]
                    # Cell should be empty or the same color
                    if cell is None or cell == color:
                        visited.add((nr, nc))
                        queue.append((nr, nc))
    
    return False


def repair_flows(full_grid, incomplete_flows):
    """
    Attempts to repair incomplete flows using backtracking.
    
    Args:
        full_grid: Merged grid with potential gaps
        incomplete_flows: List of flows to repair
    
    Returns:
        True if all flows are successfully repaired, False otherwise
    """
    
    # Use backtracking DFS to find paths for incomplete flows
    def backtrack_flows(flow_idx):
        if flow_idx == len(incomplete_flows):
            return True  # All flows repaired
        
        color, start, end = incomplete_flows[flow_idx]
        
        # Try to find a path from start to end for this color
        if find_and_mark_path(full_grid, start, end, color):
            if backtrack_flows(flow_idx + 1):
                return True
            # Backtrack: remove the path we just marked
            unmark_path(full_grid, color)
        
        return False
    
    return backtrack_flows(0)


def find_and_mark_path(grid, start, end, color):
    """
    Uses DFS to find a path from start to end and marks it on the grid.
    """
    visited = set()
    path = []
    
    def dfs(row, col):
        if (row, col) == end:
            # Mark all cells in the path
            for r, c in path:
                grid[r][c] = color
            grid[end[0]][end[1]] = color
            return True
        
        for dr, dc in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
            nr, nc = row + dr, col + dc
            if 0 <= nr < len(grid) and 0 <= nc < len(grid[0]):
                if (nr, nc) not in visited:
                    cell = grid[nr][nc]
                    if cell is None or cell == color or (nr, nc) == end:
                        visited.add((nr, nc))
                        path.append((nr, nc))
                        if dfs(nr, nc):
                            return True
                        path.pop()
        
        return False
    
    return dfs(start[0], start[1])


def unmark_path(grid, color):
    """
    Removes all cells of a specific color from the grid (for backtracking).
    """
    for i in range(len(grid)):
        for j in range(len(grid[0])):
            if grid[i][j] == color:
                grid[i][j] = None


def validate_complete_solution(grid, original_flows):
    """
    Validates that the complete solution satisfies all constraints.
    """
    # Check that every flow is connected
    for color, (start, end) in enumerate(original_flows):
        if not is_connected(grid, start, end, color):
            return False
    
    # Check that there are no overlaps (each cell belongs to at most one flow)
    # (This would depend on Flow Free rules; some variants allow overlaps)
    
    return True

bocchi peak :33
