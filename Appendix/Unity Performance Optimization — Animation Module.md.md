From performance reports, the main animation-related functions you’ll typically see are:

- `DirectorUpdateAnimationBegin`

- `DirectorUpdateAnimationEnd`

![12](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Animation_Module/12.png)

These two are key entry points. You should always inspect their call stacks—look at call counts and time distribution to narrow down bottlenecks.

Below are common performance issues and optimization strategies we’ve encountered:

------

### 1. Control the Number of Active Animators

As character count increases, CPU cost scales almost linearly across animation-related functions.

![13](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Animation_Module/13.png)

In one real-device profiling case, we inspected a single frame and found `ApplyOnAnimatorMove` was called 168 times, meaning there were 168 Animators in the Update state. That’s far too high—generally, you should keep this under 30 if possible.

**Common causes:**

- **Off-screen Animators still updating**

- **Cached character Animators left active**

- **Excessive UI Animators**

**For UI animations, if they’re simple, consider replacing them with DOTween instead of using Animator.**

------

### 2. Check Animators with Apply Root Motion Enabled

![14](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Animation_Module/14.png)

From the `Animators.Update` stack, we observed `Animator.ApplyBuiltinRootMotion` taking up to 28% of the time.

This is usually caused by Animators with Apply Root Motion enabled.

If the animation doesn’t actually require movement, disable this option.

------

### 3. Enable Optimize Game Objects

![15](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Animation_Module/15.png)

When enabled, Unity removes Transform hierarchy data during animation processing.

This can significantly reduce the cost of `Animators.Update`, freeing up main thread time for more critical gameplay logic.

------

### 4. Reduce `Animator.Initialize` Frequency

Animator.Initialize is triggered when a GameObject containing an Animator component is activated or instantiated, and it has a high performance cost. Therefore, it is not recommended to frequently Deactivate/Activate GameObjects with Animator components during combat, as shown in the figure below.

![16](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Animation_Module/16.png)

Recommendation:

- Use object pooling for frequently spawned characters

- Instead of deactivating the GameObject:

- Disable the Animator component

- Move the object off-screen

This helps reduce unnecessary `Animator.Initialize` calls.

------

### 5. Too Many Animators Using Always Animate

![19](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Animation_Module/19.png)

With Always Animate, Animators keep updating even when off-screen.

Recommended alternatives:

- `CullUpdateTransforms` → for animations that affect position

- `CullCompletely` → for animations without movement

------

### 6. `Animators.FireAnimationEventsAndBehaviours`

This reflects the cost of animation events, which mainly comes from your gameplay logic.

If this is high:

- Audit animation events

- Optimize the logic triggered by those events

![18](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Animation_Module/18.png)

------

### 7. Use GPU Skinning + GPU Instancing for Crowd Rendering

For large numbers of similar characters:

- Disable Unity’s built-in GPU Skinning (it can introduce extra overhead and thread stalls)

- Use optimized GPU Skinning + GPU Instancing solutions (e.g., open-source implementations)

Benefits:

- Reduced `Animators.Update` cost

- Better batching efficiency

------

## Additional Checks

You can quickly scan your performance report:

- Press `Ctrl + F`

- Search for "animation"

If the animation module isn’t healthy, problematic checks will show up clearly.

![20](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Animation_Module/20.png)

------

## Asset Validation Rules (Recommended)

In our local asset validation service, we use the following rules to catch animation issues:

- Animation clips with `Compression != Optimal`

- Overly high precision animation data

- Animation clips containing Scale curves

- Animator Controllers with too many AnimationStates

![11](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Animation_Module/11.png)

------

## Final Notes

These are some of the most common issues we’ve seen when optimizing animation performance. The exact approach depends on your project.

Our tool GOT Online (GameOptim) provides a wide range of automated checks and diagnostics, which can significantly speed up this process.

Hope this helps streamline your animation optimization workflow.