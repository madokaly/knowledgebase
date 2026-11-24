# **Shadow Map 阴影优化方案详解（原理 + UE5 实战 + 面试重点）**

Shadow Map（阴影贴图）是实时游戏中最常用的阴影实现技术之一，但它存在几个典型问题：

1. **性能开销大**：需要从光源视角额外渲染场景，生成深度贴图。
    
2. **锯齿明显**：Shadow Map 分辨率有限，阴影边缘容易出现阶梯。
    
3. **阴影不够真实**：普通 Shadow Map 生成的是硬阴影，无法自然模拟半影。
    
4. **阴影闪烁**：摄像机移动时可能出现阴影抖动、游走。
    
5. **大场景效果差**：一张 Shadow Map 难以同时保证近景和远景精度。
    

因此，Shadow Map 的优化主要分为两大方向：

- **性能优化**：降低阴影绘制次数、Draw Call、显存和阴影采样开销。
    
- **质量优化**：减少锯齿、闪烁、阴影失真，提高软阴影效果。
    

## **一、Shadow Map 常见优化技术总览**

先看最重要、最常见的优化方案。

|优化技术|中文名称|主要解决的问题|重要程度|
|---|---|---|---|
|CSM|级联阴影贴图|大场景阴影分辨率分配|★★★★★|
|PCF|百分比渐近过滤|阴影边缘锯齿、硬边|★★★★★|
|PCSS|百分比渐近软阴影|更真实的软阴影|★★★★★|
|Shadow Culling|阴影剔除|减少绘制物体数量|★★★★★|
|Shadow Caching|阴影缓存|避免重复生成阴影|★★★★★|
|Shadow LOD|阴影细节层次|降低远距离阴影开销|★★★★★|
|Virtual Shadow Maps|虚拟阴影贴图|大规模场景高精度阴影|★★★★★|
|Screen Space Shadows|屏幕空间阴影|补充局部细节阴影|★★★★☆|
|Shadow Atlas|阴影图集|多光源阴影资源管理|★★★★☆|
|VSM / EVSM|方差阴影贴图及扩展|大范围过滤与软阴影|★★★☆☆|

注意两个容易混淆的概念：

- **PCF、PCSS 主要用于改善阴影质量**，通常会增加采样开销，不是直接提高帧率的技术。
    
- **CSM、剔除、缓存、LOD、虚拟阴影分页**主要解决阴影资源利用率及渲染性能问题，同时也可能改善画质。
    

另外，传统 **VSM（Variance Shadow Maps，方差阴影贴图）** 与 UE5 的 **VSM（Virtual Shadow Maps，虚拟阴影贴图）** 是完全不同的技术。

## **二、Shadow Map 为什么性能开销大？**

先回顾普通 Shadow Map 的渲染流程。

### **1. 第一次渲染：生成阴影深度图**

从光源视角渲染场景：

```
// 伪代码：从光源视角渲染
SetViewProjection(LightViewProjection);

// 只渲染需要投射阴影的物体
for (Mesh* Mesh : ShadowCasters)
{
    RenderDepth(Mesh);
}

// 得到 ShadowMap
// 每个 Texel 存储光源视角下的深度
```

对于方向光，通常使用正交投影。

对于聚光灯，通常使用透视投影。

对于点光源，为了覆盖整个球形方向，传统实现通常需要渲染六个方向的 Cube Shadow Map。

### **2. 第二次渲染：阴影判断**

正常渲染场景时，将世界空间位置转换到光源空间：

$$
P_{light}=M_{LightVP}P_{world}
$$

然后比较：

$$
D_{receiver}>D_{shadow}+Bias
$$

其中：

$$
D_{receiver}：当前表面点在光源空间中的深度。
$$
$$
D_{shadow}：Shadow Map 记录的最近遮挡物深度。
$$
$$
Bias：深度偏移，减少自阴影错误。
$$

如果当前点比 Shadow Map 中记录的最近表面更远，说明它可能被其他物体遮挡。

![[局部截取_20261124_235753.png]]

### **3. 主要性能消耗**

Shadow Map 的成本可以粗略分为：

$$
T_{shadow}=T_{depth}+T_{projection}+T_{management}
$$

其中：

- `T_depth`：阴影深度图生成，涉及 Draw Call、顶点处理、光栅化、深度写入。
    
- `T_projection`：阴影投影与过滤，涉及 Shadow Map 采样和屏幕像素计算。
    
- `T_management`：阴影资源分配、剔除、缓存管理等额外工作。
    

**优化时必须先确定瓶颈发生在哪个阶段。**

例如：

- Shadow Depths 很慢：优先优化光源数量、物体剔除、LOD、缓存。
    
- Shadow Projection 很慢：优先优化 PCF/PCSS 采样数、软阴影质量和受光区域。
    
- 显存占用高：优先考虑分辨率、阴影资源复用、虚拟分页和缓存预算。
    

## **三、CSM（Cascaded Shadow Maps，级联阴影贴图）**

面试高频

### **1. CSM 解决什么问题？**

假设需要给一个开放世界场景生成太阳阴影。

如果只用一张 2048×2048 的 Shadow Map 覆盖 1000 米范围，那么地面上的每个阴影纹素会覆盖较大的世界空间区域。

结果：

- 远处建筑阴影可能还可以接受。
    
- 近处角色的脚部阴影可能模糊。
    
- 精细的武器、头发、树叶阴影细节不足。
    

能不能直接提高到 8192×8192？

可以提高精度，但代价很大。

2048 × 2048

# **4.19M**

约 419 万深度纹素

8192 × 8192

# **67.11M**

约 6711 万深度纹素，数量增加 16 倍

这里的 16 倍是像素数量和同格式存储容量的比例，并不意味着实际 GPU 耗时必然增加 16 倍。

CSM 的核心思想就是：

**将摄像机视锥体按照距离划分成多个区域，每个区域使用独立的 Shadow Map，让近处获得更高的阴影精度。**

### **2. CSM 原理示意**

![[局部截取_20261125_000006.png]]

例如将摄像机前方 200 米划分成三个级联：

|级联|摄像机距离|Shadow Map 分辨率|目标|
|---|---|---|---|
|Cascade 0|0～20 m|2048×2048|角色和近景|
|Cascade 1|20～70 m|2048×2048|中距离建筑、树木|
|Cascade 2|70～200 m|2048×2048|远景地形|

这里只是说明原理的示例参数。

虽然三个级联使用相同分辨率，但由于覆盖的世界范围不同，近处级联的每个 Texel 对应更小的世界空间面积，因此精度更高。

### **3. CSM 的具体执行流程**

**第一步：划分视锥体**

根据摄像机近裁剪面和远裁剪面计算级联分界距离。

常见方案是结合均匀划分与对数划分：

$$
C_i=\lambda C_i^{log}+(1-\lambda)C_i^{uni}
$$

其中：

$$
C_i^{log}=n\left(\frac{f}{n}\right)^{i/m}
$$

$$
C_i^{uni}=n+(f-n)\frac{i}{m}
$$

参数解释：

$$
n：近裁剪距离。
$$

$$
f：阴影覆盖的最远距离。
$$

$$
m：级联数量。
$$

$$
i：级联分界索引。
$$

$$
\lambda：两种划分方式的混合权重。
$$


对数划分通常会将更多精度分配给近处区域。

**第二步：为每个级联构造光源投影矩阵**

计算对应视锥体切片的世界空间边界，再转换到光源坐标系，建立正交投影。

注意：在光源空间构造 Shadow Map 时，不仅要考虑接收阴影的可见区域，还要考虑视锥外可能向该区域投射阴影的物体。

**第三步：分别生成 Shadow Map**

```
// 伪代码：生成多级联阴影
for (int i = 0; i < CascadeCount; i++)
{
    // 获取第 i 个级联的光源投影矩阵
    SetLightMatrix(CascadeLightMatrices[i]);

    // 只提交对该级联有影响的阴影投射物
    RenderShadowCasters(i);
}
```

**第四步：正常渲染时选择对应级联**

```
// 伪代码：根据摄像机空间深度选择级联
int CascadeIndex = 0;

if (ViewDepth < CascadeSplit0)
{
    CascadeIndex = 0;
}
else if (ViewDepth < CascadeSplit1)
{
    CascadeIndex = 1;
}
else
{
    CascadeIndex = 2;
}

// 使用当前级联对应的阴影矩阵和 Shadow Map
float Shadow = SampleCascadeShadow(CascadeIndex, WorldPosition);
```

### **4. CSM 有哪些优化点？**

**① 合理控制级联数量**

更多级联通常可以改善阴影分辨率分配，但需要更多深度渲染工作和资源。

常见做法是从 3～4 级联开始测试，再根据场景和设备预算调整。

**② 缩短动态阴影距离**

不必让高精度动态阴影一直覆盖到地图尽头。

例如：

- 近景：CSM 动态阴影。
    
- 中远景：低精度阴影、距离场阴影或静态阴影。
    
- 极远景：仅保留必要的大型物体阴影。
    

**③ 级联边界混合**

级联之间的阴影精度不同，直接切换可能出现明显分界线。

解决方法是在级联边界设置过渡区域，混合相邻级联的阴影结果。

缺点是过渡区可能需要对两个级联进行阴影查询，增加采样开销。

**④ 稳定级联（Stable CSM）**

摄像机稍微移动时，光源投影范围也可能变化，导致 Shadow Map 纹素映射不断漂移。

常见方法包括：

- 将投影中心对齐到阴影纹素网格。
    
- 避免级联投影边界随摄像机发生不必要的缩放。
    
- 使用相对稳定的投影包围区域。
    

这样可以减少阴影闪烁，但可能浪费一部分分辨率。

### **5. CSM 优缺点**

|优点|缺点|
|---|---|
|近景阴影精度高|需要额外渲染多个级联|
|适合太阳光、大场景|存在级联切换和接缝问题|
|有利于分配有限分辨率|需要处理投影稳定性|
|可结合 PCF、PCSS|级联参数需要调优|

CSM 是传统 Shadow Map 体系里非常重要的分辨率优化技术。微软的技术文档也将级联划分、级联选择和阴影过滤作为核心实现问题。

## **四、PCF（Percentage Closer Filtering，百分比渐近过滤）**

PCF 是 Shadow Map 最经典的抗锯齿方法之一。

### **1. 为什么普通 Shadow Map 会出现锯齿？**

普通 Shadow Map 每次只进行一次深度比较：

```
float ShadowDepth = ShadowMap.Sample(Sampler, UV).r;

// 返回 1 表示被光照照亮
// 返回 0 表示处于阴影
float Shadow =
    ReceiverDepth <= ShadowDepth + Bias ? 1.0 : 0.0;
```

这样的阴影判断只有 0 和 1，导致阴影边界非常生硬。

此外，由于 Shadow Map 的纹素分辨率有限，边缘还会出现明显的阶梯。

### **2. PCF 的核心原理**

PCF 不只采样一个 Texel，而是采样周围多个 Texel。

每个采样点分别执行深度比较，最后对比较结果进行平均。

例如，对周围 3×3 个采样点进行深度测试：

![[局部截取_20261125_000138.png]]

PCF 公式：

$$
V(P)=\frac{1}{N}\sum_{i=1}^{N}H(d_i-d_r+Bias)
$$

其中：

$$
V(P)：当前像素的光照可见度。
$$
$$
N：采样次数。
$$
$$
d_i：第 i 个 Shadow Map 深度。
$$
$$
d_r：当前接收点深度。
$$
$$
H：深度比较结果，满足条件时返回 1。
$$

得到的结果是 0～1 之间的连续值，使阴影边缘更平滑。

### **3. PCF 的 HLSL 实现**

```
// 示例：3×3 PCF
// 约定：ShadowMap 深度和 ReceiverDepth 采用相同的深度定义
//       此处假设深度越大，距离光源越远
float PCFShadow(
    Texture2D ShadowMap,
    SamplerState ShadowSampler,
    float2 UV,
    float ReceiverDepth,
    float Bias,
    float2 TexelSize)
{
    float Visibility = 0.0;

    // 遍历周围 9 个采样位置
    for (int y = -1; y <= 1; y++)
    {
        for (int x = -1; x <= 1; x++)
        {
            float2 Offset =
                float2(x, y) * TexelSize;

            float Depth = ShadowMap.SampleLevel(
                ShadowSampler,
                UV + Offset,
                0
            ).r;

            // 比较后累加结果
            Visibility +=
                ReceiverDepth <= Depth + Bias
                ? 1.0
                : 0.0;
        }
    }

    // 返回平均光照可见度
    return Visibility / 9.0;
}
```

这里用于说明 PCF 算法。实际工程还需要处理投影坐标、UV 边界、深度编码及平台的深度比较约定。

### **4. PCF 的常见优化**

**① 使用硬件 PCF**

现代图形 API 支持深度比较采样。

例如 HLSL：

```
// 使用 Comparison Sampler
// 采样时执行深度比较
// 线性比较过滤通常可以组合邻近 Texel 的比较结果
float Visibility = ShadowMap.SampleCmpLevelZero(
    ShadowComparisonSampler,
    UV,
    ReceiverDepth
);
```

配合线性比较过滤，硬件可对相邻纹素进行比较并插值，不必在 Shader 中手动完成全部步骤。

NVIDIA Developer+1

**② 减少采样数量**

例如：

- 3×3：9 个手动比较采样。
    
- 5×5：25 个手动比较采样。
    
- 7×7：49 个手动比较采样。
    

采样核越大，理论上可得到越宽的平滑边缘，但成本也越高。

注意，利用硬件双线性比较和特定采样核优化时，实际纹理采样指令数量可能小于上述数字。

**③ 使用 Poisson Disk Sampling（泊松盘采样）**

传统 PCF 使用规则网格采样，容易产生规律性图案。

泊松盘采样使用非规则分布，能够减少规则网格造成的视觉伪影。

再结合蓝噪声、像素旋转以及 TAA/TSR 等时间累积技术，可以使用较少的当前帧采样获得较平滑的结果。

代价是可能出现噪声、鬼影或运动时的稳定性问题。

### **5. PCF 的缺点**

最重要的一点：

**普通 PCF 的过滤半径通常是固定的，无法自然表达真实软阴影的接触硬化特性。**

例如角色脚下的阴影和角色远离地面的阴影：

- 真实世界中，贴近地面的阴影通常更清晰。
    
- 离遮挡物越远，阴影的半影通常越宽。
    
- 固定半径 PCF 则可能让两处阴影拥有近似相同的模糊程度。
    

这就是 PCSS 要解决的问题。

## **五、PCSS（Percentage Closer Soft Shadows，百分比渐近软阴影）**

PCSS 是 PCF 的进一步扩展。

它最重要的特点是：

**阴影越靠近遮挡物，边缘越清晰；阴影距离遮挡物越远，边缘越模糊。**

这种效果称为 Contact Hardening（接触硬化）。

![[局部截取_20261125_000326.png]]

PCF：近似固定软化程度

![[Pasted image 20261125000357.png]]

PCSS：随遮挡距离变化的半影

### **1. PCSS 的三个核心步骤**

**Blocker Search（遮挡物搜索）**

寻找当前接收点前方的遮挡物，并估计平均深度

**Penumbra Estimation（半影估算）**

根据光源尺寸和遮挡距离计算半影宽度

**Variable PCF（可变半径过滤）**

利用估算的半影宽度决定 PCF 过滤范围

这是经典 PCSS 的基本流程。

### **2. 第一步：Blocker Search（遮挡物搜索）**

对于当前阴影接收点，需要在 Shadow Map 的邻域中搜索比接收点更靠近光源的深度值。

```
// PCSS 第一步：查找遮挡物（简化伪代码）
float BlockerDepthSum = 0;
int BlockerCount = 0;

for (int i = 0; i < SearchSampleCount; i++)
{
    float2 SampleUV =
        UV + SearchOffsets[i] * SearchRadius;

    float Depth = SampleShadowDepth(SampleUV);

    // 遮挡物在接收点前方
    if (Depth < ReceiverDepth - Bias)
    {
        BlockerDepthSum += Depth;
        BlockerCount++;
    }
}

// 没有找到遮挡物，认为当前点被照亮
if (BlockerCount == 0)
{
    return 1.0;
}

float AvgBlockerDepth =
    BlockerDepthSum / BlockerCount;
```

这里使用线性化的光源空间深度来解释原理。

在真正实现中，搜索范围应该根据光源的尺寸、接收点深度和投影方式计算，而不是随意指定一个固定 UV 半径。

### **3. 第二步：Penumbra Estimation（半影估算）**

经典 PCSS 中，利用相似三角形关系，计算近似半影宽度。

$$
\boxed{ w_p\approx w_l\frac{d_r-d_b}{d_b} }
$$

其中：

$$
w_p：估算的半影宽度。
$$
$$
w_l：光源尺寸。
$$
$$
d_r：光源到接收面的距离。
$$
$$
d_b：光源到平均遮挡物的距离。
$$

这个公式是基于特定几何假设的简化关系，实际还需要将半影宽度转换为 Shadow Map UV 空间的过滤半径。

假设：

$$
w_l=2,\quad d_b=5,\quad d_r=6
$$

则：

$$
w_p=2\times\frac{6-5}{5}=0.4
$$

如果接收面距离增加到 10：

$$
w_p=2\times\frac{10-5}{5}=2
$$

可以看出：

**接收点距离遮挡物越远，半影越宽。**

### **4. 第三步：使用可变半径 PCF**

```
// PCSS 第三步：根据半影大小动态调整 PCF 半径
float PenumbraWidth =
    LightSize *
    (ReceiverDepth - AvgBlockerDepth) /
    AvgBlockerDepth;

// 根据当前投影关系转换为 Shadow Map UV 半径
float FilterRadius =
    ConvertPenumbraToUV(PenumbraWidth);

// 使用可变采样范围进行 PCF
float Visibility = PCF_Filter(
    UV,
    ReceiverDepth,
    FilterRadius
);
```

最终就能得到接触处清晰、远离接触处逐渐柔和的阴影。

### **5. PCSS 的性能优化**

PCSS 的主要问题是采样成本较高。

例如某个简化实现中：

- Blocker Search：16 次采样。
    
- PCF Filter：32 次采样。
    
- 每个需要计算阴影的像素总计约 48 次采样。
    

这是示例预算，并非 PCSS 的固定采样次数。

常见优化方法包括：

**① 减少 Blocker Search 的采样数**

利用泊松盘采样和时间累积，尝试减少搜索次数。

**② 自适应采样**

完全处于光照内部或阴影内部的区域，通常不需要大范围的高质量软阴影过滤。

可以先做廉价测试，仅在可能存在半影的区域增加采样。

不过要注意避免像素分支发散造成的性能损失。

**③ 降低分辨率计算软阴影，再进行重建**

软阴影可以在较低的屏幕分辨率计算，随后通过深度、法线信息以及边缘感知上采样进行恢复。

代价是细小阴影容易丢失。

**④ 时间累积与降噪**

每帧采用较少的随机采样，结合运动矢量和历史帧结果重建阴影。

这是现代实时软阴影中非常重要的优化思路，但历史数据错误会带来拖影和鬼影。

### **6. PCF 和 PCSS 对比**

|对比项|PCF|PCSS|
|---|---|---|
|阴影锯齿改善|可以|可以|
|软阴影|固定或预设过滤半径|根据半影估算动态变化|
|接触硬化|不会自然产生|可以近似实现|
|遮挡物搜索|不需要|需要|
|实现复杂度|较低|较高|
|性能成本|通常较低|通常较高|
|适用场景|通用实时阴影|高质量实时软阴影|

**面试重点：PCSS 不是简单地使用更大核的 PCF，而是在 PCF 前增加了遮挡物搜索和半影宽度估算。**

## **六、Screen Space Shadows（屏幕空间阴影优化）**

屏幕空间阴影经常被用来补充传统 Shadow Map 难以表现的小范围阴影细节。

但有两个概念需要区分。

### **1. Screen Space Shadow Mask（屏幕空间阴影遮罩）**

一种渲染组织方式是先计算主光源的阴影可见度，写入屏幕空间 Shadow Mask，再让后续光照阶段复用结果。

基本流程：

```
Camera Depth / GBuffer
        ↓
重建世界空间位置
        ↓
Shadow Map 查询（可包含 CSM、PCF）
        ↓
生成屏幕空间 Shadow Mask
        ↓
后续光照阶段使用 Shadow Mask
```

其优点是可以将昂贵的阴影查询与后续光照计算分离，并在合适情况下复用结果。

但需要注意：

**这并不意味着不再需要 Shadow Map。**

Shadow Map 仍然负责描述光源视角下的遮挡关系。

### **2. Contact Shadows（接触阴影）**

另一种更常见的屏幕空间阴影技术，是直接利用屏幕深度缓冲进行短距离 Ray Marching（光线步进）。

例如角色脚底、武器接触、手指以及细小物体的局部阴影。

基本过程：

1. 获取当前像素的世界空间位置。
    
2. 沿着指向光源的方向执行短距离步进。
    
3. 将步进位置投影回屏幕空间。
    
4. 和屏幕 Depth Buffer 进行遮挡判断。
    
5. 发现遮挡时，生成局部接触阴影。
    

```
// 接触阴影伪代码
for (int i = 0; i < StepCount; i++)
{
    // 沿光源方向前进
    float3 SampleWorldPos =
        WorldPos + LightDir * StepLength * i;

    // 投影到屏幕空间
    float2 SampleUV =
        WorldToScreenUV(SampleWorldPos);

    // 读取屏幕深度
    float SceneDepth =
        SampleSceneDepth(SampleUV);

    // 根据深度差及厚度容差判断遮挡
    if (IsOccluded(SampleWorldPos, SceneDepth))
    {
        return 0.0;
    }
}

return 1.0;
```

UE 的 Contact Shadows 就采用了基于深度缓冲的屏幕空间光线步进，并且是按光源执行的。

Epic Developer Community

### **3. 优缺点**

|优点|缺点|
|---|---|
|有利于表现微小接触阴影|只能利用屏幕中已有的深度信息|
|不需要单独渲染完整的局部 Shadow Map|屏幕外物体无法可靠参与遮挡|
|可以补充低精度阴影|遮挡信息不完整，可能漏光或产生错误|
|适合短距离细节阴影|步进次数和屏幕分辨率影响性能|

在传统 Shadow Map 体系中，可以使用 CSM 负责大范围阴影，再用 Contact Shadows 补充近距离细节。

在 UE5 的高精度 Virtual Shadow Maps 体系中，接触阴影不再总是必要；官方建议只在确有补充价值时使用，因为 VSM 本身可能给出更准确的接触阴影。

Epic Developer Community

## **七、VSM / EVSM（方差阴影贴图）**

这一节先介绍传统图形学中的 VSM，而不是 UE5 虚拟阴影贴图。

### **1. Variance Shadow Maps 的基本原理**

传统 Shadow Map 只存储深度：

$$
D
$$

而方差阴影贴图通常存储深度的一阶矩和二阶矩：

$$
M_1=E[D]
$$

$$
M_2=E[D^2]
$$

从而计算：

$$
\mu=M_1
$$

$$
\sigma^2=M_2-\mu^2
$$

其中：

$$
\mu：平均深度。
$$
$$
\sigma^2：深度方差。
$$

利用单侧切比雪夫不等式，可以估算深度测试通过的概率上界。

例如当接收点深度 \(t>\mu\) 时：

$$
p_{\max}= \frac{\sigma^2} {\sigma^2+(t-\mu)^2}
$$

这个值可用来近似阴影区域中的光照可见度。

### **2. VSM 为什么有利于优化过滤？**

普通 PCF 需要对多个 Shadow Map 深度分别进行比较。

VSM 则可以预先对一阶矩、二阶矩进行过滤，然后使用过滤后的统计结果估算阴影。

因此它能够使用：

- 双线性过滤。
    
- Mipmap。
    
- Gaussian Blur（高斯模糊）。
    
- Summed-Area Table（积分区域表）。
    

尤其在需要较大过滤范围时，可能比大量 PCF 比较采样更有优势。

### **3. VSM 的缺点：Light Bleeding（漏光）**

由于利用统计概率的上界近似实际遮挡情况，VSM 可能将某些应该完全处于阴影的区域错误地判断为部分可见。

表现为：

- 阴影内部偏亮。
    
- 遮挡物重叠处漏光。
    
- 大范围过滤时阴影不够准确。
    

此外，计算方差时也需要考虑深度精度与数值稳定性。

### **4. EVSM（Exponential Variance Shadow Maps）**

EVSM 通过对深度执行指数变换，再计算统计矩，尝试改善 VSM 的漏光问题。

其基本思想是：

$$
z'=e^{cz}
$$

然后对变换后的深度进行矩统计和过滤。

实际 EVSM 方案还可能同时处理正、负指数变换。

优点是进一步改善部分漏光问题。

缺点是：

- 需要额外纹理存储。
    
- 指数参数需要合理选择。
    
- 数值精度和溢出风险需要处理。
    
- 不能彻底消除所有漏光伪影。
    

VSM、EVSM 的核心价值是**让大范围阴影过滤更容易进行**，并不意味着它们在所有游戏场景中都优于 PCF/PCSS。

## **八、真正影响帧率的 Shadow Map 性能优化**

前面的 PCF、PCSS、VSM 更多涉及阴影过滤和画质。

商业游戏实际优化时，往往首先考虑下面这些方法，因为它们可以直接减少 Shadow Depths 的工作量。

### **1. Shadow Culling（阴影剔除）**

阴影深度渲染不应该无条件绘制场景中的全部物体。

例如一个场景存在：

- 1000 棵树。
    
- 300 栋建筑。
    
- 50 个 NPC。
    
- 20 个动态光源。
    

如果每个光源都完整绘制所有物体，成本可能非常高。

可以通过阴影投射体剔除减少参与渲染的对象。

常用方法包括：

|优化|原理|
|---|---|
|Light Frustum Culling|剔除光源阴影视锥之外的物体|
|Shadow Distance Culling|剔除超过阴影影响距离的物体|
|Small Object Culling|忽略阴影投影面积很小的物体|
|LOD Culling|让远处物体使用更简单的阴影模型|
|Occlusion-aware Culling|利用可靠的遮挡信息减少无效阴影绘制|

需要特别注意：

**不能简单地认为摄像机看不到的物体就不需要绘制进 Shadow Map。**

例如，一棵树虽然位于摄像机视野外，但它可能将阴影投射到当前可见的地面上。

因此阴影剔除必须考虑物体的投影是否影响可见的阴影接收区域，而不只是摄像机可见性。

### **2. Shadow LOD（阴影细节层次优化）**

和普通 Mesh LOD 类似，阴影也可以使用不同的几何复杂度。

例如一棵远处的树：

|距离|阴影处理|
|---|---|
|0～20 m|使用较完整的几何阴影|
|20～60 m|使用简化树叶与枝干模型|
|60～120 m|仅保留树冠、树干的大体形状|
|更远|使用低精度方案或逐渐取消阴影|

这里的距离只是策略示例，不是 UE 默认值。

这类优化可以减少：

- 顶点着色工作量。
    
- Shadow Depths Draw Call。
    
- Alpha Masked（遮罩材质）阴影绘制成本。
    
- 远距离细节阴影造成的闪烁。
    

在树木、草地密集的开放世界场景中尤其有用。

### **3. Shadow Caching（阴影缓存）**

这是非常重要的性能优化。

假设一个场景存在：

- 完全静止的建筑。
    
- 完全静止的地面。
    
- 正在移动的玩家。
    
- 正在移动的 NPC。
    

如果光源和建筑都没有变化，就没有必要每帧重新绘制建筑的全部阴影。

可以将阴影划分为两部分：

![[局部截取_20261125_000631.png]]

传统阴影缓存可以围绕静态几何和动态几何拆分来设计；UE5 Virtual Shadow Maps 则可以进一步对分页结果进行缓存。

**难点是缓存失效（Cache Invalidation）。**

当光源移动、物体移动或形变时，相关缓存必须更新，否则可能出现残留阴影。

### **4. Shadow Update Frequency（阴影更新频率）**

并非所有阴影都必须以相同的频率更新。

例如在一个 60 FPS 游戏中，可以按视觉重要性分级：

|阴影类型|示例更新策略|
|---|---|
|主角附近的关键动态阴影|每帧|
|远处低优先级动态物体|适当降低更新频率|
|光源和物体都静止的缓存阴影|尽可能复用|
|不重要的装饰阴影|简化或取消|

需要强调：这是一种可选的工程优化策略，并不是 UE 的统一默认机制。

如果阴影更新频率太低，移动物体可能产生明显的阴影拖尾、跳动或视觉滞后。

因此可以结合距离、屏幕占比、运动速度、阴影变化程度决定更新频率。

### **5. Shadow Atlas（阴影图集）**

如果场景中存在大量 Spotlight（聚光灯），每个光源都使用独立纹理，可能增加渲染资源管理和切换成本。

Shadow Atlas 的思路是：

**将多个光源的 Shadow Map 放进一张较大的纹理中，通过不同的矩形区域存储。**

```
Shadow Atlas
┌───────────────┬───────────────┐
│               │               │
│    Light A    │    Light B    │
│               │               │
├───────┬───────┴───────────────┤
│       │                       │
│   C   │       Light D         │
│       │                       │
└───────┴───────────────────────┘
```

主要优点：

- 集中管理阴影纹理资源。
    
- 可以根据光源重要性动态分配分辨率。
    
- 有利于减少部分资源绑定和切换成本。
    
- 能与阴影缓存、按需更新结合。
    

主要问题：

- 需要维护每个光源对应的 Atlas UV。
    
- 更新和重新分配空间可能产生额外成本。
    
- 不同阴影区域之间需要 Padding（边界留白），防止过滤时采样到相邻光源的阴影数据。
    

### **6. 控制动态阴影光源数量**

这一项在实际项目中非常关键。

假设一个点光源使用传统 Cube Shadow Map，需要从六个方向生成深度图。

如果一个场景有 10 个这样的动态阴影点光源，就可能涉及多达 60 个阴影面的处理。

不过不能简单将成本理解为固定的 60 次完整场景渲染，因为现代引擎可能进行合批、剔除、缓存或使用其他阴影表示方式。

具体可以优化：

- 不重要的光源关闭 Cast Shadows。
    
- 减小光源 Attenuation Radius（衰减半径）。
    
- 缩短光源 Max Draw Distance（最大绘制距离）。
    
- 对静态环境优先评估烘焙或缓存阴影。
    
- 减少同一物体同时受到的动态阴影光源数量。
    

Epic 的阴影优化文档也将光源范围、Cast Shadows 和最大绘制距离列为直接可用的优化入口。

Epic Developer Community

## **九、Virtual Shadow Maps（UE5 虚拟阴影贴图）**

这是 UE5 项目中非常值得重点掌握的部分。

### **1. 传统 CSM 和 Virtual Shadow Maps 有什么区别？**

传统 CSM 通过多个级联管理不同距离的阴影精度。

Virtual Shadow Maps 则使用高分辨率的虚拟阴影纹理，结合 Page（分页）技术，让引擎只为实际需要的部分分配和渲染物理存储。

在 Epic 文档描述的实现中：

- 单张 VSM 的虚拟分辨率为 16384×16384。
    
- 使用 128×128 的 Page。
    
- 只为需要的区域分配物理页面。
    
- 页面可在不同帧之间缓存。
    
- 方向光还会使用 Clipmap（裁剪图层）组织不同距离的阴影精度。

### **2. 虚拟分页是如何优化性能的？**

假设某张虚拟 Shadow Map 分辨率为：

$$
16384\times16384
$$

如果不进行分页，按每纹素 4 字节粗略计算，单张深度图需要：

$$
16384^2\times4=1GiB
$$

而使用 128×128 的分页后，逻辑上有：

$$
128\times128=16384\text{ 个页面}
$$

但实际只有部分页面需要物理分配并渲染。

例如：

![[局部截取_20261125_000720.png]]

如果只有 1000 个 128×128 页面需要物理存储，那么其原始深度数据量按 4 字节/纹素估算约为：

$$
1000\times128\times128\times4 \approx62.5MiB
$$

这只是用于理解分页价值的简化计算，不包含页表、额外缓存、元数据和其他渲染资源。

**虚拟分辨率很高，不代表显存中始终存在整张同尺寸的物理纹理。**

### **3. VSM 的核心优化机制**

**① 按需分配页面**

根据屏幕深度以及阴影查询需求，决定实际需要哪些页面。

避免对完全不需要的阴影区域进行完整的高分辨率渲染。

**② 页面缓存**

已经渲染过、且内容没有变化的页面可以在后续帧中复用。

**③ Clipmap**

方向光使用多个不同覆盖范围的 Clipmap Level。

越靠近摄像机，使用越精细的阴影层级。

这和传统 CSM 都是在处理大范围阴影分辨率问题，但两者的组织和渲染方式不同。

**④ Nanite 集成**

对于支持 Nanite 的几何体，可以利用 Nanite 的集群级别几何处理与剔除能力，提高复杂场景的阴影深度绘制效率。

特别是在超高面数静态环境中，可以避免传统非 Nanite Shadow Depths 的大量几何绘制成本。

### **4. UE5 Virtual Shadow Maps 最大的性能陷阱：Cache Invalidation**

这在真实商业项目里特别重要。

假设一个场景有：

- 300 棵树。
    
- 树叶使用 WPO（World Position Offset，世界位置偏移）实现风吹动画。
    
- 方向光开启 VSM。
    

由于树叶不断变形，相关阴影页面可能每帧被标记为无效，需要重新绘制。

最终可能出现：

```
树叶 WPO 动画
      ↓
阴影投射体发生变化
      ↓
相关 VSM Cache Page 失效
      ↓
重新生成页面深度
      ↓
Shadow Depths GPU 耗时增加
```

Epic 官方也特别指出，WPO、PDO、骨骼动画和移动几何会造成 VSM 缓存失效，且物体 Bounds（包围盒）过大可能放大失效范围。

### **5. VSM 缓存如何优化？**

对于树木、草地、远处地形等对象，可以采取以下措施。

|优化方式|原理|
|---|---|
|WPO Disable Distance|超过一定距离不再执行 WPO 形变|
|合理设置 Bounds|避免过大的包围盒导致大量页面失效|
|Nanite|让支持的几何采用更高效的阴影渲染路径|
|Shadow Cache Invalidation Behavior|合理控制对象的阴影缓存失效行为|
|远景阴影简化|减少远距离高频变形物体投射的阴影|
|减少光源移动|降低整个光源阴影缓存频繁失效的风险|

其中 Shadow Cache Invalidation Behavior 不能随意设置为不失效。

如果物体实际发生了会改变阴影的移动或形变，却强行阻止缓存更新，就可能出现错误阴影。

### **6. UE5 的 SMRT（Shadow Map Ray Tracing）**

UE5 虚拟阴影贴图还可以结合 SMRT，在 Shadow Map 数据上进行采样，实现软阴影与接触硬化。

它不是传统意义上直接针对场景三角形执行的硬件 Ray Tracing，而是利用阴影贴图数据来估算遮挡。

相关控制台变量包括：

```
; 方向光 SMRT 射线数量
r.Shadow.Virtual.SMRT.RayCountDirectional

; 局部光 SMRT 射线数量
r.Shadow.Virtual.SMRT.RayCountLocal

; 每条射线上的阴影采样次数
r.Shadow.Virtual.SMRT.SamplesPerRayDirectional

r.Shadow.Virtual.SMRT.SamplesPerRayLocal
```

降低射线或采样数可以节约性能，但可能增加半影噪声、降低软阴影质量。

Epic 官方文档也将这些变量作为 VSM 软阴影质量调节手段。

Epic Developer Community

在 UE5.9 的更新中，Epic 还为方向光 VSM 引入了 Stochastic PCSS（随机化 PCSS）等软阴影改进。

Epic Developer Community

### **7. CSM 和 UE5 VSM 如何选择？**

|对比项|传统 CSM|UE5 Virtual Shadow Maps|
|---|---|---|
|大场景支持|通过有限级联覆盖|通过 Clipmap 与虚拟分页|
|阴影分辨率|需要手动权衡级联|提供高虚拟分辨率与自动 LOD|
|缓存机制|可采用传统阴影缓存|支持细粒度页面缓存|
|Nanite 集成|传统路径不支持 Nanite 几何阴影|深度集成|
|动态物体成本|受多级联绘制影响|受页面失效与重新渲染影响|
|主要适用场景|传统渲染、兼容性优先项目|Nanite、大型动态场景|

**特别注意：开启 UE5 的 Virtual Shadow Maps 后，方向光不再使用传统 CSM 渲染级联。**

此外，UE5 的 Forward Shading 渲染路径不支持 Virtual Shadow Maps，需要结合项目实际渲染路径和目标平台选择。

## **十、UE5 中如何实际定位 Shadow Map 性能问题？**

在 UE 项目中，优化前应该先通过 GPU Profiler 确认真正的瓶颈。

### **1. 常用性能分析命令**

```
; GPU 各阶段统计
stat gpu

; GPU 性能分析
profilegpu

; 显示帧率
stat fps

; 显示帧耗时
stat unit

; VSM 页面统计
r.Shadow.Virtual.Stats 1

; 查看 VSM 缓存页面
r.Shadow.Virtual.Visualize cache

; 查看 VSM 虚拟页面
r.Shadow.Virtual.Visualize vpage

; 调试：关闭 VSM 缓存，观察性能差异
r.Shadow.Virtual.Cache 0
```

最后一个命令仅建议用于诊断，不应该把关闭缓存当作优化。

部分可视化命令需要先启用对应的 Virtual Shadow Map 可视化模式。测量最终性能时，应关闭调试可视化。

### **2. 重点分析 Shadow Depths 和 Shadow Projection**

|性能项|主要含义|优先优化方向|
|---|---|---|
|Shadow Depths|从光源视角生成阴影深度|阴影剔除、LOD、缓存、光源数量|
|Shadow Projection|查询阴影并生成阴影结果|PCF/PCSS 采样、软阴影质量|
|VSM Invalidated Pages|失效并需要重新绘制的页面|WPO、动态物体、光源移动|
|VSM Allocated Pages|当前需要的虚拟阴影页面|阴影分辨率、光源范围|
|VSM Non-Nanite Rendering|非 Nanite 几何的阴影渲染|Mesh LOD、Draw Call、Nanite 转换|

Epic 的 VSM 性能文档也明确区分 Shadow Depths 和 Shadow Projection 两大开销，并建议观察页面分配、缓存与失效情况。

Epic Developer Community

### **3. 常见优化场景分析**

假设一个 UE5 动作游戏中，主城存在大量建筑、树木、NPC 和动态灯光，GPU 中 Shadow Depths 耗时很高。

建议按以下顺序排查：

**第一步：检查动态光源。**

查看是否存在大量开启 Cast Shadows 的点光源、聚光灯，以及衰减半径过大的光源。

**第二步：检查 Shadow Depths 中的几何绘制量。**

对于传统 Shadow Map，观察每个级联、每个光源的阴影投射物数量。

对于 VSM，重点观察 Non-Nanite 几何的绘制成本。

**第三步：检查缓存命中情况。**

如果 VSM 页面大量 Invalidated（失效），检查 WPO 树叶、骨骼动画角色、移动光源等。

**第四步：优化远处阴影。**

降低远处细小物体阴影的更新和绘制成本，适当使用 Shadow LOD、距离剔除或其他低成本替代方案。

**第五步：调整阴影过滤质量。**

如果 Shadow Projection 开销较高，再考虑减少软阴影射线、PCF/PCSS 采样次数。

这个顺序的关键是：**不要在 Shadow Depths 已经成为瓶颈时，只去降低 PCF 采样数量。**

## **十一、Shadow Map 常见伪影及优化**

这部分也经常在图形学面试中出现。

|现象|主要原因|解决方案|
|---|---|---|
|Shadow Acne（自阴影痤疮）|深度精度误差造成错误自遮挡|Depth Bias、Slope Bias、Normal Bias|
|Peter Panning（阴影悬浮）|Bias 太大导致阴影与物体分离|减小 Bias、提高阴影精度|
|阴影锯齿|Shadow Map 分辨率不足|CSM、提高有效分辨率、PCF|
|阴影闪烁|摄像机移动导致阴影纹素映射变化|Stable CSM、Texel Snapping|
|级联接缝|不同 CSM 级联精度或过滤不一致|Cascade Transition Blend|
|阴影噪声|随机采样数量不足|增加采样、时间累积、降噪|
|Light Bleeding|深度统计近似或遮挡信息不足|改进过滤、EVSM 或其他阴影方案|

其中 Bias 的面试问题特别常见。

### **Depth Bias 和 Normal Bias 有什么区别？**

**Depth Bias（深度偏移）**：调整深度比较所使用的值，减少表面由于浮点精度和离散采样产生的错误自遮挡。

**Normal Bias（法线偏移）**：沿表面法线方向偏移阴影计算位置，以缓解部分自阴影问题。

两种方法都涉及精度和失真的权衡。

Bias 太小容易出现 Shadow Acne；Bias 太大则容易出现悬浮阴影、接触位置不准确。

注意，UE5 传统 Shadow Maps 与 Virtual Shadow Maps 的偏移处理机制并不完全一致，不能直接套用同一组调节方法。

## **十二、项目中如何选择 Shadow Map 优化方案？**

以 UE 动作游戏和开放世界项目为例，可以根据目标平台采用不同策略。

|场景|优先考虑的优化方案|
|---|---|
|PC/主机高画质开放世界|VSM + Nanite + 分页缓存 + 高质量软阴影|
|传统 UE 渲染项目|CSM + PCF + LOD + 阴影缓存|
|低性能平台或移动端|降低动态阴影距离与分辨率、简化投射物|
|大量动态灯光的室内|光源剔除、距离限制、阴影缓存|
|大量草地与树叶|WPO 距离限制、Shadow LOD、缓存失效优化|
|Boss 近距离战斗|优先保证角色与地面的接触阴影及动作阴影稳定性|

对于高速动作游戏，还需要特别关注动态阴影的**时间稳定性**。

如果角色高速移动、闪避、瞬移或进行空中连击，阴影的跳变、拖尾和缓存延迟会非常明显。

因此不能单纯为了减少 GPU 开销而降低所有动态阴影的更新频率。

## **十三、Shadow Map 优化经典面试问题**

以下是建议重点掌握的 10 道题。

1. Shadow Map 性能消耗主要来自哪里？

主要来自阴影深度图的生成与阴影投影过滤。前者涉及几何绘制、顶点处理及光栅化；后者涉及 Shadow Map 查询、深度比较及软阴影采样。

2. CSM 为什么能改善大场景阴影质量？

通过按距离划分摄像机视锥体，让近处级联覆盖更小范围，从而使有限的 Shadow Map 分辨率优先用于近景。代价是多个级联的渲染和管理成本。

3. PCF 和 PCSS 有什么区别？

PCF 对多个深度比较结果做平均，通常使用固定过滤半径。PCSS 先搜索遮挡物，再估算半影宽度，最后执行可变半径 PCF，实现近硬远软的接触硬化阴影。

4. 为什么 PCF 不能直接对深度做平均？

阴影是深度比较产生的不连续可见性结果。直接平均深度可能得到并不存在的遮挡表面，导致错误阴影。标准 PCF 是分别比较后平均可见性。

5. 如何减少 Shadow Map 的 Draw Call？

限制投射阴影的光源与物体数量，执行光源空间剔除，使用阴影 LOD、缓存，并利用合适的实例化、Nanite 等渲染路径减少几何提交成本。

6. 为什么点光源阴影通常比较昂贵？

传统全向阴影通常需要覆盖六个方向，每个方向都有阴影深度渲染成本。实际引擎还会通过缓存、剔除和合并渲染等方法优化。

7. Shadow Cache 什么时候会失效？

通常在光源移动、阴影投射体发生移动或形变、物体增删等情况下失效。UE5 VSM 中，WPO、PDO 和骨骼动画尤其需要关注。

8. UE5 Virtual Shadow Maps 和传统 VSM 有什么区别？

Virtual Shadow Maps 是虚拟分页与缓存的高分辨率阴影系统；Variance Shadow Maps 则利用深度的一阶矩和二阶矩，通过概率上界进行阴影过滤估算。两者完全不同。

9. 为什么开启 PCSS 后性能可能明显下降？

PCSS 通常需要额外执行 Blocker Search，再进行可变半径 PCF。因此比只进行少量比较采样的普通 PCF 消耗更多阴影查询。

10. Shadow Depths 很慢应该怎么排查？

先检查动态阴影光源数量和范围，再检查阴影投射物数量、Draw Call 与 Shadow LOD。对于 UE5 VSM，还应重点检查页面失效、缓存命中以及 Non-Nanite 几何绘制成本。

## **十四、最终总结：Shadow Map 优化的核心思路**

可以将所有 Shadow Map 优化归纳为四个问题：

|优化方向|核心方法|
|---|---|
|**减少生成成本**|剔除、LOD、缓存、减少阴影光源|
|**合理分配精度**|CSM、Clipmap、Virtual Shadow Maps|
|**改善阴影画质**|PCF、PCSS、EVSM、稳定级联|
|**降低查询成本**|自适应采样、硬件 PCF、降分辨率、时间累积|

对于 UE 游戏客户端面试，建议掌握的优先级是：

**第一梯队：** Shadow Map 原理、CSM、PCF、PCSS、Shadow Culling、Shadow Caching。

**第二梯队：** UE5 Virtual Shadow Maps、Nanite 阴影渲染、VSM 缓存失效、Shadow LOD、Contact Shadows。

**第三梯队：** Variance Shadow Maps、EVSM、Shadow Atlas、时间重建与高级阴影过滤。

最有价值的工程理解是：Shadow Map 优化不只是减少采样或降低分辨率，而是**尽可能减少需要绘制的阴影内容、减少重复工作，并把有限的 GPU 预算分配到玩家真正能看到的阴影细节上。**