# 2.3 Texture Resources

## 2.3.1 Texture Format

Improper texture format settings are usually one of the main reasons for large memory consumption by texture resources. Even for many projects that have established art resource standards and uniformly modified texture formats, it is still easy to find a large number of textures in formats such as RGBA32, ARGB32, RGBA Half, and RGB24. Textures in these formats not only occupy large memory but also lead to problems such as large package size, long loading time, and high texture bandwidth.

![18](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.3Texture/18.png)

The main reasons for such problems are as follows: some textures are missed during batch processing — for example, textures are not modified by callback functions due to non-standard art naming, or texture formats are not set for resources created in code; the hardware or texture itself does not support the target format, causing the texture to be parsed as uncompressed.

For the former case, after identifying problematic resources in the resource list, you need to check and modify them in the project. For the latter case, the main hardware-supported texture formats recommended by GameOptim are **ASTC** and **ETC2**.

ETC2 requires texture resolution to be a multiple of 4. For ASTC textures with Mipmap enabled, the resolution must strictly be a power of 2. Otherwise, the texture will still be parsed as uncompressed.

## 2.3.2 Mipmap

Enabling Mipmap does not reduce memory; it even increases memory usage. The main reason for discussing it here is that this setting serves as a prerequisite for extensive discussions later, so basic understanding is necessary.

When Mipmap is enabled for a texture, its memory usage increases to **1.33 times (4/3)** the original size. This is because the engine automatically generates multiple Mipmap levels in memory, and the total memory is the sum of all levels.

For example, a 1 MB texture (1024×1024, ASTC4×4) with Mipmap enabled has 11 levels. The total memory is the sum of the geometric series:

1 + 1/4 + 1/16 + … ≈ 4/3.

Each term corresponds to the memory of Mipmap 0, 1, 2, etc.

The value of Mipmap is that when the mobile GPU samples textures, it automatically selects an appropriate Mipmap level based on the distance from the rendered object to the camera.

To understand in reverse: if a 2048×2048 or higher texture is used for a distant object with small on-screen pixels, the GPU must frequently fetch large texture blocks from off-chip memory, resulting in high cache miss rates and increased bandwidth overhead.

Mipmap allows the GPU to choose a suitable resolution level, greatly **reducing GPU bandwidth**.

For 3D objects such as terrain, models, characters, particle systems, and Spine textures with varying distances to the camera, **enabling Mipmap is strongly recommended**.

For UI textures with fixed distance, Mipmap is generally unnecessary, but artists should still use reasonable resolutions.

From a memory perspective, although Mipmap increases memory by dozens of MB for some textures, the benefits of bandwidth optimization and follow-up optimization based on Mipmap outweigh the costs for most projects.

## 2.3.3 Resolution

Texture resolution (width and height in the resource list) is also a major cause of excessive memory usage. Generally, higher resolution means larger memory. Special attention should be paid to high-resolution textures (usually ≥ 1024). On mobile platforms, players can barely distinguish ultra-fine details, so excessive resolution often represents unnecessary waste.

![19](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.3Texture/19.png)

However, how to judge whether reducing resolution saves waste or sacrifices visual quality? How can programmers persuade artists to adjust resolution?

Here we introduce the second key concept in **GOT Online GPU Mode – Rendering Resource Analysis**:

**Number of resources with excessively low Mipmap 0 sampling rate**.

Similar to rendering utilization, the tool samples frequently during testing and counts the proportion of each Desired Mip level when the texture is rendered.

Example:

In 10,000 samples, Texture A is rendered in 3,000 samples.

Only 300 samples have **Desired Mip = 0** (GPU attempts to sample the highest resolution).

The other 2,700 samples use Desired Mip = 1.

Thus, the **Mipmap 0 sampling rate = 10%**.

In practice, camera distance changes dynamically. But if a texture’s **Mipmap 0 sampling rate is below 5%** throughout the test or in a single scene, the GPU rarely needs the highest resolution. Most of the memory is occupied but not used — this is clear waste.

In extreme cases, even Mipmap 0+1+2 combined utilization is below 5%, resulting in severe waste.

![20](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.3Texture/20.png)

By checking **Number of resources with low Mipmap 0 sampling rate** in the Rendering Resource Analysis panel, resources with suspected waste can be filtered. Developers can then verify and optimize them in the project.

For example, the resource `Tex_B3_Songshu` (a squirrel texture) is 1024×1024 with Mipmap enabled. However, Desired Mip 0 and 1 are **0%** during actual rendering, and Desired Mip 2 accounts for 85.71%.

This means the texture can be safely reduced to **256×256** with no visual loss while saving most memory.

![21](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.3Texture/21.png)

Using different texture resolutions on different device tiers is a practical and easy-to-implement **tiering strategy**.

This also applies to atlas textures.

Unity provides the **SpriteAtlas Variant** function, which can quickly create a scaled-down variant atlas for lower-tier devices.



## 2.3.4 Global Mipmap Limit & Texture Streaming

In addition to identifying and modifying wasteful textures one by one using the methods described above, there are two automatic approaches for controlling total texture memory in projects where most textures have Mipmap enabled and memory pressure is high. These are **Texture Quality / Global Mipmap Limit** and **Texture Streaming**, located under **Project Settings – Quality**.

![22](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.3Texture/22.png)

### (1) Texture Quality / Global Mipmap Limit

The **Texture Quality** setting available in many older Unity versions is a fairly brute-force method. When enabled, the engine forcibly removes corresponding Mipmap levels from all textures with Mipmap enabled for that quality tier. For example, setting it to **Half Res** prevents all Mipmap 0 layers from being loaded into memory.

This one-size-fits-all approach reduces memory very effectively on low-end devices under extreme memory pressure, but many projects avoid it due to significant visual impact.

![23](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.3Texture/23.png)

![24](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.3Texture/24.png)

In newer Unity versions (2022.3.60 in the figure), this function has been replaced by **Global Mipmap Limit**, which follows the same logic of forcibly removing Mipmap levels. However, beyond this global setting affecting all Mipmap textures, a new **Mipmap Limit Groups** system allows different removal levels per group.

After setting different layer removal counts for each group in Quality settings, you can enable **Apply Mipmap Limits** in a texture’s advanced settings and assign a group, allowing different Mipmap removal strategies for different textures. This may be familiar to developers with Unreal Engine experience, although Unity maintains a separate set of texture groups specifically for Mipmap control.

### (2) Texture Streaming

When project resource volume is large and flexible tiering strategies are required, the above functions may still be insufficient, with high maintenance costs and limited flexibility.

**Texture Streaming** adopts a **streaming loading** mechanism: the engine automatically decides which Mipmap levels should reside in memory, thereby reducing texture memory usage.

However, several pitfalls exist during Texture Streaming usage:

**Point 1: Enable correctly**

Texture Streaming must be enabled in **Project Settings – Quality**. However, tests show that enabling it in the Editor sometimes fails on real devices, completely disabling Streaming Mipmap for all textures. To ensure availability, you must **enable it globally via API in code**.

**Point 2: Correct parameters**

Proper values are essential for effective memory saving. Key parameters include **Memory Budget** and **Max Level Reduction**:

- **Memory Budget**: The total memory budget for all textures, default 512 MB. For typical mobile tiers, around 200 MB is sufficient for mid-to-low-end devices.

  It covers both streaming and non-streaming textures, and acts only as a reference

   for Unity to decide which Mipmap levels to load — not a hard upper memory limit.

- **Max Level Reduction**: Defines the **minimum Mipmap levels retained**. For example, a value of 2 means the engine can remove at most Mipmap 0 and 1 even if over budget.

  It also determines the initial loading level: when set to 2, the texture loads without levels 0 and 1 initially; higher levels are only loaded if budget allows.

The logic flow is:

1. Load all non-streaming textures normally.
2. Load streaming textures with high-resolution levels stripped according to Max Level Reduction.
3. Compare total texture memory with Memory Budget.
4. If under budget, gradually load higher Mipmap levels for streaming textures to improve visuals.

In short: if Memory Budget is set too high, or actual texture memory is far below budget, **Texture Streaming will have almost no effect**, and all streaming textures will retain full Mipmap chains.

**Point 3: Valid texture conditions**

Texture Streaming only affects textures that satisfy **all** of the following:

① **Generate Mipmap** and **Streaming Mipmap** are both enabled;

② Textures loaded dynamically (via AssetBundle or Resources.Load());

③ Textures **without Read/Write Enabled** (CPU-copy memory is not controlled by Texture Streaming).

**Point 4: Runtime CPU overhead**

After enabling Texture Streaming, the function **TextureStreamingManager.Update** runs continuously, causing persistent CPU overhead. The engine constantly calculates which Mipmap levels can be loaded.

On low-end devices, this cost can be significant when thousands of textures are used.

You must verify whether Texture Streaming works correctly and whether the memory saved justifies the extra CPU cost.

## 2.3.5 Read/Write Enabled

As mentioned earlier, texture memory is counted in **GFX memory (GPU memory)**. Textures with **Read/Write Enabled** keep an extra copy in **CPU memory**, effectively **doubling memory usage**.

Textures with Read/Write Enabled are clearly displayed in both the GOT Online Resource Mode list and local resource detection reports.

In practice, **textures that do not require runtime modification do not need Read/Write Enabled**. Developers should check and disable unnecessary instances to reduce memory overhead.

![25](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.3Texture/25.png)



## 2.3.6 Atlas Creation

Improper atlas creation is another common issue.

Atlas textures with high peak count may appear in the resource list, often not due to redundancy.

One typical case: many small sprites are packed into one atlas, exceeding its maximum resolution (e.g., 2048×2048). The engine then creates **additional atlas pages**.

If any small sprite in **any page** is used, **all pages of that atlas are loaded into memory**, causing severe waste. It is generally recommended to limit atlas pages to **2–3**.

![26](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.3Texture/26.png)



Even without multiple pages, many projects suffer from the **“use one, load all”** problem: only one or two sprites are needed, but the entire large atlas is loaded.

To solve this:

- **Group and package sprites strictly by usage scene and frequency**.
- Use appropriate atlas resolutions to avoid partially empty atlases, which also cause memory waste.



## 2.3.7 Use of TextMeshPro

TextMeshPro provides better visual quality and convenient functions for UI components, making it popular among many developers. However, the TMP font atlas textures generated by using TMP (textures with “SDF Atlas” in the name and in Alpha8 format) have several pitfalls worth noting.

(1) Sometimes, combined with the font resource list, it can be observed that the .ttf font files corresponding to the TMP atlas textures exist in memory. This indicates that the TMP font atlas is a **dynamic font**.

After project development is completed and all characters used in the game are confirmed to be added to the dynamic font atlas, you may consider converting the dynamic TMP to **static TMP** and removing the dependency on the .ttf file. In this way, the corresponding font resources will no longer appear in memory.

However, this method is **not recommended** if the font is still used for user input.

(2) The resolution of the atlas font texture is too large. In this case, it is recommended to check in the engine whether characters fully fill the atlas texture and whether the texture generation is reasonable.

For dynamic TMP, if the texture is not fully occupied (e.g., less than 3/4), you may enable the **Multi Atlas Textures** option and set an appropriate texture size.

For example, one 4096×4096 texture can be converted into three 2048×2048 textures, saving **32 MB – 3×8 MB = 8 MB** of memory.

![27](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/2.3Texture/27.png)

(3) TMP-related resources (such as LiberationSans SDF Atlas, EmojiOne) appear in the resource list. These are default resources of TextMeshPro.

You can remove dependencies on these default resources in **Project Settings – TextMesh Pro Settings**, so they will no longer be loaded into memory.

Since **Multi Atlas Textures** is an option exclusive to dynamic TMP, methods (1) and (2) **cannot be used simultaneously**. You may choose appropriately based on actual project conditions.