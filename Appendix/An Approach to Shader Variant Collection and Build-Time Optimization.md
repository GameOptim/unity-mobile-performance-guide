**An Approach to Shader Variant Collection and Build-Time Optimization**

------

### 1. What is a Shader Variant

From the official docs:

> In Unity, many shaders internally have multiple "variants", to account for different light modes, lightmaps, shadows and so on. These variants are indentified by a shader pass type, and a set of shader keywords.

A Unity shader asset is more than just GPU code. It also includes render states, property definitions, and multiple code paths targeting different stages of the rendering pipeline. Each code path can be compiled with different parameters to support specific rendering features.

In shader code with multiple variants, the most obvious signals are preprocessor directives like:

```
#pragma multi_compile_fwdbase          // Built-in forward pipeline keywords (lighting, shadows, etc.)
#pragma shader_feature _USE_FEATURE_A  // Custom feature toggle
#pragma multi_compile _USE_FUNCTION_A _USE_FUNCTION_B // Custom multi-compile options
```

These switches let you keep a relatively small shader codebase while generating multiple functional variants from the same “skeleton.” As features grow, the number of variants increases exponentially. Controlling variant explosion requires experience and discipline.

------

### 2. Why Collect Shader Variants

At game startup, shaders are typically preloaded to avoid runtime compilation hitches. Unity provides `Shader.WarmupAllShaders` to compile all loaded shaders and their variants.

As rendering complexity increases, the number of variants grows significantly. Blindly warming up everything leads to long startup times and poor UX.

Unity later introduced `ShaderVariantCollection` to replace this brute-force approach and enable on-demand preloading.

Key point from the docs:

> This is used for shader preloading ("warmup"), so that a game can make sure "actually required" shader variants are loaded at startup (or level load time), to avoid shader compilation related hiccups later on in the game.

In short, a variant collection represents the subset of variants actually used by the game. Loading only those can significantly improve load times.

------

### 3. Additional Notes

From the official documentation, we know that variant collections are used for preloading Shaders, but it does not mention compilation during the packaging and publishing process, nor how to filter the actually used compiled variants into AssetBundles.

In the Shader resources of the actual released game package, if a required variant is missing, the rendered result may be incorrect (When Unity loads multiple variants, it likely uses some matching method. If it cannot find the best match, it will fallback to another variant with the most matching keywords, resulting in partially different or even completely wrong rendering results). The most frustrating part is that on actual running devices, you cannot see precise information about missing variants. Once an error occurs, you may have to start over from scratch.

Shader variant loss is usually caused by the packaging of AssetBundles. Unity internally collects the actually used variants by scanning the materials that use Shaders, as well as the lighting parameters on scene renderers (it should record the Shader variants actually used in the internal rendering pipeline). To achieve AssetBundle updates, we usually place Shaders as separate resources in an independent AssetBundle, while other materials and scenes that reference these Shaders are loaded as dependencies of the AssetBundle. Once Shaders are separated from the objects that use them, Unity cannot comprehensively consider which variants need to be actually published during packaging, resulting in the annoying random occurrence of variant loss.

Some online solutions for resolving variant loss:
(1) Package the shaders together with the materials that use them into a single AssetBundle;
(2) Run through the entire project scene in the Editor. Unity will record all collected shaders and their variants, then save this information as a variant collection and package it together with the shaders.

Unity has hidden this most important feature at the bottom of the Project Settings panel:

![1](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Shader_Variant/1.png)

These approaches mostly work, but:

- **Method 1**: Materials alone don’t capture full variant usage—lighting and scene setup also matter
- **Method 2**: Manual coverage is never complete, and Unity outputs a monolithic variant collection, which complicates bundle granularity

------

### 4. My Approach

#### 4.1 Classifying Rendering Assets

In practice, rendering assets fall into three categories:

1. Scenes
2. Dynamically loaded assets (models, characters, VFX)
3. UI & UI effects

UI typically uses built-in UGUI shaders with `multi_compile`. These variants are always compiled regardless of usage, and UI shader count is small, so we can ignore them.

So we only need to handle:

- Scene static rendering
- Dynamically loaded assets

------

#### 4.2 Automated Shader Variant Collector

Steps:

1. Collect all asset paths to be included in the build (based on config: localization, distribution, etc.)
2. Resolve dependencies and gather all materials used by dynamic assets (e.g., prefabs)
3. Create an empty scene with a basic lighting setup (e.g., a real-time directional light)
4. Use reflection to call `ShaderUtil.ClearCurrentShaderVariantCollection`
5. Create a rendering camera
6. Spawn a grid of spheres and ensure they’re visible to the camera
7. Assign materials to spheres in batches and render one frame
8. Load each scene, set a proper camera view, and render
9. Save collected variants via `ShaderUtil.SaveCurrentShaderVariantCollection`
10. Done



#### 4.3 Is the Variant Collection Enough?

No — the job is only half done.

Some custom shaders, especially those referenced only via `UsePass`, do not appear in any material assets. As a result, Unity cannot collect their variants. I’m confident about this — in our production projects, we use multi-pass shaders where each pass is provided by internal shaders referenced via `UsePass`. These internal shaders are not exposed to artists, and the shaders artists use contain no actual code.

The advantage of this approach is that it allows greater flexibility in composing multi-pass shader combinations without increasing code complexity.

For example, consider three shaders:

```id="mc38m1"
ABC.shader
InternalA.shader
InternalB.shader
```

`ABC` references passes from `InternalA` and `InternalB`, but `ABC` itself contains no actual code.

In this case, Unity will collect variants only under `ABC`, without distinguishing between `InternalA` and `InternalB`. If you directly use Unity’s exported variant collection, it is very likely to result in missing variants.

Therefore, we need to split Unity’s exported variant collection into smaller, per-shader resources. This allows us both to reconstruct variant collections for dependent shaders and to improve granularity for AssetBundle packaging.



#### 4. Continue

Before proceeding, some preparation is required:

1. Use reflection to call `ShaderUtil.OpenShaderCombinations(shader, usedBySceneOnly = true)`
   This opens a Unity-generated `Library/ParsedCombinations-xxx.shader` file. By parsing it as text, we can extract all valid `builtin`, `shader_feature`, and `multi_compile` keywords, along with snippet markers.
2. Use reflection to read each shader’s variant set from `ShaderVariantCollection`.
3. Implement caching to avoid repeatedly retrieving this data later.

To make the following logic clearer, here is the pseudocode:

```
// Start by splitting the full collection into independent variant sets per shader
ShaderVariantCollection unityVAC; // Unity-exported full collection

foreach ( curSVC in unityVAC ) {

    // Current sub-collection shader
    var cur_shader = curSVC.shader;
    // All variants of the current shader
    var cur_shaderVariants = curSVC.variants;

    // Create a new independent variant collection for this shader
    var va = new ShaderVariantCollection();

    // Try to copy all variants into the new collection
    foreach ( cur_v in cur_shaderVariants ) {
        try {
            var realSV = new ShaderVariantCollection.ShaderVariant( cur_v.shader, cur_v.passType, cur_v.keywords );
            va.Add( realSV );
        } catch ( ... ) {
            // This variant does not belong to the specified pass type of the current shader
            // Usually happens because the collected variant actually belongs to a dependency
        }
    }
    Save( va );

    // Resolve dependencies introduced via UsePass and Fallback
    // Create or update variant collections for dependent shaders
    var child_shaders = GetDependencies( GetAssetPath( cur_shader ) );
    foreach ( child_shader in child_shaders ) {
        // A dependent shader may be referenced multiple times by different shaders — caching is required
        var child_va = TryGet_New_ShaderVariantCollection( child_shader );
        // Feed variants one by one to test whether they also belong to the dependent shader
        foreach ( cur_v in cur_shaderVariants ) {

            var _keywords = copy( cur_v.keywords );

            // Some keywords in this variant may not belong to the dependent shader
            // Remove them using the previously parsed ParsedCombinations data
            RemoveInvalidKeyword( _keywords, child_shader );

            try {
                var realSV = new ShaderVariantCollection.ShaderVariant( child_shader, cur_v.passType, _keywords );
                // Deduplicate before adding
                if ( !child_va.Contains( realSV ) ) {
                    child_va.Add( realSV );
                }
            } catch ( ... ) {
                // ...
            }
        }
    }

    // Save all reconstructed variant collections for dependent shaders
    // ...

    // Known limitations:
    // 1. Variant pass ownership cannot be fully determined
    //    (no passName or passIndex available)
    //    Therefore, any valid variant is assumed to be usable
    // 2. A shader can be used directly by materials (collected by Unity),
    //    or referenced by other shaders.
    //    Whether referenced variants are fully collected by Unity
    //    still requires further verification
}
```

After going through this process, we aim to build as complete a record of shader variant usage as possible, and then move on to the next stage.



#### 5. Compile-Time Optimization

After building the variant collection, I observed that Unity still spends a long time compiling shaders during the build. Even accounting for estimated `multi_compile` combinations, the compiled variant count far exceeds what is declared in the collection.

From the official documentation, I infer that `ShaderVariantCollection` is only used for preloading and defining a subset of usable variants. Compilation itself is handled in a separate resource processing stage, and must be filtered manually.

Unity 2018.2 introduced a programmable shader variant stripping pipeline:
`IPreprocessShaders.OnProcessShader`.

With this interface, we receive callbacks during shader compilation and can implement custom logic to remove unnecessary variants, reducing compile time.

Unity instantiates all classes implementing `IPreprocessShaders` and executes their callbacks during compilation. We need to filter the incoming data within these callbacks.

Example:

```csharp
/// A simple processor that strips built-in Unity variants
class BuiltinShaderPreprocessor : IPreprocessShaders {
    static ShaderKeyword[] s_uselessKeywords;
    public int callbackOrder {
        get { return 0; } // Defines execution order among multiple processors
    }
    static BuiltinShaderPreprocessor() {
        s_uselessKeywords = new ShaderKeyword[] {
            new ShaderKeyword( "DIRLIGHTMAP_COMBINED" ),
            new ShaderKeyword( "LIGHTMAP_SHADOW_MIXING" ),
            new ShaderKeyword( "SHADOWS_SCREEN" ),
        };
    }
    public void OnProcessShader( Shader shader, ShaderSnippetData snippet, IList<ShaderCompilerData> data ) {
        for ( int i = data.Count - 1; i >= 0; --i ) {
            for ( int j = 0; j < s_uselessKeywords.Length; ++j ) {
                if ( data[ i ].shaderKeywordSet.IsEnabled( s_uselessKeywords[ j ] ) ) {
                    data.RemoveAt( i );
                    break;
                }
            }
        }
    }
}
```

We need to implement a more refined and accurate stripping logic (code below is incomplete):

```csharp
class ShaderPreprocessor : IPreprocessShaders {
    public void OnProcessShader( Shader shader, ShaderSnippetData snippet, IList<ShaderCompilerData> data ) {
        // Skip system shaders
        // return;

        // Load the variant collection corresponding to this shader:
        // In the previous step, we created a dedicated collection per shader
        // Retrieve compilation data for this shader

        var comb = ShaderUtils.ParseShaderCombinations( shader, true );

        // Skip shaders that only reference others via UsePass and contain no actual code
        // return;

        // Iterate in reverse for safe removal
        for ( int i = data.Count - 1; i >= 0; --i ) {

            // Keyword list for the current compilation unit
            var _keywords = data[ i ].shaderKeywordSet.GetShaderKeywords();

            // Only process variants that have keywords to reduce complexity
            // In practice, keyword-less variants may also be unnecessary,
            // but skipping them here does not significantly increase compile cost
            if ( _keywords.Length > 0 ) {

                var keywordList = new HashSet<String>();

                for ( int j = 0; j < _keywords.Length; ++j ) {
                    var name = _keywords[ j ].GetKeywordName();
                    fullKeywords.Add( name );

                    if ( snippetCombinations.multi_compiles != null ) {
                        if ( Array.IndexOf( snippetCombinations.multi_compiles, name ) < 0 ) {
                            // Exclude multi_compile keywords — these are required and must not be stripped
                            // Only keep non-multi_compile keywords for matching
                            keywordList.Add( name );
                        }
                    }
                }

                if ( keywordList.Count > 0 ) {
                    // This variant may be eligible for stripping
                    // Next step:
                    // Check whether this keyword combination exists in the pre-collected variant set

                    // When comparing, remove multi_compile keywords
                    // Perform unordered matching:
                    // If fully matched → keep
                    // Otherwise → strip
                    // ...

                    var matched = false;

                    // Iterate over all collected variants
                    for ( int n = 0; n < rawVariants.Count; ++n ) {

                        var variant = rawVariants[ n ];
                        var matchCount = -1;
                        var mismatchCount = 0;
                        var skipCount = 0;

                        if ( variant.shader == shader && variant.passType == snippet.passType ) {
                            matchCount = 0;

                            // Important note:
                            // Matching must exclude multi_compile keywords
                            // snippetCombinations is parsed from ParsedCombinations-XXX.shader
                            // Directly using ShaderUtil.GetShaderVariantEntries may cause OOM due to full variant explosion

                            for ( var m = 0; m < variant.keywords.Length; ++m ) {

                                var keyword = variant.keywords[ m ];

                                if ( Array.IndexOf( snippetCombinations.multi_compiles, keyword ) < 0 ) {
                                    if ( keywordList.Contains( keyword ) ) {
                                        ++matchCount;
                                    } else {
                                        ++mismatchCount;
                                        break;
                                    }
                                } else {
                                    ++skipCount;
                                }
                            }
                        }

                        if ( matchCount >= 0 && mismatchCount == 0 && matchCount + skipCount == keywordList.Count ) {
                            matched = true;
                            break;
                        }
                    }

                    if ( !matched ) {
                        data.RemoveAt( i );
                    }
                }
            }
        }
    }
}
```

### 

#### 6. Conclusion

After applying all these steps, both shader variant collection and compilation time are significantly optimized.

However, the entire workflow relies heavily on less-documented Unity Editor APIs. Some intermediate data is incomplete, which may introduce subtle inconsistencies.

This approach works in practice, but it still requires further validation and iteration.