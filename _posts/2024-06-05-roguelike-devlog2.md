---
layout: post
title: "C# Roguelike, Devlog #2: Binary space partitioning (BSP) trees"
---

Coming soon..

<!--
<https://roguebasin.com/index.php/Basic_BSP_Dungeon_generation>

> Map.cs

```csharp
public class Map
{
    public readonly int width;
    public readonly int height;

    // Map data
    public BspTree tree { get; private set; }
    private Tile[,] map;

    // Constructor
    public Map(int width, int height)
    {
        this.width = width;
        this.height = height;
        this.map = new Tile[width, height];
        this.tree = new BspTree(this, width, height);
        BuildMap();
    }
```


> BspTree.cs

```csharp
public class BspTree
{
    // Fields
    public readonly int id;
    public readonly int x;
    public readonly int y;
    public readonly int width;
    public readonly int height;

    // Properties
    public static int count { get; private set; } = 0;

    // Parent
    public readonly Map map;

    // Root node
    public readonly BspNode root;

    // Constructor
    public BspTree(Map map, int width, int height, int x = 0, int y = 0)
    {
        this.id = count;
        count++;
        this.x = x;
        this.y = y;
        this.width = width;
        this.height = height;
        this.map = map;
        
        // Generate all nodes and rooms
        this.root = new BspNode(this, width, height);

        // Print info for all nodes
        VisitAllNodes(NodeInfo);
    }

    
    // Print info for a given node
    private void NodeInfo(BspNode node)
    {
        Console.WriteLine("Node (" + node.id.ToString() + "), parent: " + (node.parent != null ? node.parent.id : "null") + ", sibling: " + (node.GetSibling() != null ? node.GetSibling().id : "null") + ", children[0]: " + (node.children[0] != null ? node.children[0].id : "null") + ", children[1]: " + (node.children[1] != null ? node.children[1].id : "null"));
    }

    // Visit all nodes and call callback when visiting
    public void VisitAllNodes(Action<BspNode> callback)
    {
        VisitNodes(root, callback);
    }

    // Visit current node and all child nodes and call callback method when visiting
    private void VisitNodes(BspNode node, Action<BspNode> callback)
    {
        if (node.children[0] != null) { VisitNodes(node.children[0], callback); }
        if (node.children[1] != null) { VisitNodes(node.children[1], callback); }
        callback(node);
    }

    // Visit all leaves and call callback when visiting
    public void VisitAllLeaves(Action<BspNode> callback)
    {
        VisitLeaves(root, callback);
    }

    // Traverse current node and all child nodes and call callback method when reaching the tree leaves
    private void VisitLeaves(BspNode node, Action<BspNode> callback)
    {
        if (node.children[0] != null) { VisitLeaves(node.children[0], callback); }
        if (node.children[1] != null) { VisitLeaves(node.children[1], callback); }
        if (node.children[0] == null && node.children[1] == null) { callback(node); }
    }

    // Visit left child recursively until leaf found
    public BspNode FindLeftLeaf(BspNode node)
    {
        if (node.children[0] != null) { return FindLeftLeaf(node.children[0]); }
        else{ return node; }
    }
    
    // Visit right child recursively until leaf found
    public BspNode FindRightLeaf(BspNode node)
    {
        if (node.children[1] != null) { return FindRightLeaf(node.children[1]); }
        else{ return node; }
    }

    // Find next leaf to the right of current leaf, return null if at rightmost leaf
    public BspNode LeafToLeafRight(BspNode node)
    {
        while (true)
        {
            bool moveRight = !node.IsSecondChild();
            if (node.parent == null) { return null; }
            else { node = node.parent; }
            if (node.children[1] != null && moveRight)
            {  
                node = node.children[1];
                if (node.children[0] != null) { return FindLeftLeaf(node.children[0]); }
                return node;
            }
        }

    }
}
```


> BspNode.cs

```csharp
namespace Core;

/// <summary>
/// Binary space partitioning (BSP) node for map generation.
/// </summary>
public class BspNode
{
    // Fields
    private static int minSize = 12;
    public readonly int id; 
    public readonly int x;
    public readonly int y;
    public readonly int width;
    public readonly int height;

    // Properties
    public static int count { get; private set; } = 0;

    // Parent tree
    public readonly BspTree tree;

    // Parent node
    public BspNode parent { get; private set;}

    // Children
    public BspNode[] children = { null, null };
    
    // Node data
    public Room room { get; private set; } = null;
    public Corridor corridor { get; set; } = null;
    
    // Private

    // Constructor
    public BspNode(BspTree tree, int width, int height, int x = 0, int y = 0, BspNode parent = null)
    {
        this.id = count;
        count++;
        Logger.Log("Making new node (" + id.ToString() + ")");
        this.tree = tree;
        this.parent = parent;
        this.x = x;
        this.y = y;
        this.width = width;
        this.height = height;

        // Recursivly try to split the node into smaller nodes
        TrySplit();
    }

    // Return node depth
    public int GetLevel() 
    {
        int level = 0;
        BspNode parentCheck = parent;
        while (parentCheck != null)
        {
            parentCheck = parentCheck.parent;
            level++;
        }
        return level;
    }
    
    // Return true if this node has a parent
    public bool HasParent()
    {
        if (parent != null) { return true; }
        return false;
    }
    
    // Return true if this node is the second child
    public bool IsSecondChild()
    {
        if (HasParent()) { if (parent.children[1] == this) { return true; } }
        return false;
    }
   
    // Return the sibling node if this node has a sibling, else returns null
    public BspNode GetSibling()
    {
        if (HasParent())
        {
            if (IsSecondChild()) { return parent.children[0]; }
            else { return parent.children[1]; }
        }
        return null;
    }
    
    // Return true if this node has a corridor
    public bool HasCorridor()
    {
        if (corridor != null) { return true; }
        return false;
    }

    // Return true if this node has a room
    public bool HasRoom()
    {
        if (room != null) { return true; }
        return false;
    }

    // Return true if this node is a leaf
    public bool IsLeaf()
    {
        if (children[0] == null && children[1] == null) { return true; }
        return false;
    }

    // Try to split the node horizontally
    private bool TrySplitHorizontal() 
    {
        int splitX = x + Rand.random.Next((int)(width * 0.25), (int)(width * 0.75));
        int widthChildLeft = splitX - x;
        int widthChildRight = (x + width) - splitX;
        if (widthChildLeft > minSize && widthChildRight > minSize)
        {
            Logger.Log("Splitting node (" + id.ToString() + ") horizontally");
            children[0] = new BspNode(tree, widthChildLeft, height, x, y, parent: this);
            children[1] = new BspNode(tree, widthChildRight, height, splitX, y, parent: this);
            return true;
        }
        return false;
    }

    // Try to split the node vertically
    private bool TrySplitVertical()
    {
        int splitY = y + Rand.random.Next((int)(height * 0.25), (int)(height * 0.75));
        int heightChildLeft = splitY - y;
        int heightChildRight = (y + height) - splitY;
        if (heightChildLeft > minSize && heightChildRight > minSize)
        {
            Logger.Log("Splitting node (" + id.ToString() + ") vertically");
            children[0] = new BspNode(tree, width, heightChildLeft, x, y, parent: this);
            children[1] = new BspNode(tree, width, heightChildRight, x, splitY, parent: this);
            return true;
        }
        return false;
    }

    // Try to split the node
    private void TrySplit() 
    {
        // Set to true if node was split
        bool validSplit = false;
        
        // Decrease likelihood of splitting based on node depth
        bool trySplit = Rand.Percent(100 - Math.Min(25, (GetLevel() * 2)));

        // Randomly pick horizontal or vertical split
        bool dirHor = Rand.Percent(30);
        
        // Try to split the node
        if (trySplit)
        {
            // Try to split the node horizontally, if not possible then try vertically
            if (dirHor)
            { 
                validSplit = TrySplitHorizontal(); 
                if (!validSplit) { validSplit = TrySplitVertical(); }
            }
            
            // Try to split the node vertically, if not possible then try horizontally
            else
            {
                validSplit = TrySplitVertical(); 
                if (!validSplit) { validSplit = TrySplitHorizontal(); }
            }
        }

        // Was not able to split the node, turn the node into a room
        if (!validSplit) { MakeRoom(); }
    }

    // Make a room in this node
    private void MakeRoom()
    {
        room = new Room(this);
    }
}
```


> Room.cs

```csharp
namespace Core;

/// <summary>
/// A room.
/// </summary>
public class Room
{
    // Fields
    private static int minRoomSize = 5;
    public readonly int x;
    public readonly int y;
    public readonly int width;
    public readonly int height;
    
    // Parent node
    public readonly BspNode node;

    // Room data
    public bool?[,] area { get; private set; }
    public List<Vec2> lights { get; private set; } = new List<Vec2>();

    // Constructor
    public Room(BspNode node)
    {
        Logger.Log("Making room in node (" + node.id + ")");

        // Set padding
        int paddingVertical = Rand.random.Next(2, Math.Clamp(node.height - minRoomSize, 2, 10));
        int paddingHorizontal = Rand.random.Next(2, Math.Clamp(node.width - minRoomSize, 2, 10)); 
        int paddingTop = Rand.random.Next((int)(paddingVertical * 0.2), (int)(paddingVertical * 0.8));
        int paddingLeft = Rand.random.Next((int)(paddingHorizontal * 0.2), (int)(paddingHorizontal * 0.8));

        int width = node.width - paddingHorizontal;
        int height = node.height - paddingVertical;

        // Restrict room width/height ratio
        if (height > width) { height = Math.Min((int)(width * 8), width); }
        else { width = Math.Min((int)(height * 8), height); }

        // Set position and size
        this.node = node;
        this.x = node.x + paddingLeft;
        this.y = node.y + paddingTop;
        this.width = width;
        this.height = height;
        this.area = new bool?[width, height];

        // Generate room
        Generate();
    }
    
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
        
        // Add light at random position
        Vec2 lightPos = new Vec2(Rand.random.Next(1, width - 2), Rand.random.Next(1, height - 2));
        if (area[lightPos.x, lightPos.y] == true) { lights.Add(lightPos); }
    }
}
```
-->