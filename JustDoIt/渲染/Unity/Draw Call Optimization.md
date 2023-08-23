
# Draw Call

To draw geometry on the screen, Unity issues draw calls to the graphics API. **A draw call tells the graphics API what to draw and how to draw it.** Each draw call contains all the information the graphics API needs to draw on the screen, such as information about textures, shaders and buffers. Draw calls can be resource intensive, but often the preparation for a draw call is more resource intensive than the draw call itself.

To prepare for a draw call, the CPU sets up resources and changes internal settings on the GPU. These settings are collectively called the **render state**. 

***Changes to the render state, such as switching to a different material, are the most resource-intensive operations the graphics API performs.***


# Optimizing Draw Calls

Because render-state changes are resource intensive, it is important to optimize them. The main way to optimize render-state changes is to reduce the number of them. There are two ways to do this:

- Reduce the total number of draw calls. When you decrease the number of draw calls, you also decrease the number of render-state changes between them. 减少draw calls总数，自然就减少了不同draw calls之间的render-state切换。 
- Organize draw calls in a way that reduces the number of changes to the render state. If the graphics API can use the same render state to perform multiple draw calls, it can group draw calls together and not need to perform as many render-state changes. 尽量让相同render-state的draw calls一起被调用，以减少render-state的切换。

There are several methods you can use in Unity to optimize and reduce draw calls and render-state changes. Some methods are more suited for certain scenes than others. The following methods are available in Unity:

- [GPU instancing](https://docs.unity3d.com/Manual/GPUInstancing.html): Render multiple copies of a mesh **with the same material** in a single draw call (Render multiple copies of the same mesh at the same time). Each copy of the mesh is called an instance.
- [Draw call batching](https://docs.unity3d.com/Manual/DrawCallBatching.html): Combine meshes to reduce draw calls. Unity provides the following types of built-in draw call batching:
    - [Static batching](https://docs.unity3d.com/Manual/static-batching.html): Combine meshes of [static](https://docs.unity3d.com/Manual/StaticObjects.html) GameObjects in advance. Unity sends the combined data to the GPU, but renders each mesh in the combination individually. **Static batching doesn’t reduce the number of draw calls, but instead reduces the number of render state changes between them.** Unity can still cull the meshes individually but each draw call is less resource-intensive since the state of the data never changes.
    - [Dynamic batching](https://docs.unity3d.com/Manual/dynamic-batching.html): Transforms mesh vertices on the CPU, groups vertices that share the same configuration, and renders them in one draw call. Vertices share the same configuration if they store the same number and type of attributes. For example, `position` and `normal`.
- [Manually combining meshes](https://docs.unity3d.com/Manual/combining-meshes.html): Manually combine multiple meshes into a single mesh, using the [Mesh.CombineMeshes](https://docs.unity3d.com/ScriptReference/Mesh.CombineMeshes.html) function. Unity renders the combined mesh in a single draw call instead of one draw call per mesh.
- [SRP Batcher](https://docs.unity3d.com/Manual/SRPBatcher.html): If your Project uses a Scriptable Render Pipeline(SRP), use the SRP Batcher to reduce the CPU time Unity requires to prepare and dispatch draw calls for materials that use the same shader variant. 

![image.png](https://pic3.zhimg.com/v2-f5b8402d281543a9debe941f9762889a_r.jpg)

[参考：Unity渲染优化的4种批处理](https://zhuanlan.zhihu.com/p/432223843)


# Optimization Priority

You can use multiple draw call optimization methods in the same scene but be aware that Unity prioritizes draw call optimization methods in a particular order. If you mark a GameObject to use more than one draw call optimization method, Unity uses the highest priority method. The only exception to this is the [SRP Batcher](https://docs.unity3d.com/Manual/SRPBatcher.html). When you use the SRP Batcher, Unity also supports static batching for [GameObjects that are SRP Batcher compatible](https://docs.unity3d.com/Manual/SRPBatcher.html#gameobject-compatibility). Unity prioritizes draw call optimizations in the following order:

1. SRP Batcher and static batching
2. GPU instancing
3. Dynamic batching

If you mark a GameObject for static batching and Unity successfully batches it, Unity disables GPU instancing for that GameObject, even if the renderer uses an instancing shader. When this happens, the Inspector window displays a warning message that suggests that you disable static batching. Similarly, if Unity can use GPU instancing for a mesh, Unity disables dynamic batching for that mesh.


# Compare 

1. Static batching ***combines meshes that don’t move***. Dynamic batching ***batches moving GameObjects***.

2. Static batching is more efficient than dynamic batching because static batching doesn’t transform vertices on the CPU. 在 Unity 引擎中，***Dynamic Batching 是在运行时动态合并***多个相同的 Mesh，需要在运行时对 Mesh 进行顶点变换，以适应不同的物体位置、旋转和缩放等变换操作。而 ***Static Batching 是在场景加载时静态合并*** 多个相同的 Mesh，只需要进行一次。一旦合并完成后，就可以使用合并后的 Mesh 进行渲染，不需要再次进行合并操作，从而减少了 CPU 的负担，提高了渲染效率。

3. Static batching doesn’t reduce the number of draw calls, but instead reduces the number of render state changes between them (explain is down blow). Dynamic batching renders them in one draw call.


# Static batching

Static batching ***combines meshes that don’t move***. 

### 用法

在ProjectSettings -> Player里开启【Static Batching】

![](https://pic4.zhimg.com/v2-bd32a1fc837ba87ddceef11072661e97_b.jpg)

### 条件

1. 使用相同材质引用的静态物体  
2. 物体需为Mesh，具有MeshFilter和MeshRenderer组件  
3. Mesh 需要在ImportSettings面板勾选 read/write enabled

### ⚠️注意事项

There are limits to the number of vertices a static batch can include. Each static batch can include up to 64000 vertices. If there are more, Unity creates another batch.

### 原理

Static batching transforms the combined meshes into world space and builds one shared vertex and index buffer for them. Then, for visible meshes, Unity performs a series of simple draw calls, with almost no state changes between each one. Static batching doesn’t reduce the number of draw calls, but instead reduces the number of render state changes between them. 

就好比掏出钱包付钱，钱包已经从口袋里拿出并展开（准备vertex and index buffer的过程），那么从里面拿出钱就很快捷，即使需要拿出多次（多次draw call）。同理，需要绘制的物体并不一定在buffer中紧邻，因此CPU要向GPU指明渲染的range，如果在buffer中是断开的两段range，那么就要指示两次。

Using static batching requires additional CPU memory to store the combined geometry. If multiple GameObjects use the same mesh, Unity creates a copy of the mesh for each GameObject, and inserts each copy into the combined mesh. This means that the same geometry appears in the combined mesh multiple times. Unity does this regardless of whether you use the [editor](https://docs.unity3d.com/Manual/static-batching.html#editor) or [runtime API](https://docs.unity3d.com/Manual/static-batching.html#runtime) to prepare the GameObjects for static batching. If you want to keep a smaller memory footprint, you might have to sacrifice rendering performance and avoid static batching for some GameObjects. For example, marking trees as static in a dense forest environment can have a serious memory impact.


# Dynamic batching

Dynamic batching ***batches moving GameObjects***. 

### 用法

在ProjectSettings -> Player里开启Dynamic Batching。如果是URP，则在URP Asset下开启。

### 条件

1. 使用相同材质引用的网格实例——只有使用相同材质球的Mesh才可以被批处理。(详见拓展1)  
2. 物体之间Transform不能具有镜像关系——比如物体A的scale=+1，物体B的scale=-1，那就不能被批处理。  
3. 着色器使用的顶点属性数量不能大于900——比如，漫反射计算需要使用顶点的“位置、法线、UV”这3种属性，所以模型的顶点数不能超过300；如果着色器需要使用顶点的“位置、法线、UV0、UV1和切向量”这5种属性，那就只能批处理180顶点以下的物体。  
4. 材质的着色器不能依赖多个过程——多Pass的shader会中断批处理。(详见注意事项3) 
5. 网格实例应引用相同的光照纹理文件——如果物体使用光照纹理，需要保证它们指向光照纹理中的同一位置，才可以被动态批处理。GameObjects with light-maps have additional renderer parameters. This means that, if you want to batch light-mapped GameObjects, they must point to the same light-map location.

### ⚠️注意事项

1. 通过C#脚本修改材质，请使用Renderer.sharedMaterial以保持批处理
     - 假设一个场景里有A、B两个物体，他们共享材质球M，现在你要在脚本里获取A的材质球：
     - 如果你使用[Renderer.material](https://link.zhihu.com/?target=https%3A//docs.unity3d.com/ScriptReference/Renderer-material.html)这个语句，函数会(creates a copy of M)返回一个实例化的材质球MA，修改MA的属性只对物体A有效，A和B不再共享材质球，A和B不能被批处理，即使材质球MA和材质球M一模一样  
     - 如果你使用[Renderer.sharedMaterial](https://link.zhihu.com/?target=https%3A//docs.unity3d.com/2021.2/Documentation/ScriptReference/Renderer-sharedMaterial.html)这个语句，函数会返回M，A和B仍然共享材质球M，A和B可以被批处理  
     - If GameObjects use different material instances, Unity can’t batch them together, even if they are essentially the same. The only exception to this is shadow caster rendering.
     
     - 补充1：一个物体可以有多个材质球，所以，使用[Renderer.materials](https://link.zhihu.com/?target=https%3A//docs.unity3d.com/ScriptReference/Renderer-materials.html)、[Renderer.sharedMaterials](https://link.zhihu.com/?target=https%3A//docs.unity3d.com/ScriptReference/Renderer-materials.html)可以返回一个包含所有material的数组，而Renderer.sharedMaterial和Renderer.material都只返回第一个material  
     - 补充2：由于使用Renderer.material、Renderer.materials语句后会实例化Material，所以当你销毁object的时候，你也应该销毁实例化的material，否则会产生垃圾  
     
     - 如果是BRP项目，你也可以使用[MaterialPropertyBlock](https://link.zhihu.com/?target=https%3A//docs.unity3d.com/2021.2/Documentation/ScriptReference/MaterialPropertyBlock.html)语句修改材质，它不会破坏批处理，使用MaterialPropertyBlock要比使用多个材质球更快。但如果是URP/HDRP/SRP项目，不要使用MaterialPropertyBlock语句，因为它会阻止SRP Batcher，

2. 着色器使用的顶点属性数量不能大于900。Unity can’t apply dynamic batching to meshes that contain more than 900 vertex attributes and 225 vertices. This is because dynamic batching for meshes has an overhead per vertex. For example, if your shader uses vertex position, vertex normal, and a single UV, then Unity can batch up to 225 vertices. However, if your shader uses vertex position, vertex normal, UV0, UV1, and vertex tangent, then Unity can only batch 180 vertices.

3. Unity can’t fully apply dynamic batching to GameObjects that use multi-pass shaders.
    - Almost all Unity shaders support several lights in forward rendering. To achieve this, they process an additional render pass for each light. Unity only batches the first render pass. It can’t batch the draw calls for the additional per-pixel lights.
    - The [Legacy Deferred rendering path](https://docs.unity3d.com/Manual/RenderingPaths.html) doesn’t support dynamic batching because it draws GameObjects in two render passes. The first pass is a light pre-pass and the second pass renders the GameObjects.

### 原理

Dynamic batching works differently between meshes and geometries that Unity generates dynamically ***at runtime***, such as particle systems. There are differences between meshes and dynamic geometries, see Dynamic batching for meshes and Dynamic batching for dynamically generated geometries.

**Dynamic batching for meshes**

Dynamic batching for meshes works by transforming all vertices into ***world space***. on the CPU, rather than on the GPU. This means dynamic batching is only an optimization if the transformation work is less resource intensive than doing a draw call.

The resource requirements of a draw call depend on many factors, primarily the graphics API. For example, on consoles or modern APIs like Apple Metal, the draw call overhead is generally much lower, and often dynamic batching doesn’t produce a gain in performance. To determine whether it’s beneficial to use dynamic batching in your application, [profile](https://docs.unity3d.com/Manual/Profiler.html) your application with and without dynamic batching.

Unity can use dynamic batching for shadows casters, even if their materials are different, as long as the material values Unity needs for the shadow pass are the same. For example, multiple crates can use materials that have different textures. Although the material assets are different, the difference is irrelevant for the shadow caster pass and Unity can batch shadows for the crate GameObjects in the shadow render step.

**Dynamic batching for dynamically generated geometries**

The following renderers dynamically generate geometries, such as particles and lines, that you can optimize using dynamic batching:

- [Built-in Particle Systems](https://docs.unity3d.com/Manual/Built-inParticleSystem.html)
- [Line Renderers](https://docs.unity3d.com/Manual/class-LineRenderer.html)
- [Trail Renderers](https://docs.unity3d.com/Manual/class-TrailRenderer.html)

Dynamic batching for dynamically generated geometries works differently than it does for meshes:
1. For each renderer, Unity builds all dynamically batchable content into one large vertex buffer.
2. The renderer sets up the material state for the batch.
3. Unity then binds the vertex buffer to the GPU.
4. For each Renderer in the batch, Unity updates the offset in the vertex buffer and submits a new draw call.

*This approach is similar to how Unity submits draw calls for static batching.

### Batch breaking cause

破坏动态合批的因素

- **Additional Vertex Streams** — the object has additional vertex streams set using `MeshRenderer.additionalVertexStreams`. 通过MeshRenderer的additionalVertexStreams添加过点。 
- **Deferred Objects Split by Shadow Distance** — one of the objects is within shadow distance, the other one is not. 
- **Different Combined Meshes** — the object belongs to another combined static mesh.


# GPU Instance

GPU instancing is a draw call optimization method that renders multiple **copies of a mesh with the same material** in a single draw call (Render multiple copies of the same mesh at the same time). Each copy of the mesh is called an instance. This is useful for drawing things that appear multiple times in a scene, for example, trees or bushes.

GPU instancing renders identical meshes in the same draw call. To add variation and reduce the appearance of repetition, each instance can have different properties, such as color or scale. Draw calls that render multiple instances appear in the Frame Debugger as **Draw Mesh (instanced)**.

## 用法

如果是多Pass的shader，只有第一个Pass可以被批处理。如果shader支持GPU instancing，在材质球面板就会出现Enable Instancing。

![The Enable GPU Instancing option as it appears in the material Inspector.](https://docs.unity3d.com/uploads/Main/enable-gpu-instancing-inspector.png)

## 条件

![](https://pic2.zhimg.com/v2-3295259d08673d369b124b4b9f3a8f85_b.jpg)
和SPR Batching不同，这里创建颜色变体很麻烦。

Unity uses GPU instancing for GameObjects that share the same mesh and material. To instance a mesh and material:
- The mesh must come from one of the following sources, grouped by behavior:
    - A [MeshRenderer](https://docs.unity3d.com/Manual/class-MeshRenderer.html) component or a [Graphics.RenderMesh](https://docs.unity3d.com/ScriptReference/Graphics.RenderMesh.html) call.  
        Behavior: Unity adds these meshes to a list and then checks to see which meshes it can instance.
        Unity does not support GPU instancing for [SkinnedMeshRenderers](https://docs.unity3d.com/Manual/class-SkinnedMeshRenderer.html) or MeshRenderer components attached to GameObjects that are SRP Batcher compatible. 
    - A [Graphics.RenderMeshInstanced](https://docs.unity3d.com/ScriptReference/Graphics.RenderMeshInstanced.html) or [Graphics.RenderMeshIndirect](https://docs.unity3d.com/ScriptReference/Graphics.RenderMeshIndirect.html) call. These methods render the same mesh multiple times using the same shader. Each call to these methods issues a separate draw call. Unity does not merge these draw calls.
- The material’s shader must support GPU instancing. Unity’s Standard Shader supports GPU instancing, as do all surface shaders. To add GPU instancing support to any other shader, see [Creating shaders that support GPU instancing](https://docs.unity3d.com/Manual/gpu-instancing-shader.html).

## ⚠️注意事项

***GPU instancing isn’t compatible with the SRP Batcher.***  The SRP Batcher takes priority over GPU instancing. If a GameObject is compatible with the SRP Batcher, Unity uses the SRP Batcher to render it, not GPU instancing.

If your project uses the SRP Batcher and you want to use GPU instancing for a GameObject, you can do one of the following:
- Use [Graphics.DrawMeshInstanced](https://docs.unity3d.com/ScriptReference/Graphics.DrawMeshInstanced.html). This API bypasses the use of GameObjects and uses the specified parameters to directly draw a mesh on screen.
- Manually remove SRP Batcher compatibility. 


# Scriptable Render Pipeline Batcher

The traditional way to optimize draw calls is to reduce the number of them. Instead, the SRP Batcher reduces render-state changes between draw calls. To do this, the SRP Batcher combines a sequence of `bind` and `draw` GPU commands. Each sequence of commands is called an SRP batch.

To achieve optimal performance for your rendering, each SRP batch should contain as many `bind` and `draw` commands as possible. To achieve this, ***use as few shader variants as possible***. You can still use as many different materials with the same shader as you want.

## 用法

在URP Asset里，【SRP Batcher】是默认勾选的。

![](https://pic2.zhimg.com/v2-e7cbe0f6730f86840c2b33b43915765d_b.jpg)

## 条件

1. 被渲染的物体必须是mesh或skinned mesh，不可以是particle  
2. 被渲染的物体不能使用[MaterialPropertyBlock](https://link.zhihu.com/?target=https%3A//docs.unity3d.com/2021.2/Documentation/ScriptReference/MaterialPropertyBlock.html)修改属性，MaterialPropertyBlock不支持SRP Batcher  
3. shader需要支持SRP Batcher（HDRP和URP项目的Lit和Unlit shader都支持）

![](https://pic4.zhimg.com/v2-26b3043b8d8d80852f29710decdb6fd3_b.jpg)


## ⚠️注意事项

1. 如何使shader支持SRP Batcher：  
    - 在一个名为“UnityPerDraw”的CBuffer里声明所有的内置属性，比如 unity_ObjectToWorld、 unity_SHAr  
    - 在一个名为“UnityPerMaterial”的CBuffer里声明所有的材质属性

2. 如何使GameObject**无法**适用SRP Batcher：这里有2种方法：  
    - 方法1：使shader**不**支持SRP Batcher：新加一个材质属性，但是不在名为“UnityPerMaterial”的CBuffer里声明它  
    - 方法2：使renderer**不**支持SRP Batcher：add a MaterialPropertyBlock to the renderer 就可以使renderer**不**支持SRP Batcher

## 原理

原本，CPU每次提交DrawCall前都要【Set up Cbuffer - Upload Cbuffer】，但是在SRP Batcher里，所有材质球在显存里占有固定的CBuffer，如果材质球的内容不发生改变，CPU就不需要【SetUp-Upload】，从而降低了CPU渲染时间。SRP batcher不会减少DrawCall，而是在DrawCall与DrawCall之间减少CPU的工作量。

![](https://pic1.zhimg.com/v2-4aeeea098cf69196c17651bdb4c682dc_r.jpg)

我们总是希望一个Batch包含的DrawCall次数越多越好：  
左图：Standard Batch的次数大小取决于最后两次DrawCall的 ***Material*** 是否一致，不一致就结束这个Batch，进入SetShaderPass；  
右图：SRP Batch的次数大小取决于最后两次DrawCall的 ***Shader Variant*** 是否一致，不一致就结束这个Batch，进入SetShaderPass；

一个SPR Batch中的draw call数越多，说明效果越好。
![](https://docs.unity3d.com/uploads/Main/SRP_Batcher_batch_information.png)


# Manually Combining Meshes

If we are not able to use draw call batching, manually combining meshes that are close to each other can be a good alternative. For example, for a static cupboard with lots of drawers, it makes sense to combine everything into a single mesh.

We can manually combine multiple meshes into a single mesh as a draw call optimization technique. Unity renders the combined mesh in a single draw call instead of one draw call per mesh. This technique can be a good alternative to draw call batching in cases where the meshes are close together and don’t move relative to one another. 

**Warning**: Unity can’t individually cull meshes you combine. This means that if one part of a combined mesh is onscreen, Unity draws the entire combined mesh. If the meshes are static and you want Unity to individually cull them, use static batching instead.

There are two main ways to combine meshes:
1. In your asset generation tool while authoring the mesh.
2. In Unity using [Mesh.CombineMeshes](https://docs.unity3d.com/ScriptReference/Mesh.CombineMeshes.html).

```C#
[RequireComponent(typeof(MeshFilter))]
[RequireComponent(typeof(MeshRenderer))]
public class ExampleClass : MonoBehaviour
{
    void Start()
    {
        MeshFilter[] meshFilters = GetComponentsInChildren<MeshFilter>();
        CombineInstance[] combine = new CombineInstance[meshFilters.Length];

        int i = 0;
        while (i < meshFilters.Length)
        {
            combine[i].mesh = meshFilters[i].sharedMesh;
            combine[i].transform = meshFilters[i].transform.localToWorldMatrix;
            meshFilters[i].gameObject.SetActive(false);

            i++;
        }

        Mesh mesh = new Mesh();
        mesh.CombineMeshes(combine);
        transform.GetComponent<MeshFilter>().sharedMesh = mesh;
        transform.gameObject.SetActive(true); //有时合批会导致go的active被置成false
        
        // 上面localToWorldMatrix对transform做了修改，影响到效果，这里需要改回去
		transform.localScale = new Vector3(1, 1, 1);
		transform.rotation = Quaternion.identity; //rotation在前
		transform.position = Vector3.zero; //先缩放，再旋转，后平移
    }
}
```
