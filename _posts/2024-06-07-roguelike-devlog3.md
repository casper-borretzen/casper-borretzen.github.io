---
layout: post
title: "C# Roguelike, devlog #3: Pathfinding algorithms (Breath First Search (BFS) and A*)"
---

- [Sebastian Lague, A* Pathfinding (algorithm explanation)](https://youtu.be/-L-WgKMFuhE){:target="_blank"}
- [Amit Patel, Amit’s A* Pages](http://theory.stanford.edu/~amitp/GameProgramming/){:target="_blank"}
- [Amit Patel, Introduction to the A* Algorithm](https://www.redblobgames.com/pathfinding/a-star/introduction.html){:target="_blank"}
- [Amit Patel, Implementation of A*](https://www.redblobgames.com/pathfinding/a-star/implementation.html){:target="_blank"}
- [Amit Patel, Breadth First Search: multiple start points](https://www.redblobgames.com/pathfinding/distance-to-any/){:target="_blank"}


**BspTree.cs**

```csharp
    // Constructor
    public BspTree(Map map, int width, int height, int x = 0, int y = 0)
    {
        // ...

        // Generate pathfinding graph for all the rooms
        AddAllRoomsToPathGraph();

        // Print info for all nodes
        VisitAllNodes(NodeInfo);
    }


    // Add all the rooms to the pathfinding graph
    public void AddAllRoomsToPathGraph()
    {
        BspNode[] nodes = NodeArray();
        foreach (BspNode node in nodes) { if (node.HasRoom()) { map.pathGraph.AddRoom(node.room); }}
    }

    
    // Return an array containing all nodes
    public BspNode[] NodeArray()
    {
        BspNode[] nodeArray = new BspNode[BspNode.count];
        NodeArrayAdd(root, ref nodeArray);
        return nodeArray;
    }
    
    // Traverse all child nodes and add to array
    private void NodeArrayAdd(BspNode node, ref BspNode[] nodeArray)
    {
        nodeArray[node.id] = node;
        if (node.children[0] != null) { NodeArrayAdd(node.children[0], ref nodeArray); }
        if (node.children[1] != null) { NodeArrayAdd(node.children[1], ref nodeArray); }
    }

```

> PathGraph.cs

```csharp
namespace Core;

/// <summary>
/// A data structure that contains visitable locations and its neighbors.
/// Also contains BFS, Djikstra and A* algorithms for pathfinding.
/// </summary>
public class PathGraph
{
    // The map this pathgraph belongs to
    public readonly Map map;

    // Stores a location and its visitable neighbors
    private Dictionary<int, int[]> locations = new Dictionary<int, int[]>();
    
    // Constructor
    public PathGraph(Map map) 
    {
        this.map = map;
    }

    // A* search to check if a valid path exists between start and end locations
    public bool AstarCheck(int start, int target, Dictionary<int, int> costs = null)
    {
        // Check if start/end locations are visitable
        if (!HasLocation(start) || !HasLocation(target)) { return false; }

        // Locations to visit
        PriorityQueue<int, int> toVisit = new PriorityQueue<int, int>();
        toVisit.Enqueue(start, 0);

        // Dictionary of all reached locations and cost to get there
        Dictionary<int, int> costSoFar = new Dictionary<int, int>();
        costSoFar.Add(start, 0);

        // Start search
        while (toVisit.Count > 0)
        {
            int current = toVisit.Dequeue();
            
            // Target found
            if (current == target) { return true; }

            // Search neighbors
            foreach (int next in GetLocation(current))
            {
                int nextCost = (costs != null ? GetCost(costs, next, 1) : 1);
                int newCost = costSoFar[current] + nextCost;
                if (!costSoFar.ContainsKey(next) || newCost < costSoFar[next])
                {
                    toVisit.Enqueue(next, newCost + Heuristic(next, target));
                    costSoFar[next] = newCost;
                }
            }
        }

        // Target not found
        return false;
    }
    
    // A* search to find the shortest path between two locations
    public List<int> AstarPath(int start, int target, Dictionary<int, int> costs = null)
    {
        // Check if start/end locations are visitable
        if (!HasLocation(start) || !HasLocation(target) || start == target) { return null; }

        // Locations to visit
        PriorityQueue<int, int> toVisit = new PriorityQueue<int, int>();
        toVisit.Enqueue(start, 0);

        // Dictionary of all reached locations and the previously visited location
        Dictionary<int, int> cameFrom = new Dictionary<int, int>();
        cameFrom.Add(start, start);

        // Dictionary of all reached locations and cost to get there
        Dictionary<int, int> costSoFar = new Dictionary<int, int>();
        costSoFar.Add(start, 0);

        // Start search
        bool found = false;
        while (toVisit.Count > 0)
        {
            int current = toVisit.Dequeue();
            
            // Target found
            if (current == target) { found = true; break; }

            // Search neighbors
            foreach (int next in GetLocation(current))
            {
                int nextCost = (costs != null ? GetCost(costs, next, 1) : 1);
                int newCost = costSoFar[current] + nextCost;
                if (!costSoFar.ContainsKey(next) || newCost < costSoFar[next]) {
                    toVisit.Enqueue(next, newCost + Heuristic(next, target));
                    cameFrom[next] = current;
                    costSoFar[next] = newCost;
                }
            }
        }

        // Retrace path from target to start
        if (found)
        {
            List<int> path = new List<int>();
            int current = target;
            while (cameFrom[current] != current)
            {
                path.Add(current);
                current = cameFrom[current];
            }
            path.Reverse();

            // Return path from start location to target location
            return path;
        }

        // Target location not found
        return null;
    }

    // Dijkstra search to generate a map of weighted values to/from (depends if reverse is null or set) a list of start locations
    public Dictionary<int, int> DijkstraMap(List<int> start, Dictionary<int, int> costs = null, int? reverse = null)
    {
        // Locations to visit
        PriorityQueue<int, int> toVisit = new PriorityQueue<int, int>();

        // Dictionary of all reached locations and the previously visited location
        Dictionary<int, int> cameFrom = new Dictionary<int, int>();
        
        // Dictionary of all reached locations and cost to get there
        Dictionary<int, int> costSoFar = new Dictionary<int, int>();
        
        // Add start locations
        foreach (int location in start) 
        { 
            if (!HasLocation(location)) { return null; }
            toVisit.Enqueue(location, 0);
            cameFrom.Add(location, location);
            costSoFar.Add(location, (reverse != null ? (int)(reverse) : 0)); 
        }

        // Start search
        while (toVisit.Count > 0)
        {
            int current = toVisit.Dequeue();

            // Search neighbors
            foreach (int next in GetLocation(current))
            {
                int nextCost = (costs != null ? GetCost(costs, next, 1) : 1);
                int newCost = (reverse != null ? costSoFar[current] - nextCost : costSoFar[current] + nextCost );
                if (reverse != null ? (newCost > 0 && (!costSoFar.ContainsKey(next) || newCost > costSoFar[next])) : (!costSoFar.ContainsKey(next) || newCost < costSoFar[next]))
                {
                    toVisit.Enqueue(next, newCost);
                    cameFrom[next] = current;
                    costSoFar[next] = newCost;
                }
            }
        }

        // Return the map
        return costSoFar;
    }

    // Breath First Search (BFS) to check if a valid path exists between start and end locations
    public bool BfsCheck(int start, int target)
    {
        // Check if start/end locations are visitable
        if (!HasLocation(start) || !HasLocation(target)) { return false; }

        // Locations to visit
        Queue<int> toVisit = new Queue<int>();
        toVisit.Enqueue(start);

        // Dictionary of all reached locations and the previously visited location
        Dictionary<int, int> cameFrom = new Dictionary<int, int>();
        cameFrom.Add(start, start);

        // Start search
        while (toVisit.Count > 0)
        {
            int current = toVisit.Dequeue();
            
            // Target found
            if (current == target) { return true; }

            // Search neighbors
            foreach (int next in GetLocation(current))
            {
                if (!cameFrom.ContainsKey(next)) {
                    toVisit.Enqueue(next);
                    cameFrom.Add(next, current);
                }
            }
        }

        // Target not found
        return false;
    }
    
    // Breath First Search (BFS) to find the shortest path between two locations
    public List<int> BfsPath(int start, int target)
    {
        // Check if start/end locations are visitable
        if (!HasLocation(start) || !HasLocation(target) || start == target) { return null; }

        // Locations to visit
        Queue<int> toVisit = new Queue<int>();
        toVisit.Enqueue(start);

        // Dictionary of all reached locations and the previously visited location
        Dictionary<int, int> cameFrom = new Dictionary<int, int>();
        cameFrom.Add(start, start);

        // Start search
        bool found = false;
        while (toVisit.Count > 0)
        {
            int current = toVisit.Dequeue();
            
            // Target found
            if (current == target) { found = true; break; }

            // Search neighbors
            foreach (int next in GetLocation(current))
            {
                if (!cameFrom.ContainsKey(next)) {
                    toVisit.Enqueue(next);
                    cameFrom.Add(next, current);
                }
            }
        }

        // Retrace path from target to start
        if (found)
        {
            List<int> path = new List<int>();
            int current = target;
            while (cameFrom[current] != current)
            {
                path.Add(current);
                current = cameFrom[current];
            }
            path.Reverse();

            // Return path from start location to target location
            return path;
        }

        // Target location not found
        return null;
    }
    
    // Breath First Search (BFS) to generate a map of values to/from (depends if reverse is null or not) a list of start locations
    public Dictionary<int, int> BfsMap(List<int> start, int? reverse = null)
    {
        // Locations to visit
        Queue<int> toVisit = new Queue<int>();

        // Dictionary of all reached locations and the previously visited location
        Dictionary<int, int> cameFrom = new Dictionary<int, int>();
        
        // Dictionary of all reached location and distance to start
        Dictionary<int, int> distanceTo = new Dictionary<int, int>();

        // Add start locations
        foreach (int location in start) 
        { 
            if (!HasLocation(location)) { return null; }
            toVisit.Enqueue(location); 
            cameFrom.Add(location, location);
            distanceTo.Add(location, (reverse != null ? (int)(reverse) : 0)); 
        }

        // Start search
        while (toVisit.Count > 0)
        {
            int current = toVisit.Dequeue();

            // Search neighbors
            foreach (int next in GetLocation(current))
            {
            if (!cameFrom.ContainsKey(next)) {
                if (reverse == null || distanceTo[current] - 1 > 0)
                    {
                        toVisit.Enqueue(next);
                        cameFrom.Add(next, current);
                        distanceTo.Add(next, (reverse != null ? distanceTo[current] - 1 : distanceTo[current] + 1));
                    }
                }
            }
        }

        // Return the map
        return distanceTo;
    }

    // Return heuristic cost for a location
    private int Heuristic(int aCoord, int bCoord)
    {
        Vec2 a = map.MapCoordReverse(aCoord);
        Vec2 b = map.MapCoordReverse(bCoord);
        return Math.Abs(a.x - b.x) + Math.Abs(a.y - b.y);
    }
    
    // Return cost for a location
    public int GetCost(Dictionary<int, int> costs, int location, int defaultCost = 1) 
    { 
        int cost = 0;
        if (costs.TryGetValue(location, out cost))
        { return cost; }
        else { return defaultCost; }
    }

    // Return visitable neighbors for a given location
    public int[] GetLocation(int location) 
    { 
        return locations[location]; 
    }

    // Set visitable neighbors for a given location
    public void SetLocation(int location, int[] neighbors)
    { 
        locations[location] = neighbors;
    }

    // Check if location exists in the graph
    public bool HasLocation(int location)
    {
        return locations.ContainsKey(location);
    }

    // Try to add a new neighbor to a location, only add if location exists and doesn't have the neighbor already
    public void TryAddNeighbor(int location, int neighbor)
    {
        if (HasLocation(location))
        {
            if (!GetLocation(location).Contains(neighbor))
            {
                List<int> neighbors = GetLocation(location).ToList();
                neighbors.Add(neighbor);
                SetLocation(location, neighbors.ToArray());
            }
        }
    }
    
    // Add all visitable locations from a room
    public void AddRoom(Room room, bool modifyNeighbors = false)
    {
        AddArea(room.x, room.y, room.width, room.height, room.area, modifyNeighbors);
    }
    
    // Add all visitable locations from a room
    public void AddCorridor(Corridor corridor, bool modifyNeighbors = false)
    {
        AddArea(corridor.x, corridor.y, corridor.width, corridor.height, corridor.area, modifyNeighbors);
    }
    
    // Add all visitable locations from an area
    public void AddArea(int worldX, int worldY, int width, int height, bool?[,] area, bool modifyNeighbors = false)
    {
        // Create a temporary dictionary for new nodes to be added to the nodes dictionary
        Dictionary<int, int[]> newLocations = new Dictionary<int, int[]>();

        // Loop through the area and check locations
        for (int y = 0; y < height; y++)
        {
            for (int x = 0; x < width; x++)
            {
                if (area[x, y] == true)
                {
                    // Get the current position
                    int thisPosition = map.MapCoord(worldX + x, worldY + y);

                    // Make list to populate with visitable neighbors
                    List<int> neighbors = new List<int>();
                    
                    // Check if neighbor above the current location is a visitable location
                    bool upInBounds = worldY + y > 0;
                    if (upInBounds)
                    {
                        int upPosition = map.MapCoord(worldX + x, worldY + y - 1);
                        bool upPossible = HasLocation(upPosition) ? true : (y > 0 ? area[x, y-1] == true : false);
                        if (upPossible)
                        {
                            neighbors.Add(upPosition);
                            if (modifyNeighbors){ TryAddNeighbor(upPosition, thisPosition); }
                        }
                    }
                    
                    // Check if right-side neighbor is a visitable location
                    bool rightInBounds = worldX + x + 1 < map.width;
                    if (rightInBounds)
                    {
                        int rightPosition = map.MapCoord(worldX + x + 1, worldY + y);
                        bool rightPossible = HasLocation(rightPosition) ? true : (x + 1 < width ? area[x+1, y] == true : false);
                        if (rightPossible)
                        {
                            neighbors.Add(rightPosition);
                            if (modifyNeighbors){ TryAddNeighbor(rightPosition, thisPosition); }
                        }
                    }
                    
                    // Check if neighbor the under current location is a visitable location

                    bool downInBounds = worldY + y + 1 < map.height;
                    if (downInBounds) {
                        int downPosition = map.MapCoord(worldX + x, worldY + y + 1);
                        bool downPossible = HasLocation(downPosition) ? true : (y + 1 < height ? area[x, y+1] == true : false);
                        if (downPossible)
                        {
                            neighbors.Add(downPosition);
                            if (modifyNeighbors){ TryAddNeighbor(downPosition, thisPosition); }
                        }
                    }
                    
                    // Check if left-side neighbor is a visitable location
                    bool leftInBounds = worldX + x > 0;
                    if (leftInBounds)
                    {
                        int leftPosition = map.MapCoord(worldX + x - 1, worldY + y);
                        bool leftPossible = HasLocation(leftPosition) ? true : (x > 0 ? area[x-1, y] == true : false);
                        if (leftPossible)
                        {
                            neighbors.Add(leftPosition);
                            if (modifyNeighbors){ TryAddNeighbor(leftPosition, thisPosition); }
                        }
                    }
                    
                    // Add the current location with its neighbors to the location dictionary
                    newLocations.Add(thisPosition, neighbors.ToArray());
                }
            }
        }
        
        // Add the found nodes to the main nodes dictionary
        foreach (KeyValuePair<int, int[]> location in newLocations) 
        {
            if (!HasLocation(location.Key)) { SetLocation(location.Key, location.Value); }
        }
    }
};
```