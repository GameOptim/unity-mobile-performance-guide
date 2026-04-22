1.1 Game Performance from Players' Perspective

During gameplay, every player will encounter similar questions:

Why does the game crash after playing for a while? Why is the game so laggy? Why does my device get so hot? Why does the battery drain so quickly on mobile devices?

For players, regardless of how exquisite the art or interesting the gameplay is, once such issues occur, the gaming experience naturally deteriorates, inevitably leading to reduced user retention. This is an outcome no game developer wishes to see.

In reality, for the current mobile game market, there is a huge performance gap among various hardware models in China. If the project targets the overseas market, the performance pressure is usually even greater. Therefore, the core topic of this article is: how to ensure smooth gameplay for players with different hardware devices, especially those using low-to-mid-end devices.



1.2 Game Performance from Develops' Perspective

A common phenomenon in game development is that teams often focus entirely on feature implementation in the early and middle stages, while neglecting performance optimization. By the late development stage or near launch, testing reveals severe performance issues that ruin the user experience, yet the team has little time left to fix them. After launch, massive negative feedback on performance further damages user retention.

It is clear that performance is a critical lifeline of a game project. Developers must establish a sound optimization mindset and knowledge system to avoid detours.

Unlike players’ subjective experience (crashes, lag, overheating, high power consumption), GameOptim classifies these issues into three core categories from a technical perspective: Memory, CPU, and GPU. This article analyzes typical problems in these three areas to form a concise performance optimization methodology for Unity mobile games.

Once performance issues are identified, problem diagnosis and problem solving become the two most critical steps.



1.3 How to Identify Performance Issues

The first step after detecting performance problems is to locate and define the root cause.

For example:

- If the game crashes after prolonged running, is it due to poor caching strategies, redundant resources, and continuous memory leakage? Or is it caused by oversized art assets consuming excessive memory?
- If a scene is severely laggy with low frame rate, is it due to excessive triangle count and Draw Calls causing high rendering pressure? Or is it caused by overly complex and frequently updated UI?
- If the device overheats significantly, is it due to excessive GPU load in the current scene, or inappropriate level-of-detail (LOD) and grading strategies for the target device?

These are only a few examples among various performance issues. Common problems will be discussed in detail later. In short, identifying the root cause is itself a major challenge.



1.3.1 Various Auxiliary Tools

As the saying goes: A workman must first sharpen his tools if he is to do his work well. Using tools to obtain direct and accurate performance data can intuitively reveal performance bottlenecks and achieve twice the result with half the effort.

With the development and maturity of the game industry, major engines, IDEs, and hardware manufacturers have continuously released and updated their own performance analysis tools. Many developers or teams also prefer to build custom performance monitoring plugins tailored to their projects.

For engines, Unity Profiler is widely used by developers. It can record and view real-time consumption of modules including CPU, rendering, UI, memory, and more. Unity also provides increasingly mature and practical built-in tools such as Frame Debugger and Memory Profiler.

For IDEs, dedicated tools are available for Windows, macOS/iOS, and Android platforms, such as the Performance Profiler in Visual Studio, Instruments in Xcode, and Profiler in Android Studio.

For hardware-level debugging, tools including Snapdragon Profiler, Xcode Metal Frame Capture, and Mali Graphics Debugger are widely adopted — the list goes on.

In short, developers are often spoiled for choice when selecting performance tools, considering factors such as proficiency, cross-platform compatibility, and functional coverage.

1.3.2 GameOptim Tool

Is there a performance optimization tool that is easy to use, highly compatible, and fully functional?

The answer is yes.

GameOptim GOT Online can be quickly integrated into test projects via the official GameOptim SDK. After testing on real devices, data is uploaded and parsed within a short time, automatically generating a full set of visualized charts. Meanwhile, the tool provides performance scores based on GameOptim’s extensive optimization experience and massive database, along with targeted analysis suggestions and trend curves for each performance module.

In addition, GameOptim offers Gears, a free toolset that supports developers’ optimization needs from multiple perspectives.

1.4 How to Resolve Performance Issues

The second key step in performance tuning is to solve the problems. With sufficient knowledge, experience and the support of the aforementioned tools, the more complex and late-stage a project becomes, the longer its performance optimization list will be. The next challenge is to properly plan and shorten this list within limited time.

1.4.1 Optimization Prioritization

Before carrying out specific optimization, reasonable planning can usually save a significant amount of total time.

The first rule of planning is to grasp the main contradiction. In a long optimization list, it is often unrealistic to optimize every item completely and in parallel. Instead, the list should be reorganized and sorted by priority, with more resources allocated to issues with higher optimization cost‑effectiveness.

According to GameOptim, priority judgment mainly depends on two factors:

First, importance. If an issue contributes far more performance pressure than others and is clearly the main bottleneck — without fixing it, normal gameplay cannot be maintained — it should be resolved with concentrated effort and high priority.

Second, ease of implementation. If an issue causes relatively low performance overhead but can be fixed with minimal effort (e.g., simply toggling an option in engine settings), it should also be addressed early.

1.4.2 Trade‑off Between Performance and Visual Quality

The second point is trade‑off. Optimization related to resources and rendering often affects visual quality. Programmers therefore need to communicate closely with designers and artists to find a balanced point between performance and visuals.

This balance is dynamic and depends on actual project conditions:

- A battle royale game features frequent scene changes and requires smooth real‑time combat, so visual quality can be moderately reduced.
- A collection‑oriented game uses high‑quality characters as its core selling point, so it may maintain highly detailed models and animations even at some performance cost.

Even within the same game: prioritize visual quality in cutscenes and character showcase interfaces, but prioritize performance in combat scenes — specific analysis for specific scenarios.

In general, the ultimate goal of game optimization is to run smoothly on players’ devices. Sacrificing a certain degree of visual quality is a common choice for most development teams.

However, through extensive project practice, GameOptim has found that a large proportion of performance overhead comes from completely useless, non‑contributory issues. Solving these problems should obviously have higher priority.

1.4.3 Tiering Strategy and Standard Definition

Third, achieving the aforementioned balance on different device tiers forms our performance tiering strategy.

We can enable post‑processing, high resolution and high‑poly models on high‑end and flagship devices, while applying separate standards for mid‑range and low‑end devices.

The tiering strategy must also be adjusted according to the game genre.

1.5 How to Continuously Monitor Performance

After completing the first two steps of performance optimization, many development teams intend to take a third step: continuous performance monitoring.

To achieve this, some teams assign dedicated engine optimization engineers and develop custom DevOps tools. Once a relatively mature internal system is established, it will reduce the cost of subsequent project development and maintenance, allow experience to be inherited into new projects, and greatly improve overall development efficiency.

Other teams also have a genuine need to continue monitoring performance after the game has launched.


