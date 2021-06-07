#### [1、XRayEffect](#XRayEffect)

#### [2、Fog](#Fog)

#### [3、Mask](#Mask)

#### [4、ScreenGray](#ScreenGray)

#### [5、切块](#切块)

#### [6、Dissolve](#Dissolve)

​      轴向溶解效果



#### <a id="XRayEffect">1、XRayEffect</a>

```glsl
Shader "XRayEffect"
{
	Properties
	{
		_MainTex("MainTex",2D) = "white"{}
		_Color("MainColor",Color) = (1,1,1,1)
		_XRayColor("XRayColor",Color) = (1,1,1,1)
	}
	SubShader
	{
		Tags{"Queue" = "Transparent" "RenderType" = "Queue"}

		PASS
		{
			Blend SrcAlpha OneMinusSrcAlpha
			ZWrite On
			ZTest LEqual  //默认为此值，可不写
			CGPROGRAM
			#include "Lighting.cginc"
			#pragma vertex vert
			#pragma fragment frag

			sampler2D _MainTex;
			float4 _MainTex_ST;	
			uniform fixed4 _Color;

			struct v2f
			{
				float4 pos:SV_POSITION;
				float2 uv:TEXCOORD0;
			};

			v2f vert(appdata_base v)
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
				return o;
			}
			fixed4 frag(v2f i) :SV_Target
			{
				half4 col = tex2D(_MainTex,i.uv)*_Color;
				return col;
			}
				ENDCG
		}
		Pass
		{
			Blend One One
			ZWrite Off
			ZTest Greater

			CGPROGRAM
			#include "Lighting.cginc"
			#pragma vertex vert
			#pragma fragment frag

			sampler2D _MainTex;
			float4 _MainTex_ST;
			fixed4 _XRayColor;
			struct v2f
			{
				float4 pos:SV_POSITION;
				float rim : TEXCOORD0;
				float2 uv:TEXCOORD1;
			};
			v2f vert(appdata_base v)
			{
				v2f o;
				float4 world_pos = mul(unity_ObjectToWorld, v.vertex);
				o.pos = mul(UNITY_MATRIX_VP, world_pos);

				float3 viewDir = normalize(_WorldSpaceCameraPos.xyz - world_pos.xyz);
				float3 normal = UnityObjectToWorldNormal(v.normal);
                //利用法线和摄像机视线的点积结果再进行OneMinus,可实现颜色由中心到边缘的过渡
				o.rim = 1 - saturate(dot(normal, viewDir));

				o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
				return o;
			}
			fixed4 frag(v2f i):SV_Target
			{
				fixed4 col = tex2D(_MainTex,i.uv);
				return _XRayColor * i.rim * col;
			}
			ENDCG
		}
	}
	FallBack "Diffuse"
}
```



效果：

<img style="float:left" src="https://i.loli.net/2021/05/25/V4ArnTxyQB1GtgY.png">

>缺点：
>	1、在两个Pass中对纹理分别采样，消耗性能。
>
>变种：
>	1、注释掉第一个（正常渲染）的Pass，会裁剪掉没有被遮挡的部分，类似镜面反射。
>		  效果：
>
> <img style="margin-left:40px" src="https://i.loli.net/2021/05/25/oGWRjEbBhnZr4T7.png">

#### <a id="Fog">2、Fog</a>

```glsl
Shader "Custom/OpaqueAlpha"
{
    Properties
    {
        _MainTex("Texture",2D)="white"{}
    }
    SubShader
    {
        Tags{"RenderType"="Opaque"}
        Cull Off
        Lighting Off

        Pass
        {
            CGPROGRAM
            #include "UnityCG.cginc"
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_fog

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
                UNITY_FOG_COORDS(1)
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;

            v2f vert(appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv,_MainTex);
                UNITY_TRANSFER_FOG(o,o.vertex);
                return o;
            }

            fixed4 frag(v2f i):SV_Target
            {
                fixed4 col = tex2D(_MainTex,i.uv);
                UNITY_APPLY_FOG(i.fogCoord,col);
                return col;
            }
            ENDCG
        }
    }
}
```

效果：

<img style="margin-left:0px" src="https://i.loli.net/2021/05/27/1m9ASNqwXUaTv7r.jpg">

#### <a id="Mask">3、Mask</a>

```glsl
Shader "Custom/Unlit_Texture_Mask"
{
    Properties
    {
        _MainTex("Main Tex",2D) = "white"{}
        _MaskTex("Mask Tex",2D) = "white"{}
        _MinX("Min X",float) = -10
        _MaxX("Max X",float) = 10
        _MinY("Min Y",float) = -10
        _MaxY("Max Y",float) = -10
    }
    SubShader
    {
        Tags{"RenderType"="Opaque"}

        Pass
        {
            CGPROGRAM
            #include "UnityCG.cginc"
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_fog

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };
            struct v2f
            {
                float4 vertex : SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 vpos : TEXCOORD2;
                UNITY_FOG_COORDS(1)
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;
            sampler2D _MaskTex;
            float _MinX;
            float _MaxX;
            float _MinY;
            float _MaxY;

            v2f vert(appdata v)
            {
                v2f o;
                o.vpos = v.vertex.xyz;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv,_MainTex);
                UNITY_TRANSFER_FOG(o,o.vertex);
                return o;
            }

            fixed4 frag(v2f i):SV_Target
            {
                fixed4 col = tex2D(_MainTex,i.uv);
                UNITY_APPLY_FOG(i.fogCoord,col);

                //根据物体的定点坐标对Mask贴图的UV进行重映射
                fixed2 uv_mask = fixed2((i.vpos.x-_MinX)/(_MaxX-_MinX),(i.vpos.y-_MinY)/(_MaxY-_MinY));
                fixed4 col_mask = tex2D(_MaskTex,uv_mask);
                //把不在遮罩范围内的全部剔除掉
                col_mask.a *= (i.vpos.x >= _MinX);
                col_mask.a *= (i.vpos.x <= _MaxX);
                col_mask.a *= (i.vpos.y >= _MinY);
                col_mask.a *= (i.vpos.y <= _MaxY);
                //0.5这个值可根据具体情况进行调整
                clip(col_mask.a-0.5);
                return col;
            }
            ENDCG
        }
    }
}
```

效果：

<img title="贴图" style="margin-left:0px" src="https://i.loli.net/2021/05/27/fiM7V9opxGOaEwZ.jpg"><img title="效果图" src="https://i.loli.net/2021/05/27/93o6lhVWdEHOxCj.jpg">

```
变种：
	1、通过剔除不同的像素可实现不同的效果，例如：clip(0.5-col_mask.a)可剔除遮罩图覆盖的部位，实现镂空效果。
	2、可将o.vpos=v.vertex.xyz改为o.vpos=mul(unity_ObjectToWorld,v.vertex).xyz,可根据物体顶点的世界坐标进行遮罩效果。
```

镂空效果：

<img title="镂空效果" style="margin-left:0px" src="https://i.loli.net/2021/05/27/JwBhyUm4vYAsM35.jpg">

#### <a id="ScreenGray">4、ScreenGray</a>

```
1、通过点积将计算颜色的结果变为float类型，然后使RGB通道统一化。代码如下：
half gray = dot(mainColor.rgb,_Color.rgb);
return half4(gray,gray,gray,1);
```

效果：

置灰前：

<img title="置灰前" style="float:left" src="https://i.loli.net/2021/05/27/O3el19bDLM5q7wU.jpg">

置灰后：

<img title="置灰后" style="float:left" src="https://i.loli.net/2021/05/27/2RNEm7pGOUDZQhy.jpg">

#### <a id="切块">5、切块</a>

```
1、通过Dot获取角度，根据角度剔除相应的像素点。
代码：
	float3 _ParentDist;	 //控制切块在各个方向上的宽度
	float3 _ParentPos;	 //控制切块的位置
	float3 _PlaneDir; 	 //控制切块的方向
    _ParentPos = -_ParentDist;
    half3 viewDirection = normalize(v.posWorld.xyz - _ParentPos.xyz);
    half angle = dot(_PlaneDir, viewDirection);
    _ParentPos = +_ParentDist;
    viewDirection = normalize(v.posWorld.xyz - _ParentPos.xyz);
    half angle2 = dot(_PlaneDir, viewDirection);
    if (angle > 0 && angle2 < 0)
    {
    	clip(-1);
    }
```

原理示意图：

P点为像素点，同时在P1的右侧和P2的左侧，所以被剔除。

<img title="示意图" style="float:left" src="https://i.loli.net/2021/05/27/b49CAYJU8B6jufE.jpg">

效果：

<img title="切块效果图" style="margin-left:20px" src="https://i.loli.net/2021/05/27/jfKT1bdEu8na75H.jpg">

#### <a id="Dissolve">6、Dissolve</a>

```glsl
Shader "Custom/Disslove3"
{
	Properties
	{
		_MainTex("Main Tex",2D) = "white"{}
		_NoiseTex("Noise Tex",2D) = "white"{}
		_Threshold("Threshold",Range(0,1)) = 0.4
		_EdgeWidth("EdgeWidth",Range(0,1)) = 0.3
		_EdgeFirstColor("EdgeFirstColor",Color) = (1,1,1,1)
		_EdgeSecondColor("EdgeSecondColor",Color) = (1,1,1,1)
	}
	SubShader
	{
		Pass
		{
			CGPROGRAM
			#include "UnityCG.cginc"
			#pragma vertex vert
			#pragma fragment frag

			struct appdata
			{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};
			struct v2f
			{
				float4 vertex : SV_POSITION;
				float2 uv : TEXCOORD0;
			};

			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _NoiseTex;
			float4 _NoiseTex_ST;
			float _Threshold;
			float _EdgeWidth;
			float4 _EdgeFirstColor;
			float4 _EdgeSecondColor;

			v2f vert(appdata v)
			{
				v2f o;
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.uv = TRANSFORM_TEX(v.uv,_MainTex);
				return o;
			}
			fixed4 frag(v2f i):SV_Target
			{
                //按照噪声图的R通道进行溶解控制，正常情况下使用A通道，可灵活变通
				float r_noise = tex2D(_NoiseTex,i.uv).r;
				if(r_noise < _Threshold)
					discard;

				fixed3 albedo = tex2D(_MainTex,i.uv).rgb;
                //这里是根据噪声图R通道与指定阈值的差值进行溶解颜色的调节,在 r_noise<_EdgeWidth+_Threshold 情况下会使所有像素都会参与lerp进行颜色				//差值
                //举个例子：_EdgeWidth=0.5       _Threshold=0.6  
                //假设此时通道R值为 1 ，则 progress值为 0.2
                //针对以上情况，可根据具体需求进行效果调整
                //理想情况下溶解颜色的差值是根据像素点到溶解边缘的距离来进行差值的
				fixed progress = 1 - smoothstep(0.0, _EdgeWidth, r_noise - _Threshold);
				fixed3 burnColor = lerp(_EdgeFirstColor , _EdgeSecondColor, progress);
				burnColor = pow(burnColor, 5);
				//step(0.0001,_Threshold)防止Threshold小于0时消融残余问题
				fixed3 finalColor = lerp(albedo,burnColor, progress*step(0.0001,_Threshold));

				return fixed4(finalColor,1);
			}
			ENDCG
		}
	}
}
```

> 上述代码中阐述的情况下，比较难以实现溶解边缘的颜色过渡控制，例图如下：
>
> <img title="蜡烛火焰" style="margin-left:10px" src="https://i.loli.net/2021/05/27/LDSXs69R8dMjitc.jpg">

效果：

<img title="溶解效果图" style="margin-left:0px" src="https://i.loli.net/2021/05/27/h7k94vuXbjS3G1H.jpg">