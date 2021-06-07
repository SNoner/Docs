#### [1、ZWrite和ZTest](#ZTest)

#### [2、Blend](#Blend)

#### [3、Cull指令](#Cull)

#### [4、LOD](#LOD)

#### [5、雾效](#Fog)

#### [6、Uniform、Attribute、Varying](#Uniform)

#### [7、Time](#Time)

#### [8、噪声贴图](#Noise)

#### [9、Stencil Test](#Stencil)

#### [10、Rendering Path](#RenderingPath)





> Shader内置的API：
> [API参考1](http://www.cppblog.com/lai3d/archive/2008/10/23/64889.html)
> [API参考2](https://www.cnblogs.com/mawanli/p/7979771.html)

> 小知识：
>
> 1、tex2D(_MainTex,i.uv)   如果\_MainTex没有赋值，则获取结果为(1,1,1,1)





#### <a id="ZTest">1、ZWrite和ZTest</a>

```
1、深度：
	该像素点在3D世界中距离摄像机的距离，离摄像机越远，则深度值（Z值）越大。
2、深度缓存：
	深度缓存中存储着准备要绘制在屏幕上的像素点的深度值。如果启用了深度缓存区，在绘制每个像素前，OpenGL会把该像素的深度值和深度缓存的深度值进行比较。如果新像素深度值<缓存深度值，则新像素值会取代原先的深度值；反之，新像素值被遮挡，其颜色值和深度被丢弃。
3、ZWrite和ZTest：
ZWrite值：On/Off，默认为On，代码是否要将像素的深度写入深度缓存中（同时还要看ZTest是否通过）
ZTest值：Greater/GEqual/Less/LEqual/Equal/NotEqual/Always/Never/Off，默认为LEqual，代表通过比较深度来更改颜色缓存的值。例如当取默认值的情况下，如果将要绘制的新像素的Z值小于等于深度缓存中的值，则将用新像素的颜色值更新深度缓存中对应像素的颜色值。需要注意的是，当ZTest取值为Off时，表示的是关闭深度测试，等价于取值为Always，而不是Never！Always指的是直接将当前像素颜色（不是深度）写进颜色缓冲区中；而Never指的是不要将当前像素颜色写进颜色缓冲区中，相当于消失。

当ZWrite为On时，ZTest通过时，该像素深度写入深度缓存；ZTest通过，该像素颜色值会写入颜色缓存。
当ZWrite为On时，ZTest不通过时，该像素深度不能写入深度缓存；ZTest不通过，该像素颜色值不会写入颜色缓存。
当ZWrite为Off时，ZTest通过时，该像素深度不能写入深度缓存；ZTest通过，该像素颜色值写入颜色缓存。
当ZWrite为Off时，ZTest不通过时，该像素深度不能写入深度缓存；ZTest不通过，该像素颜色值不写入颜色缓存。
```

#### <a id="Blend">2、Blend</a>

```
混合因子：
One                      1,表示源或目标颜色完全通过
Zero                     0,表示删除源值或目标值
SrcColor                 要渲染的颜色
SrcAlpha                 要渲染的透明度
DstColor				 已经在屏幕上显示的颜色
DstAlpha				 已经在屏幕上显示的颜色透明度
OneMinusSrcColor         1-要渲染的颜色
OneMinusSrcAlpha         1-要渲染的透明度
OneMinusDstColor         1-屏幕上显示的颜色
OneMinusDstAlpha         1-屏幕上显示的透明度
```

```
常见混合类型：
Blend SrcAlpha OneMinusSrcAlpha  	//Alpha blending
Blend One One   				    //Additive
Blend OneMinusDstColor One          //Soft Additive
Blend DstColor Zero                 //Multiplicative
Blend DstColor SrcColor             //2x Multiplicative
```

```
Blend计算方式：
假设：
源颜色为：(Rs,Gs,Bs,As)
目标颜色为：(Rd,Gd,Bd,Ad)
源因子为：(Sr,Sg,Sb,Sa)
目标因为为：(Dr,Dg,Db,Da)
则混合产生的新颜色可以表示为：
源颜色*源因子+目标颜色*目标因子
(Rs*Sr+Rd*Dr,Gs*Sg+Gd*Dg,Bs*Sb+Bd*Db,As*Sa+Ad*Da)
如果颜色的某一分量超过了1.0，则会被自动截取为1.0，不需要考虑越界问题。
```

#### <a id="Cull">3、Cull</a>

```
Cull Back(默认值)		--剔除反面，渲染正面
Cull Front            --剔除正面，渲染反面
Cull Off              --正反面都渲染
```

```
1、正常情况下可以Cull Back以节省性能，但在游戏中需要根据不同情况使用此指令。
```

#### <a id="LOD">4、LOD</a>

```
这种编译方式最早是为了适应不同的显卡而准备的。目前用来适配高低配置不同的手机。
使用Shader LOD，需要编写多个SubShader。如果只有一个SubShader，无论LOD值是多少，都会加载默认的这个SubShader。
PS：多个SubShader的编写顺序按照LOD的值由大到小，然后在C#代码中使用Shader.maximumLOD=value
来进行控制。
```

#### <a id= "Fog">5、雾效</a>

```
struct v2f
{
    float4 vertex : POSITION;
    float2 uv : TEXCOORD0;
    UNITY_FOG_COORDS(1)                  --float值，保存全局雾效浓度参数
};
v2f vert(appdata v)
{
    v2f o;
    o.vertex = UnityObjectToClipPos(v.vertex);
    o.uv = TRANSFORM_TEX(v.uv,_MainTex);
    UNITY_TRANSFER_FOG(o,o.vertex);      //根据顶点位置记录该点的雾效浓度
    return o;
}
fixed4 frag(v2f i):SV_Target
{
    fixed4 col = tex2D(_MainTex,i.uv);
    UNITY_APPLY_FOG(i.fogCoord,col);     //根据fogCoord记录的雾效浓度，将雾效颜色应用到col上
    return col;
}
```

#### <a id="Uniform">6、Uniform、Attribute、Varying</a>

> Uniform : 
>
> ​			在Vertex 和 Fragment两者之间声明方式完全一样，可以在vertex和fragment中共享使用，相当于一个被vertex和fragment Shader共享的全局变量。
>
> Attribute ：
>
> ​			只能在Vertex Shader中使用的变量。（不能在fragment中声明，也不能在fragment中使用）

#### <a id="Time">7、Time</a>

| _Time          | float4 | t/20 | t    | t*2      | t*3        |
| -------------- | ------ | ---- | ---- | -------- | ---------- |
| _SinTime       | float4 | t/8  | t/4  | t/2      | t          |
| _CosTime       | float4 | t/8  | t/4  | t/2      | t          |
| nity_DeltaTime | float4 | dt   | 1/dt | smoothDt | 1/smoothDt |

#### <a id="Noise">8、噪声贴图</a>

程序生成，代码如下：

```c#
using UnityEngine;
using UnityEditor;
using System;
using System.IO;

public class NoisePicTool : EditorWindow
{
    //生成噪声图的分辨率与噪声强度
    private int width = 300;
    private int height = 300;
    private int strength = 10;

    //随机参数，使每次产生不同的噪声图
    private float xrog = 1;
    private float yrog = 1;

    private Texture2D noiseTex;
    private RenderTexture rt;
    private string contents = "./Assets/Textures/Noise";

    [MenuItem("Tools/NoisePicture")]
    static void Init()
    {
        NoisePicTool window =
            (NoisePicTool)EditorWindow.GetWindow(typeof(NoisePicTool), true, "NoisePicture");
        window.Show();
    }
    void OnEnable()
    {
        xrog = UnityEngine.Random.Range(1f, 1000);
        yrog = UnityEngine.Random.Range(1f, 1000);
        CalcNoise();
        GetRenderTexture(noiseTex);
    }
    //参数面板数值发生变化时执行此函数
    void OnInspectorUpdate()
    {
        CalcNoise();
        GetRenderTexture(noiseTex);
    }
    void OnGUI()
    {
        GUILayout.BeginHorizontal();
        GUILayout.Label("Width(pixel):", GUILayout.MaxWidth(80));
        width = EditorGUILayout.IntField(width, GUILayout.MaxWidth(60));
        GUILayout.Label("Height(pixel):", GUILayout.MaxWidth(80));
        height = EditorGUILayout.IntField(height,GUILayout.MaxWidth(60));
        GUILayout.EndHorizontal();
        GUILayout.BeginHorizontal();
        GUILayout.Label("Strength:", GUILayout.MaxWidth(60));
        strength = EditorGUILayout.IntField(strength, GUILayout.MaxWidth(60));
        GUILayout.EndHorizontal();
        if(GUILayout.Button("Spawn"))
        {
            xrog = UnityEngine.Random.Range(1, 10);
            yrog = UnityEngine.Random.Range(1, 10);
            CalcNoise();
            GetRenderTexture(noiseTex);
        }
        if(GUILayout.Button("Save"))
        {
            SaveRenderTextureToPNG(rt, contents);
        }
        if (GUILayout.Button("Close"))
        {
            this.Close();
        }
        if(rt!=null)
        {
            GUI.DrawTexture(new Rect(0, 150, width, height), rt);
        }
    }
    //计算噪声图的像素点
    void CalcNoise()
    {
        // For each pixel in the texture...
        Color[] pixel = new Color[width * height];
        noiseTex = new Texture2D(width, height);
        float y = 0.0f;
        while (y < height)
        {
            float x = 0.0f;
            while (x < width)
            {
                float xCoord = xrog + x / width * strength;
                float yCoord = yrog + y / height * strength;
                float sample = Mathf.PerlinNoise(xCoord, yCoord);
                pixel[(int)y *width + (int)x] = new Color(sample, sample, sample);
                x++;
            }
            y++;
        }
        noiseTex.SetPixels(pixel);
        noiseTex.Apply();
    }
    //获取噪声图并生成RenderTexture
    public void GetRenderTexture(Texture2D inputTex)
    {
        RenderTexture temp = RenderTexture.GetTemporary(inputTex.width, inputTex.height, 0, RenderTextureFormat.ARGB32);
        Graphics.Blit(inputTex, temp);
        rt = temp;
    }
    //按照制定路径保存噪声图为png格式
    public bool SaveRenderTextureToPNG(RenderTexture rt, string contents)
    {
        RenderTexture prev = RenderTexture.active;
        RenderTexture.active = rt;
        Texture2D png = new Texture2D(rt.width, rt.height, TextureFormat.ARGB32, false);
        png.ReadPixels(new Rect(0, 0, rt.width, rt.height), 0, 0);
        byte[] bytes = png.EncodeToPNG();
        if (!Directory.Exists(contents))
        {
            Directory.CreateDirectory(contents);
        }
        string pngName = width + "x" + height + "-" + strength;
        FileStream file = File.Open(contents + "/" + pngName + ".png",FileMode.OpenOrCreate);
        BinaryWriter writer = new BinaryWriter(file);
        writer.Write(bytes);
        file.Close();
        Texture2D.DestroyImmediate(png);
        png = null;
        RenderTexture.active = prev;
        return true;
    }
}
```

#### <a id="Stencil">9、Stencil Test</a>

```
模板缓冲区为帧缓冲区中的每个像素存储一个8位整数值。在执行片段着色器之前对于给定的像素，GPU可以将模板缓冲区中的当前值与给定的参考值进行比较，这成为模板测试。如果模板测试通过，GPU将执行深度测试，如果模板测试失败，GPU将跳过该像素的其余处理。这意味着可以使用模板缓冲区作为掩码来告诉GPU给定像素是否需要绘制。
模板测试方程为：
（ref & readMask) comparisonFunction (stencilBufferValue % readMask)

模板测试语法：
Stencil
{
	Ref <ref>
	ReadMask <readMask>
	WriteMask <writeMask>
	Comp <comparisionOperation>
	Pass <passOperation>
	Fail <failOperation>
	ZFail <zFailOperation>
	CompBack <comparisonOperationBack>
	PassBack <passOperationBack>
	FailBack <failOperationBack>
	ZFailBack <zFailOperationBack>
	CompFront <comparisonOperationFront>
	PassFront <passOperationFront>
	FailFront <failOperationFront>
	ZFailFront <zFailOperationFront>
}

Comparison Operation Values:
Never				1
Less				2
Equal				3
LEqual				4
Greater				5
NotEqual			6
GEqual				7
Always(Default)		8

Stencil Operation Values:
Keep(Default)		0		--保持不变
Zero				1		--将0写入缓冲区
Replace				2		--将参考值写入缓冲区
IncrSat				3		--增加缓冲区当前值，如大于255，则为255
DecrSat				4		--减少缓冲区当前值，如小于0，则为0
Invert				5       --按位取反
IncrWrap			7		--增加缓冲区当前值，如大于255，则变为0
DecrWrap			8		--较少缓冲区当前值，如小于0，则变为255
```

#### <a id="RenderingPath">10、Rendering Path</a>

```
Unity内置渲染管线中的渲染路径：
Unity的内置渲染管线支持不同的渲染路径，一种渲染路径是一系列与光照和着色相关的操作。不同的渲染路径具有不同的能力和性能特征。决定哪种渲染路径最合适项目取决于项目的类型和目标硬件。
如果运行项目的设备GPU不支持选择的渲染路径，Unity会自动使用较低保真度的渲染路径。

内置四大渲染路径：
1、Forward
	默认渲染路径，前向渲染中渲染实时灯光非常消耗性能。如果项目中没有大量使用实时灯光，或者照明保真度要求不高，可以考虑此渲染路径。
	
2、Deferred
	光照和阴影保真度最高的渲染路径。
	延迟渲染需要GPU支持，并且有一些限制，它不支持半透明对象、正交投影或硬件抗锯齿。如果项目中有大量实时灯光并且需要高水平的照明保真度，并且目标硬件支持延迟渲染，可以考虑此渲染路径。
3、Legacy Deferred
	类似于延迟渲染，使用不同的技术。不支持Unity5基于物理的标准着色器。
4、Legacy Vertex Lit
	具有最低照明保真度且不支持实时阴影的渲染路径，是前向渲染路径的子集。
```

