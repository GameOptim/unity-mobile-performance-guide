# 5. Software‑Level Optimization for Power Consumption, Heating and Throttling

## 5.1 Basic Principles of Power, Heating and Throttling

### 5.1.1 Mechanism of High Power Consumption and Heating on Mobile Devices

Power consumption during mobile game operation consists of four parts:

**core computing power (CPU/GPU/NPU)**, **memory access power**, **peripheral power (screen/radio/sensor/storage)**, and **system scheduling overhead**.

Common high-power scenarios for games:

- In intensive scenes (combat, open-world rendering, physics simulation, network synchronization), **CPU/GPU run at high frequency continuously**, leading to sustained high power consumption.
- Limited heat dissipation area causes **heat accumulation**. Hot modules heat up nearby components, causing cascading performance degradation.
- Power and performance do **not** have a linear relationship. Beyond the **energy efficiency inflection point**, small performance gains lead to **exponential increases in power and heat**.

When the SoC temperature reaches the thermal threshold, the Android kernel and firmware activate **multi-level thermal protection**:

- Force lower CPU/GPU frequency
- Restrict big core scheduling
- Reduce concurrent threads

This **frequency throttling** directly causes **severe frame drops**.

Notably, this issue is often **more obvious on high-end devices**:

even if raw computing power is sufficient for stable frames, overheating and throttling still cause **hot devices and low FPS**.

### 5.1.2 Quantitative Data Analysis for Power and Heating

Developers must avoid subjective judgments like “feels hot” or “bad battery life”.

Instead, **quantitative data** is required to:

- Identify the power proportion of each hardware module
- Locate timing of high-power scenes

The core difficulty:

power and heating lie **outside the game engine’s control** — they are low-level hardware behaviors.

Different chips, SoCs, and vendors have different thermal designs and protection mechanisms.

Even identical units may represent different actual costs.

Environment (temperature, charging) also strongly affects results.

Therefore:

- Use **fixed devices and stable tools**
- Perform **vertical comparison (before/after optimization)**
- Avoid **cross-device/cross-tool horizontal comparison**

### 5.1.3 Power Tuning Practice with Perfetto

#### 5.1.3.1 Review of Power Testing Tools

Accurate data collection is the first step.

(1) **GOT Online**

Collects FPS, frame time, CPU core frequency, memory, power peak, etc.

Supports screenshots, export, and comparison.

Good basic monitoring, **limited ability to break down hardware power**.

(2) **Trepn Profiler**

Low practicality; unstable accuracy.

`adb shell dumpsys batterystats` provides only battery level with low dimension.

Both are **auxiliary only**.

(3) **Android Studio Profiler (ODPM)**

Supports CPU big/LITTLE, GPU, memory, screen, WiFi power.

But requires **debuggable build**, only supports **Pixel devices**, short test duration.

**Poor generality**.

(4) **Perfetto (Best Choice)**

No debuggable build required.

Supports real-time recording via USB/WebSocket.

Google provides detailed power rails **only on Pixel devices**:

`power.rails.cpu.big`, `power.rails.gpu`, `power.rails.memory.interface`, etc.

Supports online analysis at `ui.perfetto.dev`.

**Main tool for the following tuning practice**.

Unity mobile game power is dominated by:

**CPU, GPU, memory, screen, network, SoC**.

Unified test standard:

- Fixed 30 FPS
- Fixed brightness and refresh rate
- AB comparison before/after optimization
- Perfetto for hardware power breakdown
- GOT Online for CPU time
- Gears for GPU counters
- Renderdoc/Xcode for frame analysis

#### 5.1.3.2 Impact of GPU Clocks and Bandwidth on Power



![](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/95.png)

As shown in the test chart:

**GPU is a major source of game power consumption**, strongly correlated with **graphics quality** and **bandwidth**.

Test results:

- From Battery Saver → Smooth → Standard → High Quality:

  - GPU Bandwidth: 1195 MB/s → 1483 MB/s
  - GPU Clocks: 264 MHz → 393 MHz
  - GPU Power: 335 mW → 543 mW
  - Total device power: **+270 mW**


![96](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/96.png)

**Bandwidth is the key factor**.

Transparent layers, texture format, and Mipmap have huge impact.

Comparison:

- ASTC4x4 compression reduces bandwidth and power.

- **Mipmap is far more effective**:

  - 30 layers with Mipmap: bandwidth 3.38 GB → 0.90 GB, power **−49 mW**
  - 50 layers with Mipmap: power **−160 mW+**

  

**Conclusion: All 3D textures MUST enable Mipmap.**

![97](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/97.png)

Estimated relationship:

**1 GB/s bandwidth ≈ 67 mW power**

Recommendations:

- Enable Mipmap for all 3D textures
- Use ASTC4x4, ETC2 compression; avoid uncompressed
- Tier graphics settings by device
- Reduce transparent layers and invalid rendering
- Lower GPU bandwidth

#### 5.1.3.3 Impact of Screen and Network on Power

**Screen power depends on three factors**:

**brightness**

![98](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/98.png)

**refresh rate**

![99](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/99.png)

**color complexity**

![100](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/100.png)

Test:

- 90 Hz > 60 Hz at same brightness

- Higher brightness → higher power

- Colorful image >> pure black image

  - 100% brightness: colorful 575 mW, black 140 mW (**4× difference**)


![101](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/101.png)

**Network is a hidden high-power component**.

Test:

- Idle WiFi: 62 mW
- 10 MB/s download: 359 mW
- Mobile data modem: 280 mW → 583 mW
- Total power increase: **+500 mW**

Recommendations:

- Dynamically adjust screen brightness
- Lower refresh rate in non-critical scenes
- Simplify overly colorful UI
- Reduce large-file background downloads
- Minimize continuous network transmission

#### 5.1.3.4 Impact of CPU on Power

![102](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/102.png)

CPU dynamic power formula:

**P_dynamic ∝ C × V² × f**

Power grows **exponentially** with frequency.

Test:

- Main thread intensive computing: **3.67 W**, big core 2.58 GHz
- After moving to **JobSystem**: **2.55 W**, big core 1.4 GHz

![103](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/103.png)

**Worker threads also contribute significantly** but are hard to analyze at the engine level.

**SimplePerf workflow**:

- Release build
- Generate symbol files
- Collect and analyze worker threads
- Locate high-cost functions

Example:

A **network thread** consumed more than the render thread and nearly matched the main thread — highly abnormal.

Optimization result:

![104](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/104.png)

![105](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/5Optimization/105.png)

CPU power **4820 mW → 3838 mW** (nearly **1000 mW reduction**).

Recommendations:

- Split heavy main-thread logic
- Offload physics, animation to **JobSystem**
- Avoid big core high-frequency load
- Use SimplePerf to locate high-cost worker functions
- Remove empty loops, redundant traversals
- Use **Release build**, disable debug overhead