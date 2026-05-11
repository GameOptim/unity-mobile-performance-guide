Teams often ask: Why is my game loading so slowly? Can we achieve scene transitions as fast as in *** game? What causes the game to overheat? To prevent your hard-earned game from lagging or turning into a hand warmer on real devices, we need to make full use of the Loading Module and Resource Management Module in the UWA report. We will analyze this step by step below.

### I. Key Concerns in the Loading Module

Let’s first go over some key metrics related to loading. The image below shows the loading module tab in the Overview report. On the left, we can see several important parameters:

![1](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/1.png)

**1、Loading.UpdatePreloading**
This is the main loading function of the Unity engine. It typically incurs significant overhead when switching scenes or asynchronously loading resources. Generally speaking, the more resources loaded and the more complex they are, the longer the Load.UpdatePreloading takes.

![2](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/2.png)

Before optimizing this function, it is recommended to first identify its time-consuming bottleneck. By checking the CPU call stack in the report, you can view the detailed stack trend of this function during runtime, gaining a clear understanding of its time allocation, and thus optimize it with a targeted approach.

![3](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/3.png)

**2、Resources.UnloadUnusedAssets**
This function is used to unload unused resources, and its overhead mainly depends on the number of Assets and Objects in the scene—the more there are, the higher the time cost. When optimizing performance, in addition to peak time consumption, we also need to pay attention to the number of times this function is called.

![4](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/4.png)

Generally, during scene transitions, the engine will automatically call it once. UWA recommends manually calling it once every 10 to 15 minutes.

At the same time, the development team can try using Resources.UnloadAsset during game runtime to remove a specific resource that is confirmed to be no longer needed. This API is highly efficient for unloading a single resource and can also reduce the pressure when Resources.UnloadUnusedAssets is called for batch processing.

The following image shows the stack information for loading-related functions in the report. In the stack, GarbageCollectAssetsProfile is caused by calling Resources.UnloadAssetsUnused. If this takes up too much time, it is necessary to check whether Resources.UnloadUnusedAssets is being called too frequently.

![5](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/5.png)

**3、GC.Collect**
The frequency of GC calls is primarily affected by heap memory. The more and more frequently heap memory is allocated by a function, the sooner GC will occur. Therefore, when the frequency of GC.Collect function calls is relatively high (as shown in the figure below), especially when it becomes increasingly frequent as the game runs longer, we need to pay attention to whether there are functions that allocate heap memory heavily or frequently. In this case, we can use the Mono mode of GOT Online to check if there is any phenomenon of excessively fast or high Mono allocation.

![6](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/6.png)

**4、Instantiate**

![7](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/7.png)

This section measures the time taken for resource instantiation. The more complex the project's resources and the greater the number of instantiations, the more noticeable the lag becomes. However, this aspect is often overlooked. So, how does UWA handle this issue effectively? Below, we will provide a detailed explanation using the [Resource Management] module from UWA's real-device testing report.

------

### II. Resource Management

Resource management here refers to strategies concerning the frequency and duration of resource calls, as the factors affecting loading experience are essentially two: the frequency of loading and the time consumed per load. In the real-device testing report, under the [Resource Management] label, the following inspection items are included:

![8](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/8.png)

With so many features, what details should we focus on? Let's go over a few key points:

**1. Focus on high-cost loading**
Whether it's AssetBundle or resource loading, those with high time costs require special attention. Here, if we open a resource loading tab, we can see the details of resource calls throughout the entire runtime process below, with the last column showing the time cost.

![9](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/9.png)

In the specific resource details, by selecting a resource, we can view its call details during runtime. Corresponding to the screenshot above, we can further investigate whether the AssetBundle loading really needs that much time.

![10](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/10.png)

**2. Focus on High-Frequency Calls Within a Short Period**
Whether it's AssetBundle or resource loading, attention must be paid to the frequency of loading. Typically, for objects that are loaded frequently, we can establish a cache pool: load them once, add them to the cache, and avoid subsequent loading.

As shown in the figure below, these frequently loaded AssetBundles, **which may originally have taken 5ms or 50ms each time, can be reduced to 0ms afterward**.

![11](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/11.png)

**Additionally, it's worth noting that we should also watch out for the issue of the same resource being loaded multiple times within a single frame.**
As shown in the figure below, this frame calls it 5 times, which is incorrect.

![12](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/12.png)

**3. Watch out for non-existent resources**

![13](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/13.png)

In the resource loading list, some projects may encounter resources marked as [non-existent], indicating that these resources failed to load because they were not located in the specified path. Generally, such resources arise from version iterations where resources were deleted or moved without modifying or commenting out the corresponding code.

Loading these 【not-exist】 resources only caused a small portion of CPU overhead, but more importantly, investigating these 【not-exist】 resources can prevent logical issues that lead to crashes and freezes.



**4. Frequent Instantiation/Destruction**
Resources with high operation frequency or high time consumption. Frequent Instantiate will cause a certain amount of heap memory allocation, thereby accelerating the frequency of system GC calls. More importantly, frequent instantiation will cause CPU time consumption to peak, affecting the smoothness of the game, so this is also an area we need to pay attention to.

![14](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/14.png)

For resources that are frequently instantiated, use a cache pool to reuse GameObjects that have been instantiated too many times, thereby reducing the time spent on GameObject instantiation.

**5、Activate and Deactivate**
This troubleshooting method is similar to instantiation, focusing mainly on call frequency and time consumption.

![15](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/15.png)

Compare the call counts of Activate and Deactivate. If the difference between the two is too large, it indicates the presence of unnecessary Activate/Deactivate operations.

![16](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/16.png)

For example, a certain resource has a very high number of Activate operations (such as Gold_2 and Gold_4 in the image below). Why is the count so high? Is it necessary? We can copy the resource name and search for it in the Deactivate resource list to check if such frequent state activations are truly needed.

![17](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/17.png)



Deactivate - Gold_2

![18](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/18.png)



Deactivate - Gold_4

![19](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/19.png)



**This indicates that the more than 1,000 Deactivate operations that differ are all meaningless.**

For the above resources, we can create a special cache on the C# side to record the Active state (True or False) of the object. Before calling SetActive, we first check whether the current state is already the desired state, and only call it if it is not. This is because the SetActive operation transitions from the C# layer to the C++ layer, so performing state checks in C# can reduce such cross-language operations, thereby avoiding unnecessary overhead.

**6. AssetBundle Retention Optimization**
The reason for paying attention to this parameter is that it affects memory usage during project runtime. It's important to note that part of Unity's memory is related to Serializedfile caused by AssetBundle residency. Generally, we recommend keeping the number of AssetBundle resources under 1000. Considering that this metric is related to the complexity of the project itself, everyone needs to conduct their own experiments to balance the trade-off between CPU and memory.

![20](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/20.png)

Resource loading can be optimized using a cache pool approach, and AssetBundle loading follows a similar principle. Frequently loading and unloading the same AssetBundle is usually unreasonable (as shown in the figure below). For AssetBundles that are frequently loaded and unloaded, it is recommended to add them to the cache and keep them resident in memory.

![21](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/21.png)



### III. Shader.Parse/CreateGPUProgram



If the parsing and loading strategy for Shader resources is not handled properly, it can also result in high CPU overhead. Since Shader resources have a small memory footprint but take a relatively long time to load, we recommend that, under ideal circumstances, all Shader resources be fully loaded and cached at the start of the project runtime.

**1. Shader.Parse**

![22](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/22.png)

The time consumption of this function is mainly due to the loading and parsing of Shaders, typically caused by repeated Shader loading. When optimizing, it is necessary to examine the specific Shader loading situation, which can be approached from the following three points:

(1) Avoid using Standard Shader; use other shaders instead. Pay attention to check whether the Standard Shader is loaded into the AssetBundle due to model import.

![23](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/23.png)

(2) Resolve the Shader redundancy issue. This can be examined in conjunction with the Shader memory trend, as shown in the figure below.

![24](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/24.png)

If the Shader resources are not cached in memory, they will be released when leaving a scene and reloaded when entering a scene, resulting in significant redundant overhead. To solve this problem, simply separate the Shader, package it into a separate AssetBundle based on dependencies, and cache it without unloading after loading. This way, there will be no need to reload the Shader afterward.

(3) Reduce Shader Keywords.



**2、Shader.CreateGPUProgram**

![25](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Loading_ResourceManagement/25.png)

The CPU usage of this API is the time taken to create the GPU program when the Shader is first rendered, and its duration is related to the complexity of the rendered Shader. To address this, it is recommended that development teams load Shaders through ShaderVariantCollection and Warmup them after loading, thereby avoiding the time consumption of Shader.CreateGPUProgram during game runtime.

The above are some issues to pay attention to when optimizing loading. How to proceed depends on the actual situation of your project. Additionally, combining with UWA's online assessment service can quickly help you identify performance bottlenecks.









