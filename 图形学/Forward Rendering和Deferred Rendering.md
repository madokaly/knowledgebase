可以。前向渲染（Forward Rendering）和延迟渲染（Deferred Rendering）本质上是在解决同一个问题：

> **“一个物体受到多个光源影响时，几何信息、材质计算、光照计算应该以什么顺序执行？”**

两者最大的区别可以先记成一句话：

> **前向渲染：画物体的时候直接把光照也算完。**  
> **延迟渲染：先把物体的材质/几何信息存到 GBuffer（几何缓冲区），之后再统一计算光照。**

下面详细拆开。

---

# 一、先理解整个渲染过程

假设场景里有：

- 1000 个物体
- 100 个点光源
- 每个物体有：
    - BaseColor（基础颜色）
    - Normal（法线）
    - Roughness（粗糙度）
    - Metallic（金属度）
- 最后需要算 PBR 光照

最终一个像素大概需要计算：

$$
 Color = Ambient + \sum_{i=1}^{N} Light_i 
$$

也就是：

> 最终颜色 = 环境光 + 所有影响这个像素的灯光贡献

真正麻烦的是：

> **怎么知道哪些灯影响哪些像素，并且怎么避免重复计算材质？**

Forward 和 Deferred 的区别就在这里。

---

# 二、前向渲染 Forward Rendering

## 1. 基本原理

前向渲染的流程非常直观：

```
顶点数据
   ↓
Vertex Shader
   ↓
Rasterization 光栅化
   ↓
Pixel Shader
   ↓
材质计算
   ↓
光照计算
   ↓
最终颜色
   ↓
Color Buffer
```

也就是：

```
Mesh
 ↓
BaseColor
Normal
Roughness
Metallic
 ↓
灯光1
灯光2
灯光3
...
 ↓
直接算出最终颜色
```

所以它叫：

> **Forward Rendering（前向渲染）**

因为整个计算一路“向前”走，最后直接得到最终颜色。

---

# 三、一个简单的前向渲染 Shader 思路

伪代码：

```
float3 color = BaseColor;

float3 lighting = 0;

for (每一个影响当前物体的灯)
{
    lighting += CalculateLight(
        Normal,
        ViewDir,
        LightDir,
        Roughness,
        Metallic
    );
}

return color * lighting;
```

重点在这里：

```
for (每一个灯)
```

如果一个物体受到：

```
1 个光源
```

那么还好。

如果受到：

```
50 个光源
```

Pixel Shader 就可能要算 50 次光照。

---

# 四、传统 Forward 最大的问题：大量光源

假设一个物体被 10 个灯照亮。

传统前向渲染可能需要：

```
Pass 1：主灯
Pass 2：灯2
Pass 3：灯3
...
Pass 10：灯10
```

或者在一个 Shader 中：

```
for(light : lights)
{
    CalculateLight();
}
```

于是复杂度可以粗略理解为：

$$
 物体数 \times 光源数 
$$

当然实际引擎会做裁剪，不会真的是所有灯乘所有物体。

但核心问题不变：

> **光源越多，Forward 越容易变贵。**

---

# 五、Forward Rendering 的优点

## 优点 1：架构简单

Forward 最明显的优点就是：

> **简单直接。**

一个物体：

```
几何信息
+
材质
+
光照
=
最终颜色
```

不像 Deferred 那样：

```
先写 GBuffer
再读取 GBuffer
再算光照
```

所以 Forward 的：

- Shader 逻辑容易理解
- 数据流简单
- 调试简单
- 中间 Buffer 少

---

# 六、优点 2：显存带宽压力相对较小

这是非常重要的一点。

Forward 通常直接输出：

```
Final Color
```

比如：

```
RGBA16F
```

大概只需要写：

```
一个颜色 Buffer
+
Depth Buffer
```

但是 Deferred 需要写很多 GBuffer：

```
GBufferA
GBufferB
GBufferC
GBufferD
Depth
...
```

所以 Forward 通常：

> **内存读写量更小。**

对于：

- 手机
- Switch
- VR
- 带宽有限的 GPU

非常重要。

---

# 七、优点 3：非常适合 MSAA

这是 Forward 一个非常大的优势。

MSAA：

> Multi-Sample Anti-Aliasing  
> 多重采样抗锯齿

假设：

```
4x MSAA
```

一个像素里可能有：

```
Sample 0
Sample 1
Sample 2
Sample 3
```

Forward 可以直接对这些 Sample 做覆盖判断。

流程比较自然：

```
Triangle
 ↓
Coverage
 ↓
MSAA Samples
 ↓
Shader
 ↓
颜色
```

---

# 八、为什么 Deferred 不喜欢 MSAA？

因为 Deferred 不只是一个 Color Buffer。

它有：

```
GBuffer BaseColor
GBuffer Normal
GBuffer Roughness
GBuffer Metallic
Depth
...
```

如果使用：

```
4x MSAA
```

理论上这些 Buffer 都可能需要多 Sample。

比如原本：

```
1920 × 1080 × GBuffer
```

突然变成：

```
1920 × 1080 × 4 × GBuffer
```

内存和带宽压力会明显增加。

所以：

> **传统 Deferred + MSAA 成本很高。**

因此很多 Deferred Pipeline 更偏向：

```
TAA
TSR
FXAA
```

这类后处理/时间抗锯齿方案。

---

# 九、优点 4：透明物体支持天然简单

透明物体是 Deferred 的一个老大难问题。

Forward 处理透明非常自然：

```
读取背景颜色
+
当前透明材质
+
光照
+
Alpha Blend
```

例如：

```
FinalColor =
SrcColor * Alpha +
DstColor * (1 - Alpha);
```

像：

- 玻璃
- 水
- 粒子
- 烟雾
- 半透明特效

本身就非常适合 Forward。

因此即使一个引擎主要使用 Deferred：

> **透明物体通常依然使用 Forward 路径。**

这个非常重要。

---

# 十、优点 5：材质模型更自由

Forward 的 Shader 自己完成完整光照。

所以理论上：

```
物体 A：
标准 PBR

物体 B：
卡通光照

物体 C：
特殊皮肤模型

物体 D：
特殊 BRDF
```

都比较容易处理。

因为：

> 每个材质都可以拥有自己的完整 Lighting Shader。

---

# 十一、Forward Rendering 的缺点

最大的缺点：

# 大量动态光源非常昂贵

例如一个城市夜景：

```
路灯 × 200
车灯 × 100
霓虹灯 × 100
室内灯 × 300
```

传统 Forward 就容易出现：

```
大量灯光 × 大量物体
```

的问题。

---

# 十二、Forward 的另一个问题：重复计算

假设：

```
一个像素受到 10 个灯影响
```

这个像素需要算：

```
灯1
灯2
灯3
...
灯10
```

如果设计成多 Pass，则还可能：

```
同一个物体重复 Rasterize
同一个像素重复执行 Shader
```

因此动态灯非常多时不理想。

---

# 十三、延迟渲染 Deferred Rendering

延迟渲染的思想完全不一样。

它说：

> **先别算光。**

先把屏幕上最终可见的表面信息保存下来。

例如：

```
BaseColor
Normal
Roughness
Metallic
Depth
AO
```

保存到：

> **GBuffer**

全称：

> Geometry Buffer  
> 几何缓冲区

---

# 十四、Deferred 第一阶段：Geometry Pass

流程：

```
Mesh
 ↓
Vertex Shader
 ↓
Rasterization
 ↓
Pixel Shader
 ↓
不计算灯光
 ↓
写入 GBuffer
```

例如：

```
GBuffer0：
BaseColor

GBuffer1：
Normal

GBuffer2：
Roughness
Metallic
AO

Depth：
Depth
```

屏幕最终可能变成：

```
              ┌─ BaseColor
Geometry ─────┼─ Normal
              ├─ Roughness
              ├─ Metallic
              └─ Depth
```

---

# 十五、Deferred 第二阶段：Lighting Pass

接下来才计算灯光。

例如一个点光源：

```
读取 Depth
读取 Normal
读取 BaseColor
读取 Roughness
读取 Metallic
       ↓
CalculateLighting()
       ↓
写 Final Color
```

所以：

```
第一阶段：
Geometry → GBuffer

第二阶段：
GBuffer + Lights → Final Image
```

“Deferred”的意思就是：

> **把光照计算延后了。**

---

# 十六、为什么 Deferred 特别适合很多灯？

假设屏幕：

```
1920 × 1080
```

一个小点光源实际上只照到：

```
200 × 200
```

区域。

Deferred 可以画一个：

```
Light Volume
光源体积
```

比如点光源是：

```
Sphere 球体
```

只对球体覆盖到的屏幕区域执行 Shader。

于是：

```
           点光源球体
          /        \
         /          \
--------屏幕----------------
         ██████
         ██████   ← 只有这里计算
         ██████
```

而不需要：

```
遍历所有 Mesh
```

所以 Deferred 的巨大优势是：

> **光照复杂度更接近“光源实际覆盖的像素数量”。**

而不是：

> 物体数量 × 光源数量

---

# 十七、Deferred 的一个核心优势：只给“最终可见像素”算光

假设你看到一堵墙：

```
Camera
 ↓
墙A
 ↓
墙B
 ↓
墙C
```

墙 B、墙 C 被墙 A 挡住。

经过 Depth Test 后：

```
最终 GBuffer 只有墙A
```

于是 Lighting Pass：

```
只给墙A算光
```

不会给：

```
墙B
墙C
```

继续做完整 Lighting。

因此 Deferred 对：

> **复杂场景 + 大量动态灯**

非常友好。

---

# 十八、Deferred 的优点 1：支持大量动态光源

这是它最重要的优势。

例如：

```
100 个点光源
200 个点光源
500 个点光源
```

只要每个灯实际屏幕覆盖范围比较小：

Deferred 都可以比较高效。

所以 Deferred 曾经特别适合：

```
FPS
开放世界
城市夜景
室内复杂照明
```

这种场景。

---

# 十九、优点 2：材质计算和灯光计算解耦

Deferred Pipeline：

```
材质阶段
   ↓
GBuffer

灯光阶段
   ↓
Lighting
```

所以灯光系统可以统一读取：

```
Normal
BaseColor
Roughness
Metallic
```

然后统一计算：

```
Directional Light
Point Light
Spot Light
```

结构很清晰。

---

# 二十、优点 3：Lighting Pass 可以统一优化

因为所有可见表面都变成 GBuffer。

GPU 面对的是：

```
屏幕空间数据
```

所以很多计算变成：

> Screen Space（屏幕空间）

例如：

```
SSAO
屏幕空间环境光遮蔽

SSR
屏幕空间反射

SSGI
屏幕空间全局光照

Decal
延迟贴花
```

都很容易和 GBuffer 配合。

---

# 二十一、优点 4：大量复杂材质不会重复算材质部分

比如某个材质特别复杂：

```
Texture × 10
Normal Map
Detail Normal
Material Layer
复杂节点
```

Geometry Pass 只需要计算一次然后写入：

```
GBuffer
```

后面的：

```
灯1
灯2
灯3
灯4
...
```

只读取 GBuffer。

而不需要重新执行整个 Material Shader。

这也是 Deferred 很重要的思想：

> **Material 和 Lighting 分离。**

---

# 二十二、Deferred 最大的问题：GBuffer 很贵

想象一个 2560×1440 屏幕。

可能存在：

```
GBuffer A：RGBA8
GBuffer B：RGBA16
GBuffer C：RGBA8
GBuffer D：RGBA8

Depth
Velocity
Stencil
```

每帧首先：

```
写一次
```

然后 Lighting：

```
再读一次
```

这意味着大量：

> GPU Memory Bandwidth  
> GPU 显存带宽

所以 Deferred 常常：

```
不是算力瓶颈
```

而是：

```
Bandwidth Bound
带宽瓶颈
```

---

# 二十三、为什么 Deferred 在移动端不一定理想？

移动 GPU 一个非常重要的特点是：

> 显存带宽非常宝贵。

如果 Deferred：

```
写 GBuffer A
写 GBuffer B
写 GBuffer C
写 Depth

然后

读 GBuffer A
读 GBuffer B
读 GBuffer C
读 Depth
```

就产生大量：

```
Memory Traffic
显存读写
```

所以移动 GPU 往往非常在意：

```
Bandwidth
Power Consumption
```

这也是为什么：

> Forward / Forward+ 在移动平台非常有吸引力。

当然现代移动 GPU 和 Tile-Based Deferred 技术会对这个问题进行大量优化，所以不能简单理解成“移动端绝对不能 Deferred”。

---

# 二十四、Deferred 第二大问题：透明物体不好处理

这是非常经典的面试题。

为什么？

假设一个像素：

```
玻璃
↓
墙
```

它实际上需要保存两层表面：

```
Surface A：玻璃
Surface B：墙
```

但是传统 GBuffer 一个像素只能存：

```
一套数据
```

比如：

```
Normal
BaseColor
Roughness
```

于是问题来了：

到底存：

```
玻璃的数据？
```

还是：

```
墙的数据？
```

传统 Deferred 很难表达：

```
一个 Pixel 多层 Surface
```

因此：

> **透明物体通常单独使用 Forward Rendering。**

最终现代引擎实际上经常是：

```
Opaque
→ Deferred

Transparent
→ Forward
```

属于：

> Hybrid Rendering  
> 混合渲染

---

# 二十五、Deferred 第三个问题：MSAA 很麻烦

前面说过。

假如：

```
4x MSAA
```

Forward：

```
Color × 4 Samples
```

Deferred：

```
GBufferA × 4
GBufferB × 4
GBufferC × 4
Depth × 4
...
```

内存瞬间增加很多。

并且边缘 Sample 可能属于不同物体：

```
Sample0 → 墙
Sample1 → 墙
Sample2 → 背景
Sample3 → 背景
```

因此 Lighting Pass 处理会复杂很多。

所以传统 Deferred 对：

> MSAA 不友好。

---

# 二十六、Deferred 第四个问题：GBuffer 格式限制材质

这是一个很多初学者容易忽略的问题。

Forward：

```
Shader 想要什么参数
就自己算什么
```

Deferred 不一样。

因为 Lighting Pass 只能看到：

```
GBuffer
```

如果 GBuffer 只有：

```
BaseColor
Normal
Roughness
Metallic
```

你的材质突然说：

> 我还需要一个 CustomSkinParameter。

那怎么办？

必须：

```
增加 GBuffer Channel
```

或者：

```
重新编码
```

或者：

```
特殊 Shading Model
```

所以 Deferred：

> **材质数据受到 GBuffer Layout 限制。**

---

# 二十七、一个非常重要的直观对比

假设：

```
一个像素
受到 100 个灯影响
```

## Forward

类似：

```
Material
 ↓
Light 1
Light 2
Light 3
...
Light 100
 ↓
Color
```

---

## Deferred

```
Material
 ↓
GBuffer
 ↓
Normal
BaseColor
Roughness
Metallic

然后：

Light 1 ─┐
Light 2 ─┤
Light 3 ─┤→ Lighting
...      │
Light100─┘
```

区别就是：

> **Deferred 把 Material 和 Lighting 拆开了。**

---

# 二十八、性能公式可以粗略这样理解

这不是严格数学公式，只用于建立直觉。

### Forward

传统 Forward：

 Cost \approx Geometry + Material \times Lights 

更准确一点是：

 Cost \approx VisiblePixels \times MaterialCost \times LightCount 

---

### Deferred

大概是：

 Cost \approx Geometry + GBuffer + LightingPixels 

也就是：

```
先生成一次材质信息
+
每盏灯只处理自己覆盖的像素
```

所以：

```
少量灯：
Forward 往往很好

大量动态小灯：
Deferred 往往很好
```

---

# 二十九、现代 Forward 已经不是“傻 Forward”了

这里非常重要。

很多面试资料会说：

> Forward 不适合多个灯。

这句话现在已经不够准确了。

现代引擎经常使用：

# Forward+

又叫：

```
Forward Plus
Forward+
```

它的思路是：

> **仍然使用 Forward Shader，但提前把灯进行屏幕空间分类。**

---

# 三十、Forward+ 是怎么工作的？

例如把屏幕切成很多 Tile：

```
┌──┬──┬──┬──┐
│01│02│03│04│
├──┼──┼──┼──┤
│05│06│07│08│
├──┼──┼──┼──┤
│09│10│11│12│
└──┴──┴──┴──┘
```

每个 Tile 建立一个：

```
Light List
灯光列表
```

比如：

```
Tile 01:
Light 2
Light 7

Tile 02:
Light 1

Tile 03:
Light 3
Light 6
Light 8
```

当 Pixel Shader 执行时：

```
lights = TileLightList[currentTile];

for(light : lights)
{
    CalculateLight();
}
```

于是：

> 不再遍历场景所有灯。

而只遍历：

> **当前屏幕区域真正影响自己的灯。**

---

# 三十一、进一步发展：Clustered Forward

比 Forward+ 更进一步的是：

> Clustered Rendering  
> 簇渲染

Forward+ 通常把屏幕：

```
X × Y
```

切块。

Clustered：

```
X × Y × Z
```

也就是把 Camera Frustum：

> 摄像机视锥体

切成立体网格。

例如：

```
             Camera
                ●
               / \
              /___\
             /| | |\
            /_|_|_|_\
           / | | | | \
```

每个 Cluster 保存：

```
影响它的灯光列表
```

这样就能更精确地筛选光源。

现代游戏引擎里：

```
Forward+
Clustered Forward
Clustered Deferred
```

都非常常见。

所以现在已经不能简单理解成：

```
Forward = 灯少
Deferred = 灯多
```

现代 Forward 也可以支持很多灯。

---

# 三十二、Forward 和 Deferred 的完整优缺点对比

|项目|Forward 前向渲染|Deferred 延迟渲染|
|---|---|---|
|基本思路|画物体时直接算光照|先写 GBuffer，之后统一算光|
|Pipeline|简单|更复杂|
|GBuffer|不需要|需要|
|显存占用|较低|较高|
|显存带宽|较低|较高|
|大量动态灯|传统 Forward 较差|很擅长|
|Forward+ 大量灯|较好|较好|
|MSAA|**非常友好**|**成本高**|
|透明物体|**非常友好**|**不友好**|
|自定义材质|**灵活**|受 GBuffer 限制|
|屏幕空间效果|可以|**天然方便**|
|VR|经常有优势|GBuffer 成本较高|
|移动端|经常有优势|带宽压力较大|
|GPU 带宽|较省|较吃带宽|
|大量小灯|Forward+ 后不错|**传统强项**|
|调试难度|相对简单|相对复杂|

---

# 三十三、什么时候更适合 Forward？

如果你的场景属于：

```
灯光数量不是特别多
+
很看重 MSAA
+
透明物体多
+
显存带宽有限
```

Forward 很适合。

典型情况：

### ① VR

VR 非常看重：

```
清晰度
MSAA
稳定帧率
```

Forward 往往有优势。

---

### ② 移动端

手机很在意：

```
Bandwidth
Memory
Power
```

所以 Forward / Forward+ 非常常见。

---

### ③ 风格化游戏

例如：

```
Toon Shader
特殊 BRDF
特殊光照
```

Forward 的材质灵活性比较舒服。

---

### ④ 大量透明内容

例如：

```
粒子
玻璃
水
特效
```

本来就经常需要 Forward。

---

# 三十四、什么时候适合 Deferred？

如果：

```
大型 3D 场景
+
大量动态光源
+
复杂 PBR
+
大量屏幕空间效果
```

Deferred 通常非常合适。

例如：

```
大型 FPS
开放世界
城市夜景
复杂室内
大量动态灯
```

---

# 三十五、为什么 AAA 游戏很长时间都喜欢 Deferred？

核心原因并不是：

> Deferred 图像质量更高。

这个说法不对。

Forward 和 Deferred 都可以做：

```
PBR
Shadow
GI
SSR
AO
IBL
```

真正原因是：

> **Deferred 能非常方便地处理大量动态灯，并且方便组织复杂的屏幕空间渲染管线。**

尤其过去传统 Forward 还没有成熟的：

```
Forward+
Clustered
GPU Light Culling
```

时，Deferred 优势尤其明显。

---

# 三十六、现代游戏其实经常不是二选一

这是最重要的工程结论。

现代 Renderer 经常是：

```
                Renderer
                   │
        ┌──────────┴──────────┐
        ↓                     ↓
Opaque 不透明            Transparent 透明
        ↓                     ↓
Deferred / Forward+         Forward
        ↓                     ↓
        └──────────┬──────────┘
                   ↓
              Post Process
                   ↓
                Final
```

也就是说：

> **现代引擎通常是混合式渲染架构。**

而不是纯粹：

```
100% Forward
```

或者：

```
100% Deferred
```

---

# 三十七、从 GPU 性能瓶颈角度理解

这是面试里非常值得说的一层。

## Forward 更容易成为：

```
ALU Bound
计算单元瓶颈
```

原因是 Pixel Shader 中：

```
灯光计算
BRDF
阴影
材质
```

都可能堆在一起。

特别是：

```
每像素几十盏灯
```

时计算量很高。

---

## Deferred 更容易成为：

```
Bandwidth Bound
显存带宽瓶颈
```

因为：

```
大量写 GBuffer
+
大量读 GBuffer
```

所以二者本质上某种程度是在交换：

```
Forward：

少写内存
多做 Shader 计算

        ↓↑

Deferred：

多写内存
减少重复 Shader 计算
```

这个理解非常重要。

---

# 三十八、再结合 Overdraw 理解

Overdraw：

> **同一个屏幕像素被多个物体重复绘制。**

例如：

```
Camera
 ↓
物体 A
 ↓
物体 B
 ↓
物体 C
```

如果绘制顺序不好：

```
先 C
再 B
再 A
```

同一个像素可能 Shader 跑三次。

Forward 中：

```
每一次都可能执行完整材质 + 光照
```

非常贵。

Deferred Geometry Pass：

```
虽然仍然可能有 Overdraw
```

但是 Geometry Shader 通常不用计算完整灯光。

所以复杂光照场景中 Deferred 可以降低一部分重复的 Lighting 工作。

当然配合：

```
Early-Z
Depth Prepass
Hi-Z
```

Forward 也可以明显降低 Overdraw 成本。

---

# 三十九、把 Early-Z 和这两个联系起来

你前面如果在学 Early-Z，这里正好可以联系起来。

Forward：

```
Early-Z
   ↓
尽量提前杀掉不可见像素
   ↓
避免执行昂贵的
Material + Lighting Shader
```

因此 Early-Z 对 Forward 非常重要。

Deferred：

```
Early-Z
 ↓
减少 Geometry Pass Overdraw
 ↓
减少无意义 GBuffer 写入
```

两边都有价值，但 Forward 由于单个 Pixel Shader 往往更重，所以：

> **避免无效 Pixel Shader 的收益尤其明显。**

---

# 四十、面试最容易问的几个问题

### Q1：Forward 和 Deferred 最核心的区别？

可以回答：

> Forward Rendering 在 Geometry Pass 中直接完成材质和光照计算并输出最终颜色；Deferred Rendering 则先将 BaseColor、Normal、Roughness、Metallic 等几何/材质信息写入 GBuffer，再通过后续 Lighting Pass 统一计算光照。

---

### Q2：为什么 Deferred 擅长大量光源？

答：

> 因为 Lighting Pass 可以基于屏幕空间的 Light Volume 或 Tile/Cluster，只计算灯光实际覆盖的像素，而不需要针对每个物体重复执行完整的材质和光照 Shader。

---

### Q3：Deferred 最大缺点是什么？

建议至少回答这四个：

```
① GBuffer 占用显存

② GBuffer 产生大量显存带宽

③ MSAA 成本较高

④ 透明物体不好处理
```

再深入一点：

```
⑤ 材质参数受 GBuffer Layout 限制
```

---

### Q4：Forward 最大优势？

可以答：

```
Pipeline 简单
显存带宽压力小
MSAA 友好
透明友好
材质灵活
```

---

### Q5：Deferred 为什么不能很好支持透明？

标准答案：

> 因为传统 GBuffer 一个屏幕像素通常只保存最前面的一个 Surface 的材质和几何信息，而透明物体要求同时考虑透明表面以及其后的多个 Surface，因此无法简单使用普通 Deferred GBuffer 表达。

---

### Q6：Deferred 一定比 Forward 快吗？

一定要回答：

> **不是。**

性能取决于：

```
灯光数量
灯光覆盖面积
材质复杂度
分辨率
Overdraw
带宽
MSAA
平台架构
```

例如：

```
只有一个方向光
```

Deferred 反而可能白白承担：

```
写 GBuffer
+
读 GBuffer
```

的带宽。

这种情况下 Forward 完全可能更高效。

---

# 四十一、一个最容易记住的类比

可以把 Forward 想成：

> **餐厅现点现做。**

一个顾客：

```
选菜
→ 做菜
→ 调味
→ 出菜
```

然后下一个顾客重新来。

---

Deferred 像：

> **中央厨房。**

先统一准备：

```
肉
蔬菜
调料
半成品
```

也就是：

```
GBuffer
```

然后统一进行：

```
Lighting
```

大量订单时效率很好。

但代价就是：

> 你需要一个很大的仓库。

这个仓库就是：

```
GBuffer
```

---

# 四十二、最终可以这样记忆

### Forward Rendering

核心：

> **Geometry + Material + Lighting → Final Color**

优势：

```
简单
带宽低
MSAA 强
透明强
材质灵活
```

缺点：

```
传统 Forward 大量动态灯成本高
复杂场景容易产生昂贵的重复 Pixel Shader
```

---

### Deferred Rendering

核心：

> **Geometry + Material → GBuffer → Lighting → Final Color**

优势：

```
大量动态光源
光照与材质解耦
屏幕空间效果方便
减少重复材质计算
```

缺点：

```
GBuffer 显存大
带宽高
MSAA 贵
透明困难
材质受 GBuffer 格式限制
```

最后把它们压缩成一个面试级核心：

> **Forward 更倾向于用“计算量”换“内存和带宽”；Deferred 更倾向于用“GBuffer 内存和带宽”换取更高效的大量动态光源处理。现代渲染器又通过 Forward+、Clustered Lighting 等技术模糊了两者的界限，因此实际项目通常采用混合渲染，而不是绝对二选一。**