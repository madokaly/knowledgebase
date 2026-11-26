# **PBR（基于物理的渲染）原理详解**

PBR（Physically Based Rendering，基于物理的渲染）是现代游戏引擎中非常核心的渲染技术，也是 UE5 游戏客户端、图形学、技术美术相关面试中的高频知识点。

PBR 并不是某一种固定的光照算法，而是一套遵循物理规律、模拟光线与材质相互作用的渲染方法。

传统的 Phong、Blinn-Phong 光照模型主要通过经验公式模拟光照效果，而 PBR 更关注光线在现实世界中的传播、反射、折射、吸收以及能量分布。

下面以 **UE5 常见的 Metallic-Roughness（金属度—粗糙度）工作流、Cook-Torrance BRDF 光照模型**为主进行讲解。

# **一、PBR 的核心思想**

PBR 主要解决以下几个问题：

- 光线照射到物体表面后，多少光线被反射？
- 多少光线进入物体内部，被散射或者吸收？
- 为什么金属与非金属的光照表现不同？
- 为什么光滑物体的高光集中，而粗糙物体的高光分散？
- 为什么从物体侧面观察时，反射通常更加明显？
- 如何让同一种材质在不同光照环境下依然具有合理的表现？

为了实现这些效果，PBR 主要依赖以下理论：

|核心理论|中文解释|主要作用|
|---|---|---|
|Energy Conservation|能量守恒|限制反射能量|
|Microfacet Theory|微表面理论|描述物体表面的微观粗糙程度|
|BRDF|双向反射分布函数|计算不同方向上的光线反射|
|Fresnel Effect|菲涅尔效应|计算不同观察角度下的反射比例|
|NDF|法线分布函数|描述微表面法线的分布|
|Geometry Function|几何函数|计算微表面的遮挡与阴影|
|IBL|基于图像的光照|计算环境光照与环境反射|

# **二、光线与物体表面的相互作用**

当光线照射到物体表面时，通常会发生以下几种情况。

<img width="742" height="524" alt="image" src="https://github.com/user-attachments/assets/512aa75c-f863-4d54-80a2-d6b682ec26ee" />

## **2.1 Specular Reflection（镜面反射）**

光线在物体表面发生反射，没有进入材质内部。

镜面反射主要决定物体的：

- 高光（Highlight）
- 环境倒影（Environment Reflection）
- 光泽感
- 金属质感

例如镜子、抛光金属、水面等，通常具有比较明显的镜面反射。

## **2.2 Diffuse Reflection（漫反射）**

对于普通非金属材质，部分光线进入材质内部，经过多次散射后重新从表面射出。

这个过程中，材质会吸收某些波长的光线，剩余波长决定我们看到的物体颜色。

例如红色塑料在白光照射下呈现红色，是因为材质更多地吸收其他波长的光线，而散射出较多红光。

实时渲染通常使用 Lambert（兰伯特漫反射）等模型来近似这个复杂过程。

## **2.3 Absorption（光线吸收）**

部分光线进入材质后，其能量被材质吸收，通常转换为热能等其他形式的能量。

PBR 中的 Base Color（基础颜色）就与材质对不同波长光线的反射、散射和吸收能力有关。

# **三、Energy Conservation（能量守恒）**

能量守恒是 PBR 最基本的物理原则之一。

对于不主动发光的物体，反射和透射出去的总能量不能超过接收到的入射光能量。

可以简化理解为：

$$
E_{\text{反射}}+E_{\text{透射}}+E_{\text{吸收}} =E_{\text{入射}}
$$

在常见的不透明材质模型中，通常使用以下两个系数进行近似：

$$
k_s：Specular Ratio，镜面反射比例。
$$
$$
k_d：Diffuse Ratio，漫反射分量的权重。
$$

首先计算镜面反射比例：

$$
k_s=F
$$

其中 F 是菲涅尔反射系数。

剩余可用于漫反射的比例为：

$$
k_d=(1-k_s)(1-Metallic)
$$

其中 Metallic 表示金属度。

<img width="739" height="385" alt="image" src="https://github.com/user-attachments/assets/eb354e97-add1-47bd-8c2b-b191b0adfcff" />

当 Metallic = 0 时：

$$
k_d=1-k_s
$$

表示普通非金属材质可以同时具有漫反射和镜面反射。

当 Metallic = 1 时：

$$
k_d=0
$$

表示理想的纯金属工作流中，不再计算普通漫反射，颜色主要通过镜面反射表现。

**能量守恒并不意味着最终像素的 RGB 值一定小于 1。** HDR 光照可以远大于 1，能量守恒约束的是材质对入射能量的响应，而不是屏幕颜色数值。

# **四、Microfacet Theory（微表面理论）**

## **4.1 什么是微表面理论？**

现实世界中，即使一个物体表面看起来非常光滑，在微观层面上也可能存在大量凹凸不平的结构。

PBR 将宏观物体表面近似为无数个微小的镜面，这些镜面称为：

**Microfacet（微表面、微面元）。**

每一个微表面都有自己的法线方向，并遵循镜面反射规律。

<img width="739" height="580" alt="image" src="https://github.com/user-attachments/assets/828f1d76-c98c-455b-8fe1-54759cea5a46" />

## **4.2 粗糙度如何影响光照？**

Roughness（粗糙度）描述的就是微表面法线分布的离散程度。

|对比项|低粗糙度|高粗糙度|
|---|---|---|
|微表面法线|比较集中|比较分散|
|反射方向|集中|分散|
|高光范围|小|大|
|高光峰值|通常较高|通常较低|
|环境倒影|清晰|模糊|
|典型材质|镜子、抛光金属|石头、磨砂金属|

<img width="787" height="271" alt="image" src="https://github.com/user-attachments/assets/6d76ae19-fd17-4bc6-9b38-312fa396e564" />

注意一个非常常见的误区：

**粗糙度越高，并不意味着材质本身的菲涅尔反射率越低，而是镜面反射能量的方向分布发生了改变。**

不过在实际的单次散射微表面模型中，粗糙度也可能影响积分后的总反射能量，因此现代 PBR 实现有时需要进行多重散射能量补偿。

# **五、BRDF（双向反射分布函数）**

## **5.1 什么是 BRDF？**

BRDF（Bidirectional Reflectance Distribution Function，双向反射分布函数）用于描述：

**从某个方向照射到物体表面的光线，会以多大的强度向另一个方向反射。**

之所以叫双向，是因为它同时考虑入射光方向和出射光方向。

$$
f_r(L,V)=\frac{dL_o(V)}{dE_i(L)}
$$

其中：

| 符号      | 中文解释                      |
| ------- | ------------------------- |
| \(L\)   | Light Direction，指向光源的方向   |
| \(V\)   | View Direction，指向摄像机的方向   |
| \(L_o\) | Outgoing Radiance，出射辐亮度   |
| \(E_i\) | Incoming Irradiance，入射辐照度 |
| \(f_r\) | BRDF，材质反射响应函数             |

BRDF 不是最终的像素颜色，也不是简单的反射百分比，而是描述材质如何将入射光分配到不同出射方向的函数。

## **5.2 PBR 中的四个核心向量**

<img width="741" height="546" alt="image" src="https://github.com/user-attachments/assets/8bcbaf05-b56d-4c1f-a03a-c3c0771b8080" />

|向量|中文名称|作用|
|---|---|---|
|N|Normal，表面法线|表示当前表面的朝向|
|L|Light，光源方向|表示光源相对于表面的方向|
|V|View，观察方向|表示摄像机相对于表面的方向|
|H|Half Vector，半程向量|表示能够将 L 方向的光反射到 V 方向的理想微表面法线|

半程向量的计算：

$$
H=\frac{L+V}{\|L+V\|}
$$

Shader 中通常会使用以下几个点积：

```
float3 H = normalize(L + V);  // 半程向量

float NoL = saturate(dot(N, L)); // 表面朝向光源的程度
float NoV = saturate(dot(N, V)); // 表面朝向摄像机的程度
float NoH = saturate(dot(N, H)); // 理想微表面与宏观法线的接近程度
float VoH = saturate(dot(V, H)); // 观察方向与微表面法线的夹角余弦
```

其中 `saturate()` 是 HLSL 中将数值限制在 0～1 范围内的函数。

## **5.3 Cook-Torrance BRDF 模型**

现代实时渲染中非常经典的方案是将漫反射模型与 Cook-Torrance 微表面镜面反射模型结合。

$$
\boxed{ f_r=k_d\frac{C}{\pi} + \frac{D\cdot F\cdot G} {4(N\cdot L)(N\cdot V)} }
$$

其中：

|参数|中文名称|核心作用|
|---|---|---|
|\(C\)|Base Color，基础颜色|决定材质的颜色|
|\(k_d\)|漫反射权重|控制漫反射能量分配|
|\(D\)|Distribution，法线分布函数|有多少微表面满足反射方向|
|\(F\)|Fresnel，菲涅尔函数|满足反射条件的微表面能反射多少光|
|\(G\)|Geometry，几何函数|有多少反射光没有被其他微表面遮挡|

可以将其拆分为：

$$
BRDF=DiffuseBRDF+SpecularBRDF
$$

<img width="746" height="214" alt="image" src="https://github.com/user-attachments/assets/68679c81-ead2-4111-b319-b17e919f99b4" />

其中 D、F、G 三个函数是 PBR 最重要的知识点，也是面试中最经常被问到的部分。

# **六、D（Normal Distribution Function，法线分布函数）**

## **6.1 D 函数解决什么问题？**

假设当前光源方向为 L，摄像机方向为 V。

根据镜面反射规律，只有法线方向为 H 的理想微表面，才能将来自 L 方向的光准确反射到 V 方向。

但是一个粗糙表面有大量方向不同的微表面。

因此我们需要计算：

**法线方向为 H 的微表面分布密度有多大？**

这就是 D 函数的作用。

<img width="740" height="394" alt="image" src="https://github.com/user-attachments/assets/8ade96bb-38dd-44e2-a43e-a0472de64e56" />

## **6.2 GGX 法线分布函数**

目前游戏引擎中非常常用的是：

**GGX（Trowbridge-Reitz）法线分布函数。**

公式如下：

$$
D_{GGX}= \frac{\alpha^2} {\pi\left[(N\cdot H)^2(\alpha^2-1)+1\right]^2}
$$

其中：

$$
\alpha=Roughness^2
$$

注意，这里 \(\alpha\) 已经是粗糙度的平方，而公式中的 \(\alpha^2\) 实际上对应 Roughness 的四次方。

## **6.3 GGX 函数与粗糙度的关系**

<img width="739" height="684" alt="image" src="https://github.com/user-attachments/assets/ac0606a9-72b3-4058-bed0-58cb18605d38" />

当粗糙度很低时，微表面法线高度集中在 N 附近，D 函数的峰值很高，但是偏离 N 后会快速下降，最终形成狭窄、明亮的高光。

当粗糙度很高时，D 函数的分布更加平缓，更多方向上都存在能够反射光线的微表面，最终形成范围更广、峰值较低的高光。

## **6.4 HLSL 实现**

```
// GGX 法线分布函数
float DistributionGGX(float NoH, float Roughness)
{
    // 将美术使用的粗糙度转换为微表面粗糙度
    float a = Roughness * Roughness;
    float a2 = a * a;

    float d = NoH * NoH * (a2 - 1.0) + 1.0;

    // 防止分母为 0
    return a2 / max(PI * d * d, 1e-8);
}
```

实际使用时通常还需要对极低粗糙度进行限制，并根据像素覆盖范围进行高光抗锯齿处理。

# **七、F（Fresnel，菲涅尔函数）**

## **7.1 什么是菲涅尔效应？**

物体表面的反射率会随着光线与表面法线之间的夹角发生变化。

例如观察平静的水面：

- 垂直向下看，能够比较清楚地看到水底。
- 从接近水平方向观察，水面反射的天空和周围建筑更加明显。

这是因为当观察方向接近掠射角时，表面镜面反射率通常会显著增加。

<img width="741" height="393" alt="image" src="https://github.com/user-attachments/assets/c5f248fb-242e-4c50-be11-c2caa40b5885" />

## **7.2 Fresnel-Schlick 近似公式**

完整的菲涅尔方程计算比较复杂，实时渲染中经常使用 Schlick 近似公式：

$$
\boxed{ F=F_0+(1-F_0)(1-V\cdot H)^5 }
$$

其中：

$$
F_0：Normal Incidence Reflectance，垂直入射时的基础镜面反射率。
$$
$$
V\cdot H：观察方向与微表面法线的夹角余弦。
$$
指数 5：用于近似菲涅尔反射率随角度变化的曲线。

在微表面 BRDF 中，这里使用的是 V·H，而不是直接使用 V·N，因为实际发生镜面反射的是微表面。

## **7.3 不同角度下的菲涅尔反射率**

$$
以普通非金属材质 F_0=0.04为例：
$$

<img width="742" height="461" alt="image" src="https://github.com/user-attachments/assets/a5d72528-b295-4f49-9165-8c938cd4b956" />

可以看出：

|夹角|反射率（约）|
|---|---|
|0°|4%|
|30°|4%|
|60°|7%|
|80°|41%|
|90°|100%|

因此，即使是塑料、石头等非金属，也会在适当角度出现明显的镜面反射。

## **7.4 F0 是如何得到的？**

对于空气与普通非吸收电介质之间的界面，垂直入射时的反射率可以通过折射率计算：

$$
F_0=\left(\frac{n_1-n_2}{n_1+n_2}\right)^2
$$

假设空气折射率约为 1，玻璃折射率约为 1.5：

$$
F_0=\left(\frac{1-1.5}{1+1.5}\right)^2 =0.04
$$

这就是很多 PBR 实现将非金属基础镜面反射率设置为 0.04 的原因。

金属的折射率需要考虑复数形式，其反射率通常更高，并且不同波长的反射率差异较大，因此金属的镜面反射往往带有颜色。

## **7.5 HLSL 实现**

```
// Fresnel-Schlick 菲涅尔近似
float3 FresnelSchlick(float VoH, float3 F0)
{
    // VoH 越小，越接近掠射角，反射率越高
    float factor = pow(1.0 - saturate(VoH), 5.0);

    return F0 + (1.0 - F0) * factor;
}
```

注意这里的 F0 和返回值是 `float3`，因为红、绿、蓝三个颜色通道的反射率可以不同。

# **八、G（Geometry Function，几何函数）**

## **8.1 G 函数解决什么问题？**

微表面之间不是完全独立的。

即使某个微表面的法线方向正好可以将光线反射到摄像机，也可能发生两种情况：

- **Shadowing（阴影遮挡）：** 入射光还没有照射到目标微表面，就已经被其他微表面挡住。
- **Masking（观察遮挡）：** 光线成功照射目标微表面，但是反射光在到达摄像机之前，被其他微表面挡住。

<img width="745" height="379" alt="image" src="https://github.com/user-attachments/assets/c42e755b-48b3-43c2-bdda-a7a7e0fdfdcb" />

因此 G 函数就是计算微表面同时对光源和观察者可见的程度。

通常：

$$
0\leq G\leq1
$$

G = 1：没有因微表面遮挡而损失该次反射的贡献。 
G = 0：该次反射贡献完全被遮挡。
G 越小：镜面反射受到的微观遮挡越明显。

## **8.2 Smith 几何函数**

一种常见近似方式是分别计算光源方向和观察方向的可见性，然后将两者相乘：

$$
G(N,V,L)=G_1(N,V)G_1(N,L)
$$

其中单方向的 Schlick-GGX 几何函数为：

$$
G_1(N,X)= \frac{N\cdot X} {(N\cdot X)(1-k)+k}
$$

对于一种常见的直接光照近似：

$$
k=\frac{(Roughness+1)^2}{8}
$$

而经典 IBL 近似中可以使用：

$$
k=\frac{Roughness^2}{2}
$$

不同引擎以及不同版本可以使用其他 Smith GGX 可见性函数，例如 Height-Correlated Smith GGX（高度相关的 Smith GGX 模型）。

## **8.3 HLSL 实现**

```
// 计算单个方向上的微表面可见性
float GeometrySchlickGGX(float NoX, float Roughness)
{
    float r = Roughness + 1.0;
    float k = r * r / 8.0;

    return NoX / max(NoX * (1.0 - k) + k, 1e-6);
}

// Smith 几何函数
float GeometrySmith(float NoV, float NoL, float Roughness)
{
    // 观察方向的遮挡
    float masking = GeometrySchlickGGX(NoV, Roughness);

    // 光源方向的遮挡
    float shadowing = GeometrySchlickGGX(NoL, Roughness);

    return masking * shadowing;
}
```

**注意：G 函数与 Shadow Map（阴影贴图）不是一回事。**

G 函数计算的是材质内部的微观表面遮挡；Shadow Map 计算的是场景物体之间的宏观遮挡。

两者分别解决不同尺度上的光照问题。

# **九、D、F、G 三者的关系**

这是 PBR 面试最需要掌握的部分。

|函数|解决的问题|主要影响因素|
|---|---|---|
|D|有多少符合反射方向的微表面|Roughness、N·H|
|F|每个微表面能够反射多少光|F0、V·H|
|G|有多少微表面反射能够被观察到|Roughness、N·L、N·V|
|分母|将微表面反射转换为宏观表面的 BRDF|N·L、N·V|

$$
SpecularBRDF= \frac{D\cdot F\cdot G}{4(N\cdot L)(N\cdot V)}
$$

可以简单记忆为：

**D 管分布，F 管反射率，G 管遮挡。**

分母中的 4(N·L)(N·V) 则来自微表面反射模型中的方向变换及投影面积关系，并不是单纯为了让结果不超过 1。

# **十、PBR 渲染方程（Rendering Equation）**

前面介绍的 BRDF 只解决了材质如何反射光线的问题。

真正计算最终像素颜色时，还需要考虑光源颜色、光照强度、光照方向、环境光照等。

完整的不透明表面反射渲染方程为：

$$
L_o(p,V)=L_e(p,V)+ \int_{\Omega} f_r(p,L,V)L_i(p,L) (N\cdot L)d\omega_i
$$

$$
p     当前着色点的位置
$$
$$
L_o    ， 朝向观察者的出射辐亮度
$$
$$
L_e   ，  Emissive；自发光辐亮度
$$
$$
L_i  ，   来自某个方向的入射辐亮度
$$
$$
f_r   ，  BRDF，材质反射响应
$$
$$
\Omega  ，   表面法线方向的上半球
$$
$$
d\omega_i，    微小立体角
$$
$$
N\cdot L ，     入射光的余弦投影项
$$

## **10.1 为什么要对半球进行积分？**

<img width="741" height="480" alt="image" src="https://github.com/user-attachments/assets/6772128a-5ccc-4d28-a79f-3e682426f41b" />

因为现实中物体不只是受到一个光源照射。

它可能同时受到太阳、天空、灯光、墙壁反射光、地面反射光等影响。

理论上，需要对半球内所有入射方向进行积分。

对于实时游戏而言，逐像素对所有方向进行积分的开销太大，因此通常拆分为：

$$
Lighting=DirectLighting+IndirectLighting
$$

即直接光照和间接光照。

## **10.2 直接光照（Direct Lighting）**

直接光照来自可以明确计算的光源，例如 Directional Light（方向光）、Point Light（点光源）、Spot Light（聚光灯）。

对于离散光源，可以将光照计算近似为：

$$
L_{direct}=\sum_{j=1}^{n} f_r(L_j,V)I_j A_j S_j \max(N\cdot L_j,0)
$$

其中
$$
I_j 为有效光源强度及颜色，
$$
$$
A_j 为距离等衰减项，
$$
$$
S_j 为场景阴影可见性。
$$

每一个光源都需要计算相应的 BRDF，然后将结果累加。

# **十一、PBR 常用材质参数详解**

UE5 中经典 Default Lit（默认光照）材质主要包含以下参数。

<img width="1200" height="1200" alt="image" src="https://github.com/user-attachments/assets/4b5e52c0-1987-4529-8d11-36dfa5fa0121" />

**Base Color（基础颜色）**

非金属材质的漫反射颜色；纯金属材质的基础镜面反射颜色。不应该直接烘焙高光、场景阴影等光照信息。

<img width="900" height="900" alt="image" src="https://github.com/user-attachments/assets/d3964a9c-a136-461b-a700-5dab8b205ef5" />

**Metallic（金属度）**

控制材质属于金属还是非金属。0 为非金属，1 为纯金属。

<img width="1024" height="1024" alt="image" src="https://github.com/user-attachments/assets/7df97962-659f-47a7-9a51-f9c648d01906" />

**Roughness（粗糙度）**

控制微表面法线分布，从而影响高光宽度和环境反射的清晰程度。

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/6f8082eb-12d1-46d3-b5a5-9e5cc5a42bac" />

**Normal Map（法线贴图）**

修改每个着色点参与光照计算的法线方向，在不增加实际模型几何面数的情况下表现表面细节。

<img width="512" height="512" alt="image" src="https://github.com/user-attachments/assets/a64fc9a5-acce-4c5c-b64d-74b662ba5dc8" />

**AO（Ambient Occlusion，环境光遮蔽）**

近似描述凹槽、缝隙等位置接收间接光照的困难程度，不应简单地将所有直接光照一起变暗。

<img width="512" height="512" alt="image" src="https://github.com/user-attachments/assets/f65454f0-f954-41f5-bd6a-54ed12c62585" />

**Emissive（自发光）**

表示材质自身发出的光，例如武器能量条、角色发光纹路、灯管等。

## **11.1 Metallic（金属度）的实现原理**

金属度不仅仅决定物体是否具有金属光泽，还参与漫反射颜色和镜面反射颜色的计算。

对于常见的金属度工作流：

```
// 计算基础镜面反射率
float3 F0 = lerp(
    float3(0.04, 0.04, 0.04),
    BaseColor,
    Metallic
);

// 计算漫反射颜色
float3 DiffuseColor = BaseColor * (1.0 - Metallic);
```

其中 `lerp` 是线性插值函数。

### **非金属材质（Dielectric，电介质）**

当 Metallic = 0：

```
F0 = float3(0.04, 0.04, 0.04);
DiffuseColor = BaseColor;
```

特征是具有明显的漫反射颜色，同时通常具有较低的、接近无色的基础镜面反射率。

例如塑料、木头、石头、陶瓷等。

### **金属材质（Conductor，导体）**

当 Metallic = 1：

```
F0 = BaseColor;
DiffuseColor = float3(0, 0, 0);
```

特征是普通漫反射近似为零，颜色主要来自带有颜色的镜面反射。

例如黄金、铜、铁、铝等。

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/47bc10ba-41a3-4586-a34e-5f4ae57a0c9a" />

**非金属材质**

红色漫反射，接近白色的高光

<img width="750" height="527" alt="image" src="https://github.com/user-attachments/assets/725c1f2f-876b-43df-b3ef-12c5b022873a" />

**金属材质**

镜面反射本身具有颜色

## **11.2 Metallic 和 Roughness 的关系**

Metallic 与 Roughness 是两个不同的属性。

- Metallic：决定材质反射的性质和颜色。
    
- Roughness：决定镜面反射的方向分布。
    

因此，金属也可以非常粗糙，非金属也可以非常光滑。

<img width="1024" height="576" alt="image" src="https://github.com/user-attachments/assets/6388cff6-6398-407d-8757-fc8f37a8551d" />

|材质|Metallic 参考值|Roughness 参考值|
|---|---|---|
|光滑塑料|0|0.15|
|磨砂塑料|0|0.7|
|木头|0|0.65|
|石头|0|0.9|
|抛光黄金|1|0.1|
|抛光钢铁|1|0.2|
|磨砂铝|1|0.6|
|粗糙裸铁|1|0.85|

这些是用于理解材质表现的示例值，不是固定的物理测量数据。

## **11.3 UE5 中的 Specular 参数是什么？**

UE 的经典材质工作流除了 Metallic 和 Roughness，还提供了 Specular（镜面反射）参数。

对于普通非金属材质，可以近似理解为：

$$
F_0=0.08\times Specular
$$

UE 默认：

```
Specular = 0.5;
```

因此：

$$
F_0=0.08\times0.5=0.04
$$

结合 Metallic：

```
float3 F0 = lerp(
    0.08 * Specular,
    BaseColor,
    Metallic
);
```

所以 UE 中的 Specular = 0.5 并不表示反射 50% 的光，而是对应约 4% 的非金属垂直入射镜面反射率。

通常想要让物体看起来更光滑，应该优先修改 Roughness，而不是 Specular。

# **十二、IBL（Image Based Lighting，基于图像的光照）**

## **12.1 为什么 PBR 需要 IBL？**

只计算直接光照，无法表现来自周围环境的光照和反射。

尤其是金属材质，由于没有普通漫反射，在缺少直接光照贡献和环境反射的区域，很容易表现为黑色。

IBL 的核心思想是：

**将整个环境看成一个能够从不同方向提供光照的大型光源。**

<img width="2448" height="1224" alt="image" src="https://github.com/user-attachments/assets/d37b7658-006a-4bdd-84e6-8c9a092c8758" />

<img width="2500" height="1668" alt="image" src="https://github.com/user-attachments/assets/9c1462ba-140f-4cae-90d3-62cda4e1be7c" />

IBL 通常将环境光照划分为：

- Diffuse IBL（环境漫反射光照）。
- Specular IBL（环境镜面反射光照）。

## **12.2 Diffuse IBL（环境漫反射）**

环境漫反射需要计算表面法线朝向的整个半球所提供的光照。

由于 Lambert 漫反射主要依赖法线方向，可以提前对环境贴图进行卷积积分，生成：

**Irradiance Map（辐照度贴图）。**

其数学含义为：

$$
E(N)=\int_{\Omega}L_i(L) \max(N\cdot L,0)d\omega
$$

运行时只需要根据 N 方向采样辐照度贴图即可。

```
// 根据表面法线采样已经预积分的环境光
float3 Irradiance = IrradianceMap.Sample(
    LinearSampler, N
).rgb;

// 计算环境漫反射
float3 DiffuseIBL = DiffuseColor * Irradiance / PI;
```

这里假设 Irradiance Map 存储的是未除以 π 的辐照度。如果预计算阶段已经包含了该归一化项，运行时就不能再次除以 π。

## **12.3 Specular IBL（环境镜面反射）**

环境镜面反射更加复杂，因为它同时受到以下属性影响：

- 表面法线 N。
- 观察方向 V。
- 粗糙度 Roughness。
- 菲涅尔反射率 F0。
- 环境光照分布。

经典实时 PBR 方案通常使用：

**Split-Sum Approximation（分离求和近似）。**

将复杂的环境镜面反射积分近似拆分为两个部分。

<img width="738" height="757" alt="image" src="https://github.com/user-attachments/assets/1786f47a-8c0d-4b58-989e-59e3b95568a1" />

**Specular IBL**

$$
L_{prefiltered}(F_0 A+B),运行时采样两种预计算资源并组合
$$

### **Prefiltered Environment Map（预过滤环境贴图）**

使用不同的 Mipmap（多级渐远纹理）存储不同粗糙度对应的环境反射结果。

注意这里不是简单地生成普通纹理 Mipmap，而是根据 GGX 等反射分布进行预过滤。

|Mip Level|对应粗糙度|环境反射|
|---|---|---|
|Mip 0|很低|清晰|
|Mip 1|较低|略微模糊|
|Mip 2|中等|比较模糊|
|Mip 3|较高|高度模糊|
|Mip 4|很高|非常分散|

具体 HLSL 示意：

```
// 根据观察方向和法线计算反射方向
float3 R = reflect(-V, N);

// 根据粗糙度选择预过滤环境贴图的 Mip
float MipLevel = Roughness * MaxReflectionMip;

// 采样对应粗糙度的环境反射
float3 PrefilteredColor = PrefilteredMap.SampleLevel(
    LinearSampler,
    R,
    MipLevel
).rgb;
```

### **BRDF LUT（BRDF 查找表）**

LUT（Look-Up Table，查找表）用于存储预计算结果。

经典 BRDF LUT 的输入为：

```
float2 UV = float2(NoV, Roughness);
```

输出通常为两个系数 A 和 B。

```
float2 EnvBRDF = BRDF_LUT.Sample(
    LinearSampler,
    float2(NoV, Roughness)
).rg;

// Split-Sum 镜面反射近似
float3 SpecularIBL =
    PrefilteredColor *
    (F0 * EnvBRDF.x + EnvBRDF.y);
```

这样就将复杂的实时积分转换成了纹理采样和少量数学运算，大幅降低了计算开销。

Specular IBL+1

# **十三、完整的 PBR Shader 实现**

下面将之前的公式组合起来，实现一个简化版的 PBR 直接光照 Shader。

```
static const float PI = 3.14159265359;

// ============================================
// D：GGX 法线分布函数
// 计算有多少微表面符合镜面反射方向
// ============================================
float DistributionGGX(float NoH, float Roughness)
{
    float a = Roughness * Roughness;
    float a2 = a * a;

    float d = NoH * NoH * (a2 - 1.0) + 1.0;

    return a2 / max(PI * d * d, 1e-8);
}

// ============================================
// F：菲涅尔函数
// 计算不同角度下的镜面反射率
// ============================================
float3 FresnelSchlick(float VoH, float3 F0)
{
    float factor = pow(1.0 - saturate(VoH), 5.0);

    return F0 + (1.0 - F0) * factor;
}

// ============================================
// G：单方向几何遮挡函数
// ============================================
float GeometrySchlickGGX(float NoX, float Roughness)
{
    float r = Roughness + 1.0;
    float k = r * r / 8.0;

    return NoX / max(NoX * (1.0 - k) + k, 1e-6);
}

// ============================================
// G：Smith 几何函数
// 同时考虑光源和观察方向的微表面遮挡
// ============================================
float GeometrySmith(
    float NoV,
    float NoL,
    float Roughness)
{
    float Gv = GeometrySchlickGGX(NoV, Roughness);
    float Gl = GeometrySchlickGGX(NoL, Roughness);

    return Gv * Gl;
}

// ============================================
// 计算单个光源的 PBR 直接光照
// ============================================
float3 CalculatePBRLighting(
    float3 N,            // 世界空间法线
    float3 V,            // 指向摄像机的方向
    float3 L,            // 指向光源的方向
    float3 BaseColor,    // 线性空间基础颜色
    float Metallic,     // 金属度
    float Roughness,    // 粗糙度
    float Specular,     // UE 风格镜面反射参数
    float3 LightColor,   // 光源颜色及有效强度
    float Attenuation,   // 光照衰减
    float Shadow)        // 阴影可见性，0~1
{
    N = normalize(N);
    V = normalize(V);
    L = normalize(L);

    Metallic = saturate(Metallic);
    Specular = saturate(Specular);

    // 示例：限制极低粗糙度，减少数值不稳定
    Roughness = clamp(Roughness, 0.045, 1.0);

    float NoL = saturate(dot(N, L));
    float NoV = saturate(dot(N, V));

    // 背向光源或观察者时不计算普通表面反射
    if (NoL <= 0.0 || NoV <= 0.0)
        return float3(0.0, 0.0, 0.0);

    // 计算半程向量
    float3 H = normalize(L + V);

    float NoH = saturate(dot(N, H));
    float VoH = saturate(dot(V, H));

    // ----------------------------------------
    // 1. 计算基础镜面反射率 F0
    // ----------------------------------------
    float3 F0 = lerp(
        0.08 * Specular,
        BaseColor,
        Metallic
    );

    // ----------------------------------------
    // 2. 计算 D、F、G
    // ----------------------------------------
    float D = DistributionGGX(NoH, Roughness);
    float3 F = FresnelSchlick(VoH, F0);
    float G = GeometrySmith(NoV, NoL, Roughness);

    // ----------------------------------------
    // 3. Cook-Torrance 镜面反射 BRDF
    // ----------------------------------------
    float3 SpecularBRDF =
        (D * F * G) /
        max(4.0 * NoL * NoV, 1e-6);

    // ----------------------------------------
    // 4. 计算漫反射权重
    // ----------------------------------------
    float3 Ks = F;
    float3 Kd = (1.0 - Ks) * (1.0 - Metallic);

    // ----------------------------------------
    // 5. Lambert 漫反射 BRDF
    // ----------------------------------------
    float3 DiffuseBRDF = Kd * BaseColor / PI;

    // ----------------------------------------
    // 6. 计算光源贡献
    // ----------------------------------------
    float3 Radiance = LightColor * Attenuation;

    return (DiffuseBRDF + SpecularBRDF)
         * Radiance
         * NoL
         * Shadow;
}
```

上述代码包含了经典 PBR 直接光照的核心计算流程，但它是便于学习的教学实现，并非 UE5 引擎源码的逐行复刻。实际引擎会根据材质模型、平台和渲染路径采用不同的近似、可见性函数及能量补偿方法。

# **十四、PBR 在 UE5 渲染管线中的位置**

以传统 Default Lit 材质与 Deferred Rendering（延迟渲染）为例。

<img width="745" height="828" alt="image" src="https://github.com/user-attachments/assets/33df7f15-ff59-48c5-bf1e-a488c2834167" />

## **14.1 UE 源码相关文件**

如果需要深入研究 UE 的 PBR 实现，可以重点关注经典非 Substrate 着色路径中的这些 Shader 文件：

|文件|主要作用|
|---|---|
|`BRDF.ush`|BRDF 基础数学函数|
|`ShadingModels.ush`|不同材质光照模型的实现|
|`DeferredLightingCommon.ush`|延迟光照相关公共计算|
|`DeferredLightPixelShaders.usf`|延迟光照像素着色器|
|`BasePassPixelShader.usf`|Base Pass 像素着色器|
|`ReflectionEnvironmentShared.ush`|环境反射相关计算|

重点关注以下函数：

```
D_GGX();                 // GGX 法线分布函数
F_Schlick();             // 菲涅尔函数
Vis_SmithJointApprox();  // Smith 联合可见性近似
Diffuse_Lambert();       // Lambert 漫反射
SpecularGGX();           // GGX 镜面反射
DefaultLitBxDF();        // 默认光照模型
```

其中 UE 的部分实现会将几何函数 G 和 Cook-Torrance 的分母合并为 Visibility（可见性）函数：

$$
Vis=\frac{G}{4(N\cdot L)(N\cdot V)}
$$

因此源码中可能看到：

```
Specular = D * Vis * F;
```

它在数学结构上仍然属于前面所讲的微表面镜面反射模型。

具体文件与函数会随 UE 版本、Substrate 设置和渲染路径变化，应以实际项目使用的引擎源码为准。

## **14.2 PBR 和 Lumen 有什么区别？**

这是 UE 面试中容易混淆的两个概念。

|对比项|PBR|Lumen|
|---|---|---|
|主要作用|计算材质如何响应光线|计算场景间接光照和反射|
|解决的问题|光照到达物体后如何反射|光线从哪里来、如何在场景中传播|
|核心技术|BRDF、微表面理论|场景表示、光线追踪、光照缓存等|
|是否必须配合使用|否|否|

关闭 Lumen 不代表关闭 PBR。

使用烘焙光照、Reflection Capture（反射捕获）、传统 Sky Light（天空光）等方案，也可以实现 PBR 渲染。

# **十五、PBR 与传统光照模型的区别**

|对比项|Phong|Blinn-Phong|PBR|
|---|---|---|---|
|漫反射|通常 Lambert|通常 Lambert|Lambert、Burley 等|
|镜面反射|R·V|N·H|DFG 微表面模型|
|高光控制|Shininess|Shininess|Roughness|
|能量守恒|经典形式不保证|经典形式不保证|作为设计原则|
|菲涅尔效应|默认没有|默认没有|通常支持|
|金属与非金属|手动调整|手动调整|材质工作流统一描述|
|环境适应性|相对较差|相对较差|通常较好|
|计算开销|较低|较低|相对较高|

Blinn-Phong 的高光公式：

$$
Specular=K_s(N\cdot H)^n
$$

PBR 的高光公式：

$$
Specular=\frac{DFG}{4(N\cdot L)(N\cdot V)}
$$

两者都可以使用半程向量 H，但 Blinn-Phong 主要通过指数控制高光形状，PBR 则进一步考虑了微表面分布、反射率及遮挡等物理因素。

PBR 的最大优势不仅仅是画面更加真实，而是**材质参数在不同光照环境中的表现更加一致，降低了美术针对不同场景反复调整材质的成本。**

# **十六、PBR 性能优化方案**

对于大型动作游戏，尤其是同时支持 PC 和移动端的 UE 项目，PBR 的性能优化也很重要。

|优化方案|实现原理|
|---|---|
|BRDF 函数近似|使用 Schlick、Smith 等低成本数学近似|
|BRDF LUT|提前计算部分复杂积分|
|环境贴图预过滤|使用预计算 Mip 减少运行时积分|
|ORM 贴图合并|将 AO、Roughness、Metallic 打包到一张纹理|
|光源数量裁剪|减少每个像素需要计算的光源数量|
|Tiled / Clustered Lighting|按屏幕区域或空间集群筛选光源|
|Material LOD|远距离物体使用低复杂度材质|
|Half Precision|在适合的平台和计算中使用半精度|
|简化移动端光照|减少复杂反射、间接光照及特殊材质层|
|高光抗锯齿|缓解低粗糙度和高频法线导致的闪烁|

## **ORM 贴图合并**

ORM 是常见的纹理通道打包方案：

<img width="746" height="117" alt="image" src="https://github.com/user-attachments/assets/2acbd509-5f11-4cb6-b1e2-d6b5c8f0e804" />

```
// 一次纹理采样获取三个材质参数
float3 ORM = ORMTexture.Sample(Sampler, UV).rgb;

float AO        = ORM.r;
float Roughness = ORM.g;
float Metallic  = ORM.b;
```

可以减少纹理采样次数，合理设置压缩格式时也能降低纹理内存与带宽开销。

注意，这些是数值数据而不是颜色数据，因此 ORM 纹理通常需要关闭 sRGB。

# **十七、PBR 高频经典面试问题及答案**

## **1. 什么是 PBR？**

PBR 是一套基于物理规律的渲染方法，通过能量守恒、微表面理论、菲涅尔效应和 BRDF 模型模拟光线与材质的相互作用，使材质在不同光照环境下具有更加一致、合理的表现。

## **2. PBR 的核心原理有哪些？**

主要包括能量守恒、微表面理论、BRDF、菲涅尔效应、法线分布函数和几何遮挡函数。

实时渲染中通常还结合 IBL 等环境光照技术。

## **3. Cook-Torrance BRDF 中 D、F、G 分别是什么？**

D 是法线分布函数，描述符合反射方向的微表面分布密度。

F 是菲涅尔函数，描述不同角度下微表面的镜面反射率。

G 是几何函数，描述微表面之间的阴影遮挡和观察遮挡。

$$
f_{specular}=\frac{DFG}{4(N\cdot L)(N\cdot V)}
$$

## **4. 为什么粗糙度越大，高光越模糊？**

因为粗糙度越大，微表面法线的分布越分散，导致反射光向更多方向传播，从而使高光范围扩大，峰值通常降低，环境反射更加模糊。

## **5. 为什么金属没有普通漫反射？**

光进入金属后会受到自由电子等因素的影响，并在很短距离内快速衰减，难以像普通非金属那样通过内部散射重新射出。

因此常见金属度工作流将纯金属的普通漫反射近似为零，颜色主要由镜面反射决定。

## **6. 为什么非金属的 F0 通常为 0.04？**

因为很多普通非金属的折射率接近 1.5。

根据垂直入射的菲涅尔反射公式：

$$
F_0=\left(\frac{1-1.5}{1+1.5}\right)^2=0.04
$$

这是常用近似值，并不是所有非金属的实际反射率都等于 4%。

## **7. Metallic 可以设置为 0.5 吗？**

可以，但对于单一、均匀的普通材质，通常应该接近 0 或 1。

中间值主要用于材质混合、纹理边界过滤、金属与非金属在一个像素内的混合覆盖等情况。

例如生锈的铁：

- 裸露铁表面：Metallic 接近 1。
- 铁锈覆盖区域：Metallic 接近 0。
- 两者混合边界：可能存在中间值。

## **8. Roughness 和 Normal Map 有什么区别？**

Roughness 控制微观法线的统计分布，影响反射光的分散程度。

Normal Map 修改着色点使用的法线方向，改变局部光照方向。

两者都可以表现表面细节，但作用尺度和数学含义不同。

## **9. PBR 为什么要使用 HDR？**

PBR 的光照计算涉及实际或近似物理的光照强度，最终辐亮度可能远大于 1。

HDR（高动态范围）可以保留强光、高光以及多个光源叠加产生的高亮信息，再通过 Tone Mapping（色调映射）转换到显示设备能够表现的范围。

但 HDR 并不是 PBR 在数学上的必要定义条件。

## **10. PBR 为什么要在线性颜色空间中计算？**

光照叠加、能量分配、颜色乘法等运算应该在线性空间中进行。

sRGB 颜色经过非线性编码，直接在 sRGB 空间计算光照会导致亮度和颜色关系不正确。

因此通常需要：

|纹理类型|sRGB 设置|
|---|---|
|Base Color|开启（常规 sRGB 颜色纹理）|
|Emissive Color|视颜色资源编码而定|
|Normal Map|关闭|
|Metallic|关闭|
|Roughness|关闭|
|AO|关闭|
|ORM|关闭|

## **11. PBR 和延迟渲染是什么关系？**

PBR 是材质光照模型相关的渲染方法，延迟渲染是组织渲染流程的技术。

延迟渲染通过 GBuffer 存储材质参数，然后在光照阶段使用 PBR 模型进行计算。

前向渲染也可以使用 PBR，两者不存在必然绑定关系。

## **12. 为什么金属材质在某些场景中看起来是黑色的？**

因为纯金属普通漫反射近似为零，如果缺少环境反射和有效的直接镜面反射，金属就缺少足够的出射光，因此看起来很暗甚至是黑色。

应该检查天空光、反射环境、Lumen Reflections、光源配置等，而不是直接降低 Metallic 来掩盖问题。

## **13. G 函数和 AO 有什么区别？**

|对比项|G 函数|AO|
|---|---|---|
|作用尺度|微观表面|相对宏观的局部几何|
|主要影响|微表面镜面反射|间接光照可见性|
|主要输入|N、L、V、Roughness|几何信息或 AO 贴图|
|是否属于 Cook-Torrance BRDF|是|不是|

## **14. 为什么低粗糙度材质容易出现高光闪烁？**

低粗糙度对应非常尖锐的 GGX 分布。

当高频法线贴图、物体移动或摄像机移动导致 N·H 发生微小变化时，镜面反射强度可能剧烈变化，引起 Specular Aliasing（镜面反射走样）。

常用解决方案包括 Specular AA（高光抗锯齿）、根据法线方差增加有效粗糙度、合理设置法线贴图 Mipmap，以及结合时域抗锯齿。

## **15. PBR 是否完全符合现实物理规律？**

不是。

实时 PBR 为了满足游戏帧率要求，使用了大量物理近似，例如 Lambert 漫反射、Schlick 菲涅尔近似、RGB 光谱近似、单次微表面散射、Split-Sum 环境光积分近似等。

PBR 的目标是在可接受的计算开销下，获得符合物理规律且视觉可信的结果，并不是完全精确地模拟现实世界。

# **十八、PBR 核心知识总结**

<img width="736" height="695" alt="image" src="https://github.com/user-attachments/assets/9b82ca52-fe50-4a33-bad9-343ad86b8473" />

对于 UE 游戏客户端开发面试，建议按照以下优先级掌握：

|优先级|知识点|
|---|---|
|★★★★★|PBR 核心思想、能量守恒|
|★★★★★|微表面理论、粗糙度原理|
|★★★★★|Cook-Torrance BRDF、D/F/G|
|★★★★★|金属与非金属的区别|
|★★★★★|菲涅尔效应、F0|
|★★★★☆|IBL、Split-Sum、BRDF LUT|
|★★★★☆|PBR 在 UE 渲染管线中的位置|
|★★★★☆|PBR 与传统光照模型的区别|
|★★★★☆|PBR 性能优化与高光走样|
|★★★☆☆|渲染方程的完整数学推导|

最重要的是理解：**PBR 的本质，是通过微表面理论描述材质表面的统计特征，通过 BRDF 描述光线的反射行为，并结合能量守恒等物理规律，计算最终进入摄像机的光线。**
