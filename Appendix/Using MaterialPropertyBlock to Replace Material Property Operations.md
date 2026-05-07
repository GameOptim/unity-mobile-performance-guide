# Using MaterialPropertyBlock to Replace Material Property Operations

Author: Li Xing

In Unite 2017, optimization suggestions for using Material Property Blocks were mentioned. The author has conducted special research, testing, and verification on this topic, and shares the experimental conclusions and test project here, hoping to help everyone with development and optimization.

## 1. Official Documentation

In the foreign technology session of Unite 2017, Arturo Núñez stated in the session *Shader Performance and Optimization*:

**Use MaterialPropertyBlock**

Is faster to set properties using a MaterialPropertyBlock rather than material.SetFloat(); Material.SetColor();

First, I specifically looked up the official documentation for MaterialPropertyBlock, which states:

Material Property Blocks are used with the `Graphics.DrawMesh` and `Renderer.SetPropertyBlock` APIs. They can be used when you want to draw many objects with the same material but different properties. For example, you can change the color of each mesh drawn without altering the state of the renderer.

Let's look at the `Renderer` class. It contains two properties: `Material` and `SharedMaterial`; and two functions: `GetPropertyBlock` and `SetPropertyBlock`.

The properties are used to access and modify materials, while the functions are used to set and get Material Property Blocks.

We know that when operating on shared material properties, we can use the `SharedMaterial` property. Changing this property will affect all objects using this material. When we need to modify a single material instance, we use the `Material` property. The first time `Material` is accessed, it creates a copy of the material, known as `Material (Instance)`.

## 2. Experiment

First, declare two arrays: one for handling materials, and another for handling Material Property Blocks.

```
GameObject[] listObj = null;
GameObject[] listProp = null;
```

Declare a public variable to control the array length, and a MaterialPropertyBlock.

```
public int objCount = 100;
MaterialPropertyBlock prop = null;
```

Then perform initialization in the `Start` function.

Generate `objCount` spheres on the left side of the screen to handle material operations, and `objCount` spheres on the right side to handle MaterialPropertyBlock operations.

```
void Start () {
    colorID = Shader.PropertyToID("_Color");
    prop = new MaterialPropertyBlock();
    var obj = Resources.Load("Perfabs/Sphere") as GameObject;
    listObj = new GameObject[objCount];
    listProp = new GameObject[objCount];
    
    for (int i = 0; i < objCount; ++i)
    {
        int x = Random.Range(-6,-2);
        int y = Random.Range(-4, 4);
        int z = Random.Range(-4, 4);
        GameObject o = Instantiate(obj);
        o.name = i.ToString();
        o.transform.localPosition = new Vector3(x,y,z);
        listObj[i] = o;
    }
    
    for (int i = 0; i < objCount; ++i)
    {
        int x = Random.Range(2, 6);
        int y = Random.Range(-4, 4);
        int z = Random.Range(-4, 4);
        GameObject o = Instantiate(obj);
        o.name = (objCount + i).ToString();
        o.transform.localPosition = new Vector3(x, y, z);
        listProp[i] = o;
    }
}
```

Then handle the operations in the `Update` function. Here, the Up and Down arrow keys are used for testing.

```
void Update () {
    if (Input.GetKeyDown(KeyCode.DownArrow))
    {
        Stopwatch sw = new Stopwatch();
        sw.Start();
        for (int i = 0; i < objCount; ++i)
        {
            float r = Random.Range(0, 1f);
            float g = Random.Range(0, 1f);
            float b = Random.Range(0, 1f);
            listObj[i].GetComponent<Renderer>().material.SetColor("_Color", new Color(r, g, b, 1));
        }
        sw.Stop();
        UnityEngine.Debug.Log(string.Format("material total: {0:F4} ms", (float)sw.ElapsedTicks *1000 / Stopwatch.Frequency));
    }
    
    if (Input.GetKeyDown(KeyCode.UpArrow))
    {
        Stopwatch sw = new Stopwatch();
        sw.Start();
        for (int i = 0; i < objCount; ++i)
        {
            float r = Random.Range(0, 1f);
            float g = Random.Range(0, 1f);
            float b = Random.Range(0, 1f);
            listProp[i].GetComponent<Renderer>().GetPropertyBlock(prop);
            prop.SetColor(colorID, new Color(r, g, b, 1));
            listProp[i].GetComponent<Renderer>().SetPropertyBlock(prop);
        }
        sw.Stop();
        UnityEngine.Debug.Log(string.Format("MaterialPropertyBlock total: {0:F4} ms", (float)sw.ElapsedTicks * 1000 / Stopwatch.Frequency));
    }
}
```

Now let's look at the comparison data:

![1](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Material/1.png)

From the results, using MaterialPropertyBlock is indeed faster than operating directly on Material, with a cost nearly **one-quarter** that of material operations.

Additionally, both operations take longer on the **first call** than on subsequent calls, especially for materials. This shows that the copy operation during the first material property modification is very costly.

Of course, the code above can be further optimized, as `GetComponent<Renderer>()` is called every frame. We can cache these references in the `Start` method.

```
Renderer[] listRender = null;
Renderer[] listRenderProp = null;

...
listRender[i] = o.GetComponent<Renderer>();
...
listRenderProp[i] = o.GetComponent<Renderer>();
...
```

Let's look at the optimized runtime comparison data:

![2](https://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/GameOptim/Material/2.png)



I also sampled using the **Memory** module in the Profiler with the **Detailed** view enabled. It can be seen that under `Scene Memory`, there are material copies (caused by material operations, but not by MaterialPropertyBlock operations).

This confirms that material operations create instances, while MaterialPropertyBlock operations do not.

## 3. In-Game Implementation

As the official documentation describes, the Unity Terrain Engine uses MaterialPropertyBlock to draw trees. All trees share the same material but have different colors, scales, and wind factors.

For large open worlds, we dynamically load maps, and we can combine this with **GPU Instancing** for further performance improvements.

GPU Instancing has two advantages:

1. Eliminates the overhead of GameObject instances themselves.
2. Reduces Draw Calls, as well as CPU overhead from dynamic batching and memory overhead from static batching.

Unfortunately, it only works on devices supporting **OpenGL ES 3.0 or higher**.

For games with custom character skin color systems, the advantages of MaterialPropertyBlock are obvious:

If you want 100 different players on screen, using material color operations creates **100 material instances**. Furthermore, material property operations are slower than MaterialPropertyBlock.

In performance optimization, every millisecond saved counts, and the cumulative effect is significant.

## 4. Related Projects

Download link for Arturo Núñez's Shader Performance and Optimization project:

https://github.com/ArturoNereu/ShaderProfilingAndOptimization

