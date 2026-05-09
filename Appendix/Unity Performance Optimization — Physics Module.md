In Unity's built-in physics engine, the time consumption of the physics module mainly comes from FixedUpdate.PhysicsFixedUpdate and ray detection, collision detection, etc., in the logic code. The time consumption of the FixedUpdate.PhysicsFixedUpdate function consists of two main parts: one is Physics.Processing, and the other is Physics.Simulate. Generally, we need to focus on the stacks of these two functions and further identify the causes by analyzing the call counts and time consumption ratios of the stack functions. Common factors affecting performance and optimization strategies include the following:

### I. Whether the project actually requires the physics module

First, we need to understand the mechanism of the Auto Simulation option in Unity. Before Unity 2017.4, this option did not exist, and the engine would automatically enable or disable it based on whether the project contained physics content. Starting from Unity 2017.4, this option became available and is enabled by default.

![img](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog/UWA_ReportModule7/2.png)



Many projects often overlook whether the physics module is enabled or disabled, resulting in unnecessary performance waste. It is recommended that development teams consider whether to enable the physics module based on the project's requirements. Alternatively, they can explore other solutions to replace the role of the physics module to save time spent on physics-related tasks.

### II. Controlling the Number of Physics Calls

In the physics module of GameOptim's real-device performance evaluation reports, we can see statistics on the number of physics updates.

![3](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Physics_Module/3.png)

When it comes to the number of physics calls, the following two parameters require special attention:

![img](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog/UWA_ReportModule7/4.png)



**1）Fixed Timestep**
This determines the update interval for FixedUpdate. The larger this value, the fewer physics update calls per frame. However, if the physics update frequency is too low, it may cause certain mechanisms to behave abnormally. It is recommended that teams increase the Fixed Timestep value as much as possible within an acceptable range.

**2）Maximum Allowed Timestep**
Here we need to first understand the inherent characteristics of the physics system, which is **when the previous frame of the game experiences lag, Unity will call FixedUpdate.PhysicsFixedUpdate multiple times consecutively at a very early stage of the current frame. The significance of Maximum Allowed Timestep lies in limiting the number of physics updates.**

![img](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog/UWA_ReportModule7/5.png)



Maximum Allowed TimeStep determines the maximum number of physics calls per frame. The smaller this value, the fewer the maximum physics calls per frame. With a Fixed Timestep of 20ms, reducing the Maximum Allowed Timestep from 333ms to 100ms means that, regardless of any stuttering, the frame will call physics at most 5 times instead of up to 17 times.

Additionally, it is recommended to optimize other modules first, as this can reduce the number of physics updates in the next frame.

### III. Is the Number of Contacts Reasonable?

Abnormal contact counts typically fall into the following two scenarios:

**1) Excessive Number of Contacts**
Consider whether there are unnecessary physical collision detections, such as collisions between a shield and the ground.

![6](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Physics_Module/6.png)

**2) The number of contacts is 0 and there is physics time consumption**
A contact count of 0 indicates that there are no collisions or triggers in the project. If other physical features such as cloth simulation are not being used, consider turning off the physics module, specifically by disabling AutoSimulation.

![7](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Physics_Module/7.png)

### IV. Reduce the Use of Raycasts

Raycast, commonly referred to as ray detection, has a processing time proportional to the number of rays. The simplest way to control the time spent on Raycasts is to limit the number of rays.

![img](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog/UWA_ReportModule7/8.png)



However, some projects indeed require a significant number of Raycasts. For instance, in bullet hell games, the number of Raycasts is often high. In such cases, the RaycastCommand under the Job System can be used to transfer the time-consuming Raycast operations from the main thread to sub-threads, thereby reducing their overall impact on performance.

### V. Auto Sync Transforms

When Auto Sync Transforms is checked, Unity synchronizes the Transform changes of Rigidbody/Collider, such as Position and Scale, to the physics engine during Physics Queries. Additionally, when AutoSimulation is checked, Unity automatically synchronizes Rigidbody and Collider once per physics update. Therefore, if AutoSimulation is turned off and the project uses ray detection or NGUI, Auto Sync Transforms should typically be checked; otherwise, issues such as inaccurate ray detection results or unresponsive UI events may occur.

The performance overhead of synchronization operations is generally low. High time consumption issues typically only arise in extremely large scenes where a large number of objects are loaded into the scene.

![img](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog/UWA_ReportModule7/9.png)



### VI. Logic Code Related

The logic code here refers to the void FixedUpdate function in the MonoBehavior script, which is typically displayed as xxx.FixedUpdate in the Profiler. The number of calls to this function is influenced by the frequency of physics updates.

![7](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Physics_Module/10.png)

For example, if there are 10 small monsters in the scene that have the Monster.cs script attached, and this script contains logic code in the void FixedUpdate function, then when the physics update count for the current frame is 3, you will see that Monster.FixedUpdate is called 10 * 3 = 30 times in the Profiler for this frame.

We generally recommend writing code in the Update function as much as possible, as this can reduce the number of times the corresponding logic is executed, thereby decreasing the time spent on logic code.

The above are some issues and corresponding methods to pay attention to when optimizing the performance of the physics module. How to implement them will depend on the specific circumstances of your project. Of course, GameOptim has already developed GOT Online and local resource detection tools, which provide a wide range of detection features. We hope these tools can serve as powerful aids in optimizing your physics module.