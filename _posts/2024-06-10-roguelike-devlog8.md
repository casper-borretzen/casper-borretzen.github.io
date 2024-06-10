---
layout: post
title: "C# roguelike, devlog 8: Mesh generation and cube to sphere projection"
#published: false
#hidden: true
---

![devlog](/img/devlog8.png){:target="_blank"}

### Introduction
---

text coming soon..

- [Amit Patel, Wraparound square tile maps on a sphere](https://www.redblobgames.com/x/1938-square-tiling-of-sphere/){:target="_blank"}
- [Jasper Flick, Cube Sphere: Going from Cube to Sphere](https://catlikecoding.com/unity/tutorials/procedural-meshes/cube-sphere/){:target="_blank"}
- [Zucker, M., & Higashi, Y. (2018): Cube-to-sphere projections for procedural texturing and beyond](https://www.jcgt.org/published/0007/02/01/paper-lowres.pdf){:target="_blank"}
- [Dimitrijević, A., Lambers, M., & Rančić, D. (2016): Comparison of spherical cube map projections used in planet-sized terrain rendering](http://casopisi.junis.ni.ac.rs/index.php/FUMathInf/article/viewFile/871/pdf_75){:target="_blank"}
- [Lambers, M. (2019): Survey of Cube Mapping Methods in Interactive Computer Graphics](https://marlam.de/publications/cubemaps/lambers2019cubemaps.pdf){:target="_blank"}
- [Lu, F., Song, Z., & Fang, X. (2014): Generation of orthogonal curvilinear grids on the sphere surface based on Laplace-Beltrami equations](https://iopscience.iop.org/article/10.1088/1755-1315/19/1/012012/pdf){:target="_blank"}

### Implementation
---

The planet texture for UV testing can be downloaded [here](/files/uv_checker_cubemap_1024.png){:target="_blank"}.

I made the texture from an image found [here](
https://github.com/ThexXTURBOXx/UVCheckerMapGenerator){:target="_blank"}.

{% include folder_tree.html root="Roguelike" content="assets|+uv_checker_cubemap_1024.png,Makefile,Roguelike.csproj,src|BspNode.cs|BspTree.cs|Corridor.cs|Game.cs|LogEntry.cs|Logger.cs|Map.cs|PathGraph.cs|+Planet.cs|Rand.cs|Room.cs|Vec2.cs" %}

<div class="block-title">Game.cs:</div>

```diff
    ...

    public static Map map { get; private set; }
    public static Player player { get; private set; }
+   public static Planet planet { get; private set; }

    ...

    // Initialize
    private static void Init()
    {
        // Raylib & Imgui initialization
        Raylib.InitWindow(1280, 720, "Roguelike");
        Raylib.SetTargetFPS(30);
        rlImGui.Setup(true);

        // Create new objects
-       map = new Map(96, 48);
-       player = new Player();
+       //map = new Map(96, 48);
+       //player = new Player();
+       planet = new Planet(10);
        
        // Camera setup
        camera.Position = Vector3.Zero;
        camera.Target = Vector3.Zero;
        camera.Up = Vector3.UnitY;
        camera.FovY = 45.0f;
        camera.Projection = CameraProjection.Perspective;
    }

    ...
    
    // Exit game
    private static void Exit()
    {
+       planet.Exit();
        rlImGui.Shutdown();
        Raylib.CloseWindow();
    }

    // Handle user input
    private static void Input()
    {
        // System
        if (Raylib.IsKeyPressed(KeyboardKey.Tab)) { debugMode = !debugMode; }
        if (Raylib.IsKeyPressed(KeyboardKey.F)) { Raylib.ToggleFullscreen(); }
        
-       if (Raylib.IsKeyPressed(KeyboardKey.Up)) { player.MoveUp(); }
-       else if (Raylib.IsKeyPressed(KeyboardKey.Down)) { player.MoveDown(); }
-       else if (Raylib.IsKeyPressed(KeyboardKey.Left)) { player.MoveLeft(); }
-       else if (Raylib.IsKeyPressed(KeyboardKey.Right)) { player.MoveRight(); }
+       //if (Raylib.IsKeyPressed(KeyboardKey.Up)) { player.MoveUp(); }
+       //else if (Raylib.IsKeyPressed(KeyboardKey.Down)) { player.MoveDown(); }
+       //else if (Raylib.IsKeyPressed(KeyboardKey.Left)) { player.MoveLeft(); }
+       //else if (Raylib.IsKeyPressed(KeyboardKey.Right)) { player.MoveRight(); }
    }

    ...

    // Update things in the game
    private static void Update()
    {
        float deltaTime = Raylib.GetFrameTime();

+       planet.Update(deltaTime);

        // Camera
-       Vector3 cameraTargetGoal = new Vector3((float)player.x, 0f, (float)player.y);
+       //Vector3 cameraTargetGoal = new Vector3((float)player.x, 0f, (float)player.y);
        Vector3 cameraTargetGoal = planet.pos;
        camera.Target = Raymath.Vector3Distance(camera.Target, cameraTargetGoal) > 0.1f ? Raymath.Vector3Lerp(camera.Target, cameraTargetGoal, 0.05f) : camera.Target;
-       camera.Position = camera.Target + new Vector3(0f, 16.0f, 12.0f);
+       camera.Position = camera.Target + new Vector3(0f, 18.0f, 18.0f);
    }

    // Render things on screen
    private static void Render()
    {
        // Start render
        Raylib.BeginDrawing();

        // Set background color
        Raylib.ClearBackground(Color.Black);

        // 3D rendering
        Raylib.BeginMode3D(camera);
        if (debugMode) { Raylib.DrawGrid(300, 1.0f); }
-       map.Render3D();
-       player.Render3D();
+       //map.Render3D();
+       //player.Render3D();
+       planet.Render3D();
        Raylib.EndMode3D();

        // 2D rendering
-       map.Render2D();
+       //map.Render2D();
+       planet.Render2D();
        Raylib.DrawFPS(2,2);
-       Raylib.DrawText("POSITION: " + player.x.ToString() + "x" + player.y.ToString(), 2, Raylib.GetRenderHeight() - 16, 16, Color.White);
+       //Raylib.DrawText("POSITION: " + player.x.ToString() + "x" + player.y.ToString(), 2, Raylib.GetRenderHeight() - 16, 16, Color.White);
        
        // 2D rendering (debug mode)
        if (debugMode) {
            Raylib.DrawText("DEBUG MODE", 2, 20, 16, Color.White);
            RenderImGui();
        }

        // End render
        Raylib.EndDrawing();
    }

    ...
```

<div class="block-title">Planet.cs:</div>

```csharp
using Raylib_cs;
using System.Numerics;

namespace Roguelike;

/// <summary>
/// A round astronomical body.
/// </summary>
public class Planet
{
    public readonly int size;
    public Vector3 pos { get; private set; }
    private Vector3 rotation = Vector3.Zero;
    private Model model;
    private Texture2D texture;

    // Constructor
    public Planet(int size)
    {
        this.size = size;
        this.pos = Vector3.Zero;
        Generate();
    }

    // Generate the planet
    private unsafe void Generate()
    {
        // Generate mesh
        model = Raylib.LoadModelFromMesh(MakeMesh(flat: false));

        // Set model texture
        texture = Raylib.LoadTexture("./assets/uv_checker_cubemap_1024.png");
        Raylib.SetMaterialTexture(ref model, 0, MaterialMapIndex.Albedo, ref texture);
    }
   
    // Called every frame
    public void Update(float deltaTime)
    {
        Rotate(new Vector3(0.1f, 0f, 0.1f) * deltaTime);
    }

    // Render 3D graphics
    public void Render3D()
    {
        Vector3 renderOffset = new Vector3(size * 0.7f, 0f, 0f);
        Raylib.DrawModel(model, pos + renderOffset, 1.0f, Color.White);
    }
    
    // Render 2D graphics
    public void Render2D()
    {
        float texDisplaySize = 0.175f;
        Raylib.DrawTextureEx(texture, new Vector2(0f, 0f), 0f, texDisplaySize, Color.White);
    }

    // Free allocated memory
    public void Exit()
    {
        Raylib.UnloadTexture(texture);
        Raylib.UnloadModel(model);
    }

    // Rotate the planet model
    public unsafe void Rotate(Vector3 newRotation)
    {
        rotation += newRotation;
        model.Transform = Raymath.MatrixRotateXYZ(rotation);
    }

    // Transform a given cube face index & grid position to a 3D point in local space
    private unsafe Vector3 Transform2Dto3D(int face, Vector2 pos, bool flat = false)
    {
        // Flat mode
        if (flat) { return Transform2DToFlat(face, pos); }

        // Sphere mode
        return TransformCubeToSphere(Transform2DToCube(face, pos));
    }

    // Project the planet as a 3D unfolded cube
    private Vector3 Transform2DToFlat(int face, Vector2 pos)
    {
        Vector3 result = new Vector3(pos.X - (float)size * 2, 0f, pos.Y - (float)size);
        switch (face)
        {
            case 1: result += new Vector3((float)size, 0.0f, 0.0f); break;
            case 2: result += new Vector3((float)size * 2, 0.0f, 0.0f); break;
            case 3: result += new Vector3((float)size * 3, 0.0f, 0.0f); break;
            case 4: result += new Vector3((float)size, 0.0f, -(float)size); break;
            case 5: result += new Vector3((float)size, 0.0f, (float)size); break;
        }
        return result;
    }

    // Project the planet as a 3D cube
    private Vector3 Transform2DToCube(int face, Vector2 pos)
    {
        Vector3 result = Vector3.Zero;
        Vector3 offset = new Vector3(-(float)size / 2, (float)size / 2, -(float)size / 2);
        switch (face)
        {
            case 0: result = offset + new Vector3(0f, pos.X - (float)size, pos.Y); break;
            case 1: result = offset + new Vector3(pos.X, 0f, pos.Y); break;
            case 2: result = offset + new Vector3((float)size, ((float)size - pos.X) - (float)size, pos.Y); break;
            case 3: result = offset + new Vector3((float)size - pos.X, -(float)size, pos.Y); break;
            case 4: result = offset + new Vector3(pos.X, (pos.Y - (float)size), 0f); break;
            case 5: result = offset + new Vector3(pos.X, ((float)size - pos.Y) - (float)size, (float)size); break;
        }
        return result;
    }

    // Cube to sphere projection
    private Vector3 TransformCubeToSphere(Vector3 p)
    {
        p /= size / 2f;
        float x = p.X * (float)Math.Sqrt(1f - (((p.Y * p.Y) + (p.Z * p.Z)) / (2f)) + (((p.Y * p.Y) * (p.Z * p.Z)) / (3f)));
        float y = p.Y * (float)Math.Sqrt(1f - (((p.X * p.X) + (p.Z * p.Z)) / (2f)) + (((p.X * p.X) * (p.Z * p.Z)) / (3f)));
        float z = p.Z * (float)Math.Sqrt(1f - (((p.X * p.X) + (p.Y * p.Y)) / (2f)) + (((p.X * p.X) * (p.Y * p.Y)) / (3f)));
        return new Vector3(x, y, z) * (size * 0.75f);
    }

    // Generate the 3D mesh for the planet
    private unsafe Mesh MakeMesh(bool flat = true)
    {
        // Set mesh specs
        int faceNumVerts = (size + 1) * (size + 1);
        int numVerts = 6 * faceNumVerts;
        int numTris = 2 * (6 * (size * size));
        
        // Allocate memory for the mesh
        Mesh mesh = new(numVerts, numTris);
        mesh.AllocVertices();
        mesh.AllocTexCoords();
        mesh.AllocNormals();
        mesh.AllocIndices();
        
        // Contigous regions of memory set aside for mesh data
        Span<Vector3> vertices = mesh.VerticesAs<Vector3>();
        Span<Vector2> texcoords = mesh.TexCoordsAs<Vector2>();
        Span<ushort> indices = mesh.IndicesAs<ushort>();
        Span<Vector3> normals = mesh.NormalsAs<Vector3>();
        
        // Make lookup table for cube face vert index start position
        int[] faceVertIndex = new int[6];
        for (int face = 0; face < 6; face++){
            faceVertIndex[face] = faceNumVerts * face;
        }

        // Loop through all cube faces
        ushort vertIndex = 0;
        int triIndex = 0;
        Color color = Color.White;
        for (int face = 0; face < 6; face++)
        {
            for (int y = 0; y < size; y++)
            {
                for (int x = 0; x < size; x++)
                {
                    // Set UV cordinates for vertices on the current grid position
                    float texCoordXStart = 0f;
                    float texCoordYStart = 0f;
                    float texCoordXSize = 1f / 3f;
                    float texCoordYSize = 1f / 2f;
                    switch (face)
                    {
                        case 0:
                            texCoordXStart = 0f;
                            texCoordYStart = 0f;
                            break;
                        case 1:
                            texCoordXStart = texCoordXSize;
                            texCoordYStart = 0f;
                            break;
                        case 2:
                            texCoordXStart = texCoordXSize * 2;
                            texCoordYStart = 0f;
                            break;
                        case 3:
                            texCoordXStart = 0f;
                            texCoordYStart = texCoordYSize;
                            break;
                        case 4:
                            texCoordXStart = texCoordXSize;
                            texCoordYStart = texCoordYSize;
                            break;
                        case 5:
                            texCoordXStart = texCoordXSize * 2;
                            texCoordYStart = texCoordYSize;
                            break;
                    }  
                    float texCoordLeft   = texCoordXStart + ((float)x * (texCoordXSize / (float)size));
                    float texCoordRight  = texCoordXStart + (((float)x + 1f) * (texCoordXSize / (float)size));
                    float texCoordTop    = texCoordYStart + ((float)y * (texCoordYSize / (float)size));
                    float texCoordBottom = texCoordYStart + (((float)y + 1f) * (texCoordYSize / (float)size));
                    
                    // Set vertex indexes
                    int vertIndexStart = vertIndex;
                    ushort vertTopLeft = (ushort)(vertIndex - (size + (x == 0 ? 1 : 2)));
                    ushort vertTopRight = (ushort)(vertTopLeft + 1);
                    ushort vertBottomLeft = (ushort)(vertIndex - 1);
                    if (y == 0)
                    {
                        vertTopLeft = (ushort)(vertIndex - (x == 1 ? 3 : 2));
                    }
                    if (y == 1)
                    {
                        vertTopLeft += (ushort)((x == 0 ? x + 1 : x) - size);
                        vertTopRight = (ushort)(vertTopLeft + (x == 0 ? 1 : 2));
                    }

                    // Make top-left vertex
                    if (y == 0 && x == 0)
                    {
                        vertices[vertIndex] = Transform2Dto3D(face, new Vector2(x, y), flat: flat);
                        normals[vertIndex] = Vector3.Normalize(vertices[vertIndex]);
                        texcoords[vertIndex] = new(texCoordLeft, texCoordTop);
                        vertTopLeft = (ushort)(vertIndex);
                        vertIndex++;
                    }

                    // Make top-right vertex
                    if (y == 0)
                    {
                        vertices[vertIndex] = Transform2Dto3D(face, new Vector2(x + 1, y), flat: flat);
                        normals[vertIndex] = Vector3.Normalize(vertices[vertIndex]);
                        texcoords[vertIndex] = new(texCoordRight, texCoordTop);
                        vertTopRight = (ushort)(vertIndex);
                        vertIndex++;
                    }

                    // Make bottom-left vertex
                    if (x == 0)
                    {
                        vertices[vertIndex] = Transform2Dto3D(face, new Vector2(x, y + 1), flat: flat);
                        normals[vertIndex] = Vector3.Normalize(vertices[vertIndex]);
                        texcoords[vertIndex] = new(texCoordLeft, texCoordBottom);
                        vertBottomLeft = (ushort)(vertIndex);
                        vertIndex++;
                    }
                    
                    // Make bottom-right vertex
                    vertices[vertIndex] = Transform2Dto3D(face, new Vector2(x + 1, y + 1), flat: flat);
                    normals[vertIndex] = Vector3.Normalize(vertices[vertIndex]);
                    texcoords[vertIndex] = new(texCoordRight, texCoordBottom);
                    ushort vertBottomRight = (ushort)(vertIndex);
                    vertIndex++;

                    // Triangle 1 (Counter-clockwise winding order)
                    indices[triIndex]     = vertTopLeft;
                    indices[triIndex + 1] = vertBottomRight;
                    indices[triIndex + 2] = vertTopRight;
                    triIndex += 3;

                    // Triangle 2 (Counter-clockwise winding order)
                    indices[triIndex]     = vertTopLeft;
                    indices[triIndex + 1] = vertBottomLeft;
                    indices[triIndex + 2] = vertBottomRight;
                    triIndex += 3;
                }
            }
        }

        // Upload mesh data from CPU (RAM) to GPU (VRAM)
        Raylib.UploadMesh(ref mesh, false);

        // Return the finished mesh
        return mesh;
    }
}
```

### Conclusion
---

The planet generator needs more features, like a coordinate system, support for heightmap and some procedural terrain generation algorithms. But for now this is a start.

{% include bash_command.html bash_command="make run" bash_dir="~/Roguelike" %}

[![screenshot](/img/screenshot_2024-06-10-013930.png)](/img/screenshot_2024-06-10-013930.png){:target="_blank"}

Here's a little video: [https://youtu.be/0PV7XQGF4gY](https://youtu.be/0PV7XQGF4gY){:target="_blank"}

Download the source code: [roguelike-devlog8.zip](/files/roguelike-devlog8.zip){:target="_blank"}

Find the project on GitHub: [LASER-WOLF/Roguelike](https://github.com/LASER-WOLF/Roguelike){:target="_blank"}