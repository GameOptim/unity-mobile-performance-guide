On mobile platforms, rendering optimization is an unavoidable topic. As one of the major sources of performance overhead, almost all games rely on rendering scenes, objects, and visual effects. Achieving the best balance between high-quality visual presentation and smooth runtime performance has always been a challenge for designers, artists, and engineers alike.

 

## 1. Two Fundamental Parameters Affecting Rendering Efficiency: DrawCall and Triangle

 

### 1. DrawCall

 

In the **Overview mode of GOT Online**, we can view the **DrawCall curve** within the Rendering module. This curve displays both the specific **DrawCall count** and the **Batch count**, as shown in the figure below.

 ![1](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/2.png)

Currently, we recommend that on **mid-range and low-end devices**, the main range (5%–95%) of **Batch** should be controlled within **[0, 250]**.

 

In Unity, it is important to distinguish between **DrawCall** and **Batch**. A single **Batch** may contain multiple **DrawCalls**. As shown in the example below in **FrameDebugger**, two default **ParticleSystem** instances are combined into a single Batch. Therefore, this **Dynamic Batch** contains **2 DrawCalls**.

 ![3](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/3.png)

Common approaches to reducing **Batch** include the following four methods:

-  Dynamic Batching 
-  Static Batching 
-  GPU Instancing 
-  SRP Batcher 

 

### 2. Triangle

 

Generally, a higher **Triangle count** leads to higher rendering time. Therefore, our reports provide **Triangle usage statistics**, distinguishing between **opaque** and **transparent** triangles.

 

It is generally recommended to use **LOD (Level of Detail) tools** to reduce the number of triangles in a scene, thereby lowering rendering overhead.

 ![4](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/4.png)

It should be noted that the **Triangle count displayed here does not represent the number of triangles in the scene models for the current frame**, but rather the **number of triangles actually rendered in the current frame**. This value depends not only on the model's triangle count but also on the **number of rendering passes**.

 

For example:

If a mesh in the scene contains **10,000 triangles**, and the **Shader used by the mesh contains two rendering passes**, or if **two cameras render the object simultaneously**, then the Triangle count shown here will be **20,000**.

 

## 2. Camera.Render Function Stack Analysis

 

In rendering optimization, an effective approach to locating performance bottlenecks is to analyze the **specific call stack of the** `Camera.Render` **function**. These functions can be viewed in **Code Efficiency** both in **real-device tests** and in **GOT Online reports**.

 

Below are several commonly encountered functions during optimization.

 

### 1. RenderForward.RenderLoopJob

 

When expanding the **Camera.Render** call stack, the **self time of** `RenderForward.RenderLoopJob` may appear relatively high. This is typically caused by a **large number of Batches**. 

![5](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/5.png) 

### 2. High Culling Cost

 ![6](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/6.png)

Generally speaking, a **Culling cost within 10%–20%** is considered reasonable.

 

If the **Culling time is relatively high**, the following aspects can be investigated:

 

**1) A strong correlation exists between Culling cost and the number of small GameObjects in the scene.**

 

In such cases, it is recommended that the development team optimize the **scene production workflow** and check whether there are **too many small objects** in the scene, which can increase Culling overhead.

 

Possible optimization approaches include:

 

-  Dynamic loading 
-  Chunk-based rendering 
-  Culling Group 
-  Culling Distance 

 

These techniques can help reduce Culling overhead.

 

**2) If the project uses multi-threaded rendering with Occlusion Culling enabled**, the workload on the **worker threads** may become excessive, leading to increased overall Culling cost.

 

Since **Occlusion Culling** requires calculating occlusion relationships among objects in the scene, although it can reduce rendering overhead, the **feature itself also introduces performance costs**. Therefore, it may not be suitable for all scenarios.

 

In this case, it is recommended that the development team **selectively disable certain Occlusion Culling configurations** and compare the overall rendering performance before deciding whether to enable this feature.

 

 

 

### 3. Render.Mesh

 

`Render.Mesh` represents the rendering cost of **objects that cannot be batched**. The **number of calls** corresponds directly to the **number of Batches**.

 

In the example below, the **call count of** `Render.Mesh` **is 269**, which indicates that **269 opaque objects in the scene were not batched**, a relatively high number.

 ![7](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/7.png)

Excessive **Render.Mesh cost** is typically caused by **a large number of objects that cannot be batched**. Optimization can be carried out in the following ways:

 

**1) For the opaque rendering queue**

 

It is recommended to check for **redundant Materials**. Sometimes materials that are logically identical cannot be batched because they are **different instances**.

 

This can be investigated through **online AssetBundle inspection**, checking for **Material redundancy within AssetBundles**.

 

**2) For the transparent rendering queue**

 

It is necessary to distinguish between **NGUI** and **non-NGUI** scenarios.

 

- **NGUI case** The `Render.Mesh` calls are very likely caused by **UI DrawCalls**. A high number of `Render.Mesh` calls may indicate **excessive UI DrawCalls**, which should be investigated—particularly whether **texture atlases are packed properly**. 
- **Non-NGUI case** It should be checked whether **transparent objects overlap excessively**. Adjusting the **RenderQueue** can increase batching opportunities for objects using the **same Material**. 

 

### 4. ParticleSystem.ScheduleGeometryJobs and ParticleSystem.Draw

 

**1) ParticleSystem.ScheduleGeometryJobs**

 

This function indicates that **before Culling occurs**, the **main thread must wait for worker threads to calculate particle positions**.

 

This overhead is often higher in **combat scenes**.

 ![9](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/9.png)

To optimize this function, it is recommended that the development team:

 

-  Reduce the **complexity of particle systems**, especially on **mid-range and low-end devices**
-  Use **frustum-based pre-culling**
- **Deactivate particle systems outside the camera frustum**

 

This can reduce unnecessary **particle scheduling overhead**.

 

**2) ParticleSystem.Draw**

 

The **number of calls to** `ParticleSystem.Draw` corresponds to the **DrawCall count of particle systems**.

 ![10](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/10.png)

If this function is called too frequently, it is recommended to **reduce the number of particle systems**.

 

Developers can further analyze and optimize by referring to the list in:

 

**Real-device Test Report → Memory Management → Detailed Resource Information → Particle Systems**

![11](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/11.png) 

In addition, batching probability can be increased and DrawCalls reduced by:

 

-  Using **TextureSheetAnimation**
-  Adjusting **Order in Layer** to reduce particle overlap 

 

### 5. Shader.CreateGPUProgram

 

The CPU time consumed by this API occurs when a **Shader is rendered for the first time**. The cost is related to the **complexity of the Shader being rendered**.

 

As shown in the example below, the **execution time of** `Shader.CreateGPUProgram` **reached 203.87 ms in a certain frame**, which caused noticeable **frame stuttering**.

 ![12](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/12.png)

To address this issue, the Shader can be **preloaded using** `ShaderVariantCollection`.

 

After loading, `ShaderVariantCollection.WarmUp` can be called to trigger `Shader.CreateGPUProgram`, and the **ShaderVariantCollection (SVC)** can then be cached.

 This approach prevents the API from being triggered during runtime, thereby avoiding **localized CPU spikes during gameplay**.



## 3. Enabling Multi-threaded Rendering



After enabling multi-threaded rendering, the rendering time on the main thread will significantly decrease. It is recommended that development teams enable this feature.

![13](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/13.png)

However, it is important to note that since our online reports only account for the CPU time on the main thread, if multi-threaded rendering is enabled in a version, the reports will only reflect the main thread's time consumption, which is not conducive to analyzing rendering bottlenecks. Therefore, we typically recommend that during internal testing, two versions be submitted: one with multi-threaded rendering enabled, to serve as a reference for rendering time in the release version, and one with multi-threaded rendering disabled, for detailed analysis of rendering bottlenecks.



## 4. GPU Instancing



Using GPU Instancing allows rendering multiple copies of the same mesh in a single draw call, but each instance can have different parameters (such as Color or Scale) to add variation. When rendering objects that appear repeatedly in a scene, such as buildings, trees, grass, etc., GPU Instancing can effectively reduce the number of draw calls per scene and significantly improve rendering performance.

However, there are the following considerations when using GPU Instancing:

- Compatible platforms and APIs
- The meshes and materials of the rendered instances are the same

- Shader supports GPU Instancing
- SkinnedMeshRenderer is not supported

In some special cases, GPU Instancing rendering for a large number of semi-transparent objects may result in significant performance overhead. 



## 5. SRP Batcher

An increasing number of teams are adopting URP as their rendering pipeline, leveraging SRP Batcher to significantly expand the batching range and improve rendering efficiency. When using URP, the rendering function stack changes as follows:

![14](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Rendering_Module/14.png)

However, when using SRP Batcher, it is still important to note:

- Shader needs to be compatible with SRP
- SRP Batcher currently does not support particle systems
- Shader variants can break DrawCall batching

**The above are some issues to pay attention to when optimizing the rendering module. How to operate still requires everyone to combine the actual situation of the project. At the same time, using GameOptim services can quickly help everyone locate performance bottlenecks.**

