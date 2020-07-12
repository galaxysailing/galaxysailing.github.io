---
title: '[Unity Shader] 小谈深度法线纹理'
date: 2020-07-12 12:08:36
categories:
    - Unity Shader
tags:
    - Unity
    - Unity Shader
    - 深度法线纹理
mathjax: true
---

在实现一些特定的效果，我们需要获取场景中的深度纹理和法线纹理。虽然，在ShaderLab中，获取它们很简单，但在使用他们之前，应该了解他们根源。

<!-- more -->

### 1 概念

法线信息是通过顶点输入阶段而得的，这节我们只讨论深度纹理，因为深度信息比较特殊，需要通过计算得到的。

#### 1.1 深度纹理

**深度纹理**实际上来自深度缓冲，它包含了一个介于 0.0 和 1.0 之间的深度值，通常用于做深度测试(depth test)。
实际上这个深度值并不是均匀分布的，这个深度值是依据下列公式计算：

$$
depth = \frac{\frac{1}{z}-\frac{1}{near}}{\frac{1}{far}-\frac{1}{near}}
$$

这个 z 值是**观察空间(view space)**下 $z$ 分量的相反数（由于观察空间为右手坐标系）。深度值和 $\frac{1}{z}$ 成正比，函数图如下所示：

![深度值计算函数](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/12/1594527317597.png)

可以看到 $z$ 值在 $[1,2]$ 范围中精度占了50%，现实中我们对近距离的深度精度需求要高于远距离的精度，这样可以缓解[深度冲突(Z-fighting)](https://en.wikipedia.org/wiki/Z-fighting) 问题。

那么，问题来了，这个深度值计算公式又是怎么得到的。

![1](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/12/1594456714767.png)

首先，这个深度值来自于顶点变化后的 NDC 的 z 分量， 由于经过了投影变换 (通常是透视投影)，这个深度值则是非线性。还有， NDC z 分量的范围为 $[-1,1]$ 我们需要对其进行处理：

$$
depth = 0.5 * z_{ndc} + 0.5
$$

特别的，在 DirectX 接口中 z 分量的范围为 $[0,1]$

接着，我们根据视锥矩阵，来得到裁剪 $z_{clip}$ 分量和观察 $z_v$ 之间的关系：

$$
 M_{frustum} = \begin{bmatrix} 
\frac{ \cot{ \frac{FOV}{2} } }{Aspect} & 0 & 0 & 0 \\
0 & \cot{ \frac{FOV}{2} } & 0 & 0 \\
0 & 0 & -\frac{Far + Near}{Far - Near} & -\frac{2 * Near * Far}{Far - Near} \\
0 & 0 & -1 & 0
\end{bmatrix}  
$$

![摄像机](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/12/1594524191297.png)

由此可得：

$$
z_{clip} = - z_{view}\frac{(Far + Near) }{Far - Near} -\frac{2 * Near * Far}{Far - Near}
$$

$$
w_{clip} = -z_{view}
$$

同时，我们可以得到 NDC 和 $z_{view}$ 的关系：

$$
z_{ndc} = \frac{z_{clip}}{w_{clip}} = -\frac{(Far + Near) }{Far - Near} -\frac{2 * Near * Far}{(Far - Near) * z_{view}}
$$

由于我们知道了 NDC 的 $z$ 分量和深度值之间的关系，我们就能推出深度值最开始计算深度值的函数关系（我们需要对 $z_{view} 取反$，因为他是观察坐标系，总是为负数）

### 2 深度法线可视化

在Unity中，深度纹理可以直接来自于真正的深度缓存，也可以由一个单独的Pass进行渲染，取决于使用的渲染路径和硬件。在延迟渲染中，深度和纹理信息会直接渲染到G-buffer中，所以可以直接访问。而前向渲染中，是不会创建法线纹理的，因此，Unity底层使用了一个单独的 Pass （在 **buildin_shaders-xxx/DefaultResources/Camera-DepthNormalTexture.shader** 中）把整个场景都渲染了一遍
我们可以让摄像机生成一张深度纹理（精度为24位或16位，取决于深度缓存的精度）或者深度+法线纹理（共32位，深度、法线纹理各占16位）。

#### 2.1 Unity API

我们需要在脚本中设置摄像机纹理的渲染模式：

```cs
// 渲染深度信息 在shader中通过 _CameraDepthTexture访问
camera.depthTextureMode = DepthTextureMode.Depth; 

// 渲染深度和法线信息，通过_CameraDepthNormalsTexture
camera.depthTextureMode = DepthTextureMode.DepthNormals;
```

```cs
// 可以同时产生深度和深度+法线纹理
camera.depthTextureMode |= DepthTextureMode.Depth;
camera.depthTextureMode |= DepthTextureMode.DepthNormals; 
```

大多情况，我们可以通过 `tex2D` 函数直接采样，由于某些平台（如 PS3和PS2）上，我们需要一些特殊处理。因此，需要使用宏 `SAMPLE_DEPTH_TEXTURE` 进行采样：

```glsl
float d = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv);
```

还有类似的宏如：`SAMPLE_DEPTH_TEXTURE_PROJ`、`SAMPLE_DEPTH_TEXTURE_LOD`

通过纹理采样得到的深度值是非线性的，即为ndc坐标下的深度值，我们可以通过`LinearEyeDepth`将采样结果转化为观察空间下的深度值。还有`Linear01Depth`则会返回一个范围在 $[0,1]$ 的线性深度值。这两个函数使用了内置的`_ZBufferParams`变量来得到远近裁剪平面的距离。

获取深度+法线纹理，可以直接使用`tex2D`函数对`_CameraDepthNormalsTexture`进行采样。Unity提供了辅助函数`DecodeDepthNormal`来为这个采样结果进行解码，从而获得深度值和法线信息，它在`UnityCG.cginc`被定义：

```glsl
inline void DecodeDepthNormal(float4 enc, out float depth, out float3 normal){
	// 范围为[0,1]的线性深度值
	depth = DecodeFloatRG(enc.zw);
	
	// 观察空间下的法线信息
	normal = DecodeViewNormalStereo(enc);
}
```

当我们开启摄像机渲染深度+法线纹理时，我们可以在 Frame Debugger 观察深度信息和法线信息。

![渲染结果](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/12/1594525110967.png)

![深度信息](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/12/1594525135835.png)

![发信信息](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/12/1594525173375.png)

当然，可以通过编写 Shader 的方式来观察深度和法线信息。

##### 2.2 深度信息可视化

```glsl
fixed4 frag(v2f i): SV_TARGET{
	float d = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv_depth);
	float depth = 1-d;
	return fixed4(depth,depth,depth,1.0);
}

```

![原图](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/12/1594526457797.png)

运行效果

![运行效果](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/12/1594526523563.png)


##### 2.3 法线信息可视化

```glsl
fixed4 frag(v2f i): SV_TARGET{
	half4 depthNormal = tex2D(_CameraDepthNormalsTexture, i.uv);
	half3 normal = DecodeViewNormalStereo(depthNormal);   
	return fixed4(normal,1.0);
}
```

![原图](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/12/1594526249850.png)

运行效果

![运行效果](https://galaxysailing-blog.oss-cn-shanghai.aliyuncs.com/img/2020/07/12/1594526291413.png)
