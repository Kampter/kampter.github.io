---
title: Games 202 环境光照
date: 2023-01-21 14:17:21
featureImage: images/blog/image-20210414115518543.png
categories: 
- 课程笔记
tags: 
- Computer Graphic
- Games 202
- Math
---



## IBL实时环境光照

IBL：Image-Based Lighting

典型的保存方式：cube map，spherical map

在不考虑阴影的情况下（Visibility term）真实的渲染方程

{{< figure src="image-20230224111730627.png" width="50%">}}

真实求解需要用蒙特卡洛积分求解path tracing, 但是速度太慢

使用之前的近似方案

- 一点点小的区别，我们只需要对 BRDF 覆盖的范围 ΩG 进行积分即可

{{< figure src="image-20230224105959628.png" width="50%">}}

### **第一部分的积分**

红色区域就是对光源的入射方向（上面的 r ）进行了一个滤波

prefilter，在 rendering 之前预先处理

- 类似于 MIPMAP 的思想
- 预先生成多张使用不同滤波核 filter 的环境贴图
- 之后在 shading 的时候进行一个查询，双线性插值
- 如果查询的值不是一个预先设置的滤波核的大小，三线性插值

{{< figure src="image-20210411153933349.png" width="50%">}}

### **第二部分的积分**

蓝色部分的积分：预先计算 precompute

假定是 microfacet 的 BRDF

- 只需要知道菲涅尔项、微表面的法线分布（roughness）
  - Precompute its value for all possible combinations of variables roughness, color (Fresnel term), etc.
- 还是很难求积分 而且保存结果需要很大的内存

{{< figure src="image-20230224121919975.png" width="40%">}}

菲涅尔项可以用一个函数近似

- Schlick’s approximation
- 认为不同的材质是两个参数的函数：入射光夹角、基础反射率（基础颜色）

{{< figure src="image-20230224121704103.png" width="40%">}}

{{< figure src="image-20210411161156645.png" width="40%">}}

D 项可以定义一个法线分布

- Beckmann distribution
- α 定义 roughness，分布的胖瘦
- θh 表示法线和半角矢量的夹角

{{< figure src="image-20230224121853466.png" width="40%">}}

这样就可以简化为一个三维的预计算，但是计算量依旧很大

### Unreal Engine的降维思路

显式把上面的菲涅尔项写进去，试图把 R0 分离开来

- 分母的 F 会被消掉
- [完整解释和求导公式](https://learnopengl-cn.github.io/07%20PBR/03%20IBL/02%20Specular%20IBL/#brdf)

{{< figure src="image-20230224123946363.png" width="60%">}}

- 这样的好处是降维成二维纹理预计算

{{< figure src="image-20230224124241620.png" width="40%">}}

## 球面谐波函数

1. 回顾Games101的知识点，diffuse一般保留低频信息

Spherical Harmonics

球面谐波函数的可视化

- 每一行的频率是一样的，第 l 阶的 SH
- m=2l+1
- 前 n 阶一共有 n2 个基函数

{{< figure src="image-20210414115518543.png" width="50%">}}

某天某个大佬想到用这个SH函数来描绘环境光照产生的diffuse信息。发现只需要使用前三阶的SH函数就可以把误差缩小到1%

{{< figure src="1.png" width="50%">}}

而SH可以支持旋转的性质解决了旋转问题：

同一行的SH可以从同一行其他的SH旋转表示出来

## Precomputed Radiance Transfer

- 利用球谐函数的性质进行预计算
- 用另一种角度看待渲染方程
- **我们假设场景是不变的，改变的只是光照**

{{< figure src="image-20210414142001805.png" width="50%">}}

- 把 Lighting 表示为SH函数, 预计算Light transport部分

### 公式推演

将 diffuse 常数项从渲染方程中提取出来

{{< figure src="image-20230224160643950.png" width="40%">}}

光照使用 SH 表示

{{< figure src="image-20230224160658065.png" width="20%">}}

渲染方程变为

{{< figure src="image-20230224160711216.png" width="50%">}}

红色部分与光照无关，可以预计算。渲染方程变为

{{< figure src="image-20230224160748654.png" width="20%">}}

-   **场景是不能动的**，因为动了，visibility 项就变了，预计算失效
-   可以利用SH 的旋转性解决光源移动：如果光源做了一个旋转操作，很快就能够得到新的 SH

搜SH文章的时候看到这篇，对于这个观点有点感兴趣：[声音是2D的信号，图像是3D的信号](https://www.zhihu.com/column/c_1046337272613527552)



## 总结

无论 IBL 还是 PRT 都属于实现环境光照的方案，它们的区别在于：

- IBL 是一种从预计算环境光照出发的环境关照渲染方案：
  - 采用环境贴图：存储占用空间较大，同时也占采样 I/O。
  - 能保留高频信息，常用于 diffuse/glossy/specular 物体的渲染。
- PRT 是一种从预计算 transfer function 出发的环境光照渲染方案：
  - 采用 SH lighting：存储开销和重建环境光照的开销极低。
  - 只能保留低频信息，常用于 diffuse/glossy 物体的渲染。
  - 物体不可局部形变，材质不可动态：若发生变化，那么其 transfer 就需要更新。
  - 只考虑了物体局部 transfer 效果，没有考虑完整场景 transfer 效果，不过在其它 PRT 方案中有支持完整场景的 transfer 效果。

## 参考

1. https://learnopengl-cn.github.io/07%20PBR/03%20IBL/02%20Specular%20IBL/

2. https://www.cnblogs.com/KillerAery/p/15335369.html

3. https://www.zhihu.com/column/c_1046337272613527552

    