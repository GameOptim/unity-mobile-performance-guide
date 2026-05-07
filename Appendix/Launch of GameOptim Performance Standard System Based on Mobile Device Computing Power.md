# Launch of GameOptim Performance Standard System Based on Mobile Device Computing Power

During daily game optimization, we are constantly asked such questions by R&D teams:

“How low should we keep our PSS memory on a 4GB RAM device? What are the normal upper limits for texture memory, Mono memory, and Shader memory?”

“What is an appropriate number of triangles and reasonable upper limit for Draw Calls on XXX device?”

“On XXX device, Animators.Update takes 6.8ms. Is this high or low? Do we need further optimization, and what is the reasonable target?”

“The temperature on XXX device reaches 55°C. Is this considered high?”

…

Do these questions look familiar? They essentially relate to **standardization**—a critical aspect of game development. Almost all commercial games consider this at project initiation: defining development standards, including production specifications and quality guidelines. The questions above are just a small part of the performance and quality standards that R&D teams care about most. Everyone expects clear answers: specific values or ranges for performance metrics on a given device. However, this seemingly simple demand is extremely difficult to fulfill.

First, we need **reasonable computing power tiers** for a vast number of devices. There are tens of thousands of mobile devices worldwide, with numerous new models released every year. Meanwhile, computing power varies drastically—for example, the Xiaomi 5S and Xiaomi 12 have completely different performance capabilities. We must first tier these massive devices properly, with different performance standards for each tier.

To solve this, we designed and implemented a **device tiering solution**. GameOptim Device Tiering comprehensively considers device configurations, latest hardware parameters, user feedback, and massive test device distribution data. It supports timely updates to reflect changes in hardware environments and device distribution.

Second, standardization requires **continuous extraction of key metrics**. A game project is highly complex, with over 30 modules including rendering, UI, logic code, loading, physics, particle systems, memory, ALU, bandwidth, etc. Each module can be broken down into more detailed indicators—much like a human physical exam, which first covers major organs and then enzymes, proteins, and other biomarkers. We must identify which metrics reflect rendering performance and which indicate physics bottlenecks. This demands deep expertise in performance optimization and continuous iteration across different engines and devices.

Third, development teams have **unique project requirements**. Some projects target stable 30 FPS, others 60 FPS. Some aim for smooth operation on 2GB RAM devices, while others prioritize visual quality by loading more resources and abandoning devices below 3GB RAM. These differences lead to varied performance standards. Therefore, we conducted grouped analysis on massive performance data, mined statistical patterns and importance of each metric across device tiers, and built an impact model between metrics and target FPS to derive threshold ranges for different devices and goals.

Finally, **verifying rationality and universality** is the core of standardization. Our optimization experts review metric ranges for rationality. We build test projects for key metrics across engine versions and configurations, then validate and adjust thresholds based on real-device data. The final performance standard excludes outliers and is fully feasible. We continuously compare and track the latest test data to maintain timeliness.

By solving the above four major challenges and countless detailed issues, we officially launch our **performance standard system**. It provides specific performance standards for different mobile device tiers, project goals, and game modules.

When teams test and analyze game performance on internal mobile devices, we **automatically tier devices** and provide matching performance standards based on tier and target FPS.

The figure below shows recommended metric values for a game scene on two different mobile devices.

![1](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/PA_normalvalue/1.png)

Obviously, the R&D team has properly graded devices:

- On the low-end Meizu M6 Note, triangle peaks at 238,000. GameOptim recommends **below 150,000**, indicating overuse. The team can locate high-triangle areas via reports and simplify models.
- On the mid-to-high-end Huawei Mate 20, triangle peaks at 315,000, while the recommendation is **below 450,000**—within a reasonable range. The team may discuss adding model details with artists for better visuals.

Similarly, recommended values for Draw Calls, opaque/transparent rendering time, animation, physics, and other rendering module indicators vary by device tier.

For memory usage standards, the figure below shows texture memory usage and recommendations on Redmi 4X and Xiaomi 8.

- On low-end Redmi 4X: texture memory peaks at 136.9MB, recommended **< 140MB**.
- On Xiaomi 8: texture memory peaks at 219.5MB, recommended **< 280MB**—no crash risk.

![2](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/PA_normalvalue/2.png)

In addition, teams can set **target FPS** in projects. Recommended performance metrics change accordingly, with high-priority alerts for items exceeding thresholds, as shown below.

![3](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/PA_normalvalue/3.png)

Naturally, the GameOptim Performance Standard will **continuously iterate** as performance data expands, hardware computing power improves, and engine versions evolve.

To help teams customize project-specific standards, we provide a **user-defined threshold function**. After modifying recommended values, optimization tasks with corresponding priorities appear when thresholds are exceeded. Thresholds are divided into three categories:

## Recommended Memory Usage

Resource memory and Lua memory are major memory consumers. R&D teams can set graded rules (resolution, post-processing, etc.) based on target device RAM.

![4](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/PA_normalvalue/4.png)

## Recommended Engine Module Metrics

Includes common optimization indicators: Draw Call count, triangle count, and various engine module parameters.

![5](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/PA_normalvalue/5.png)



## Recommended Runtime Information

Covers runtime performance metrics: CPU time, FPS, network transmission, Mono allocation, etc.

With these parameters, teams can quickly configure a threshold system tailored to their project needs.

![6](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/PA_normalvalue/6.png)

In addition, the latest version of the GameOptim scoring system has undergone iterative optimization to provide a clearer direction for performance improvements.

![7](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/PA_normalvalue/7.png)

1. The project score is composed of three components: "Runtime Performance," "Project Resources," and "Package Resources."
2. The project score is updated whenever any of these module scores are revised.
3. Each day, the above scoring modules are validated. If the scores have not changed in the past 15 days, the scores for each module will be recalculated according to the established rules, and the project score will be updated accordingly.