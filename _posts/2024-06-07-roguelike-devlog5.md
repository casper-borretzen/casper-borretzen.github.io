---
layout: post
title: "C# Roguelike, devlog #5: Djikstra maps"
---

Coming soon..

<!--

<https://www.roguebasin.com/index.php/The_Incredible_Power_of_Dijkstra_Maps>
<https://www.roguebasin.com/index.php/Dijkstra_Maps_Visualized>

> ?.cs

```csharp
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
```
-->