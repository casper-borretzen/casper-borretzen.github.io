---
layout: post
title: "C# Roguelike, devlog #5: Djikstra maps"
---

### Introduction
---

text coming soon..

- [Brett Gildersleeve and Patrick Kenney, Applications of Dijkstra Maps in Roguelikes](https://youtu.be/2ExLEY32RgM)
- [Brian Walker, The Incredible Power of Dijkstra Maps](https://www.roguebasin.com/index.php/The_Incredible_Power_of_Dijkstra_Maps)
- [Derrick S Creamer, Dijkstra Maps Visualized](https://www.roguebasin.com/index.php/Dijkstra_Maps_Visualized)

### Theory
---

text coming soon..

### Implementation
---

{% include folder_tree.html root="Roguelike" content="Roguelike.csproj,src|BspNode.cs|BspTree.cs|Corridor.cs|Map.cs|PathGraph.cs|Program.cs|Rand.cs|Room.cs|Vec2.cs" %}

<div class="block-title">Room.cs:</div>

```diff
    ...

    // Room data
    public bool?[,] area { get; private set; }
+   public List<Vec2> lights { get; private set; } = new List<Vec2>();

    ...

    // Generate room
    private void Generate() {
        for (int y = 0; y < height; y++) 
        {
            for (int x = 0; x < width; x++) 
            {
                if (x == 0 || y == 0 || x == width - 1 || y == height - 1) { area[x,y] = false; }
                else { area[x,y] = true; }
            }
        }
        
+       // Add light at random position
+       Vec2 lightPos = new Vec2(Rand.random.Next(1, width - 2), Rand.random.Next(1, height - 2));
+       if (area[lightPos.x, lightPos.y] == true) { lights.Add(lightPos); }

    }

    ...
```

<div class="block-title">Map.cs:</div>

```diff
    ...

    // Map data
    public BspTree tree { get; private set; }
    private bool?[,] map;
    public readonly PathGraph pathGraph;

+   // Lighting system
+   private List<int> lights = new List<int>();
+   private Dictionary<int, int> lightMap = new Dictionary<int, int>();
+   private Dictionary<int, int> blocksLight = new Dictionary<int, int>();

    ...

+   // Return the light intensity for a given point on the map
+   private int GetLightIntensity(int x, int y)
+   {
+       int coord = MapCoord(x, y);
+       if (lightMap.ContainsKey(coord))
+       {
+           return lightMap[coord];
+       }
+       return 0;
+   }

    // Build the map
    private void BuildMap() {

        // Build all rooms
        tree.VisitAllNodes(BuildRoom);

        // Build all corridors
        tree.VisitAllNodes(BuildCorridor);

+       // Make lightmap
+       lightMap = pathGraph.DijkstraMap(lights, blocksLight, 24);
    }

+   // Add a lightsource to the map
+   public void AddLight(int x, int y)
+   {
+       int coord = MapCoord(x, y);
+       if (!lights.Contains(coord) && map[x, y] == true)
+       { 
+           lights.Add(coord); 
+       }
+   }

    // Add a door to the map
    public void AddDoor(int x, int y)
    {
        int coord = MapCoord(x, y);
        if (map[x, y] == true)
        { 
            // TODO: Add door here
+           blocksLight[coord] = 12;
        }
    }

    // Build room from a node
    private void BuildRoom(BspNode node)
    {
        if (node.HasRoom())
        {
            Room room = node.room;
            BuildSpace(room.x, room.y, room.width, room.height, room.area);

+           // Add lights
+           foreach (Vec2 light in room.lights)
+           {
+               AddLight(room.x + light.x, room.y + light.y);
+           }
        }
    }

    ...

    // Render map as ascii characters
    public void Render() {
    Console.WriteLine("GENERATED MAP " + width.ToString() + "x" + height.ToString());
        for (int y = 0; y < height; y++)
        {
            for (int x = 0; x < width; x++)
            {
                char tileChar = '.';

                if (map[x, y] == true)
                {
                    tileChar = ' ';
                }
                else if (map[x, y] == false)
                {
                    tileChar = '#';
                }
                
+               // Visualize light intensity
+               int lightIntensity = GetLightIntensity(x, y);
+               if (lightIntensity > 16) { Console.BackgroundColor = ConsoleColor.Yellow; }
+               else if (lightIntensity > 8) { Console.BackgroundColor = ConsoleColor.Gray; }
+               else if (lightIntensity > 0) { Console.BackgroundColor = ConsoleColor.DarkGray; }
+               else { Console.BackgroundColor = ConsoleColor.Black; }

                // Write char to console
                Console.Write(tileChar);
            }
            
            // Go to next line
            Console.Write(Environment.NewLine);
        }
    }

    ...
```

<div class="block-title">PathGraph.cs:</div>

```diff
    ...

+   // Dijkstra search to generate a map of weighted values to/from (depends if reverse is null or set) a list of start locations
+   public Dictionary<int, int> DijkstraMap(List<int> start, Dictionary<int, int> costs = null, int? reverse = null)
+   {
+       // Locations to visit
+       PriorityQueue<int, int> toVisit = new PriorityQueue<int, int>();
+       
+       // Dictionary of all reached locations and the previously visited location
+       Dictionary<int, int> cameFrom = new Dictionary<int, int>();
+       
+       // Dictionary of all reached locations and cost to get there
+       Dictionary<int, int> costSoFar = new Dictionary<int, int>();
+       
+       // Add start locations
+       foreach (int location in start) 
+       { 
+           if (!HasLocation(location)) { return null; }
+           toVisit.Enqueue(location, 0);
+           cameFrom.Add(location, location);
+           costSoFar.Add(location, (reverse != null ? (int)(reverse) : 0)); 
+       }
+       
+       // Start search
+       while (toVisit.Count > 0)
+       {
+           int current = toVisit.Dequeue();
+           
+           // Search neighbors
+           foreach (int next in GetLocation(current))
+           {
+               int nextCost = (costs != null ? GetCost(costs, next, 1) : 1);
+               int newCost = (reverse != null ? costSoFar[current] - nextCost : costSoFar[current] + nextCost );
+               if (reverse != null ? (newCost > 0 && (!costSoFar.ContainsKey(next) || newCost > costSoFar[next])) : (!costSoFar.ContainsKey(next) || newCost < costSoFar[next]))
+               {
+                   toVisit.Enqueue(next, newCost);
+                   cameFrom[next] = current;
+                   costSoFar[next] = newCost;
+               }
+           }
+       }
+       
+       // Return the map
+       return costSoFar;
+   }
    
    ...
```

### Conclusion
---

text coming soon..

{% include bash_command.html bash_command="dotnet run" bash_dir="~/Roguelike" %}

[![screenshot](/img/screenshot_2024-06-08-144624.png)](/img/screenshot_2024-06-08-144624.png)

Download the source code: [roguelike-devlog5.zip](/files/roguelike-devlog5.zip)

Find the project on GitHub: [LASER-WOLF/Roguelike](https://github.com/LASER-WOLF/Roguelike)