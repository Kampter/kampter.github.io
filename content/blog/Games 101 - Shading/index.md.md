---
title: Games 101 - Shading
date: 2022-09-11 16:31:55
featureImage: images/blog/shading.jpg
categories: 
- 课程笔记
tags: 
- Computer Graphic
- Math
- HLSL
- Unity
- Games 101
---

前言：只记录自己需要的内容

---

## 深度测试 （Z buffer or Depth buffer）

{{< figure src="image-20230221163720934.png" width="50%">}}

把深度视为无限远，然后遍历每一个山脚行，再遍历每一个三角形的光栅化过程，同时记录光栅化的深度信息。如果光栅化后当前像素点的深度信息小于之前记录过的信息，则替换原本的像素点信息。

{{< figure src="Z-buffer.png" width="50%">}}

Z-buffer：对每个像素多存一个深度

复杂度：O(n) for n triangles 并不是排序，而是只要最值

需要保证三角形进入顺序和结果无关

无法处理透明物体

## Blinn-Phong Reflectance Model 光照模型 着色模型

### Diffuse

-   Lambertian (Diffuse) Shading 
    $$
    L_{d}=k_{d}\left(I / r^{2}\right) \max (0, \mathbf{n} \cdot \mathbf{l})
    $$

-   可以简化为：

    K_d 漫反射系数

    I/r2 衰减后的灯光，LightColor * AttenuationLight

    Max(0, normal dot LightDirction)

### Specular

{{< figure src="specular.png" width="50%">}}

p一般100～200，控制高光大小

### Ambient

$$
L_{a}=k_{a} I_{a}
$$

一般可以认为ambient为一个常数，要计算真实光照要考虑GI

### Combine

$$
\begin{aligned} L &=L_{a}+L_{d}+L_{s} \\ &=k_{a} I_{a}+k_{d}\left(I / r^{2}\right) \max (0, \mathbf{n} \cdot \mathbf{l})+k_{s}\left(I / r^{2}\right) \max (0, \mathbf{n} \cdot \mathbf{h})^{p} \end{aligned}
$$

--- 2022.12.20 更新 ---

URP实现

```HLSL
Shader "Custom/Blinn-Phong"
{
    Properties
    {
        _BaseColor ("Base Color", Color) = (1.0, 1.0, 1.0, 1.0)
        _BaseMap ("Main Texture", 2D) = "white"{}
        _SpecColor("Specular Color", Color) = (1.0, 1.0, 1.0, 1.0)
        _Smoothness("Smoothness", range(8, 256)) = 20
    }
    SubShader
    {
        Tags {
            "RenderType"="Opaque" 
            "RenderPipeline"="UniversalRenderPipeline" 
            "LightMode" = "UniversalForward"
            "ShaderModel"="4.5"
        }
        LOD 100
        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
            
            struct Attributes
            {
                half4 positionOS : POSITION;
                half3 normal : NORMAL;
                half4 tangetOS : TANGENT;
                half2 uv : TEXCOORD0;
            };

            struct Varyings
            {
                half4 positionCS : SV_POSITION;
                half3 positionWS : POSITION_WS;
                half2 uv : TEXCOORD0;
                half3 normalWS : NORMAL_WS;
            };

            TEXTURE2D(_BaseMap);
            SAMPLER(sampler_BaseMap);
            
            CBUFFER_START(UnityPerMaterial)
                half4 _BaseMap_ST;
                half4 _BaseColor;
                half4 _SpecColor;
                half _Smoothness;
            CBUFFER_END
            
            Varyings vert (Attributes IN)
            {
                const VertexPositionInputs vertex_position_inputs = GetVertexPositionInputs(IN.positionOS);
                const VertexNormalInputs vertex_normal_inputs = GetVertexNormalInputs(IN.normal);

                Varyings OUT;
                OUT.positionWS = vertex_position_inputs.positionWS;
                OUT.positionCS = vertex_position_inputs.positionCS;
                OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
                OUT.normalWS = vertex_normal_inputs.normalWS;
                return OUT;
            }
            
            half4 frag (Varyings IN) : SV_Target
            {
                Light mainLight = GetMainLight();
                half3 lightColor = mainLight.color;
                half3 lightDir = normalize(mainLight.direction);
                half3 lightAtten = mainLight.distanceAttenuation;
                // diffuse
                half3 albedo = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv) * _BaseColor;
                half kd = albedo;
                half light = lightColor * lightAtten;
                half ndotL = max(0, dot(IN.normalWS, lightDir));
                half3 diffuse = kd * light * ndotL;
                // specular
                half ks = 1 - kd;
                half3 viewDirWS = normalize(GetCameraPositionWS() - IN.positionWS);
                half3 halfDir = normalize(viewDirWS + lightDir);
                half nDotH = max(0, dot(halfDir, IN.normalWS));
                half3 specular = ks * light * pow(nDotH, _Smoothness) * saturate(_SpecColor);
                // Ambient
                half3 ambient = 0.1;
                // Combine
                half4 finalColor = half4(ambient + diffuse + specular, 1.0);
                return finalColor;
            }
            ENDHLSL
        }
    }
}
```

效果如下

{{< figure src="image-20230222162300339.png" width="50%">}}

## Real Time Rendering

{{< figure src="Real_Time_Rendering_Pipeline.png" width="50%">}}

1. 应用阶段：CPU传递数据给GPU（包括纹理，材质，着色器，是否Culling）。Drawcall 即CPU每调用一次GPU就计算一次Drawcall

2. 几何阶段：顶点着色器，几何着色器，曲面细分着色器，可编程。几何阶段还进行投影，裁切和屏幕映射等操作。

    几何着色器常见应用：法线可视化，绘制图形（增加删除点）。曲面细分着色器常见应用: displacement 置换

    投影：观察空间转换到**裁剪空间**（又被称为齐次裁剪空间）裁切：对透视裁剪空间来说，GPU需要对裁剪空间中的顶点执行齐次除法（其实就是将齐次坐标系中的w分量除x、y、z分量），得到顶点的**归一化的设备坐标（Normalized Device Coordinates, NDC**）屏幕映射：最终转化为**屏幕空间**，Z分量即Z-buffer

3. 光栅化阶段：包括图元组装（三个点组成一个三角形），三角形遍历（参考光栅化笔记），片元着色器，逐片元操作包括**裁剪测试（Scissor Test）**、**透明度测试（Alpha Test）**、**模板测试（Stencil Test）**以及**深度测试（Depth Test）**，可自定义。

    通过测试的片元可以进入合并，考虑是否颜色混合 （alpha blend）在经过上面的层层测试后，片元颜色就会被送到颜色缓冲区。GPU会使用**双重缓冲（Double Buffering）**的策略，即屏幕上显示**前置缓冲（Front Buffer）**，而渲染好的颜色先被送入**后置缓冲（Back Buffer）**，再替换前置缓冲，以此避免在屏幕上显示正在光栅化的图元。

## 纹理映射

纹理映射即漫反射系数k_d

{{< figure src="image-20230222160942261.png" width="50%">}}

每一个3D模型的表面展开都是1个2d平面，2d平面的x y坐标用来查询纹理的uv坐标

插值方式，V是三角形abc的重心
$$
\begin{aligned} \alpha &=\frac{-\left(x-x_{B}\right)\left(y_{C}-y_{B}\right)+\left(y-y_{B}\right)\left(x_{C}-x_{B}\right)}{-\left(x_{A}-x_{B}\right)\left(y_{C}-y_{B}\right)+\left(y_{A}-y_{B}\right)\left(x_{C}-x_{B}\right)} \\ 
\beta &=\frac{-\left(x-x_{C}\right)\left(y_{A}-y_{C}\right)+\left(y-y_{C}\right)\left(x_{A}-x_{C}\right)}{-\left(x_{B}-x_{C}\right)\left(y_{A}-y_{C}\right)+\left(y_{B}-y_{C}\right)\left(x_{A}-x_{C}\right)} \\ 
\gamma &=1-\alpha-\beta \end{aligned}
$$
投影前后的重心坐标可能会变化，所以需要在对应时间计算对应的重心坐标来做插值，不能随意复用！

### 纹素

每张纹理贴图的一个像素点。

如果纹素太小，把多个pixel 映射同一个纹素

- 解决：
    - Nearest
    - Bilinear
        - Bilinear 插值 lerp
        - 水平+竖直插值→双线性插值
        - 最近的四个点插值
    - Bicubic 双向三次插值
        - 周围16个点做三次插值
        - 运算量更大，结果更好

如果纹素太大，会产生摩尔纹

{{< figure src="image-20230222161014635.png" width="50%">}}

用Mipmap降低纹理精度

{{< figure src="image-20230222161030909.png" width="50%">}}

各向异性过滤

把贴图的xy方向分别使用不同的mipmap，比如1024x1024的贴图可以变成128 x 256或者64 x 128

- 怎么知道层数D？约为相邻pixel的映射uv之间的距离取2的对数
- 如果计算出来需要的D是整数，就很方便，直接查找
- 如果计算出来需要的D不是整数→Trilinear Interpolation三线性插值

## 参考

1.   https://zhuanlan.zhihu.com/p/137780634



