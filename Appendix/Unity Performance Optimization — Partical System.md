Whether it's the CPU or GPU, the impact of particle systems on them is not to be underestimated. As projects become more intensive and AAA-level, players' tastes become more demanding, gameplay complexity increases, and visual effects become more intricate... Therefore, we must approach particle systems with even greater caution.

### I. How Particle Systems Affect the CPU

In the particle module of the UWA report, there are several parameters that require our attention.
"ParticleSystem.Update": The average CPU time consumed by particle systePartical System.m updates.
"ParticleSystem.Draw": The average CPU time per frame spent by the particle system submitting Draw Calls;
"ParticleSystem.ScheduleGeometryJobs": This function is related to the scheduling of multi-threaded update tasks for particle systems. Generally, a higher value indicates a greater number of active particle systems in the project.

![2](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/2.png)

Based on GameOptim's experience, the main factors typically affected are as follows:



**1. Number of particle systems**
In GameOptim's performance report, directly search (ctrl+f) for "particle system count," and these two detection results will appear:

![3](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/3.png)



(1) Particle system count: UWA recommends no more than 600 (for 1G devices). This count refers to the total number of all ParticleSystem instances in memory, including those currently playing and those in the object pool.

(2) Playing particle system count: This refers to the number of ParticleSystem components currently playing, including both on-screen and off-screen instances. We recommend that the peak number appearing in a single frame should not exceed 50 (for 1G devices).

So, how can you check which particle systems are cached during your project's runtime? Are these playing particle systems reasonable? Here’s a tip: you can view them in the report under 【Specific Resource Information】-【Particle Systems】.

![4](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/4.png)



As shown in the figure above, the blue line represents all particle systems loaded into memory, the purple line indicates potentially redundant ones, and the yellow line shows the actual number of particle systems being played. Following the lifecycle chart of the game's runtime, you can take a screenshot at any frame, especially during periods with higher counts, and then switch to the [Selected Frame] mode to view all ParticleSystems and all playing ParticleSystems for that frame.

For the two issues mentioned above, optimization and analysis can also focus on these two points:

(1) Pay attention to whether the peak number of particle systems (i.e., the blue curve) is too high. You can select a specific peak frame to check which particle systems are cached, whether they are all reasonable, and if there is any excessive caching.

(2) Pay attention to whether the peak number of playing particle systems (i.e., the yellow curve) is too high. You can select a specific peak frame to check which particle systems are playing, whether they are all reasonable, and if any production optimizations can be made (see details below).

**2. Regarding ParticleSystem.Prewarm**
In the key performance parameters of the GameOptim report, we can note a function: ParticleSystem.Prewarm, which indicates that a particle system has the "Prewarm" option enabled in the current frame. When a particle system with this option enabled is instantiated in the scene or transitions from deactivated to active, it immediately performs a complete simulation. Taking "fire" as an example, when Prewarm is enabled, the "big fire" can be seen in the first frame after loading, rather than starting from a "small flame" and gradually growing.

![img](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog/UWA_ReportModule3/5.png)



![6](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/6.png)



However, the Prewarm operation typically consumes a certain amount of time, so it is recommended to turn it off when it is not necessary.

![img](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog/UWA_ReportModule3/7.png)



------

### II. Impact of Particle Systems on the GPU Side

If particle system issues are severe, they can also affect GPU performance. We can also use the Overdraw data from GameOptim's real-device testing reports to investigate and pinpoint issues. Generally, we recommend keeping Overdraw below 5 on mid-to-low-end devices.

![8](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/8.png)



By analyzing the Overdraw trend chart alongside corresponding in-game screenshots, we can identify issues related to resource specifications, such as whether particle effects cover too large an area or have excessive overlapping layers.

![9](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/9.png)



![10](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/10.png)



**Here are some additional common optimization approaches**
For low-end devices, try to reduce the complexity and screen coverage of particle systems as much as possible to lower rendering overhead and improve the smoothness of operation on low-end devices. Specific approaches are as follows:

(1) Reduce the number of particles and on-screen particles on mid-to-low-end devices, such as displaying only "key" particle effects or particle effects released by the player's own character, thereby lowering CPU overhead during Update.

(2) Attempt to disable particle systems that are far from the current view frustum or camera, and enable them when they come closer, to avoid unnecessary Update overhead for particle systems.

(3) Minimize the screen coverage area of particle effects as much as possible. Larger coverage areas and more overlapping layers result in higher rendering overhead.



------

### III. How to Standardize Particle Systems for Everyone

The above points address the optimization of particle systems, but for most teams, this is typically done during the mid-to-late stages of a project. However, if we lack comprehensive and scientific detection and prediction of the performance pressure of particle effects, this can indeed pose risks. So, how can we ensure smooth and fluid performance on actual devices during daily development by implementing scientific art standards and detection methods?

**1. Local Resource Detection is a monitoring service launched by GameOptim for resource standardization. During daily development, it can perform performance detection on art resources such as textures, meshes, animation resources, particle effects, etc., as shown in the figure below.**

![12](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/12.png)



The detection principle of particle effects is essentially similar to ParticleEffectProfiler. The scanning process requires rendering in the Game View, which can intuitively display the detection results, making it very user-friendly for developers.

The detection rules in the report are as follows, and they will be gradually improved and expanded in the future.

**(1) Average Overdraw rate is too high during special effects playback**
Calculate the average Overdraw of pixels involved in rendering per frame and take the highest value during the process. The higher this value, the greater the likelihood that the special effects will cause GPU pressure, and it is recommended to check them.

**(2) DrawCall peak is too high during special effects playback**
Count the number of DrawCalls per frame and take the highest value during the process. When this value is high, it needs to be checked.

**(3) Total texture memory for effects is too large**
Calculate the total texture memory included in the effects. When this value is high, it may indicate excessive texture usage, which needs to be checked.

**(4) Excessive number of ParticleSystem components included during effect runtime**
Count the number of ParticleSystem components included in the special effects. When this value is high, it can easily lead to higher rendering-related metrics, as well as increased serialization time, requiring inspection.

**(5) Excessive maximum particle count during special effects playback**
Calculate the total number of particles per frame and take the highest value during the process. The larger this value, the greater the update overhead of the special effects may be.

**(6) Excessive total number of textures in the special effects**
Count the total number of textures included in the special effects. A high value can easily lead to elevated rendering-related metrics, which requires inspection.

**(7)Activate the ParticleSystem for Collision or Trigger**
It is recommended not to enable the Collision or Trigger functions in the particle system, as doing so may result in significant physics overhead.

**2. Utilize the GPU time consumption feature of GOT Online to monitor and optimize each skill effect one by one.**
(1) Run the skill effects sequentially on a real device. The size and position of the skill effects can be set by the technical team using the camera. This allows for automatic testing on the real device. By analyzing the GPU time consumption feedback from the device, it becomes possible to quickly identify which skill effects are causing high GPU pressure on devices of different performance levels.

![13](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/13.png)



![14](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/14.png)



Similarly, the same approach applies to Draw Call and Triangle counts, enabling rapid identification of bottlenecks.

![15](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/15.png)



Some teams go a step further by combining the instantiation of effects with Active/Deactive detection, which helps identify which skill effects may pose performance risks during runtime.

![16](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Partical_System/16.png)



This process has been regularly implemented across multiple teams, with positive feedback on its effectiveness.

These are some of the issues and corresponding methods to focus on when optimizing particle systems. How to implement them depends on the specific circumstances of your project. At the same time, leveraging GameOptim services can quickly help you identify performance bottlenecks.