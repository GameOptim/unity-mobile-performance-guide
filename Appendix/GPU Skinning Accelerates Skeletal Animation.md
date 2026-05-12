# GPU Skinning Accelerates Skeletal Animation



When there are many character animation models in a scene, performance incurs significant overhead, a large portion of which comes from skeletal animation. The method recommended in this article involves transferring skinning tasks from the CPU to the GPU, which can greatly improve the operational efficiency of multi-character scenes. This approach has substantial applicability in large-scale group animation simulations, such as in MMO and RTS games.

------

## **I. Cause**

We know that when there are many character animation models in a scene, performance incurs significant overhead. Besides Draw Calls, a large portion of this overhead comes from skeletal animation. Unity has a built-in GPU Skinning feature, but based on my tests, it did not improve overall performance at all; instead, it increased it significantly. There are many methods to reduce the overhead of skeletal animation, each with its own pros and cons, and none are a one-size-fits-all solution. The method introduced here is no exception. Essentially, it involves implementing GPU Skinning ourselves, but it differs from Unity's built-in GPU Skinning.

![1](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/1.gif)
![2](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/2.png)
*Using character models from ShadowGun*

![3](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/3.png)



*Enabling Unity's built-in GPU Skinning*

As can be seen from the figure above, Unity calls the Transform Feedback interface of OpenGL ES, which is only available in OpenGL ES 3.0 or later. ~~As I understand it, Transform Feedback involves passing large amounts of data to the Vertex Shader, returning the GPU-calculated results to the CPU via a Buffer Object, and then having the CPU read the data from the Buffer Object (or directly pass the Buffer Object to the next step) for use in subsequent operations. Clearly, in skeletal animation, Transform Feedback handles bone transformations, and Unity uses the transformed results to perform GPU skinning operations.~~

This time, we are going to implement this process ourselves, but without using Transform Feedback, to ensure it runs smoothly on OpenGL ES 2.0. Moreover, the Unity engine does not provide such low-level interfaces.

The general steps are as follows:

- Serialize the skeletal animation data into a custom data structure. This approach allows us to completely break free from the constraints of Animation and enables optimization.
  GameObjects (a feature in Unity that completely removes the skeletal hierarchy GameObjects without losing binding points, reducing overhead);
- Perform skeletal transformations on the CPU;
- Pass the results of the skeletal transformations to the GPU for skinning.

These three simple steps are nothing out of the ordinary for traditional skeletal animation. Below, I will elaborate on each step and describe the details clearly.

------

## **II. Implementation**

**Extracting Skeletal Animation Data**
![4](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/4.png)
*Animation Data in Unity*

The purpose of this step is to extract this data and store it in a custom data structure. The code is roughly as follows:
![5](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/5.png)
There are two key points to note. First, it is important to clarify whether the rotation extracted from the AnimationCurve is in Euler angles or quaternions. I initially made a mistake here, assuming it was Euler angles, which led to incorrect results in subsequent calculations. Second, the quaternion used for rotation must be a unit quaternion (with a magnitude of 1); otherwise, you will receive an error message from Unity.

In the code above, I directly sampled the data for each frame at a frequency of 30fps. Alternatively, you could avoid sampling in advance and instead sample from the AnimationCurve only when needed. This approach would result in smoother animations but would also increase the computational load at runtime.

**Bone Transformation**

The transformation of bones is the core part of all the code. It may seem quite complex, but once you think it through, the amount of code is actually minimal
![6](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/6.png)

Simply put, bone transformation is a matrix multiplication. For example, bone0 (abbreviated as b0) is the parent bone of bone1 (abbreviated as b1):
![7](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/7.png)
Note that this is matrix left multiplication (read from right to left), where trs is Matrix4x4.TRS, meaning the data sampled from AnimationCurve.

The role of Bindpose is to transform vertex coordinates from model space into bone space (which is the inverse matrix of the bone matrix), and then apply the current bone's transformation, proceeding layer by layer along the hierarchy.

**Skinning**

The code for the CPU part of skinning is as follows:

![8](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/8.png)
![9](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog/Sparkle_GPUSkinning/9.png)
![10](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/10.png)

*Since the number of bones is fixed at 24, the 96 in the diagram equals 24 multiplied by 4*

Using SetMatrixArray is actually a bit wasteful because for a 4x4 matrix (four float4s), the last dimension is always (0, 0, 0, 1). Therefore, it can be replaced with a 3x4 matrix (three float4s), which reduces the pressure on data transfer.

Now that all the bone transformation matrices have been passed to the Shader, we can use this data to perform skinning (transforming vertex coordinates).
![11](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/11.png)

------

## **Three, Improvement**

At this point, all characters' movements are synchronized. Next, we will make improvements by no longer using uniform arrays to pass data. Instead, we will store skeletal animation data in textures and introduce some variation to avoid the issue of all characters' movements being completely synchronized. At the very beginning of the runtime, we will store the animation data for all frames into textures. The code is as follows:
![12](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/12.png)

The skinning code in the Shader changes accordingly to:
![13](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog/Sparkle_GPUSkinning/13.png)

The above details the author's implementation of GPU Skinning. However, no method is perfect. As one of the alternative solutions to reduce the overhead of skeletal animation, using it appropriately can significantly improve performance.

------

## **IV. Testing**

To further verify the feasibility of this solution on mobile devices, UWA conducted the following experiments on real devices.

We placed a certain number of models playing animations in an empty scene to compare the performance efficiency of Mecanim and GPU Skinning. The models were taken from ShadowGun, featuring 2600 polygons and 24 bones. When using Mecanim, the models were set to Generic mode with Optimize GameObject enabled. The data for 1000 frames running on Redmi Note2 is as follows:

![14](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/14.png)
*FPS variation*

![15](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/15.png)
*Test scene CPU time consumption data*

The above image shows the CPU time consumption data of the main thread when there are 300 characters in the scene using the GPU Skinning solution. The average CPU time consumption per frame (main thread) for different numbers of characters is as follows:
![16](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/GPU%20Skinning/16.png)

From the data, it can be seen that GPU Skinning demonstrates better performance in terms of overall FPS and average CPU time consumption per frame on the main thread, thereby allowing valuable CPU time to be allocated to more game logic.

------

## **V. Advantages and Limitations**

This method transfers the skinning workload from the CPU to the GPU. Testing data on real devices has verified that this method can significantly improve the operational efficiency of multi-character scenes. The method offers the following advantages:

> 1. Significantly reduces the CPU time consumption of MeshSkinning.Render, while also eliminating the dependency on the Animator component, thereby completely avoiding the CPU usage of MeshSkinning.Update and Animator.Update;
> 2. By storing animation data in textures, only a small amount of memory overhead is required to achieve a substantial improvement in runtime efficiency;
> 3. Suitable for large-scale crowd animation simulations, such as MMO, RTS, and other game genres.

Of course, the method currently has the following limitations:

> 1. Increases the burden on GPU computation;
> 2. The current Shader implementation uses tex2Dlod, which may have compatibility issues on certain low-end devices;
> 3. Currently, it cannot directly handle animation events, animation blending, and other operations, requiring further development by the R&D team.



