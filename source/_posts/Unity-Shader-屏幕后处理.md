---
title: '[Unity Shader] 屏幕后处理'
date: 2020-07-06 23:22:07
categories:
    - Unity Shader
tags:
    - Unity Shader
    - ShaderLab
    - 后处理
mathjax: true
---

游戏中的**屏幕后处理(posting-processing)**实际上是通过**离屏渲染(off-screen Rendering)**技术来实现的。
在OpenGL中游戏场景渲染到一个指定的帧缓冲附加的纹理（即颜色缓冲）中，最后将纹理绑定到默认帧缓冲中再进行渲染。
我们可以在绑定默认帧缓冲时我们可以通过定义好的shader，逐像素地访问这个纹理，对其进行处理，这就是所谓的后处理

<!-- more -->

### 1 Unity 中的后处理

对于Unity的内置渲染管线，官方对后处理有这样一段描述。
> 	The Built-in Render Pipeline does not include a post-processing solution by default. To use post-processing 
effects with the Built-in Render Pipeline, download the Post-Processing Version 2 package. For information on using 
post-processing effects in the Built-in Render Pipeline, see the Post-Processing Version 2 documentation.
 
简单直译过来就是Unity的内置管线对于后处理没有默认的解决方案。介绍后处理和常见效果的实现也是本文的意图。

### 2 Unity API

```cs
MonoBehaviour::void OnRenderImage(RenderTexture src,RenderTexture dest);
```
Unity把当前渲染的结果存储在参数一 `src` 中，通过函数中的一系列操作后，再把目标渲染到参数二 `dest` 中，最后将 `dest` 渲染到屏幕上。在这个函数中我们需要调用**Graphics.Blit**来处理：
```cs
// 1. 如果dest为null，直接将结果渲染到屏幕上
Graphics.Blit(Texture src,RenderTexture dest); 
// 2. mat会对src进行后处理，pass为-1则是一次调用Shader的所有Pass，否则调用给定索引的Pass
Graphics.Blit(Texture src,RenderTexture dest,Material mat, int pass = -1);
Graphics.Blit(Texture src,RenderTexture dest, int pass = -1);
```

### 3 Unity实现后处理效果
在相机上添加一个用于屏幕后处理的脚本。在这个脚本里编写`OnRenderImage`并调用`Graphics.Blit`，用指定的Shader对纹理进行处理。

首先，我们定义一个父类来处理摄像机的shader。
```cs
public class PostProcessBase : MonoBehaviour
{
    protected Material CheckShaderAndCreateMaterial(Shader shader, Material material)
    {
        if (shader == null || !shader.isSupported)
        {
            return null;
        }
        if (material != null && material.shader == shader)
        {
            return material;
        }
        material = new Material(shader);
        material.hideFlags = HideFlags.DontSave;
        return material;
    }
}
```
我们在后处理都要通过父类的`CheckShaderAndCreateMaterial`方法来检查`material`类是否正确。下面是挂载在相机上处理后处理shader的样板代码

```cs
public class PostProcessScript : PostProcessBase {
	public Shader shader;

	private Material mat;

	public Material material
    {
        get
        {
            this.mat= CheckShaderAndCreateMaterial(shader, mat);
            return mat;
        }
    }

	void OnRenderImage(RenderTexture src, RenderTexture dest){
		if(material == null){
			// 如果材质为空直接返回
			Graphics.Blit(src,dest)
			return;
		}
		// todo 这里可以通过material和shader通信
		Graphics.Blit(src,dest,material);
	}
}

```

`shader`中要关闭深度写入和面剔除，由于是对纹理进行操作，没必要进行深度写入和面剔除。深度测试时让所有的片元通过。

```
ZTest Always Cull Off ZWrite Off
```

### 4 各种后处理效果

#### 4.1 亮度/饱和度/对比度

首先我们在shader中定义分别对**亮度、饱和度、对比度**做控制的变量

```glsl
sampler2D _MainTex; // 由material传入渲染好的纹理
half _Brightness;
half _Saturation;
half _Contrast;
```

##### 4.1.1 顶点着色器

```glsl
struct v2f {
	float4 pos : SV_POSTION;
	half2 uv : TEXCOORD0; // 片元着色器将逐像素访问纹理
};

v2f vert(appdata_img v) {
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.uv = v.texcoord;
	return o;
}
```

##### 4.1.2 片元着色器

```glsl
fixed4 frag(v2f i) : SV_TARGET{
	fixed4 renderTex = tex2D(_MainTex, i.uv);

	// apply brightness
	fixed3 finalColor = renderTex.rgb * _Brightness;

	// apply saturation
	fixed luminance = 0.2125 * renderTex.r + 0.7154 * renderTex.g + 0.0721 * renderTex.b;
	fixed3 luminanceColor = fixed3(luminance,luminance,luminance);
	// luminanceColor + _Saturation * (finalColor - luminanceColor)
	finalColor = lerp(luminanceColor, finalColor, _Saturation);

	// apply contrast
	fixed3 avgColor = fixed3(0.5,0.5,0.5);
	finalColor = lerp(avgColor, finalColor, _Contrast);

	return finalColor;
}
```

##### 4.1.3 为什么这样处理

- **亮度：**图像中的RGB各个值越大，表示越亮，反之则越暗
`texColor * _Brightness`像素的各个颜色同比例缩放，表示了亮度的增减

- **饱和度：**指色彩的鲜艳程度，饱和度取决于该色中**含色成分**和**消色成分**（灰度）的比例。即含色成分越高，饱和度越高；消色成分越高，饱和度越低。对于**纯的颜色都是高饱和度**，如鲜红，鲜绿等。但如果我们对其**混杂上白色，灰色或其他色调的颜色，是不饱和的颜色**，如粉红，黄褐等。完全不饱和根本没有色调，如灰色。
`luminance`表示像素的亮度，`0.2125 * renderTex.r + 0.7154 * renderTex.g + 0.0721 * renderTex.b`表示人眼对不同颜色的敏感度（亮度相同的**蓝光**和**绿光**，**蓝光**看起来会更暗）。当`_Saturation<1`时，整体的色调会偏灰，当`_Saturation>1`时，色调会比原来的更鲜艳
- **对比度：**表示一张图片的明暗差异，即对比度越高，亮的像素越亮，暗的像素越暗。
当`_Contrast>1`时，`_Contrast*finalColor<(_Contrast-1)*avgColor`的结果的像素会变得更暗。反之，会更亮
当`_Contrast<1`时，总体上会趋向于灰色


#### 4.2 边缘检测

**边缘检测**是利用边缘检测算子对图像进行卷积操作。

##### 4.2.1 什么是卷积

简单来说，首先取一个**n * n的卷积矩阵(Convolution Matrix)，也叫核**，
矩阵的中心为当前像素的权重，矩阵周围则是中心像素边上的像素。**卷积**就是将像素**乘**矩阵对应的**权重**进行**作和**。

![卷积](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/06/20200706231751.png)

##### 4.2.2 边缘算子

通常**边**的产生是由于像素到像素间的颜色、亮度有突变。这种相邻像素之间的差值称为**梯度**，因此边缘的梯度值会比较大。下面介绍三种算子：

**1. Sobel**

$$
G_{x} = \begin{bmatrix} 
-1 & -2 & -1 \\
0 & 0 & 0 \\
1 & 2 & 1
\end{bmatrix} 
G_{y}  = \begin{bmatrix} 
-1 & 0 & 1 \\
-2 & 0 & 2 \\
-1 & 0 & 1
\end{bmatrix}  
$$

**2. Prewitt算子**

$$
G_{x} = \begin{bmatrix} 
-1 & -1 & -1 \\
0 & 0 & 0 \\
1 & 1 & 1
\end{bmatrix} 
G_{y}  = \begin{bmatrix} 
-1 & 0 & 1 \\
-1 & 0 & 1 \\
-1 & 0 & 1
\end{bmatrix}  
$$

**3. Roberts算子**
 
$$
G_{x} = \begin{bmatrix} 
-1 & 0 \\
0 & 1 
\end{bmatrix} 
G_{y}  = \begin{bmatrix} 
0 & -1 \\
1 & 0 
\end{bmatrix}  
$$
它们包含了两个方向，分别检测水平和垂直上的边缘信息。对算子进行遍历，分别对像素的水平和垂直信息进行卷积。下列是卷积公式：

$$
G = \sqrt{G_{x}^2 + G_{y}^2}
$$

出于效率考虑，替换为绝对值公式：

$$
G = |{G_{x}}| + |G_{y}|
$$

得到梯度值就可以进行计算了，梯度值越大越有可能是边缘点
关于更详细的算子可以看[几种图像边缘检测算子](https://zhuanlan.zhihu.com/p/56728333)

##### 4.2.3 顶点着色器
```glsl

sampler2D _MainTex;
// 访问纹理的纹素大小
fixed4 _MainTex_TexelSize;

struct v2f{
	half2 uv : TEXCOORD0;
	half2 offset[9] : TEXCOORD1;
	float4 pos : SV_POSITION;
};

v2f vert (appdata_img v){
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.uv = v.texcoord;

	// 左上
	o.offset[0] = _MainTex_TexelSize.xy * half2(-1,-1);
    o.offset[1] = _MainTex_TexelSize.xy * half2(0,-1);
    o.offset[2] = _MainTex_TexelSize.xy * half2(1,-1);
    o.offset[3] = _MainTex_TexelSize.xy * half2(-1,0);
    o.offset[4] = _MainTex_TexelSize.xy * half2(0,0);
    o.offset[5] = _MainTex_TexelSize.xy * half2(1,0);
    o.offset[6] = _MainTex_TexelSize.xy * half2(-1,1);
    o.offset[7] = _MainTex_TexelSize.xy * half2(0,1);
    o.offset[8] = _MainTex_TexelSize.xy * half2(1,1);
	
	return o;
}
```

##### 4.2.4 片元着色器

```glsl
fixed _EdgeOnly;
fixed4 _EdgeColor;
fixed4 _BackgroundColor;

fixed luminance(fixed4 color){
	return 0.2125 * color.r + 0.7154 * color.g + 0.0721 * color.b;
}
half Sobel(v2f i){
	const half Gx[9] = {
		-1,-2,-1,
		0,0,0,
		1,2,1
	};
	const half Gy[9] = {
		-1,0,1,
        -2,0,2,
        -1,0,1
	}
	half texColor;
    half edgeX = 0;
    half edgeY = 0;
    // 卷积操作
	for(int it = 0; it < 9; ++it){
		texColor = luminance(tex2D(_MainTex, i.uv + i.offset[it]));
        edgeX += texColor * Gx[it];
        edgeY += texColor * Gy[it];
	}
	// edge 越小越有可能是边缘
	half edge = 1 - abs(edgeX) - abs(edgeY);
	return edge;
}

fixed4 frag (v2f i) : SV_Target
{
	// 1. 根据Sobel算子求出像素的梯度
	half edge = Sobel(i);
	
	// 2. edge越小，withEdgeColor值越趋向_EdgeColor
	fixed4 withEdgeColor = lerp(_EdgeColor, tex2D(_MainTex, i.uv), edge);
	// 3. 边缘趋向边缘颜色，不是边缘则趋近背景颜色
	fixed4 onlyEdgeColor = lerp(_EdgeColor, _BackgroundColor, edge);
	// 4. _EdgeOnly为0就是原图带有边缘，_EdgeOnly为1就是把原图替换成背景色只留了边缘
	return lerp(withEdgeColor, onlyEdgeColor, _EdgeOnly);
}
```

#### 4.3 模糊

后处理模糊可以通过**均值模糊**和**中值模糊**进行实现。**均值模糊**通过卷积实现想下面这样的核可以实现：
$$
kernel = \begin{bmatrix} 
1 & 2 & 1 \\
2 & 4 &  2\\
1 & 2 & 1
\end{bmatrix}  /16
$$
**中值模糊**将选择区域的像素进行排序，选择中值替换掉原来的颜色。

##### 4.3.1 高斯模糊

高斯模糊同样使用了卷积，它使用的核被称为**高斯核**，他由以下公式获得：
$$
G(x,y) = \frac{1}{2\pi \sigma^{2}}e^{-\frac{x^{2}+y^{2}}{2\sigma^{2}}}
$$
其中$\sigma$为标准差，通常取值为1，x 和 y 对应当前位置到核中心的整数距离
高斯方程表示了领域每个像素对中心像素的影响程度：距离越近，影响越大。本节，我们用以下的高斯核进行卷积：
$$
kernel = \begin{bmatrix} 
0.0030 & 0.0133 & 0.0219 & 0.0133 & 0.0030 \\
0.0133 & 0.0596 &  0.0983 & 0.0596 & 0.0133\\
0.0219 & 0.0983 & 0.1621 & 0.0983 & 0.0219 \\
0.0133 & 0.0596 &  0.0983 & 0.0596 & 0.0133\\
0.0030 & 0.0133 & 0.0219 & 0.0133 & 0.0030
\end{bmatrix}
$$
可以发现矩阵的值和距离有关，因此加上出于性能考虑，我们需要把这个二维的高斯核拆解成两个一维核先后对图像进行卷积。这样，采样次数从 `N * N * W * H`  变成了 `2 * N * W * H`，我们可以先后调用两个Pass，第一个Pass用竖直方向的高斯核对图像进行卷积，第二个Pass用水平方向的高斯核。

##### 4.3.2 脚本

对图像我们可以通过多次卷积迭代，获得更模糊的图像。`OnRenderImage`的代码如下：

```cs
// 迭代次数
[Range(0, 4)]
public int iterations = 3;

// 模糊范围
[Range(0.2f, 3.0f)]
public float blurSpread = 0.6f;

// 降采样
[Range(0, 8)]
public int downSample = 3;

void OnRenderImage(RenderTexture src, RenderTexture dest){
	if(material == null){
		Graphics.Blit(src, dest);
		return;
	}
	int rtW = src.width/downSample; 
	int rtH = strc.height/downSample;
	RenderTexture buffer0 = RenderTexture.GetTemporary(rtW, rtH, 0);
	// 根据纹理坐标周围的像素，计算出插值
	buffer0.filterMode = FilterMode.Bilinear;
	Graphics.Blit(src,buffer0);
	
	for(int i = 0;i<iterations;++i){
		material.SetFloat("_BlurSize",1.0f + i*blurSpread);

		RenderTexture buffer1 = RenderTexture.GetTemporary(rtW,rtH,0);
		
		// Vertical
		Graphics.Blit(buffer0,buffer1,0);

		RenderTexture.ReleaseTemporary(buffer0);
		buffer0 = buffer1;
		buffer1 = RenderTexture.GetTemporary(rtW,rtH,0);
		
		// Horizontal
		Graphics.Blit(buffer0,buffer1,1);
		RenderTexture.ReleaseTemporary(buffer0);
		buffer0 = buffer1;
	}
	Graphics.Blit(buffer0, dest);
	RenderTexture.ReleaseTemporary(buffer0);
}
```

##### 4.3.3 顶点着色器

```glsl
sampler2D _MainTex;
half4 _MainTex_TexelSize;
float _BlurSize;

struct v2f{
	float4 pos: SV_POSTION;
	half2 uv[5]: TEXCOORD0;
}

v2f vertBlurVertical(appdata_img v){
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	half2 uv = v.texcoord;
	
	o.uv[0] = uv;
	// 下一格
	o.uv[1] = uv + float2(0.0,_MainTex_TexelSize.y) * _BlurSize;
	// 上一格
	o.uv[2] = uv - float2(0.0,_MainTex_TexelSize.y) * _BlurSize;
	o.uv[3] = uv + float2(0.0,_MainTex_TexelSize.y * 2.0) * _BlurSize;
	o.uv[4] = uv - float2(0.0,_MainTex_TexelSize.y * 2.0) * _BlurSize;
	return o;
}

v2f vertBlurHorizontal(appdata_img v){
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	half2 uv = v.texcoord;
	
	o.uv[0] = uv;
	// 下一格
	o.uv[1] = uv + float2(_MainTex_TexelSize.x, 0.0) * _BlurSize;
	// 上一格
	o.uv[2] = uv - float2(_MainTex_TexelSize.x, 0.0) * _BlurSize;
	o.uv[3] = uv + float2(_MainTex_TexelSize.x * 2.0, 0.0) * _BlurSize;
	o.uv[4] = uv - float2(_MainTex_TexelSize.x * 2.0, 0.0) * _BlurSize;
	return o;
}
```

定义了两个顶点着色器，一个保存垂直方向的纹理坐标，另一个保存水平方向上的。

##### 4.3.4 片源着色器

```glsl
fixed4 fragBlur(v2f i) : SV_TARGET{
	float weight = {0.4026,0.2442,0.0545};
	
	float3 sum = tex2D(_MainTex,i.uv[0]) * weight[0];

	for(int i = 1;i<3;++i){
		sum += tex2D(_MainTex,i.uv[i*2-1]) * weight[i];
		sum += tex2D(_MainTex,i.uv[i*2]) * weight[i];
	}
	return fixed4(sum, 1.0);
}
```
片元着色器做卷积操作

##### 4.3.5 最后一步

```glsl
Pass {
	NAME "GAUSSIAN_BLUR_VERTICAL"
			
	CGPROGRAM
	
	#pragma vertex vertBlurVertical  
	#pragma fragment fragBlur
			  
	ENDCG  
}
		
Pass {  
	NAME "GAUSSIAN_BLUR_HORIZONTAL"
			
	CGPROGRAM  
			
	#pragma vertex vertBlurHorizontal  
	#pragma fragment fragBlur
			
	ENDCG
}
```

定义两个Pass，分别应用对应的着色器，设置`NAME`属性，便于后期Bloom引用Pass

#### 4.4 Bloom

Bloom特效可以模拟真实相机的图像效果，它让画面中较亮的区域**扩散**到周围区域中。
**Bloom基本原理**很简单，就是将屏幕纹理中的亮区提取到一个纹理上，对这个亮区的纹理进行高斯模糊，最后与原屏幕混合，就有Bloom的效果了。


##### 4.1.1 脚本

```cs
void OnRenderImage(RenderTexture src, RenderTexture dest)
{
	if (material == null)
	{
		Graphics.Blit(src, dest);
		return;
	}

	material.SetFloat("_LuminanceThreshold", luminanceThreshold);

	int rtW = src.width / downSample;
	int rtH = src.height / downSample;

	RenderTexture buffer0 = RenderTexture.GetTemporary(rtW, rtH, 0);
	buffer0.filterMode = FilterMode.Bilinear;

	Graphics.Blit(src, buffer0, material, 0);

	for (int i = 0; i < iterations; ++i)
	{
		material.SetFloat("_BlurSize", 1.0f + i * blurSpread);

		RenderTexture buffer1 = RenderTexture.GetTemporary(rtW, rtH, 0);

		// Render the vertical pass
		Graphics.Blit(buffer0, buffer1, material, 1);

		RenderTexture.ReleaseTemporary(buffer0);
		buffer0 = buffer1;
		buffer1 = RenderTexture.GetTemporary(rtW, rtH, 0);

		// Render the horizontal pass
		Graphics.Blit(buffer0, buffer1, material, 2);

		RenderTexture.ReleaseTemporary(buffer0);
		buffer0 = buffer1;
	}

	material.SetTexture("_Bloom", buffer0);
	Graphics.Blit(src, dest, material, 3);

	RenderTexture.ReleaseTemporary(buffer0);
}
```

上述代码在原有的高斯模糊的基础上，先设置了提取亮区的阈值，接着进行高斯模糊，最后再设置纹理，和原图进行叠加


##### 4.4.2 处理光区

```glsl
sampler2D _MainTex;
half4 _MainTex_TexelSize; // 调用高斯模糊的Pass需要
float _BlurSize; // 调用高斯模糊的Pass需要
float _LuminanceThreshold;

struct v2f{
	float4 pos:SV_POSITION;
	float2 uv: TEXCOORD0;
};

// 正常处理
vertExtractBright(appdata_img v){
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.uv = v.texcoord;
	return o;
}

fixed luminance(fixed4 color){
	return 0.2125 * color.r + 0.7154 * color.g + 0.0721 * color.b;
}

fixed4 fragExtractBright(v2f i): SV_TARGET{
	fixed4 c = tex2D(_MainTex, i.uv);
	fixed val = clamp(luminance(c) - _LuminanceThreshold,0,1);
	return c * val;
}
```

处理光区主要在片元着色器部分，对采样的像素进行光值计算，结果与阈值作差，取`[0,1]`之间，只要val不为0，即为亮区


##### 4.4.3 混合

```glsl
sampler2D _Bloom;

struct v2fBloom{
	float4 pos:SV_POSITION;
	half4 uv: TEXCOORD0; 
}；


v2fBloom vertBloom(appdata_img v){
	v2fBloom o;
	o.pos =  UnityObjectToClipPos(v.vertex);
	o.uv.xy = v.texcoord;
	o.uv.zw = v.texcoord;
	
	#if UNITY_UV_STARTS_AT_TOP
	if(_MainTex_TexelSize.y < 0.0)
		o.uv.w = 1.0 - o.uv.w;
	#endif
	
	return o;
}

fixed4 fragBloom(v2fBloom i) : SV_TARGET{
	return tex2D(_MainTex, i.uv.xy) + tex2D(_Bloom, i.uv.zw);
}

```

混合部分的顶点着色器做了平台差异化处理，片元着色器则进行简单的混合。


##### 4.4.4 最后一步

```glsl
ZTest Always Cull Off ZWrite Off

Pass {
	CGPROGRAM

	#pragma vertex vertExtractBright
	#pragma fragment fragExtractBright

	ENDCG
}

UsePass "Custom/GaussianBlur/GAUSSIAN_BLUR_VERTICAL"
UsePass "Custom/GaussianBlur/GAUSSIAN_BLUR_HORIZONTAL"

Pass {
	CGPROGRAM

	#pragma vertex vertBloom
	#pragma fragment fragBloom

	ENDCG
}
```



### 5 参考资料

1. Unity Shader 入门精要
2. [LearnOpenGL CN](https://learnopengl-cn.github.io/)
