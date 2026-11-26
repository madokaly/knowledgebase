# **NPR 卡通渲染（Non-Photorealistic Rendering）原理详解**

NPR（Non-Photorealistic Rendering，非真实感渲染）是一种不以还原真实世界的物理光照为主要目标，而是通过人为控制光照、阴影、颜色、轮廓线、材质等效果，实现卡通、动漫、水墨、油画等艺术风格的渲染技术。

其中，**Toon Shading（卡通着色）是 NPR 最常见的实现方式之一**。

对于《鸣潮》《原神》《崩坏：星穹铁道》等二次元游戏，角色渲染通常会采用风格化渲染技术，并结合部分 PBR（基于物理的渲染）技术，使角色既具有动漫风格，又拥有一定的空间感、材质质感和环境光照表现。

下面重点从图形学底层原理、Shader 实现、UE5 工程应用以及游戏客户端高频面试问题几个方面进行讲解。

# **一、NPR 与 PBR 的区别**

![[Pasted image 20261126233035.png]]

**PBR（基于物理的渲染）**

连续的光照过渡、真实的材质与反射

![[Pasted image 20261126233047.png]]

**NPR（卡通风格渲染）**

离散的明暗分层、风格化阴影与描边

|对比项|PBR 渲染|NPR 卡通渲染|
|---|---|---|
|核心目标|模拟真实世界的光照|实现指定的艺术风格|
|漫反射|连续变化|通常分段变化|
|阴影|追求物理合理性|强调形状、颜色与可控性|
|高光|基于物理 BRDF|可自定义高光形状|
|能量守恒|通常需要遵循|可以主动打破|
|描边|通常不需要|常用轮廓线增强表现|
|法线|尽量表达真实几何细节|可以人为修改|
|面部光照|根据法线计算|常结合 SDF 等特殊方案|
|美术可控性|受物理模型约束|非常高|

需要注意：**NPR 和 PBR 并不是互斥的。**

例如，一个二次元角色可以同时使用：

- 皮肤：NPR 二分色光照。
- 面部：SDF 面部阴影。
- 头发：风格化各向异性高光。
- 衣服：Ramp 光照与自定义阴影颜色。
- 金属饰品：PBR 金属反射。
- 武器：PBR 与 NPR 混合渲染。
- 环境：PBR 光照与风格化后处理。

这种混合渲染方案能够兼顾角色的动漫表现和场景的空间真实感。

# **二、NPR 卡通渲染的核心技术**

在大型二次元动作游戏中，角色 NPR 渲染通常由以下技术组成。

|核心技术|中文解释|重要程度|
|---|---|---|
|Cel Shading|卡通分层着色|★★★★★|
|Ramp Shading|渐变贴图光照|★★★★★|
|SDF Face Shadow|SDF 面部阴影|★★★★★|
|Outline|角色描边|★★★★★|
|Custom Normal|自定义法线|★★★★★|
|Stylized Specular|风格化高光|★★★★|
|Rim Light|边缘光、轮廓光|★★★★|
|Shadow Mask|阴影控制遮罩|★★★★|
|Hair Shadow|头发对面部投影|★★★★|
|Anisotropic Highlight|各向异性高光|★★★★|
|Stylized PBR|风格化物理渲染|★★★★|
|Post Processing|风格化后处理|★★★|

整体渲染流程可以概括为：

![[局部截取_20261126_233111.png]]

这是一个概念流程，不同渲染管线的实际 Pass 顺序可能有所不同。

# **三、Cel Shading（卡通分层着色）**

## **3.1 基本原理**

Cel Shading 是 NPR 卡通渲染最基础、最核心的技术。

传统漫反射使用 Lambert（兰伯特漫反射）光照模型：

$$
Diffuse=C_d \cdot C_l\cdot \max(0,N\cdot L)
$$

其中：

$$
N，表面单位法线方向
$$

$$
L，从表面指向光源的单位方向
$$

$$
N\cdot L，法线与光照方向的点积
$$

$$
C_d，材质漫反射颜色
$$

$$
C_l，光源颜色及强度
$$

由于：

$$
N\cdot L=\cos\theta
$$

所以法线与光源方向越接近，表面就越亮。

![[局部截取_20261126_233312.png]]

Lambert 的问题在于，光照强度会连续变化：

```
// N：世界空间法线
// L：指向光源的方向
float NdotL = saturate(dot(N, L));

// 连续的漫反射光照
float3 Diffuse = BaseColor * LightColor * NdotL;
```

而动漫角色通常没有这么连续的明暗过渡，而是存在明显的亮面与暗面分界线。

因此，Cel Shading 的核心操作就是：

**将连续的光照强度离散化，划分为有限数量的明暗区域。**

## **3.2 二分色光照（Two-Tone Shading）**

最简单的方式是使用 Step（阶跃函数）。

$$
S= \begin{cases} 1,&N\cdot L\geq T\\ 0,&N\cdot L<T \end{cases}
$$

其中 T 是光照阈值（Threshold）。

```
float NdotL = dot(N, L);

// 光照阈值
float Threshold = 0.3;

// 大于等于阈值返回1，否则返回0
float ShadowMask = step(Threshold, NdotL);

// 根据明暗区域选择颜色
float3 FinalColor = lerp(
    ShadowColor,    // 暗部颜色
    LightColor,     // 亮部颜色
    ShadowMask
);

FinalColor *= BaseColor;
```

下面可以直接调整光照阈值，观察明暗分界线如何变化。

![[局部截取_20261126_233437.png]]

可以看到：
- Threshold 越大，暗部区域越大。
- Threshold 越小，亮部区域越大。
- 明暗过渡宽度越小，卡通分界线越明显。

## **3.3 多分色光照（Multi-Tone Shading）**

二分色只有亮部和暗部。如果想让角色具备更强的立体感，可以增加第三层甚至第四层颜色。

![[局部截取_20261126_233457.png]]

```
float NdotL = dot(N, L);

// 三分色光照
float3 Color = ShadowColor;

// 中间色区域
Color = lerp(
    Color,
    MidColor,
    step(0.1, NdotL)
);

// 高亮区域
Color = lerp(
    Color,
    BrightColor,
    step(0.6, NdotL)
);

return BaseColor * Color;
```

分层越多并不一定越好。对于二次元角色来说，分层过多可能会失去干净、明确的动漫明暗关系。

## **3.4 SmoothStep（平滑阶跃函数）**

直接使用 Step 会导致明暗交界处过于锐利，并且随着角色运动、摄像机移动，容易产生锯齿和闪烁。

因此实际项目中，经常使用 SmoothStep 替代硬阶跃。

```
float NdotL = dot(N, L);

// 分界线位置
float Threshold = 0.3;

// 过渡区域宽度
float Softness = 0.02;

float Mask = smoothstep(
    Threshold - Softness,
    Threshold + Softness,
    NdotL
);

float3 Color = lerp(ShadowColor, BrightColor, Mask);
```

SmoothStep 使用三次 Hermite 插值：

$$
t=\operatorname{saturate} \left(\frac{x-a}{b-a}\right)
$$

$$
SmoothStep=t^2(3-2t)
$$

也可以使用 `fwidth` 根据相邻像素的变化率动态调整过渡宽度：

```
// 根据屏幕空间变化率估算抗锯齿宽度
float Width = max(fwidth(NdotL), 0.0001);

float Mask = smoothstep(
    Threshold - Width,
    Threshold + Width,
    NdotL
);
```

这样既能保留卡通渲染的分层效果，又能减轻明暗交界处的走样。

## **3.5 Half Lambert（半兰伯特光照）**

传统 Lambert 会把所有背光区域截断为 0。

Half Lambert 常用的重映射方式是：

$$
H=\frac{N\cdot L+1}{2}
$$

```
float NdotL = dot(N, L);

// 将 [-1, 1] 映射为 [0, 1]
float HalfLambert = NdotL * 0.5 + 0.5;

// 某些 Half Lambert 变体还会进行平方
// HalfLambert *= HalfLambert;
```

它可以保留背光区域的方向信息，并将整个点积范围映射到 0～1，方便后续采样 Ramp 贴图。

注意：Half Lambert 本身仍然是连续光照，**它不等于 Cel Shading**，通常需要进一步使用 Step、SmoothStep 或 Ramp 进行风格化处理。

# **四、Ramp Shading（渐变贴图光照）**

## **4.1 为什么需要 Ramp 贴图？**

直接在 Shader 中编写多个 Step 函数存在以下问题：

- 光照层数、阈值和颜色需要通过参数或者代码控制。
- 不同角色、皮肤、服装的阴影颜色存在差异。
- 美术人员难以自由设计复杂的色彩过渡。
- 难以统一管理不同材质的光照风格。

因此，大型项目经常使用 Ramp Texture（光照渐变贴图）。

Ramp 的本质是 **LUT（Look-Up Table，查找表）**。

它并不是直接根据模型 UV 采样，而是将光照计算结果作为贴图采样坐标。

![[局部截取_20261126_233540.png]]

## **4.2 实现原理**

首先计算法线与光源方向的点积，并将其映射到 0～1。

$$
u=\frac{N\cdot L+1}{2}
$$

然后使用该数值采样 Ramp 贴图。

```
float NdotL = dot(N, L);

// 将光照结果映射为贴图U坐标
float RampU = saturate(NdotL * 0.5 + 0.5);

// 采样光照渐变贴图
float3 RampColor = RampTexture.Sample(
    RampSampler,
    float2(RampU, 0.5)
).rgb;

// 最终漫反射颜色
float3 Diffuse = BaseColor * RampColor;
```

![[局部截取_20261126_233603.png]]

## **4.3 多行 Ramp 贴图**

实际项目中，可以在一张 Ramp 贴图中存储多种材质的光照风格。

![[局部截取_20261126_233622.png]]

通过材质 ID 或 Mask 贴图来决定采样哪一行。

```
float RampU = saturate(dot(N, L) * 0.5 + 0.5);

// 材质类型：0皮肤，1头发，2布料……
float MaterialID = 1;

// Ramp贴图中共有5行光照配置
float RowCount = 5;

// 采样对应行的中心位置
float RampV = (MaterialID + 0.5) / RowCount;

float3 RampColor = RampTexture.Sample(
    RampSampler,
    float2(RampU, RampV)
).rgb;
```

实际工程中需要特别注意纹理过滤、MipMap 和不同行之间的颜色串扰，可以通过增加行间填充、控制采样层级等方式解决。

Ramp Shading 最大的优势是：**将光照风格设计从程序逻辑转移到美术资源中。**

# **五、SDF Face Shadow（SDF 面部阴影）**

这是二次元角色 NPR 渲染中非常重要的一项技术，也是面试中经常被问到的技术点。

## **5.1 为什么面部不能直接使用 NdotL？**

人体面部存在很多复杂的几何结构，例如：

- 鼻梁、鼻翼。
- 眼窝、眉骨。
- 嘴唇、下巴。
- 脸颊。

这些结构会导致面部法线方向变化剧烈。

如果直接使用 NdotL 进行二分色光照，就可能产生大量零碎、不规则的阴影。

![[Pasted image 20261126233643.png]]

**普通法线光照**
容易出现复杂、破碎的面部阴影

![[Pasted image 20261126233653.png]]

**SDF 面部光照**
明暗形状更加稳定且可控


而动漫人物的面部阴影通常由美术人员按照特定规律绘制，并不完全符合真实几何光照。

因此，需要一种可以控制面部阴影形状，同时能够随着光源方向动态变化的技术。

这就是 SDF Face Shadow。

## **5.2 什么是 SDF？**

SDF（Signed Distance Field，有符号距离场）表示某个位置到最近边界的距离，并通过正负号区分边界内外。

$$
SDF(P)= \begin{cases} -d(P,\partial\Omega),&P\text{ 在内部}\\ 0,&P\text{ 在边界上}\\ +d(P,\partial\Omega),&P\text{ 在外部} \end{cases}
$$

但是，游戏中俗称的 SDF 面部阴影贴图，并不一定是严格数学意义上的单一距离场。

它通常由多个不同光照角度下的面部阴影 Mask，通过距离场转换、插值、合成等方式生成。

最终贴图可以理解为：

**每个面部像素保存一个光照方向阈值，决定这个位置在什么光照角度下进入亮部或者暗部。**

## **5.3 SDF 面部阴影贴图如何制作？**

![[Pasted image 20261126233740.png]]


一种常见制作流程：

1. 美术绘制正面光照下的面部阴影 Mask。
2. 绘制斜前方光照下的阴影 Mask。
3. 绘制侧面光照下的阴影 Mask。
4. 绘制斜后方及背面光照下的阴影 Mask。
5. 将各个 Mask 转换为距离场并进行插值、合成。
6. 生成最终的灰度面部阴影贴图。

Shader 根据光源方向与灰度贴图进行比较，就可以还原不同角度下的阴影区域。

## **5.4 如何计算光源相对于面部的方向？**

首先建立角色头部的局部坐标系。

- Forward：头部正前方。
- Right：头部右方向。
- Up：头部上方向。
- LightDir：指向光源的方向。

UE 默认坐标系通常是 X 轴向前、Y 轴向右、Z 轴向上，但头部骨骼的局部轴需要根据具体模型确认。

![[局部截取_20261126_233814.png]]

通常只考虑光照方向在头部水平面上的投影，避免光源高度变化导致面部阴影形状失控。

## **5.5 SDF 面部阴影 Shader 实现**

以下假设 SDF 贴图的灰度值越大，表示该位置能够在更大的偏转角度下保持亮部。

```
// 头部坐标系，要求为单位向量
float3 F = FaceForward;
float3 R = FaceRight;
float3 U = FaceUp;

// 光照方向：从表面指向光源
float3 L = normalize(LightDirection);

// 将光照投影到头部水平面
float3 ProjectedL = L - dot(L, U) * U;

// 避免光源正上方或正下方时归一化零向量
float LenSq = dot(ProjectedL, ProjectedL);

float3 HorizontalL = LenSq > 0.000001
    ? ProjectedL * rsqrt(max(LenSq, 0.000001))
    : F;

// 判断光源来自左侧还是右侧
float Side = dot(R, HorizontalL);

// 计算正前方与光源的夹角
float CosAngle = clamp(
    dot(F, HorizontalL),
    -1.0,
    1.0
);

// 将0~180度映射到0~1
float Threshold = acos(CosAngle) / 3.14159265;

// 根据光源左右方向镜像UV
float2 FaceUV = UV;

if (Side < 0)
{
    FaceUV.x = 1.0 - FaceUV.x;
}

// 采样SDF面部阴影贴图
float SDF = FaceShadowTexture.Sample(
    FaceSampler,
    FaceUV
).r;

// 根据贴图和光源角度决定明暗区域
float Mask = smoothstep(
    Threshold - 0.01,
    Threshold + 0.01,
    SDF
);

// 混合面部亮色和阴影色
float3 FaceColor = lerp(
    FaceShadowColor,
    FaceBrightColor,
    Mask
);
```

这里的比较方向、UV 镜像规则以及阈值公式必须与贴图的制作约定一致。

如果贴图是按照余弦值编码，也可以直接使用点积重映射，避免 `acos` 运算。

## **5.6 SDF 面部阴影的优缺点**

|优点|缺点|
|---|---|
|阴影形状完全可控|需要额外制作面部贴图|
|不依赖复杂面部法线|不完全符合真实光照|
|阴影过渡稳定|垂直方向光照需要额外处理|
|能够实现动漫风格的面部阴影|不对称面部可能需要独立左右贴图|
|运行时计算成本相对较低|无法独立解决其他物体遮挡|

**SDF 面部阴影不能替代 Shadow Map。**

SDF 主要控制面部自身的风格化明暗分布；Shadow Map 则负责头发、武器、建筑等几何体遮挡光源所产生的真实投影。

# **六、Outline（描边技术）**

描边是区分普通 3D 渲染与卡通渲染的重要视觉特征。

目前常用的描边实现主要有两种：

|方案|中文名称|实现位置|
|---|---|---|
|Inverted Hull|反向外壳描边、背面扩张描边|模型空间|
|Screen Space Outline|屏幕空间描边|屏幕空间后处理|

## **6.1 Inverted Hull（反向外壳描边）**

![[局部截取_20261126_233843.png]]

基本原理：

1. 再绘制一次角色模型，作为描边 Pass。
2. 在顶点着色器中，让模型顶点沿法线方向向外扩张。
3. 剔除正面三角形，只渲染扩张模型的背面。
4. 使用深色材质绘制。
5. 扩张模型的中间区域被原始模型遮挡，只有边缘露出，形成描边。

该方法也被《罪恶装备 Xrd》的公开技术分享采用。

顶点扩张公式：

$$
P'=P+N\cdot Width
$$

```
// 世界空间顶点位置
float3 PositionWS = Input.PositionWS;

// 描边使用的平滑法线
float3 NormalWS = normalize(Input.OutlineNormalWS);

// 描边宽度
float OutlineWidth = 0.02;

// 沿法线向外扩张
PositionWS += NormalWS * OutlineWidth;

// 转换到裁剪空间
Output.PositionCS = mul(
    ViewProjectionMatrix,
    float4(PositionWS, 1.0)
);
```

描边 Pass 的主要渲染状态：

```
Cull Front     // 剔除正面，保留背面
Depth Test     // 开启深度测试
OutlineColor   // 描边颜色
```

深度写入是否开启，需要根据描边 Pass 的绘制顺序、遮挡关系及后续渲染需求决定。

## **6.2 反向外壳描边存在的问题**

### **问题一：硬边处描边断裂**

模型的硬边、UV 接缝等位置可能存在拆分顶点，不同顶点的法线方向不一致。

沿各自法线扩张后，就可能出现描边裂缝。

![[局部截取_20261126_233909.png]]

解决方案：为描边单独生成 Smooth Normal（平滑法线），将其存储在额外 UV、顶点颜色或其他顶点属性中。

这样：

- 光照使用原始法线或美术修改后的法线。
- 描边扩张使用独立的平滑法线。

### **问题二：描边近粗远细**

如果在世界空间中固定扩张 1cm，那么角色距离摄像机越远，屏幕上的描边就越细。

解决方案包括：

- 根据相机距离动态调整扩张宽度。
- 根据投影矩阵计算屏幕空间宽度。
- 直接在裁剪空间中沿投影后的描边方向进行偏移。

例如，假设已经计算出归一化的屏幕像素方向 `OutlineDir`：

```
// 希望描边固定为2像素
float WidthPixels = 2.0;

// 将像素偏移转换为NDC偏移
float2 OffsetNDC =
    OutlineDir * WidthPixels * 2.0 / ScreenSize;

// 转换为裁剪空间偏移
PositionCS.xy += OffsetNDC * PositionCS.w;
```

该方法可以让描边宽度在不同观察距离下更加稳定。

### **问题三：所有部位描边一样粗**

可以通过顶点颜色控制不同部位的描边宽度：

```
float Width = OutlineWidth * VertexColor.r;
```

例如脸部描边较细、身体外轮廓较粗、细小饰品不描边。

## **6.3 Screen Space Outline（屏幕空间描边）**

屏幕空间描边不需要对原始模型进行顶点扩张，而是根据屏幕空间中相邻像素的数据差异检测边缘。

常用的数据有：

- Depth Buffer（深度缓冲）。
- Normal Buffer（法线缓冲）。
- Custom Depth（自定义深度）。
- Custom Stencil（自定义模板值）。
- Object ID（物体标识）。

![[Pasted image 20261126233938.png]]

Depth 深度缓冲

![[Pasted image 20261126233944.png]]

Normal 法线缓冲

基本原理：

如果当前像素与相邻像素之间的深度差异较大，可能表示物体轮廓。

如果相邻像素的法线差异较大，可能表示物体内部存在折角或几何边缘。

```
// 采样当前像素深度
float CenterDepth = SampleDepth(UV);

// 采样相邻像素深度
float RightDepth = SampleDepth(UV + float2(TexelSize.x, 0));
float UpDepth = SampleDepth(UV + float2(0, TexelSize.y));

// 深度边缘检测
float DepthEdge = max(
    abs(CenterDepth - RightDepth),
    abs(CenterDepth - UpDepth)
);

// 法线边缘检测
float3 N0 = SampleNormal(UV);
float3 N1 = SampleNormal(UV + float2(TexelSize.x, 0));

float NormalEdge = 1.0 - dot(N0, N1);

// 判断是否需要描边
float Edge = max(
    step(DepthThreshold, DepthEdge),
    step(NormalThreshold, NormalEdge)
);

// 混合描边颜色
float3 FinalColor = lerp(
    SceneColor,
    OutlineColor,
    Edge
);
```

实际项目一般还会使用线性深度、相对深度阈值以及更多方向的采样。

常见边缘检测算法还有 Sobel（索贝尔算子）：

**水平方向梯度**

$$
G_x=\begin{bmatrix}-1&0&1\\-2&0&2\\-1&0&1\end{bmatrix}
$$

**垂直方向梯度**

$$
G_y=\begin{bmatrix}-1&-2&-1\\0&0&0\\1&2&1\end{bmatrix}
$$

通过卷积计算两个方向的梯度，然后得到边缘强度：

$$
Edge=\sqrt{G_x^2+G_y^2}
$$

## **6.4 两种描边方案对比**

|对比项|反向外壳描边|屏幕空间描边|
|---|---|---|
|实现原理|模型扩张、背面绘制|深度和法线边缘检测|
|是否增加几何绘制|通常需要|不需要额外绘制扩张模型|
|性能主要开销|顶点处理、Draw Call、Overdraw|屏幕像素与纹理采样|
|描边宽度|需要特殊处理|容易保持像素宽度|
|模型内部边缘|不擅长一般内部边缘|比较容易检测|
|细节控制|顶点级控制方便|依赖缓冲数据和 Mask|
|常见问题|裂缝、穿插、宽度变化|噪点、误描边、画面闪烁|
|适用场景|角色外轮廓|场景描边、交互高亮、内部细节|

商业项目可以同时使用两种方式，发挥各自的优势。

# **七、Custom Normal（自定义法线）**

## **7.1 为什么 NPR 要修改法线？**

在 PBR 中，法线主要用于尽可能准确地表达模型表面的几何朝向。

而在 NPR 中，法线还承担着控制阴影形状的作用。

因为：

$$
N\cdot L
$$

直接决定了亮部和暗部的划分。

如果模型表面的法线变化十分复杂，那么卡通阴影的形状也会非常复杂。

![[局部截取_20261126_234030.png]]

因此，NPR 可以主动修改模型法线，让光照计算产生更符合美术要求的阴影形状。

## **7.2 常见修改方式**

|方式|原理|
|---|---|
|Smooth Normal|将相邻顶点法线进行平滑处理|
|Normal Transfer|从代理模型传递法线到原始模型|
|Spherical Normal|使用近似球面的法线方向|
|Custom Vertex Normal|美术手动编辑顶点法线|
|Normal Blending|在 Shader 中混合真实法线与风格化法线|

例如，将角色头部近似为一个球体：

```
// 根据球心计算近似球面法线
float3 SphereNormal = normalize(
    PositionWS - HeadCenterWS
);

// 混合真实法线与球面法线
float3 FinalNormal = normalize(
    lerp(
        OriginalNormal,
        SphereNormal,
        NormalBlendWeight
    )
);
```

这种方案对于头发、脸颊等区域，可以减少过多的几何明暗细节。

在《罪恶装备 Xrd》的公开技术分享中，美术可控的模型法线也是其实现二维动画风格的重要组成部分。

gdcvault.com+1

一个角色可以拥有多套不同用途的法线：

|法线类型|作用|
|---|---|
|Geometric Normal|几何表面方向|
|Shading Normal|漫反射与高光计算|
|Outline Normal|描边顶点扩张|
|Stylized Normal|控制特定区域的卡通阴影|

**法线的修改只改变参与计算的方向，不会直接修改模型的几何形状与碰撞体。**

# **八、Stylized Specular（风格化高光）**

## **8.1 传统高光计算**

以 Blinn-Phong 高光模型为例：

$$
H=\frac{L+V}{|L+V|}
$$

$$
Specular=K_s\max(0,N\cdot H)^n
$$

其中：

- L：光源方向。
- V：观察方向。
- H：半程向量（Half Vector）。
- N：表面法线。
- Ks：高光强度。
- n：高光指数，越大高光越集中。

传统高光是连续衰减的，而动漫中的高光可能是明确的白色块、条形或者星形。
![[局部截取_20261126_234153.png]]
## **8.2 卡通高光实现**

和漫反射一样，可以对高光进行离散化处理。

```
float3 H = normalize(L + V);

float NdotH = saturate(dot(N, H));

// 传统Blinn-Phong高光
float Specular = pow(NdotH, SpecularPower);

// 对高光进行卡通分层
float ToonSpecular = smoothstep(
    SpecularThreshold - 0.01,
    SpecularThreshold + 0.01,
    Specular
);

// 通过贴图控制哪些区域允许出现高光
ToonSpecular *= SpecularMask;

float3 SpecularColor =
    ToonSpecular * HighlightColor * SpecularIntensity;
```

也可以直接对 NdotH 使用阈值，以减少不必要的幂运算。

通过 Specular Mask（高光遮罩）控制皮肤、头发、布料、金属等不同材质的高光表现。

# **九、Anisotropic Highlight（各向异性高光）**

## **9.1 为什么头发高光需要特殊处理？**

动漫角色头发的高光通常不是一个圆形光斑，而是沿着发丝分布形成带状高光。

![[Pasted image 20261126234248.png]]

这是因为头发由大量具有方向性的发丝组成，不同方向上的光照反射表现并不相同。

这种现象称为 Anisotropy（各向异性）。

## **9.2 Kajiya-Kay 头发高光模型**

普通高光主要依赖法线 N，而 Kajiya-Kay 的经典发丝高光形式主要使用发丝切线 T。

$$
\sin\theta=\sqrt{1-(T\cdot H)^2}
$$

$$
Specular=(\sin\theta)^n
$$

```
// 发丝方向
float3 T = normalize(HairTangent);

// 半程向量
float3 H = normalize(L + V);

float TdotH = dot(T, H);

// 计算发丝切线与半程向量夹角的正弦值
float SinTH = sqrt(
    saturate(1.0 - TdotH * TdotH)
);

// 发丝高光
float Specular = pow(SinTH, HairSpecularPower);

// 将头发高光风格化为明显的条带
float ToonHighlight = smoothstep(
    0.45,
    0.55,
    Specular
);
```

实际项目还可以结合 Tangent Shift（切线偏移）、高光 Mask、手绘高光贴图等方式实现双层高光、彩色高光。

对于动漫角色来说，头发高光并不一定要遵循真实的发丝散射模型，只需要达到稳定且符合角色设计的视觉效果。

# **十、Rim Light（边缘光）**

## **10.1 基于 Fresnel 的边缘光**

边缘光能够增强角色轮廓，使角色从复杂场景中更加突出。

一种最常见的实现方式是利用法线与观察方向的点积：

$$
Rim=(1-\max(0,N\cdot V))^n
$$

当表面朝向摄像机时，N·V 接近 1，边缘光较弱。

当表面接近垂直于观察方向时，N·V 接近 0，边缘光较强。

![[局部截取_20261126_234324.png]]

```
float NdotV = saturate(dot(N, V));

// 基于视角的边缘光
float Rim = pow(1.0 - NdotV, RimPower);

// 卡通化边缘光
Rim = smoothstep(
    RimThreshold - 0.02,
    RimThreshold + 0.02,
    Rim
);

// 可以额外使用光照方向限制边缘光区域
Rim *= saturate(dot(N, L));

float3 RimColor = Rim * RimLightColor * RimIntensity;
```

这种方式也常被称为 Fresnel-like（类菲涅尔）边缘光，但不等同于严格的物理 Fresnel 反射公式。

## **10.2 Screen Space Rim Light（屏幕空间边缘光）**

Fresnel 边缘光可能在模型内部的曲面产生多余的亮线，并且光带宽度不稳定。

因此还可以使用深度缓冲实现屏幕空间边缘光。

![[局部截取_20261126_234346.png]]

实现步骤：

1. 获取当前像素的屏幕位置和线性深度。
2. 沿投影后的法线方向偏移一定像素。
3. 采样偏移位置的深度。
4. 如果深度差超过阈值，说明可能处于角色边缘。
5. 使用边缘光颜色对当前像素进行着色。

```
float CenterDepth = SampleLinearDepth(UV);

// NormalSS：投影到屏幕空间的法线方向
float2 OffsetUV =
    UV + NormalSS * RimWidthPixels / ScreenSize;

float OffsetDepth = SampleLinearDepth(OffsetUV);

// 假设线性深度越大代表离相机越远
float DepthDiff = OffsetDepth - CenterDepth;

// 根据深度差判断是否处于边缘
float RimMask = smoothstep(
    DepthThreshold,
    DepthThreshold + Softness,
    DepthDiff
);
```

这种方法可以更直接地控制边缘光的屏幕像素宽度，但会增加深度纹理采样开销。

# **十一、Shadow Mask（阴影遮罩）与实时阴影**

## **11.1 Shadow Mask 阴影遮罩**

有些部位即使正对光源，也希望保持一定的阴影，例如头发根部、衣服褶皱、下巴和脖子的交界处。

可以通过 Shadow Mask 控制。

```
float NdotL = dot(N, L);

// 采样美术绘制的阴影控制贴图
float ShadowMask = MaskTexture.Sample(
    MaskSampler, UV
).r;

// 根据遮罩调整光照阈值
float Threshold = BaseThreshold
                + (1.0 - ShadowMask) * ShadowStrength;

// 计算最终明暗分层
float ToonMask = smoothstep(
    Threshold - Softness,
    Threshold + Softness,
    NdotL
);
```

这里 ShadowMask 越小，越容易进入阴影区域。

实际项目经常采用 Channel Packing（纹理通道打包），将多种控制数据存放在同一张 RGBA 贴图中。

例如：

|通道|示例用途|
|---|---|
|R|高光强度 Mask|
|G|阴影控制 Mask|
|B|材质类型 ID|
|A|边缘光或描边 Mask|

这样可以减少纹理资源数量以及采样次数。具体通道定义由项目自身决定，并没有统一标准。

## **11.2 NPR 如何结合 Shadow Map？**

Cel Shading 和 Shadow Map 解决的是两个不同的问题。

|技术|解决的问题|
|---|---|
|Cel Shading|表面朝向光源时应该有多亮|
|Shadow Map|光源到表面之间是否存在遮挡|

例如，角色面部正对光源，NdotL 很大，但如果头发挡住了光线，那么面部依然应该产生阴影。

![[局部截取_20261126_234415.png]]

可以使用 Shadow Map 得到 Shadow Attenuation（阴影衰减值）。

```
// 卡通明暗分层
float ToonMask = smoothstep(
    Threshold - Softness,
    Threshold + Softness,
    dot(N, L)
);

// Shadow Map计算结果
// 0：完全被遮挡
// 1：完全可见光源
float RealTimeShadow = GetShadowAttenuation();

// 综合表面明暗和几何遮挡
float FinalLightMask = ToonMask * RealTimeShadow;

// 也可以按照艺术需求采用min等组合方式
// float FinalLightMask = min(ToonMask, RealTimeShadow);

float3 Color = lerp(
    ShadowColor,
    BrightColor,
    FinalLightMask
);
```

这样就能同时实现卡通分层阴影以及动态几何遮挡。

## **11.3 Hair Shadow（刘海阴影）**

对于二次元角色，刘海投射到脸部的阴影需要特别控制。

常见方案包括：

|方案|原理|
|---|---|
|普通 Shadow Map|直接计算头发对面部的投影|
|Per-Object Shadow|使用角色专属的高精度阴影|
|Screen Space Hair Shadow|利用屏幕空间深度、模板值计算|
|Fake Hair Shadow|使用额外几何体或贴图模拟|

屏幕空间刘海阴影的一种实现思路：

1. 将面部和头发分别进行标记。
2. 获取头发的屏幕空间深度或遮罩。
3. 根据光照方向偏移采样头发数据。
4. 判断偏移区域是否覆盖面部。
5. 仅在面部区域混合刘海阴影颜色。

这种方案具有很强的美术可控性，但并不是严格的三维阴影计算，可能受到屏幕边缘、视角和遮挡变化的影响。

# **十二、NPR 与 PBR 混合渲染**

在大型二次元动作游戏中，通常不需要将整个游戏都使用纯 NPR 方式渲染。

一个角色的不同材质可以采用不同的光照算法。
![[Pasted image 20261126234440.png]]

**面部**

SDF 阴影、固定阴影色、刘海阴影

**头发**

Ramp 光照、各向异性高光、平滑法线

**皮肤**

二分色光照、风格化高光

**衣服**

Ramp 光照、法线贴图、材质遮罩

**金属与饰品**

GGX 高光、环境反射、金属度和粗糙度

最终颜色的概念公式可以表示为：

$$
C_{final}= C_{diffuse}+C_{specular} +C_{indirect}+C_{rim}+C_{emissive}
$$

其中漫反射、高光、间接光等分量都可以进行不同程度的风格化处理。

例如：

```
// NPR卡通漫反射
float3 NPRDiffuse = CalculateToonDiffuse();

// PBR漫反射
float3 PBRDiffuse = CalculatePBRDiffuse();

// 根据材质类型混合两种漫反射
float3 Diffuse = lerp(
    NPRDiffuse,
    PBRDiffuse,
    PBRWeight
);

// 混合风格化高光和PBR高光
float3 Specular = lerp(
    ToonSpecular,
    GGXSpecular,
    PBRWeight
);

// 最终HDR颜色
float3 FinalColor =
      Diffuse
    + Specular
    + IndirectLighting
    + RimLighting
    + Emissive;
```

这里是概念性示例，并非完整的物理 BRDF，需要根据项目要求处理能量守恒、金属度、Fresnel、光照衰减等因素。

# **十三、UE5 中如何实现 NPR 卡通渲染？**

对于 UE 游戏客户端开发，需要重点了解以下三种实现方案。

|实现方案|难度|灵活性|是否需要修改引擎|
|---|---|---|---|
|Post Process Material|低|较低|否|
|Unlit + 自定义光照|中|较高|否|
|Custom Shading Model|高|高|传统实现通常需要|

## **13.1 方案一：Post Process Material（后处理材质）**

使用 UE 的后处理材质，对已经渲染完成的画面进行风格化处理。

![[局部截取_20261126_234518.png]]


UE 中常用的材质节点和功能：

|节点或功能|作用|
|---|---|
|SceneTexture|获取场景相关缓冲数据|
|PostProcessInput0|获取当前后处理输入颜色|
|SceneDepth|获取场景深度|
|WorldNormal|获取世界空间法线，取决于管线和位置|
|CustomDepth|获取指定物体的自定义深度|
|CustomStencil|区分不同角色或物体|
|Post Process Volume|控制后处理生效范围|

UE 官方支持使用后处理材质以及 Custom Depth、Custom Stencil 实现物体分类和屏幕空间特效。

优点是实现简单、无需修改引擎。

缺点是后处理阶段通常无法准确分离原始漫反射、高光、环境光以及不同光源的贡献，因此不适合实现高度可控的角色级 NPR 光照。

## **13.2 方案二：Unlit Material + 自定义光照**

将材质的 Shading Model 设置为 Unlit（无光照），然后在材质中自行计算 NPR 光照，并输出到 Emissive Color。

![[局部截取_20261126_234544.png]]

光源方向可以通过 MPC（Material Parameter Collection，材质参数集合）传入。

UE C++ 示例：

```
// 获取MPC实例
UMaterialParameterCollectionInstance* MPCInstance =
    GetWorld()->GetParameterCollectionInstance(MPC);

// 获取方向光的Actor朝向
// 取反得到从物体指向光源的方向
FVector LightDir = -DirectionalLight->GetActorForwardVector();

if (MPCInstance)
{
    // 将光源方向传递给材质
    MPCInstance->SetVectorParameterValue(
        TEXT("MainLightDirection"),
        FLinearColor(
            LightDir.X,
            LightDir.Y,
            LightDir.Z,
            0.0f
        )
    );
}
```

角色头部朝向等独立数据可以通过 MID（Dynamic Material Instance，动态材质实例）或 Custom Primitive Data 等方式传入。

优点：

- 无需修改引擎源码。
- 可以自由实现 Ramp、SDF、高光等算法。
- 适合快速原型验证与固定主光源的游戏。

缺点：

- 不会自动获得 UE 默认 Lit 材质的完整动态光照结果。
- 实时阴影、局部光源、间接光照需要额外设计。
- 普通 Surface Material 的 Custom HLSL 节点不能随意访问延迟光照阶段的逐光源数据。

## **13.3 方案三：Custom Shading Model（自定义着色模型）**

这是需要深入 UE 渲染管线时重点关注的方案。

对于复杂的 NPR 项目，可以扩展引擎着色模型，让角色直接在渲染管线的光照阶段执行自定义的 NPR BRDF。

以传统 UE 延迟渲染管线为例：

![[局部截取_20261126_234607.png]]

传统非 Substrate 渲染路径下，新增一个 Toon Shading Model 通常涉及以下模块：

|引擎文件或模块|主要作用|
|---|---|
|EngineTypes.h|定义材质着色模型枚举|
|ShadingCommon.ush|定义 Shader 中的着色模型标识|
|ShadingModels.ush|实现自定义 BRDF 光照计算|
|ShadingModelsMaterial.ush|处理材质相关的着色模型数据|
|Material 编译模块|支持自定义着色模型编译|
|GBuffer 编解码模块|保存并读取 NPR 自定义数据|
|Deferred Lighting 模块|在光照阶段调用 NPR 算法|

具体文件和修改位置会随 UE 版本、Substrate 开关、前向或延迟渲染路径而变化。上表是常见的源码研究入口，不是一份可以直接应用于所有 UE5 版本的补丁。

UE 官方材质系统本身已经包含多种 Shading Model，它们决定材质属性如何参与最终光照计算。

自定义 Toon BxDF 的概念代码：

```
FDirectLighting ToonBxDF(
    FGBufferData GBuffer,
    float3 N,
    float3 V,
    float3 L,
    float3 LightColor,
    float ShadowAttenuation
)
{
    FDirectLighting Lighting;
    Lighting = (FDirectLighting)0;

    float NdotL = dot(N, L);

    // 从GBuffer读取自定义光照参数
    float Threshold = GBuffer.CustomData.x;
    float Softness  = max(GBuffer.CustomData.y, 0.001);

    // 计算卡通漫反射
    float ToonMask = smoothstep(
        Threshold - Softness,
        Threshold + Softness,
        NdotL
    );

    float LightMask = ToonMask * ShadowAttenuation;

    Lighting.Diffuse =
        GBuffer.BaseColor * LightColor * LightMask;

    // 计算卡通高光
    float3 H = normalize(L + V);

    float Specular = step(
        0.95,
        saturate(dot(N, H))
    );

    Lighting.Specular =
        Specular * LightColor * ShadowAttenuation;

    return Lighting;
}
```

这是用于解释原理的伪 HLSL。实际 UE 的 BxDF 接口和光源颜色、阴影衰减的应用位置需要遵循对应版本的渲染管线，避免重复乘以光照强度和阴影系数。

## **13.4 多光源情况下需要注意什么？**

PBR 通常可以将多个光源产生的贡献进行累加。

但是 NPR 如果对每个光源独立进行二分色计算，然后直接相加，可能会出现：

- 多条明暗分界线。
- 暗部被其他光源全部填亮。
- 角色脸部的 SDF 阴影方向不稳定。
- 不同光源叠加导致颜色过曝。

一种常见的项目设计是区分 Key Light（主光源）和 Fill Light（辅助光源）。

![[局部截取_20261126_234633.png]]

对于角色战斗、日夜变化、室内外切换，还需要管理主光源的选择、光照权重及过渡过程，避免光照方向瞬间切换造成脸部阴影跳变。

# **十四、NPR 卡通渲染的性能优化**

## **14.1 常用优化方案**

|优化方案|主要作用|
|---|---|
|Channel Packing|减少纹理数量和采样次数|
|Shared Ramp Texture|多角色共享 Ramp 贴图|
|Shader Permutation 管理|减少不必要的着色器变体|
|Material Instance|复用基础材质和 Shader|
|LOD（细节层级）|远距离角色简化光照与描边|
|减少描边 Pass|降低额外顶点处理及绘制开销|
|简化屏幕空间检测|减少深度、法线采样|
|限制逐像素光源数量|降低多光源着色成本|
|简化远距离角色阴影|降低阴影贴图渲染成本|

## **14.2 按角色重要程度进行渲染分级**

|功能|主控角色|近距离角色|远距离角色|
|---|---|---|---|
|Ramp 光照|完整|完整|简化|
|SDF 面部阴影|开启|开启|可关闭|
|头发高光|双层高光|单层高光|简化高光|
|刘海阴影|高精度|普通|关闭|
|描边|完整|简化|按需关闭|
|实时阴影|高质量|普通|低质量或关闭|
|屏幕空间边缘光|开启|按需开启|关闭|

## **14.3 NPR 一定比 PBR 性能好吗？**

不一定。

单独来看，Step、SmoothStep 或 Ramp 光照通常比完整的复杂 PBR BRDF 更简单。

但高质量 NPR 可能增加：

- 额外的描边 Pass。
- 额外的 SDF、Ramp、Mask 纹理采样。
- 角色专属的阴影 Pass。
- 屏幕空间边缘检测。
- 多种独立高光与材质分支。

所以必须结合 GPU Profiling 进行判断。

在 UE 中，可以使用 `stat GPU`、`ProfileGPU`、Unreal Insights、RenderDoc 等工具分析具体开销。

# **十五、NPR 卡通渲染高频面试问题及答案**

## **1. NPR 和 PBR 的核心区别是什么？**

PBR 以物理合理的光照和材质表现为核心，通常遵循能量守恒、微表面模型和物理反射规律。

NPR 以艺术表现为核心，通过光照离散化、自定义法线、描边、阴影贴图等技术主动控制最终画面，可以打破物理约束。

两者可以混合使用。

## **2. Cel Shading 的底层原理是什么？**

基于法线与光源方向的点积计算光照强度，然后通过 Step、SmoothStep 或 Ramp 将连续光照离散为多个明暗区域。

核心就是对传统连续光照进行分段映射。

## **3. 为什么使用 Ramp 贴图而不是多个 Step？**

Ramp 贴图能够将光照曲线、颜色、分层数量和过渡范围交给美术人员控制。

它可以支持多种材质的光照风格，避免编写大量固定的分层逻辑。

本质上是使用光照强度作为索引的查找表。

## **4. 为什么二次元角色面部要使用 SDF 阴影？**

因为角色面部几何结构复杂，传统 NdotL 光照会产生破碎、不规则的阴影。

SDF 将光照角度与预先设计的面部阴影数据进行比较，能够产生稳定、平滑并且符合动漫风格的面部阴影。

## **5. SDF 面部阴影如何判断光源来自左侧还是右侧？**

使用头部 Right 向量和投影后的光源方向进行点积。

$$
Side=Right\cdot L_{horizontal}
$$

根据结果的正负判断光源所在侧，并选择镜像 UV 或对应的左右阴影贴图。

## **6. SDF 阴影和 Shadow Map 有什么区别？**

SDF 面部阴影属于预先设计的风格化光照方案，主要控制面部自身的明暗分布。

Shadow Map 是基于场景几何与光源之间的深度关系计算实时遮挡。

SDF 无法直接计算动态物体对面部的真实投影，因此需要结合 Shadow Map 或其他动态阴影方案。

## **7. NPR 常用的描边实现有哪些？**

主要有两种：

- Inverted Hull：顶点沿法线扩张，剔除正面，绘制深色背面。
- Screen Space Outline：使用深度、法线或物体 ID 检测屏幕空间边缘。

前者适合角色外轮廓，后者适合内部边缘、交互高亮与场景描边。

## **8. 反向外壳描边为什么会出现断裂？**

因为模型硬边和拆分顶点可能拥有不同的法线。

沿不同方向扩张后，顶点之间产生裂缝。

解决方案是为描边生成独立的平滑法线，并存储在顶点属性中。

## **9. 如何使描边在不同距离下保持相同粗细？**

使用屏幕空间描边，或者在模型描边方案中，根据投影矩阵、裁剪空间 W 和屏幕分辨率计算偏移，使扩张宽度对应固定的屏幕像素数量。

## **10. 如何解决卡通阴影分界线的锯齿和闪烁？**

主要方法有：

- 使用 SmoothStep 替代硬 Step。
- 使用 `fwidth` 计算适当的屏幕空间过渡宽度。
- 优化模型法线。
- 合理使用 TAA、TSR 等时域抗锯齿。
- 优化 Ramp 贴图过滤和 MipMap。
- 保持主光源方向随时间稳定变化。

## **11. 为什么需要自定义法线？**

因为 NPR 中法线不仅表达几何方向，还决定卡通阴影的形状。

通过修改法线，可以使阴影更加连续、简洁，并且符合美术人员设计的明暗关系。

## **12. Rim Light 的实现原理是什么？**

使用法线 N 和观察方向 V 的点积：

$$
Rim=(1-\operatorname{saturate}(N\cdot V))^n
$$

当表面接近视角切线方向时，边缘光增强。

也可以通过屏幕空间深度差实现固定像素宽度的边缘光。

## **13. 头发为什么使用各向异性高光？**

因为发丝具有明确的方向性，不同方向上的反射表现不同。

通过发丝切线方向参与高光计算，可以得到沿头发分布的带状高光，而不是普通的圆形高光。

## **14. UE 实现 NPR 是否一定需要修改引擎源码？**

不一定。

后处理材质、Unlit 自定义光照以及材质图中的风格化处理都可以在不修改引擎源码的情况下实现。

但如果需要在传统渲染管线的逐光源光照阶段新增 Toon Shading Model，并与 GBuffer、实时阴影、动态光照深度集成，往往需要扩展引擎源码和 Shader。

## **15. NPR 如何处理多光源？**

通常可以指定主光源控制 Cel Shading、Ramp 和 SDF 的明暗分布。

其他光源作为辅助光源，提供受控的颜色、亮度和高光贡献。

避免多个光源独立产生大量卡通明暗分界线，破坏角色整体风格。

## **16. NPR 渲染的主要性能瓶颈有哪些？**

不一定是卡通光照公式本身，更多可能来自额外描边绘制、阴影 Pass、纹理采样、屏幕空间后处理、多光源计算以及高复杂度 Shader。

需要结合具体平台和 GPU 性能分析工具进行优化。

# **十六、NPR 渲染核心知识总结**

![[局部截取_20261126_234743.png]]

对于类似《鸣潮》的 UE 动作游戏客户端岗位，建议优先掌握以下知识：

|学习优先级|技术|
|---|---|
|P0|Cel Shading、Ramp Shading|
|P0|SDF 面部阴影|
|P0|反向外壳描边、屏幕空间描边|
|P0|自定义法线|
|P1|风格化高光、边缘光|
|P1|NPR 与 Shadow Map 的结合|
|P1|UE 自定义 Shading Model|
|P1|NPR 与 PBR 混合渲染|
|P2|刘海阴影、各向异性高光|
|P2|多光源管理、渲染性能优化|

以上介绍的是业界常见的 NPR 实现方案，并非对《鸣潮》等商业游戏私有渲染源码的确认。

**NPR 渲染真正的技术核心，并不是简单地给 Lambert 光照增加一个 Step 函数，而是通过光照、阴影、法线、高光、描边和材质的协同控制，让三维模型在动态光照与不同观察角度下，始终保持稳定的二维动漫艺术风格。**