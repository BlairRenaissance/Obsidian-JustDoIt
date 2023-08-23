
MeshFilter 和 Material 组件之间的关系是，MeshFilter 定义了 3D 模型的形状和结构，而 Material 定义了 3D 模型的材质和外观，MeshRenderer 则将这两者结合起来，将 3D 模型渲染到屏幕上。



### Data

Meshes contain vertices and multiple triangle arrays.  
  
Conceptually, all vertex data is stored in separate arrays of the same size. For example, if you have a mesh of 100 Vertices, and want to have a position, normal and two texture coordinates for each vertex, then the mesh should have [vertices](https://docs.unity3d.com/ScriptReference/Mesh-vertices.html), [normals](https://docs.unity3d.com/ScriptReference/Mesh-normals.html), [uv](https://docs.unity3d.com/ScriptReference/Mesh-uv.html) and [uv2](https://docs.unity3d.com/ScriptReference/Mesh-uv2.html) arrays, each being 100 in size. Data for i-th vertex is at index "i" in each array.  
  
For every vertex there can be a vertex position, normal, tangent, color and up to ***8 texture coordinates***. 






### SubMesh

在Unity中，Mesh的SubMesh（子网格）是指将一个Mesh对象分割成多个较小的几何体集合的一种方式。每个子网格都有自己的顶点、三角形索引和材质等属性。使用子网格可以实现对一个Mesh对象的不同部分应用不同的材质或渲染设置。

在一个Mesh对象中，可以包含一个或多个子网格。每个子网格都由一个独立的三角形索引列表组成，用于定义该子网格的几何形状。子网格的数量通常由物体的几何结构决定，例如一个物体的不同部分可能具有不同的材质或需要不同的渲染设置。

通过使用子网格，可以实现以下几种常见的应用场景：

1.  多材质渲染：每个子网格可以使用不同的材质，从而实现一个物体上不同部分的不同外观。例如，在一个角色模型的Mesh中，头部、身体和腿部可以是不同的子网格，每个子网格应用不同的材质以呈现不同的纹理或颜色。

2.  碰撞检测：可以为每个子网格单独设置碰撞器，这样在进行碰撞检测时可以针对具体的子网格进行处理。这对于复杂的物体，如汽车模型，可以更精确地处理碰撞。

3.  局部渲染优化：如果一个物体只有一部分需要经常更新或者在特定条件下才会显示，那么可以将这部分作为单独的子网格，以便在渲染时只更新或渲染需要的子网格，从而提高性能。

总而言之，子网格是Unity中用于将一个Mesh对象划分为多个较小部分的机制，使得可以对这些部分应用不同的材质、碰撞器或者进行局部的渲染优化。






### Properties

[bindposes](https://docs.unity3d.com/ScriptReference/Mesh-bindposes.html)
The bind poses. The bind pose at each index refers to the bone with the same index.

[blendShapeCount](https://docs.unity3d.com/ScriptReference/Mesh-blendShapeCount.html)

Returns BlendShape count on this mesh.

[boneWeights](https://docs.unity3d.com/ScriptReference/Mesh-boneWeights.html)
The BoneWeight for each vertex in the Mesh, which represents 4 bones per vertex.

[bounds](https://docs.unity3d.com/ScriptReference/Mesh-bounds.html)
The bounding volume of the Mesh.

[colors](https://docs.unity3d.com/ScriptReference/Mesh-colors.html)

Vertex colors of the Mesh.

[colors32](https://docs.unity3d.com/ScriptReference/Mesh-colors32.html)

Vertex colors of the Mesh.

[indexBufferTarget](https://docs.unity3d.com/ScriptReference/Mesh-indexBufferTarget.html)

The intended target usage of the Mesh GPU index buffer.

indexBufferTarget属性决定了索引数据是如何存储和使用的，有以下两个选项：

1.  IndexBufferTarget.Default：这是默认的索引缓冲区目标。在此模式下，索引数据将存储在系统内存中，并在需要时传输到图形设备的显存中。这种模式适用于大多数情况，并提供了灵活性和易用性。

2.  IndexBufferTarget.GPU：这种模式将索引数据直接存储在图形设备的显存中。这样做可以减少数据传输和内存占用，从而提高渲染性能。但是，索引数据将无法在CPU上进行修改，因此只适用于静态或只读的Mesh数据。

[indexFormat](https://docs.unity3d.com/ScriptReference/Mesh-indexFormat.html)

Format of the mesh index buffer data.

[isReadable](https://docs.unity3d.com/ScriptReference/Mesh-isReadable.html)

Returns true if the Mesh is read/write enabled, or false if it is not.

[normals](https://docs.unity3d.com/ScriptReference/Mesh-normals.html)

The normals of the Mesh.

[subMeshCount](https://docs.unity3d.com/ScriptReference/Mesh-subMeshCount.html)

The number of sub-meshes inside the Mesh object.

[tangents](https://docs.unity3d.com/ScriptReference/Mesh-tangents.html)

The tangents of the Mesh.

[triangles](https://docs.unity3d.com/ScriptReference/Mesh-triangles.html)

An array containing all triangles in the Mesh.

[uv](https://docs.unity3d.com/ScriptReference/Mesh-uv.html) - [uv8](https://docs.unity3d.com/ScriptReference/Mesh-uv8.html)

The texture coordinates (UVs) in the first channel. 

[vertexAttributeCount](https://docs.unity3d.com/ScriptReference/Mesh-vertexAttributeCount.html)

Returns the number of vertex attributes that the mesh has. (Read Only)

[vertexBufferCount](https://docs.unity3d.com/ScriptReference/Mesh-vertexBufferCount.html)

Gets the number of vertex buffers present in the Mesh. (Read Only)

[vertexBufferTarget](https://docs.unity3d.com/ScriptReference/Mesh-vertexBufferTarget.html)

The intended target usage of the Mesh GPU vertex buffer.

[vertexCount](https://docs.unity3d.com/ScriptReference/Mesh-vertexCount.html)

Returns the number of vertices in the Mesh (Read Only).

[vertices](https://docs.unity3d.com/ScriptReference/Mesh-vertices.html)

Returns a copy of the vertex positions or assigns a new vertex positions array.






## Simple vs Advanced Mesh API  
  
The Mesh class has two sets of methods for assigning data to a Mesh from script. The "simple" set of methods provide a basis for setting the indices, triangle, normals, tangents, etc. These methods include validation checks, for example to ensure that you are not passing in data that would include out-of-bounds indices. They represent the standard way to assign Mesh data from script in Unity.  
  
**The "simple" methods are: [SetColors](https://docs.unity3d.com/ScriptReference/Mesh.SetColors.html), [SetIndices](https://docs.unity3d.com/ScriptReference/Mesh.SetIndices.html), [SetNormals](https://docs.unity3d.com/ScriptReference/Mesh.SetNormals.html), [SetTangents](https://docs.unity3d.com/ScriptReference/Mesh.SetTangents.html), [SetTriangles](https://docs.unity3d.com/ScriptReference/Mesh.SetTriangles.html), [SetUVs](https://docs.unity3d.com/ScriptReference/Mesh.SetUVs.html), [SetVertices](https://docs.unity3d.com/ScriptReference/Mesh.SetVertices.html), [SetBoneWeights](https://docs.unity3d.com/ScriptReference/Mesh.SetBoneWeights.html).**  
  
There is also an "advanced" set of methods, which allow you to directly write to the mesh data with control over whether any checks or validation should be performed. These methods are intended for advanced use cases which require maximum performance. They are faster, but allow you to skip the checks on the data you supply. If you use these methods you must make sure that you are not supplying invalid data, because Unity will not check for you.  
  
**The "advanced" methods are: [SetVertexBufferParams](https://docs.unity3d.com/ScriptReference/Mesh.SetVertexBufferParams.html), [SetVertexBufferData](https://docs.unity3d.com/ScriptReference/Mesh.SetVertexBufferData.html), [SetIndexBufferParams](https://docs.unity3d.com/ScriptReference/Mesh.SetIndexBufferParams.html), [SetIndexBufferData](https://docs.unity3d.com/ScriptReference/Mesh.SetIndexBufferData.html), [SetSubMesh](https://docs.unity3d.com/ScriptReference/Mesh.SetSubMesh.html), and you can use the [MeshUpdateFlags](https://docs.unity3d.com/ScriptReference/Rendering.MeshUpdateFlags.html) to control which checks or validation are performed or omitted.** 

Use [AcquireReadOnlyMeshData](https://docs.unity3d.com/ScriptReference/Mesh.AcquireReadOnlyMeshData.html) to take a read-only snapshot of Mesh data that you can use with C# Jobs and Burst, and [AllocateWritableMeshData](https://docs.unity3d.com/ScriptReference/Mesh.AllocateWritableMeshData.html) with [ApplyAndDisposeWritableMeshData](https://docs.unity3d.com/ScriptReference/Mesh.ApplyAndDisposeWritableMeshData.html) to create Meshes from C# Jobs and Burst. 

（在Unity游戏引擎中，Mesh代表了一个物体的几何形状和拓扑结构。Mesh数据可以包含顶点坐标、法线、纹理坐标等信息，它们对于渲染物体是至关重要的。而C# Jobs和Burst是Unity提供的用于优化游戏性能的工具。

AcquireReadOnlyMeshData是一个函数，它允许你获取一个只读的Mesh数据快照。这个快照可以在C# Jobs和Burst中使用，这意味着你可以在并行处理中访问和读取Mesh数据，而不会导致冲突或竞争条件。通过使用只读快照，你可以在多个任务之间安全地共享Mesh数据。

AllocateWritableMeshData是另一个函数，它配合ApplyAndDisposeWritableMeshData函数使用，用于在C# Jobs和Burst中创建可写的Mesh。这意味着你可以在并行任务中创建、修改和操作Mesh数据。使用AllocateWritableMeshData分配可写的Mesh数据，然后使用ApplyAndDisposeWritableMeshData将修改后的数据应用到Mesh中，并进行清理工作。

综合起来，通过使用AcquireReadOnlyMeshData和AllocateWritableMeshData函数，结合C# Jobs和Burst，你可以在游戏开发中高效地处理Mesh数据，实现并行处理和优化性能的目标。）



### Simple Mesh API 







### Advanced Mesh API 

