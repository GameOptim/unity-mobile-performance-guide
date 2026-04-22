# 2.1 Overview



## 2.1.1 Runtime Memory Usage



First, before discussing memory-related parameters and setting standards, we need to clarify the actual meaning of various memory parameters commonly seen in performance tool statistics.

In many performance tools including GameOptim, the most important and frequently observed memory metric is **PSS Total (Proportional Set Size)**.

It represents the **actual physical memory used by a process in RAM**, with shared libraries allocated proportionally across processes. In short, it is the **actual runtime memory footprint of the game process**.

Some tools use **RSS Total** instead. Like PSS, RSS represents physical memory usage, but includes **the entire size of shared libraries**, often leading to an overestimated value with reduced reference value.

Notably, shared library memory does **not** cause the system to kill the process. This will be marked separately if it appears later.

In practice, both values are affected by Android OS versions, device models, and other hardware factors.

In any case, **in-game crashes** or **process killing when the game is backgrounded** (e.g., switching to social apps) severely damage user experience.

Based on GameOptim’s experience, **these two phenomena are almost always caused by excessively high runtime memory**.

When system memory reaches a high-usage warning threshold, the system automatically terminates processes according to priority to free memory.

This is the main cause of **Out-of-Memory (OOM) crashes** that developers often see in crash logs.

In the system process priority hierarchy, user applications including games are usually in a relatively low tier.

Within the same priority level, **the higher the memory usage, the higher the probability of being killed**.

Therefore, game processes are often at high risk of being terminated by the system.

Since we can barely reschedule or adjust process priorities at the engine level,

**controlling the overall PSS Total** remains the most common and effective optimization method in real projects.

Once we identify an excessively high total memory value, we must break it down to locate issues.

We decompose PSS Total from **two perspectives**:

First, the widely used **Memory Profiler**. Its memory classification and naming change across versions.

This article analyzes versions **0.7.1** and **1.1.0**, which are both popular and complementary; developers often use both together.

Below is the same snapshot parsed in both versions:

**Memory Profiler 0.7.1**

![1](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.1Overview/1.png)

The old Summary view splits memory into three clear dimensions.

It also shows **In use** and **Reserved** memory, displaying reserved memory overhead.

![2](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.1Overview/2.png)

However, the old detailed analysis page has poor integration.

Important categories such as Manager and SerializedFile are not clearly distinguished, even though they appear in Take Example.

Users must manually expand categories, hide irrelevant fields, and sort by size to obtain useful information.

**Memory Profiler 1.1.0**

![3](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.1Overview/3.png)

The new Summary removes low-value top-level categories and splits memory into **five asset types**.

The main drawback is that **reserved memory is no longer displayed**; everything (including heap reservation) is merged into **Native**.

![4](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.1Overview/4.png)

![5](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.1Overview/5.png)

The new detailed view is significantly improved: clearer classification, automatic sorting, and basic visualization.

However, reserved memory (including IL2CPP VM memory) is grouped into **Unknown**.

In GameOptim’s debugging experience, **Unknown can become extremely large** in certain cases —

for example, when AssetBundles are compressed in **LZMA format**.

Although this part usually does not require targeted optimization, hundreds of MB of unknown memory is disturbing and confusing for many developers.

Comparing both versions often helps identify problems.



![1](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.1Overview/1.png)



We will next elaborate on the Summary view in version 0.7.1. As shown in the first section of the figure, Unity divides the total statistical runtime memory (in this case, Unity captures RSS memory) into two major parts: Tracked Memory and Untracked Memory.

Tracked Memory refers to memory allocated by the Unity engine and traceable by the engine itself. It is essentially equivalent to the Reserved Total memory we have been discussing, and usually accounts for the majority of the game process memory footprint. In contrast, Untracked Memory refers to memory allocated at the system level that cannot be tracked by the engine, including some shared library memory.

The former is the main part analyzable by the Memory Profiler. Tracked Memory includes two values: In use and Reserved. Generally, the engine does not request memory from the operating system on demand. Instead, it first allocates a block of continuous memory for internal use. Only when the free memory is insufficient will the engine request another block of continuous memory from the system.

This means that in addition to the actually used memory, there is also reserved memory. The sum of the two constitutes the final memory footprint. This scenario will reappear later and will not be explained repeatedly. In general, the larger the In use memory, the larger the reserved memory. Therefore, optimization should prioritize the actual In use memory.

In the second section of the figure, Unity further decomposes Tracked Memory. We need to focus on the following four components: Managed Heap, Graphics & Graphics Driver, Audio, and Other Native Memory. Other items such as virtual machine, Profiler, and DLLs memory are either absent or negligible in the final IL2CPP Release build, and generally do not require attention.

To clarify: Graphics & Graphics Driver mainly includes textures, meshes, RenderTextures, and other rendering resources submitted to the GPU. Other Native Memory includes other resources besides the above rendering and audio resources, such as fonts, particle systems, as well as Scene Objects, serialized files, and other asset types.

All the above memory can be classified and analyzed in detail in Tree Map \ Object and Allocations. Although the Memory Profiler provides relatively complete information, it is limited to single-frame snapshots, with a maximum of two-frame comparison. In practical memory analysis, the lifecycle of memory objects and memory statistics trends are extremely important. In this regard, GOT Online Resource Mode, with its automatic sampling and interactive asset list design, has clear advantages.

However, we have found in an increasing number of projects that Untracked Memory is unexpectedly high, sometimes accounting for approximately 50% of total memory. This requires us to analyze from a second perspective.

Although similar analysis can be performed using Android Studio, we use the most direct data obtained via adb commands as an example. After connecting a real device, enter the command:

```adb shell dumpsys meminfo package_name/pid```

to obtain a memory breakdown sample as shown in the figure.

![6](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.1Overview/6.png)



The figure clearly shows the PSS Total mentioned earlier and its components at the current sampling point. Generally, developers need to focus on three parts: Native Heap, Gfx dev/EGL mtrack/GL mtrack, and Unknown.

Among them, only rendering resources in Unity Tracked Memory are recognizable at the system level, so their memory usage is distributed between Gfx dev/EGL mtrack/GL mtrack and Unknown. Unity Untracked Memory usually comes from external code, third-party libraries, plugins, etc., and is scattered between Native Heap and Unknown.

Therefore, further investigation is necessary if we find that Gfx dev/EGL mtrack/GL mtrack + Unknown is much larger than Tracked Memory, or if Native Heap is excessively high.

If the team has clear management of external code and plugins, comparative testing can be used: build packages with plugins enabled or disabled to observe memory changes. Alternatively, built-in profilers of plugins such as Wwise can be used directly. For projects with heavy Lua usage, GOT Online Lua Mode is recommended for targeted analysis.

However, in more and more projects, Native Heap memory is high and shows an increasing trend, which requires assistance from the Perfetto tool.

In summary, we have comprehensively dissected the runtime memory of a Unity mobile project. Based on this foundation, we can target problems accurately. Cases of detecting, locating, and solving memory issues in projects using the above workflow will be presented in detail in corresponding sections later.



## 2.1.2 Memory Parameter Standards



After understanding the meaning of various memory-related parameters, we know that the key to avoiding game crashes lies in controlling the **PSS memory peak**.

The majority of PSS memory comes from **asset memory** and **Mono heap memory** within Reserved Total.

For projects using Lua, Lua memory should also be monitored.

According to GameOptim’s experience, the crash risk is low only when the **PSS memory peak is controlled below 0.5–0.6 times the total device RAM**.

For example:

- For devices with 2 GB RAM, PSS memory should optimally be controlled below 1 GB.
- For devices with 3 GB RAM, PSS memory should be controlled below 1.5 GB.

In particular, GameOptim emphasizes that **Mono heap memory** requires close attention.

In many projects, apart from high resident size and potential leakage risks, the Mono heap size also affects **GC time**.

The table below provides GameOptim’s recommended standards broken down by each asset type, which are relatively strict.


<table border="1" cellpadding="8" cellspacing="0" style="border-collapse: collapse; width: 100%;">
  <tr>
    <!-- 核心：内存类型 跨2行+2列，合并目标3个格子 -->
    <td rowspan="2" colspan="2" style="text-align: center; font-weight: bold;">内存类型</td>
    <!-- 推荐值 跨3列（2G/3G/4G） -->
    <td colspan="3" style="text-align: center; font-weight: bold;">推荐值</td>
  </tr>
  <tr>
    <!-- 2G/3G/4G 列头 -->
    <td style="text-align: center;">2G</td>
    <td style="text-align: center;">3G</td>
    <td style="text-align: center;">4G</td>
  </tr>
  <tr>
    <!-- 总内存 - 跨2行 -->
    <td rowspan="2" style="text-align: center; font-weight: bold;">总内存</td>
    <td style="text-align: center;">PSS Total</td>
    <td style="text-align: center;">1 GB</td>
    <td style="text-align: center;">1.5 GB</td>
    <td style="text-align: center;">2 GB</td>
  </tr>
  <tr>
    <td style="text-align: center;">Reserved Total</td>
    <td style="text-align: center;">700 MB</td>
    <td style="text-align: center;">1.2 GB</td>
    <td style="text-align: center;">1.5 GB</td>
  </tr>
  <tr>
    <!-- 资源内存 - 跨4行 -->
    <td rowspan="4" style="text-align: center; font-weight: bold;">资源内存</td>
    <td style="text-align: center;">Texture</td>
    <td style="text-align: center;">140 MB</td>
    <td style="text-align: center;">210 MB</td>
    <td style="text-align: center;">280 MB</td>
  </tr>
  <tr>
    <td style="text-align: center;">Mesh</td>
    <td style="text-align: center;">60 MB</td>
    <td style="text-align: center;">100 MB</td>
    <td style="text-align: center;">140 MB</td>
  </tr>
  <tr>
    <td style="text-align: center;">Shader</td>
    <td style="text-align: center;">40 MB</td>
    <td style="text-align: center;">50 MB</td>
    <td style="text-align: center;">60 MB</td>
  </tr>
  <tr>
    <td style="text-align: center;">Animation Clip</td>
    <td style="text-align: center;">40 MB</td>
    <td style="text-align: center;">60 MB</td>
    <td style="text-align: center;">80 MB</td>
  </tr>
  <tr>
    <!-- Mono 堆内存 -->
    <td style="text-align: center; font-weight: bold;">Mono 堆内存</td>
    <td></td>
    <td style="text-align: center;">80 MB</td>
    <td style="text-align: center;">100 MB</td>
    <td style="text-align: center;">120 MB</td>
  </tr>
  <tr>
    <!-- Lua 内存 -->
    <td style="text-align: center; font-weight: bold;">Lua 内存</td>
    <td></td>
    <td style="text-align: center;">100 MB</td>
    <td style="text-align: center;">100 MB</td>
    <td style="text-align: center;">100 MB</td>
  </tr>
</table>


![7](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.1Overview/7.png)

![8](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.1Overview/8.png)



Nevertheless, developers should adjust them based on the actual conditions of their own project.

For instance, a 2D project consumes almost no mesh resources, so the standards for other resources can be relaxed significantly.

After establishing memory standards based on actual project conditions,

further coordination with artists and designers is generally required to define reasonable art specification parameters and document them.

Once specifications are finalized, regularly check all art assets in the project for compliance, and modify or update them in a timely manner.

The compliance checking process can be automated via tools developed by engineers inside the engine to improve efficiency.

Alternatively, GameOptim’s local asset detection feature (described later) can be used to screen assets with incorrect settings.

If assets cannot be batch-processed into high, medium, and low versions,

artists must create different resource variants for each quality level.



## 2.1.3 Local Resource Detection Service – Project Resource Detection



The engine settings for various resource memories are complicated, and not all of them can be collected at runtime. Although the content mentioned below are common and important issues in many projects, real project situations are more complex. More resource setting problems can often be found through the project resource detection interface of the local resource detection service. They not only affect the memory footprint of related resources, but also impose varying degrees of pressure on CPU time and GPU depending on the situation.

For this reason, GameOptim has designed detection rules and thresholds based on experience. On this basis, it collects and counts resources with such problems and provides corresponding optimization suggestions, helping developers conduct more in-depth investigation and optimization on resources.

![9](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.1Overview/9.png)





