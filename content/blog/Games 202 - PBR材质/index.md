---
title: Games 202 - PBR材质
date: 2023-02-25 10:50:14
featureImage: images/blog/image-20230226110129681.png
categories: 
- 课程笔记
tags: 
- Computer Graphic
- Math
- Games 202
---



## PBR

PBR包括Materials, lighting, camera, light transport等等任何与渲染有关的基于物理的内容。而工业界习惯叫PBR都是指PBR材质，虽然之前已经在unity中看过URP中PBR的实现原理并且尝试手动还原，这次看过Games 202后还是更加深化对PBR材质的了解。

基于表面的材质

-   Microfacet models微表面模型（不是完全基于物理的）

-   Disney Principled BRDF能够用于离线渲染, 但也可以运用在实时渲染中，但也不是PBR，是基于artist的角度来考虑的。

基于体积的材质

-   实时渲染中并没有什么特别好的解决方案，常见的诸如云，头发，皮肤。



## Microfacet BRDF

{{< figure src="image-20230225222721675.png" width="50%" >}}

**F项：菲涅尔项**，表示观察角度与反射的关系(从一个角度看去会有多少的能量被反射) 当视线与反射表面夹角越小，反射越明显。水体是菲涅尔效应最明显的现实物体之一（当站在湖边看到脚下的湖水是透明的，而远处湖面的水则是不透明的，并且反射非常强烈）。

标准菲尼尔项计算公式：

{{< figure src="image-20230226105412627.png" width="50%" >}}

而工业界主流使用采用 Schlick 的 Fresnel 近似，因为计算成本低廉，而且精度也足够。其中，n1、n2分别为两种介质的折射率。一般假设 n1=1 近似于空气折射率，而 n2取决于被渲染的物体介质。

{{< figure src="image-20230226105515745.png" width="30%" >}}

**D项：微表面的法线分布**决定这一项的是不同微表面朝向的法线分布；简单来说，当微平面的法线分布比较集中（各法线朝向大致相同）时，那么物体表面材质会更容易表现出高光；当微平面的法线分布比较散开（各法线朝向差异比较大）时，那么物体表面材质将表现的非常 diffuse。

{{< figure src="image-20230226105622230.png" width="50%" >}}

传统的 Blinn-Phong 模型也是法线分布模型

{{< figure src="image-20230226105927827.png" width="25%" >}}

**Beckmann NDF** 是第一批微平面模型中使用的法线分布，也是 Cook-Torrance BRDF 在最初提出时选择的NDF。其中，α∈[0,1] 表示为表面的粗糙程度。

{{< figure src="image-20230226110000794.png" width="35%" >}}

**GGX/Trowbridge-Reitz NDF** 是URP现在采用的法线分布函数

{{< figure src="image-20230226110036570.png" width="35%" >}}

GGX / Trowbridge-Reitz NDF 与 Beckmann NDF 的主要区别在于前者函数具有更长的尾巴，这样就可以让高光部分过渡部分更加缓和，从而更加自然。

{{< figure src="image-20230226110129681.png" width="50%" >}}

**Generalized-Trowbridge-Reitz（GTR）**

{{< figure src="image-20230226110153956.png" width="30%" >}}

其中， γ参数用于控制尾部形状。 当 γ=2 时，GTR等同于GGX。 随着 γ的值减小，分布的尾部变得更长。而随着 γ 值的增加，分布的尾部变得更短。

**G项：几何函数**体现了光在物体微平面上反射时的损耗，一般指两种损耗：阴影（Shadowing）和遮蔽（Obstruction）。

阴影来自光照射在微表面的遮挡，而遮蔽来自光反弹后被微表面遮挡。

{{< figure src="image-20230226110641358.png" width="50%" >}}

{{< figure src="image-20230226110622330.png" width="40%" >}}

### 能量补偿项（Energy Compensation Term）

几何函数表示了光在物体微平面上反射时的损耗，但实际上这些光线并不是损耗了，而是变成了微平面之间的互反射或多次表面反射的光线，但是Microfacet理论忽略了这些反射，这样做的缺点是会造成越 diffuse 的物体能量损失越多，从而使粗糙物体渲染偏暗。

解决办法是使用 The Kulla-Conty Approximation 补偿能量在BRDF上面

{{< figure src="image-20230226110849478.png" width="70%" >}}

使用能量补偿后渲染方程变为

{{< figure src="image-20230226111145169.png" width="70%" >}}



闫神说现在有人用diffuse + brdf 来描述能量补偿，也就是粗暴的用diffuse来补偿能量，但是不符合能量守恒定律。

这里的补偿是指单独对specular BRDF的补偿，而不包括diffuse部分的，因此所谓弹幕中说opengl中cook-torrance brdf多加diffuse错误的理解是不对的。闫神指出这种错误其实在最终计算会叠加2次diffuse因此是能量不守恒的。

## Linearly Transformed Cosines (LTC)

LTC:**在不考虑遮挡和阴影情况下做微表面模型的shading.**

我的理解是为了实现多边形光源对原Microfacet BRDF的改进，主要改进GGX发现分布项。

但是这个模型并不能很好的支持阴影



## Disney’s Principled BRDF

Disney 总体上采用 microfacet Cook-Torrance BRDF 着色模型，并对其中一些项进行了改造，总体公式如下：

{{< figure src="image-20230226113414818.png" width="70%" >}}

参考[Disney 源码](https://github.com/wdas/brdf/blob/main/src/brdfs/disney.brdf)：

漫反射（Diffuse）

{{< figure src="image-20230226123706382.png" width="70%" >}}

高光部分与Cook-Torrance BRDF类似

{{< figure src="image-20230226123632386.png" width="25%" >}}

菲涅尔项（Fresnel Term）

{{< figure src="image-20230226123846683.png" width="70%" >}}

法线分布项（Normal Distribution Term）

使用了 Generalized-Trowbridge-Reitz（GTR）γ=2 的版本，并根据各向异性做了一定改进。anisotropicanisotropic 表述了各向异性的程度，Xa,Ya则描述了各向异性的旋转方向。

{{< figure src="image-20230226124105179.png" width="50%" >}}

几何函数 （Geometry Function）Disney 在几何函数上使用 Smith 遮蔽函数，并基于各向异性 GGX 分布来得到如下：

{{< figure src="image-20230226124328841.png" width="50%" >}}

次表面散射SSS

{{< figure src="image-20230226123722840.png" width="50%" >}}

Sheen 布料光泽

{{< figure src="image-20230226123722840.png" width="50%" >}}

 清漆项（Clearcoat）简单粗暴的使用cook-torrance高光 * clearcoat

{{< figure src="image-20230226130639413.png" width="30%" >}}

使用 γ=1的各向同性 GTR 法线分布，并通过 clearcoatGlossclearcoatGloss 来控制清漆的光滑程度：

{{< figure src="image-20230226130724042.png" width="40%" >}}

使用 f0 = 0.04 的菲尼尔项和使用a = 0.25的ggx各向同性GGX

{{< figure src="image-20230226130811312.png" width="30%" >}}



--- 2023.03.16 更新---

[在URP中搭建一个Disney Principled BRDF](http://localhost:1313/p/在urp中搭建一个disney-principled-brdf/)
