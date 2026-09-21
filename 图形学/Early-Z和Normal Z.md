可以把 **Early-Z（提前深度测试）** 和你说的 **Normal Z** 理解成：

- **Normal Z / 普通深度测试**：先执行像素着色器（Pixel Shader / Fragment Shader），再进行深度测试。
- **Early-Z / 提前深度测试**：尽量把深度测试提前，在执行像素着色器之前，就把被挡住的像素剔除掉。

它们最核心的区别就是：

> **深度测试发生在像素着色器之前，还是之后。**

这会直接影响 **Overdraw（重复绘制）**、像素着色器开销、透明物体处理，以及某些 Shader 功能能不能正常工作。

---

# 一、先理解普通的 Z-Test 是什么

现代 3D 渲染中，每个像素除了颜色以外，一般还会保存一个：

**Depth Buffer（深度缓冲 / Z-Buffer）**

里面记录：

> 当前这个屏幕像素位置，离摄像机最近的物体深度。

假设摄像机看到两个物体：

```
摄像机

   A           B
   ↓           ↓
距离 5        距离 10

────────── 屏幕
```

A 挡住了 B。

如果某个像素：

```
DepthBuffer = 5
```

后来 B 想画进来：

```
B.depth = 10
```

进行深度测试：

```
if (B.depth < DepthBuffer)
{
    // 更靠近摄像机
    DrawPixel();
}
else
{
    // 被挡住
    DiscardPixel();
}
```

于是 B 被丢弃。

这就是：

**Depth Test / Z-Test（深度测试）**

---

# 二、Normal Z：传统/普通深度测试流程

这里的 Normal Z 可以理解成：

**Late-Z / 普通 Z-Test**

流程大致是：

```
顶点阶段
    ↓
光栅化 Rasterization
    ↓
Pixel Shader
    ↓
Depth Test
    ↓
写入颜色
```

即：

```
Pixel Shader
先计算颜色

然后

Z-Test
判断这个像素到底能不能看见
```

问题就来了。

例如场景中：

```
墙
│
│    敌人
│    模型
│
摄像机
```

敌人完全被墙挡住。

GPU 如果使用普通流程：

```
敌人的 Pixel Shader
↓
计算光照
↓
采样纹理
↓
计算法线
↓
计算阴影
↓
计算 PBR
↓
计算完毕
↓
Depth Test
↓
发现：

“哦，原来被墙挡住了。”

↓
全部结果丢弃
```

前面的 Shader 计算就全部浪费了。

---

# 三、Early-Z 是什么

Early-Z：

**Early Depth Test（提前深度测试）**

就是把：

```
Depth Test
```

尽量提前到：

```
Pixel Shader
```

之前。

流程变成：

```
顶点阶段
    ↓
光栅化 Rasterization
    ↓
Early Depth Test
    ↓
Pixel Shader
    ↓
写颜色
```

即：

```
先判断你能不能被看见
```

如果已经被挡住：

```
直接杀掉 Fragment
```

根本不执行 Pixel Shader。

---

# 四、Normal Z 和 Early-Z 最直观对比

假设屏幕某一块区域：

```
摄像机
   ↓

████████████  墙

▒▒▒▒▒▒▒▒▒▒▒▒  墙后面一个复杂角色
```

角色完全看不见。

## Normal Z

GPU：

```
角色像素
↓
纹理采样
↓
法线贴图
↓
PBR
↓
阴影
↓
GI
↓
反射
↓
算完
↓
Depth Test
↓
被墙挡住
↓
丢弃
```

相当于：

> 先干活，再发现这活白干了。

---

## Early-Z

GPU：

```
角色像素
↓
Depth Test
↓
发现墙已经在前面
↓
直接丢弃
```

Pixel Shader：

```
根本不执行
```

相当于：

> 先检查需不需要干活，再决定要不要干。

---

# 五、Early-Z 为什么性能会明显更好

最重要的原因就是减少：

# Overdraw（重复绘制）

Overdraw 指：

> 同一个屏幕像素被多个物体重复计算。

例如一个像素位置：

```
摄像机
   ↓

角色A
角色B
草
墙
地面
```

可能同一个屏幕像素被算了：

```
地面
↓
墙
↓
草
↓
角色B
↓
角色A
```

总共：

```
5 次 Pixel Shader
```

最终只有最前面的角色 A 被看见。

那么：

```
Overdraw = 5
```

如果 Pixel Shader 很贵，这个性能损耗会非常严重。

---

# 六、举一个真实一点的 Shader

比如 UE 中某个角色材质：

```
BaseColor Texture
Normal Map
Roughness
Metallic
AO

+

Dynamic Lighting
Shadow
Reflection
SSAO
IBL
```

一个像素可能需要：

```
10~几十次纹理采样
+
大量数学运算
```

如果角色被墙挡住：

普通 Z：

```
几十次 Texture Sample
+
大量光照运算

全部做完

↓
Depth Test Fail
↓
丢弃
```

Early-Z：

```
Depth Test Fail
↓
直接丢弃
```

所以 Early-Z 对于：

```
复杂 PBR Shader
复杂场景
高 Overdraw
```

收益特别明显。

---

# 七、Normal Z 和 Early-Z 核心流程区别

可以记这一张。

```
Normal Z

Rasterizer
    ↓
Pixel Shader
    ↓
Depth Test
    ↓
Color Buffer
```

而 Early-Z：

```
Early-Z

Rasterizer
    ↓
Depth Test
    ↓
Pixel Shader
    ↓
Color Buffer
```

中文：

```
普通 Z：

先算像素
再判断能不能看见

Early-Z：

先判断能不能看见
再决定算不算像素
```

---

# 八、Early-Z 最大优点：减少 Pixel Shader 执行

这是面试最重要的一点。

Early-Z 本质上：

> 利用已有的深度信息提前剔除不可见像素，从而减少 Pixel Shader 的执行数量。

例如：

```
100 万个 Fragment
```

其中：

```
60 万
```

被其他物体遮挡。

普通：

```
执行 100 万次 Pixel Shader
```

Early-Z：

```
执行约 40 万次 Pixel Shader
```

可能直接少：

```
60%
```

的像素计算。

当然实际 GPU 还有：

```
Hi-Z
Tile
Quad
Wave
Early/Late Test
```

等优化，所以不会这么简单，但原理就是这样。

---

# 九、Early-Z 第二个优点：降低纹理采样压力

Pixel Shader 常见瓶颈：

```
Texture Sampling
```

例如：

```
float4 baseColor = Texture.Sample(...);
float3 normal = NormalMap.Sample(...);
float roughness = RoughnessMap.Sample(...);
float ao = AOMap.Sample(...);
```

如果 Fragment 被 Early-Z 剔除：

这些：

```
Texture Sample
```

全部不用执行。

因此可以减少：

```
Texture Bandwidth
纹理带宽
```

以及：

```
Texture Cache
纹理缓存压力
```

---

# 十、Early-Z 第三个优点：特别适合不透明物体

对于：

```
Opaque（不透明）
```

Early-Z 特别有效。

比如：

```
建筑
墙壁
地面
角色
岩石
车辆
```

因为这些物体通常：

```
ZWrite = On
ZTest = Less / LessEqual
```

所以深度关系非常明确。

---

# 十一、Early-Z 的问题：不是所有 Shader 都能提前测试

这里是面试很容易继续问的地方。

Early-Z 有一个非常重要的限制：

> Pixel Shader 不能随意改变最终深度或者决定像素是否存在。

例如 Shader 使用：

```
discard;
clip();
```

就可能影响 Early-Z。

---

# 十二、为什么 discard / clip 会影响 Early-Z

假设草：

```
一个 Quad
```

上面是一张草纹理。

真实几何：

```
┌─────────┐
│    /\   │
│   /  \  │
│  /草  \ │
│        │
└─────────┘
```

实际上 Quad 很大，但纹理透明区域很多。

Shader：

```
if (alpha < 0.5)
{
    discard;
}
```

或者：

```
clip(alpha - 0.5);
```

意思：

```
透明区域不要画
```

问题来了。

如果 GPU 提前写入深度：

```
整个 Quad
```

都写进 Z Buffer。

但是 Pixel Shader 后面：

```
discard
```

掉了一部分。

那么原本透明区域：

```
颜色没画
```

但是：

```
深度却已经写进去了
```

后面的物体就会被错误遮挡。

所以某些情况下：

```
discard
clip
Alpha Test
```

会让 GPU：

```
关闭 Early-Z
```

或者变成：

```
Early-Z Test
+
Late-Z Write
```

具体由 GPU 决定。

---

# 十三、修改深度也可能导致 Early-Z 失效

Pixel Shader 可以写：

```
SV_Depth
```

例如：

```
float PS(...) : SV_Depth
{
    return customDepth;
}
```

这意味着：

```
Pixel Shader
```

决定最终深度。

那 GPU 在 Pixel Shader 之前根本不知道：

```
最终 Z 是多少
```

于是无法安全完成 Early-Z。

一般：

```
写 SV_Depth
```

可能导致：

```
Early-Z 优化受限
```

---

# 十四、透明物体通常不能直接使用 Early-Z

透明物体是另外一个典型问题。

例如：

```
玻璃
烟雾
粒子
水
```

通常：

```
ZWrite = Off
```

因为透明物体需要：

```
Alpha Blend
```

例如：

```
最终颜色 =
前景颜色 * Alpha
+
背景颜色 * (1 - Alpha)
```

所以：

```
背景必须存在
```

不能简单：

```
前面透明物体一写深度
↓
后面全部 Early-Z Kill
```

否则背景都没了。

因此透明物体通常：

```
ZTest = On
ZWrite = Off
```

Early-Z 的收益也会受到限制。

这也是为什么：

> 粒子、烟雾、毛发、半透明材质很容易产生严重 Overdraw。

---

# 十五、Normal Z 的优点是什么？

很多人看到 Early-Z 会觉得：

> 那为什么不永远 Early-Z？

因为普通的 Late-Z 更灵活。

Normal / Late Z 的优势主要是：

### 1. 支持 Pixel Shader 修改深度

例如：

```
SV_Depth
```

可以正常决定最终深度。

---

### 2. 支持 discard / clip

Shader 可以：

```
先判断 Alpha
↓
决定这个像素是否存在
↓
再决定写不写 Depth
```

逻辑更加自然。

---

### 3. 对特殊材质更加兼容

例如：

```
Alpha Test
特殊透明
Parallax
Depth Offset
Pixel Depth Offset
```

可能需要：

```
Pixel Shader
```

先计算最终效果。

---

# 十六、Normal Z 最大缺点

就是：

> 被遮挡的 Fragment 也可能执行完整 Pixel Shader。

所以性能浪费非常大。

尤其：

```
复杂场景
+
复杂 Pixel Shader
+
严重 Overdraw
```

性能问题特别明显。

---

# 十七、Early-Z 和 Normal Z 优缺点总结

|对比|Early-Z|Normal / Late Z|
|---|---|---|
|深度测试|Pixel Shader 前|Pixel Shader 后|
|Pixel Shader 数量|少|可能很多|
|Overdraw|显著降低|较严重|
|Shader 性能|好|相对差|
|Texture Sampling|少|多|
|Bandwidth|较低|较高|
|discard / clip|可能受限制|支持好|
|SV_Depth|可能破坏 Early-Z|正常支持|
|不透明物体|非常适合|可以|
|透明物体|作用有限|更常见|
|Shader 灵活度|相对低|高|
|GPU 优化|很重要|基础方案|

一句话总结：

```
Early-Z

牺牲一部分 Shader 灵活性

换取：

少执行大量 Pixel Shader。
```

---

# 十八、实际上现代 GPU 并不是只有 Early-Z 和 Late-Z 两种

真实 GPU 会复杂很多。

通常可能是：

```
Early Depth Test
      ↓
Pixel Shader
      ↓
Late Depth Test
```

即：

```
Early-Z
+
Late-Z
```

同时存在。

例如：

```
Early-Z
```

负责：

> 尽可能早地剔除明显不可见 Fragment。

而：

```
Late-Z
```

负责：

> Shader 执行后，根据最终结果再次确认深度。

所以现代 GPU 更像：

```
                ┌→ Early-Z Kill
Rasterization ──┤
                ↓
          Pixel Shader
                ↓
             Late-Z
                ↓
           Color Write
```

GPU Driver 会根据 Shader 决定：

```
Early-Z 能不能开
```

以及：

```
能开到什么程度
```

---

# 十九、你还应该区分：Early-Z 和 Z-Prepass

这是游戏开发面试里非常容易混淆的一对。

它们不是完全一回事。

---

## Early-Z

是一种：

```
GPU 深度测试优化
```

核心：

```
Depth Test
放在 Pixel Shader 之前
```

---

## Z-Prepass

也叫：

```
Depth Prepass
深度预通过
```

是一种：

```
渲染策略
```

即：

第一遍：

```
只画 Depth
```

第二遍：

```
正式画颜色
```

---

# 二十、Z-Prepass 为什么可以强化 Early-Z

假设场景：

```
建筑
树
角色
石头
地面
```

第一遍：

```
Depth Prepass

只执行简单 Shader
↓
建立完整 Depth Buffer
```

得到：

```
完整深度
```

第二遍真正画：

```
BaseColor
Normal
PBR
Shadow
Reflection
```

这时候：

```
Depth Buffer
```

已经有完整数据。

所以大量被挡住的 Fragment：

```
Early-Z
↓
直接 Kill
```

不会执行复杂 Pixel Shader。

流程：

```
Pass 1

Geometry
↓
Depth Only
↓
Z Buffer

Pass 2

Geometry
↓
Early-Z
↓
Pixel Shader
↓
Color
```

这就是：

**Z-Prepass + Early-Z**

---

# 二十一、那 Z-Prepass 不是要画两遍？为什么还会更快

对。

这是它最大的 trade-off（权衡）。

Z-Prepass：

```
几何绘制两次
```

第一遍：

```
Depth
```

第二遍：

```
Color
```

所以增加：

```
Vertex Shader
Draw Call
Triangle Rasterization
```

成本。

但是可能减少大量：

```
Pixel Shader
```

成本。

所以是否划算取决于场景。

---

# 二十二、什么场景适合 Z-Prepass + Early-Z

例如：

```
大型建筑
城市
室内
复杂角色
复杂 PBR 材质
大量遮挡
```

假设：

```
Vertex Shader 成本 = 1
Pixel Shader 成本 = 20
```

即使 Vertex 做两遍：

```
1 + 1
```

也可能比：

```
Pixel Shader 重复计算 20 × N
```

划算很多。

---

# 二十三、什么时候 Early-Z 收益很小

例如：

## 场景 1：几乎没有 Overdraw

```
大片天空
简单地面
少量物体
```

每个像素：

```
基本只画一次
```

那 Early-Z 没什么可剔除的。

---

## 场景 2：Pixel Shader 很便宜

Shader：

```
return float4(1,0,0,1);
```

Pixel Shader 成本非常低。

这时：

```
Z-Prepass
```

反而可能增加额外 Draw Call 和 Geometry 成本。

---

## 场景 3：大量透明物体

例如：

```
烟雾
火焰
粒子
透明玻璃
```

因为：

```
ZWrite Off
```

很多情况下：

```
无法依靠完整深度遮挡
```

所以 Early-Z 效果有限。

---

# 二十四、Early-Z 和 Hi-Z 也不要混淆

再补一个高频概念：

**Hi-Z（Hierarchical Z）**

中文：

**层次化深度缓冲**

它是对 Z Buffer 做层次结构。

比如屏幕：

```
1920×1080 Depth
```

GPU 不一定一个像素一个像素检查。

而是建立类似：

```
            整个屏幕
               ↓
        ┌──────┴──────┐
       区域           区域
       ↓              ↓
    小区域          小区域
     ↓
    Pixel
```

如果 GPU 发现：

```
这一整个区域
```

都被前面的墙挡住了，

就可以：

```
整个 Tile
```

一起剔除。

不用：

```
一个 Fragment
一个 Fragment
```

地做 Depth Test。

所以：

```
Hi-Z
```

可以理解成：

> Early-Z 的大范围、高效率版本之一。

---

# 二十五、Early-Z、Hi-Z、Z-Prepass 三者关系

非常重要，可以这么记：

```
Z-Prepass
    ↓
提前建立 Depth Buffer
    ↓
Early-Z
    ↓
利用 Depth Buffer 提前剔除 Fragment
    ↓
Hi-Z
    ↓
批量剔除大块区域
```

但它们是三个不同概念：

|技术|中文|本质|
|---|---|---|
|Z Buffer|深度缓冲|存深度|
|Z-Test|深度测试|比较深度|
|Early-Z|提前深度测试|Pixel Shader 前做深度测试|
|Late-Z|后期深度测试|Pixel Shader 后测试|
|Z-Prepass|深度预通过|提前单独画一遍 Depth|
|Hi-Z|层次化深度|按区域快速做深度剔除|

---

# 二十六、UE 里怎么理解 Early-Z

如果你现在主要在学 UE，那么可以重点记：

UE 中很多场景会使用：

```
PrePass
```

也就是：

**Depth Prepass（深度预通过）**

例如：

```
PrePass
↓
BasePass
↓
Lighting
```

PrePass：

```
主要建立 Scene Depth
```

后面的 Base Pass：

```
BaseColor
Normal
Roughness
Metallic
...
```

就可以利用：

```
Early-Z
```

减少无效 Pixel Shader。

UE 中你有时会看到相关设置类似：

```
Early Z-pass
```

例如：

```
None

Opaque Meshes

Opaque and Masked Meshes

Automatic
```

意思本质上就是：

> 哪些物体参与提前的 Depth Pass。

---

# 二十七、为什么 UE 的 Masked 材质要特别注意 Early-Z

UE 中：

```
Masked Material
```

比如：

```
树叶
草
铁丝网
```

通常使用：

```
Opacity Mask
```

相当于：

```
clip(alpha - threshold);
```

例如：

```
贴图：

████░░░░████

█ = 实体
░ = 透明
```

几何实际上可能是：

```
Quad
```

但只有一部分像素真正存在。

因此 PrePass 必须：

```
执行 Alpha Test
```

才能得到正确 Depth。

这会让：

```
Depth Prepass Shader
```

变贵。

因此 UE 是否把：

```
Masked Material
```

加入 Early-Z / PrePass，

是一个典型性能权衡。

---

# 二十八、一个特别重要的性能规律

你可以把 GPU 渲染成本粗略理解成：

```
GPU Cost

≈

Geometry Cost
+
Pixel Cost
```

其中：

```
Geometry Cost
≈
三角形数量
×
Vertex Shader Cost
```

而：

```
Pixel Cost
≈
屏幕像素
×
Overdraw
×
Pixel Shader Cost
```

Early-Z 主要优化的是：

```
Overdraw
```

也就是：

```
Pixel Cost
```

所以：

> Early-Z 对 Pixel Bound（像素瓶颈）场景效果最明显。

---

# 二十九、面试怎么回答

如果面试官问：

> 什么是 Early-Z？

可以这样回答：

> Early-Z 是 GPU 的提前深度测试机制。正常情况下 Fragment 经过光栅化后需要执行 Pixel Shader，再进行深度测试；Early-Z 会尽量将深度测试提前到 Pixel Shader 之前，如果当前 Fragment 已经被 Depth Buffer 中更近的物体遮挡，就可以直接剔除，从而避免执行昂贵的 Pixel Shader。它主要用于降低 Overdraw 和像素着色开销。

然后继续补：

> 但如果 Pixel Shader 中存在 discard、clip、SV_Depth、Pixel Depth Offset 等可能改变像素存活状态或者最终深度的操作，Early-Z 可能受到限制或者退化为 Late-Z。

这基本就是一个比较完整的面试答案。

---

# 三十、最后用一张图彻底记住

## 普通 Z / Late-Z

```
Triangle
   ↓
Rasterization
   ↓
Fragment
   ↓
Pixel Shader          ← 先做昂贵计算
   ↓
Depth Test
   ↓
发现被挡住
   ↓
Discard

问题：

Pixel Shader 白算了
```

---

## Early-Z

```
Triangle
   ↓
Rasterization
   ↓
Fragment
   ↓
Depth Test            ← 提前判断
   ↓
被挡住？
   │
   ├── Yes → Discard
   │
   └── No
        ↓
   Pixel Shader
        ↓
   Color Buffer
```

因此你可以记成一句非常简单的话：

> **Normal/Late-Z：先算颜色，再判断这个像素有没有资格显示。**  
> **Early-Z：先判断有没有资格显示，有资格才算颜色。**

而在 UE / 游戏引擎实际项目里，更常见的是：

```
Depth PrePass
       ↓
建立 Z Buffer
       ↓
Early-Z / Hi-Z
       ↓
减少 Overdraw
       ↓
BasePass Pixel Shader
```

这几个概念基本是连在一起考的。