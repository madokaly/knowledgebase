可以，下面我按“**游戏开发高频 + UE 常见 + 面试容易问**”重新整理，只保留最重要的 Buffer。

你可以把它们分成 3 个等级来记：

1. **必须掌握**：Depth、Stencil、GBuffer、Color/SceneColor、Shadow Map
2. **现代引擎高频**：Velocity、History、Hi-Z
3. **UE 高频**：Custom Depth / Custom Stencil

---

# 一、Color Buffer / Scene Color

这是最基础的颜色缓存。

它保存：

> 当前屏幕像素的颜色结果。

最简单理解：

```
Pixel(x, y)
↓
RGBA
```

例如：

```
R = 0.8
G = 0.2
B = 0.1
A = 1.0
```

## Color Buffer 和 SceneColor 的区别

在现代引擎里，经常会区分：

```
SceneColor
```

和：

```
BackBuffer
```

### SceneColor

通常表示：

> 场景经过光照后的 HDR 颜色。

例如：

```
Lighting
   ↓
SceneColor HDR
   ↓
Bloom
   ↓
TAA
   ↓
Tone Mapping
   ↓
BackBuffer
```

SceneColor 可能出现：

```
RGB > 1
```

因为它是 HDR。

例如：

```
Sun Reflection = 8.4
```

---

### BackBuffer

最终真正准备：

```
Present
```

到屏幕的颜色缓存。

所以可以简单记：

```
SceneColor
=
后处理前的场景颜色

BackBuffer
=
最后显示到屏幕的结果
```

### 面试常问

**Q：SceneColor 和 BackBuffer 有什么区别？**

答：

> SceneColor 一般是场景光照完成后的 HDR 中间结果，之后还会经过 TAA、Bloom、Tonemapping 等后处理；BackBuffer 是最终用于 Present 到显示器的图像。

---

# 二、Depth Buffer

这是最重要的 Buffer 之一。

也叫：

```
Z Buffer
```

它保存：

> 每个屏幕像素当前最近物体的深度。

例如：

```
Camera
 |
 |--- A    Depth = 0.2
 |
 |------- B Depth = 0.7
```

如果它们覆盖同一个 Pixel：

```
A 更近
```

所以：

```
DepthBuffer = 0.2
```

---

## Depth Test

假设现在 DepthBuffer：

```
0.3
```

新的 Pixel：

```
Depth = 0.6
```

由于：

```
0.6 > 0.3
```

说明它更远。

因此：

```
Depth Test Fail
```

这个 Pixel 不写入。

于是 GPU 可以正确处理：

```
前后遮挡关系
```

---

# 三、Depth Buffer 为什么极其重要

它不仅用于：

```
谁挡住谁
```

现代渲染里还大量用于：

```
SSAO
SSR
SSGI
Fog
Depth Of Field
TAA
Soft Particle
Occlusion
```

因为通过：

```
Screen UV
+
Depth
```

可以恢复：

```
View Space Position
World Space Position
```

这点非常重要。

---

# 四、为什么 GBuffer 不存 Position

这是一个很经典的面试题。

假设 Deferred Rendering 需要：

```
Position
Normal
BaseColor
Roughness
Metallic
```

看起来可以直接把 Position 存进 GBuffer。

但：

```
Position = float3
```

非常占显存和带宽。

实际上：

```
Screen UV
+
Depth
+
Inverse Projection
```

就可以恢复 View Position。

再：

```
Inverse View
```

恢复 World Position。

所以通常：

```
不单独存 Position
```

而使用：

```
Depth Buffer
```

恢复。

### 面试答案

> Deferred Rendering 通常不会在 GBuffer 中保存完整 World Position，因为 Position 占用较高带宽，而通过屏幕坐标、Depth 和逆 ViewProjection 矩阵就可以重建世界坐标。

这是非常值得记住的一题。

---

# 五、Early-Z

Depth Buffer 和性能优化直接相关。

正常情况下可能是：

```
Rasterization
     ↓
Pixel Shader
     ↓
Depth Test
```

问题是：

Pixel Shader 很贵。

可能涉及：

```
Texture Sample
Normal Map
PBR
Shadow
```

但这个 Pixel 最后可能：

```
被前面的物体挡住
```

那么前面的计算全浪费了。

所以 GPU 会尽可能：

```
Rasterization
     ↓
Depth Test
     ↓
失败？
 ↓
直接丢弃
```

再执行：

```
Pixel Shader
```

这就是：

```
Early-Z
```

### 面试常问

**Q：Early-Z 有什么作用？**

答：

> 在 Pixel Shader 执行前提前进行深度测试，对最终一定被遮挡的 Fragment 直接剔除，从而减少 Pixel Shader 的执行次数。

---

# 六、Depth PrePass

UE 里经常会遇到：

```
PrePass
Early Z Pass
Depth PrePass
```

本质是：

> 提前只渲染一次深度。

第一遍：

```
Mesh
 ↓
Depth Only
 ↓
Depth Buffer
```

不计算复杂：

```
BaseColor
Normal
Lighting
```

之后真正 Base Pass：

```
Depth Buffer
↓
Early-Z
↓
大量不可见 Pixel 被提前剔除
```

适合：

```
复杂材质
高 Overdraw
大场景
```

### 面试要理解

Depth PrePass 的代价是：

```
几何会额外绘制一次
```

所以不是永远稳赚。

它本质上是：

```
增加 Vertex / Draw 成本
```

换：

```
降低 Pixel Shader 成本
```

---

# 七、Stencil Buffer

Stencil Buffer：

> 给屏幕上的 Pixel 打“标签”。

它一般不是浮点深度，而是：

```
整数
```

例如：

```
0
1
2
3
...
```

典型格式：

```
D24S8
```

表示：

```
24bit Depth
8bit Stencil
```

---

# 八、Stencil Buffer 最常见用途

例如：

```
角色描边
```

第一遍：

```
角色区域
Stencil = 1
```

第二遍绘制放大的模型：

```
只有 Stencil != 1 的地方绘制
```

就能得到：

```
Outline
```

除此之外常用于：

```
Mask
Portal
Mirror
特殊后处理
区域限制
```

---

# 九、Depth 和 Stencil 的区别

这是很基础，但面试容易问。

可以直接记：

```
Depth
=
距离信息

Stencil
=
标签信息
```

Depth 回答：

> 这个 Pixel 离摄像机多远？

Stencil 回答：

> 这个 Pixel 属于哪一类？

---

# 十、GBuffer

如果是 UE / Deferred Rendering，GBuffer 是必须掌握的。

GBuffer：

```
Geometry Buffer
```

它不是单独一个 Buffer，而通常是一组 Render Target。

主要保存：

> 当前屏幕 Pixel 的几何和材质信息。

典型包括：

```
BaseColor
Normal
Roughness
Metallic
Specular
AO
Shading Model
```

不同引擎具体布局不同。

---

# 十一、为什么需要 GBuffer

Forward Rendering：

```
Mesh
 ↓
Material
 ↓
Lighting
 ↓
Final Color
```

也就是：

> 渲染物体的时候直接计算光照。

Deferred Rendering：

```
Geometry Pass
     ↓
GBuffer
     ↓
Lighting Pass
     ↓
SceneColor
```

第一步：

```
先记录材质数据
```

第二步：

```
统一算光照
```

所以叫：

```
Deferred Shading
延迟着色
```

因为：

> 把 Lighting / Shading 延迟到了 Geometry Pass 之后。

---

# 十二、GBuffer 典型数据

假设屏幕某个 Pixel：

```
Pixel(500, 300)
```

GBuffer 可能存：

```
BaseColor
= (0.8, 0.2, 0.1)

Normal
= (0, 0, 1)

Roughness
= 0.4

Metallic
= 0.8

AO
= 1.0
```

同时：

```
Depth Buffer
=
0.46
```

Lighting Pass：

```
GBuffer
+
Depth
+
Lights
+
Shadow
↓
PBR
↓
SceneColor
```

---

# 十三、Deferred Rendering 最大优缺点

这个也是面试高频。

## 优点

大量动态光源时非常适合。

因为：

```
几何处理
```

和：

```
光照计算
```

被拆开了。

Lighting 可以直接在屏幕空间进行。

尤其适合：

```
大量 Point Light
大量 Spot Light
复杂场景
```

---

## 缺点

### 1. GBuffer 占显存

例如：

```
BaseColor
Normal
Material
Velocity
Depth
```

一堆 Render Target。

---

### 2. 带宽压力大

因为：

```
Geometry Pass
写 GBuffer

Lighting Pass
读 GBuffer
```

涉及大量：

```
显存读写
```

所以 Deferred 很依赖：

```
Memory Bandwidth
```

---

### 3. 透明物体不好处理

这是面试特别爱问的。

透明物体通常不能简单写 GBuffer。

原因是：

一个 Pixel 可能同时有：

```
玻璃
+
玻璃后面的角色
+
背景
```

但 GBuffer 一个 Pixel：

```
只能保存一套材质属性
```

所以传统 Deferred Renderer：

```
Opaque
→ Deferred

Translucency
→ Forward
```

这在 UE 里也是很常见的思路。

---

# 十四、Shadow Map

Shadow Map 本质非常简单：

> 从 Light 视角生成的 Depth Buffer。

普通 Depth：

```
Camera
↓
Scene
↓
Depth
```

Shadow Map：

```
Light
↓
Scene
↓
Depth
```

---

# 十五、Shadow Map 怎么判断阴影

假设 Light 看到：

```
ShadowMap Depth = 5m
```

现在某个 Pixel：

```
Light → Pixel = 10m
```

说明：

```
Light
↓
5m处有物体
↓
当前Pixel在10m
```

所以：

```
当前Pixel被挡住
```

因此属于阴影。

核心就是：

```
CurrentDepth
vs
ShadowMapDepth
```

比较。

---

# 十六、Shadow Acne 和 Peter Panning

如果面试稍微深入 Shadow Map，很容易继续问：

```
Shadow Acne
```

原因主要是：

```
浮点精度
Shadow Map 分辨率
Depth 比较误差
```

于是使用：

```
Depth Bias
Slope Bias
```

避免自阴影。

但 Bias 太大又会出现：

```
Peter Panning
```

也就是：

> 阴影像和物体分离了。

所以 Shadow Bias 本质上是在：

```
Acne
```

和：

```
Peter Panning
```

之间做权衡。

---

# 十七、Velocity Buffer / Motion Vector

现代游戏非常重要。

它保存：

> 当前 Pixel 相比上一帧移动了多少。

例如：

上一帧：

```
(500, 300)
```

这一帧：

```
(510, 305)
```

那么：

```
Velocity
=
(10, 5)
```

实际通常会使用归一化屏幕坐标。

---

# 十八、Velocity Buffer 用在哪里

现在最常见：

```
TAA
TSR
DLSS
FSR
Motion Blur
```

比如 TAA：

当前 Pixel：

```
CurrentUV
```

通过：

```
Velocity
```

寻找上一帧：

```
PrevUV
=
CurrentUV - Velocity
```

然后读取上一帧颜色。

这叫：

```
Reprojection
重投影
```

---

# 十九、History Buffer

History Buffer：

> 保存上一帧或历史帧结果。

例如 TAA：

```
Current Frame
+
Previous History
↓
TAA
↓
New History
```

现代渲染大量使用：

```
Temporal
时间域
```

因为一帧信息不够，就：

```
积累多帧
```

常见：

```
TAA
TSR
DLSS
SSR
SSGI
Lumen
```

都会大量依赖历史数据。

---

# 二十、Velocity 和 History 为什么经常一起出现

可以这样理解：

History 告诉你：

```
上一帧长什么样
```

Velocity 告诉你：

```
上一帧对应 Pixel 在哪里
```

所以：

```
History
+
Velocity
=
Temporal Reprojection
```

这是现代渲染必须理解的基础。

---

# 二十一、Hi-Z / Hierarchical Z

这个比前面的稍微进阶，但面试和 UE / GPU Driven Rendering 里很重要。

Hi-Z：

```
Hierarchical Z Buffer
```

本质：

> Depth Buffer 的 Mipmap / 金字塔结构。

例如：

```
1920×1080
↓
960×540
↓
480×270
↓
240×135
↓
...
```

每一级保存某种：

```
Min Depth
```

或者：

```
Max Depth
```

取决于 Z 的定义。

---

# 二十二、Hi-Z 最重要用途

主要两个：

```
Occlusion Culling
SSR Ray Marching
```

特别是：

```
GPU Occlusion Culling
```

例如一个 Object：

```
Camera
↓
Building
↓
Object
```

通过 Hi-Z 可以快速发现：

```
Object 整体都在 Building 后面
```

于是：

```
不用真正渲染 Object
```

---

# 二十三、UE 的 Custom Depth / Custom Stencil

这是 UE 非常高频的。

普通：

```
SceneDepth
```

是：

> 整个正常场景的 Depth。

UE 还允许某些 Actor 单独写：

```
CustomDepth
```

---

## 最典型用途

### 角色被墙挡住时描边

角色：

```
CustomDepth
```

墙：

```
SceneDepth
```

如果：

```
CustomDepth > SceneDepth
```

说明：

> 角色在正常场景表面后面。

也就是说：

```
被挡住
```

然后 Post Process 就可以：

```
画 X-Ray
画 Outline
角色高亮
```

---

# 二十四、Custom Stencil

Custom Stencil 可以进一步：

```
给对象分类
```

例如：

```
敌人 = 1
队友 = 2
交互物体 = 3
Boss = 4
```

Post Process：

```
Stencil == 1
→ 红色描边

Stencil == 2
→ 绿色描边
```

因此 UE 里：

```
CustomDepth
+
CustomStencil
```

经常组合使用。

---

# 二十五、Back Buffer / Double Buffer

这个也是图形 API 基础面试经常问。

GPU 通常不会直接往当前显示中的画面写。

而是：

```
Front Buffer
=
当前显示

Back Buffer
=
GPU正在渲染
```

GPU 画完：

```
Present
```

然后交换。

可以理解：

```
GPU
↓
BackBuffer
↓
Present
↓
Display
```

---

# 二十六、Swap Chain

DX / Vulkan / UE 里你经常看到：

```
SwapChain
```

它管理多个：

```
BackBuffer
```

例如：

```
Buffer A
Buffer B
Buffer C
```

用于：

```
双缓冲
三缓冲
```

目的包括：

```
避免显示器读取和 GPU 写入冲突
提高流水线并行度
```

---

# 二十七、最推荐你记住的 9 个 Buffer

如果你现在是为了：

```
UE开发
+
图形学
+
面试
```

优先掌握这几个就够了：

|Buffer|保存什么|核心用途|
|---|---|---|
|Color / SceneColor|RGB颜色|场景颜色|
|BackBuffer|最终颜色|显示|
|Depth Buffer|深度|遮挡、Early-Z|
|Stencil Buffer|标签|Mask / 描边|
|GBuffer|材质+几何属性|Deferred Lighting|
|Shadow Map|Light视角深度|阴影|
|Velocity Buffer|Pixel运动|TAA / TSR / MotionBlur|
|History Buffer|上一帧结果|Temporal算法|
|Hi-Z|分层Depth|Occlusion / SSR|

UE 再额外掌握：

```
CustomDepth
CustomStencil
```

---

# 二十八、最核心的一张图

把它们串起来：

```
                    Scene
                      │
                      ▼
                Depth PrePass
                      │
                      ▼
                Depth Buffer
                      │
                      ▼
                  Base Pass
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      BaseColor     Normal      Material
          └───────────┼───────────┘
                      │
                   GBuffer
                      │
                      │
Shadow Map ───────────┤
                      │
                      ▼
              Deferred Lighting
                      │
                      ▼
                 SceneColor
                      │
          ┌───────────┴──────────┐
          │                      │
      Velocity               History
          │                      │
          └──────────┬───────────┘
                     ▼
                  TAA / TSR
                     │
                     ▼
              Tone Mapping
                     │
                     ▼
                BackBuffer
                     │
                   Present
                     │
                     ▼
                   Screen
```

---

# 二十九、面试最容易问的 Buffer 题

你至少要能直接回答这些：

1. **Depth Buffer 是什么？Depth Test 怎么工作？**
2. **Early-Z 为什么能提高性能？**
3. **Depth PrePass 有什么优缺点？**
4. **Stencil Buffer 和 Depth Buffer 有什么区别？**
5. **GBuffer 里面一般存什么？**
6. **为什么 Deferred Rendering 叫延迟渲染？**
7. **Deferred 和 Forward 的优缺点？**
8. **为什么 GBuffer 通常不存 World Position？**
9. **为什么透明物体不适合传统 Deferred Rendering？**
10. **Shadow Map 本质是什么？**
11. **Velocity Buffer 有什么作用？**
12. **TAA 为什么需要 Velocity 和 History？**
13. **Hi-Z 是什么？为什么可以做 Occlusion Culling？**
14. **UE 的 CustomDepth / CustomStencil 有什么作用？**
15. **SceneColor 和 BackBuffer 有什么区别？**

如果这 15 个问题你都能比较顺畅地讲清楚，**普通游戏客户端 / UE 渲染相关面试里的 Buffer 基础基本就覆盖得比较完整了。**

下面我直接按“**面试回答口径**”来给。重点不是背定义，而是让你能在面试里用 30 秒～2 分钟讲清楚。

1. **Depth Buffer 是什么？Depth Test 怎么工作？**  
    Depth Buffer，也叫 Z-Buffer，用来记录屏幕上每个像素当前已经写入的“最近深度值”。当新的 Fragment 到来时，会把它的深度和 Depth Buffer 中已有的深度进行比较。如果新 Fragment 更靠近摄像机，就通过 Depth Test，并可能更新颜色和深度；如果更远，就被丢弃。
    
    它解决的核心问题就是：**谁挡住谁**。
    
    例如：
    
    ```
    当前Depth = 0.3
    新Fragment = 0.6
    
    0.6更远
    → Depth Test失败
    → 不写Color Buffer
    ```
    
    在普通 Z 中通常越小越近；UE 等现代引擎常使用 Reverse-Z，此时判断方向会反过来，但原理完全一样。
    
2. **Early-Z 为什么能提高性能？**  
    Early-Z 是指尽量在 Pixel Shader 执行之前做深度测试。因为 Pixel Shader 往往会执行纹理采样、Normal Map、PBR 计算等昂贵操作，如果一个像素最后一定会被前面的物体挡住，那执行 Pixel Shader 就是浪费。
    
    所以理想流程是：
    
    ```
    Rasterization
        ↓
    Early Depth Test
        ↓
    被挡住？
        ↓ 是
    直接丢弃
    
        ↓ 否
    Pixel Shader
    ```
    
    它主要降低的是 **Overdraw 带来的 Pixel Shader 开销**。
    
    面试时可以补一句：Alpha Test、修改 Depth、某些 Shader 副作用等情况，可能限制 Early-Z 优化。
    
3. **Depth PrePass 有什么优缺点？**  
    Depth PrePass 是在正式 Base Pass 之前，先只渲染一次场景深度。
    
    第一遍：
    
    ```
    Mesh
    ↓
    Depth Only
    ↓
    Depth Buffer
    ```
    
    第二遍真正渲染材质时，就可以充分利用深度进行 Early-Z，把大量被遮挡像素提前剔除。
    
    优点是：减少复杂 Pixel Shader 的无效执行，特别适合复杂材质、高 Overdraw 场景。
    
    缺点是：**几何会额外画一遍**，会增加 Draw Call、Vertex Shader、Rasterization 等开销。
    
    所以它本质上是：
    
    ```
    增加几何阶段成本
    换取
    减少像素阶段成本
    ```
    
    是否划算取决于场景。
    
4. **Stencil Buffer 和 Depth Buffer 有什么区别？**  
    Depth Buffer 保存的是深度，用于判断前后遮挡；Stencil Buffer 保存的是一个整数标签，用于分类和 Mask。
    
    最简单的记法：
    
    ```
    Depth   = 这个像素有多远？
    Stencil = 这个像素属于谁/哪一类？
    ```
    
    Stencil 常用于：
    
    ```
    Outline
    Portal
    Mirror
    Mask
    特殊后处理
    ```
    
    例如：
    
    ```
    敌人 Stencil = 1
    队友 Stencil = 2
    ```
    
    后处理就可以针对不同值做不同效果。
    
5. **GBuffer 里面一般存什么？**  
    GBuffer 是 Deferred Rendering 中保存几何和材质信息的一组 Render Target。
    
    常见数据包括：
    
    ```
    BaseColor
    World/View Normal
    Roughness
    Metallic
    Specular
    AO
    ShadingModel
    ```
    
    Depth 通常单独存在 Depth Buffer 中。
    
    Geometry Pass：
    
    ```
    Mesh
    ↓
    GBuffer
    ```
    
    Lighting Pass：
    
    ```
    GBuffer
    + Depth
    + Light
    + Shadow
    ↓
    PBR Lighting
    ↓
    SceneColor
    ```
    
    具体每个 GBuffer 通道怎么打包，不同引擎不一样。
    
6. **为什么 Deferred Rendering 叫延迟渲染？**  
    因为它把真正的 Lighting/Shading 延迟到了 Geometry Pass 之后。
    
    Forward 是：
    
    ```
    Mesh
    ↓
    Material
    ↓
    Lighting
    ↓
    Color
    ```
    
    Deferred 是：
    
    ```
    Mesh
    ↓
    先写GBuffer
    ↓
    所有几何完成
    ↓
    再统一计算Lighting
    ```
    
    所以“Deferred”的核心就是：
    
    > 几何阶段先不算最终光照，只记录材质和几何信息，把光照计算推迟到后面的屏幕空间阶段。
    
7. **Deferred 和 Forward 的优缺点？**  
    Forward 的优点是流程直接、GBuffer 带宽压力小、透明物体处理自然，而且 MSAA 通常更容易支持。缺点是物体和大量动态光源组合时，光照计算容易重复。
    
    Deferred 的优点是几何和光照解耦，大量动态光源时效率通常更好，而且屏幕空间光照处理方便。缺点是 GBuffer 占用显存和带宽，透明物体不好处理，MSAA 成本也更高。
    
    面试可以总结成：
    
    ```
    Forward
    → 简单、透明友好、带宽低
    → 大量动态灯时可能贵
    
    Deferred
    → 多动态灯友好
    → GBuffer显存/带宽高、透明麻烦
    ```
    
    现代引擎实际还会混合使用，例如不透明 Deferred，透明 Forward。
    
8. **为什么 GBuffer 通常不存 World Position？**  
    因为 World Position 是一个 float3，直接存每个像素的位置会占大量显存和内存带宽。
    
    但我们已经知道：
    
    ```
    Screen UV
    +
    Depth
    +
    Inverse Projection
    ```
    
    就可以恢复 View Space Position，再通过：
    
    ```
    Inverse View
    ```
    
    恢复 World Position。
    
    所以：
    
    ```
    不保存Position
    ↓
    保存Depth
    ↓
    需要时重建Position
    ```
    
    这样通常比增加一个 Position Render Target 更划算。
    
    这是典型的：
    
    > **用少量计算换显存和带宽。**
    
9. **为什么透明物体不适合传统 Deferred Rendering？**  
    因为传统 GBuffer 对于屏幕上的一个 Pixel，通常只能保存一套表面材质信息。
    
    但透明像素可能同时包含：
    
    ```
    玻璃
    +
    玻璃后面的角色
    +
    背景
    ```
    
    一个像素实际上对应多个表面层。
    
    而传统 Deferred GBuffer：
    
    ```
    一个Pixel
    → 一套Normal
    → 一套BaseColor
    → 一套Roughness
    ```
    
    没办法自然表达多个透明层。
    
    所以一般：
    
    ```
    Opaque
    → Deferred
    
    Transparent
    → Forward
    ```
    
    同时透明物体通常还需要按距离排序并进行 Alpha Blend，这也和传统 Deferred 的工作方式不太匹配。
    
10. **Shadow Map 本质是什么？**  
    Shadow Map 本质上就是：
    
    > 从光源视角渲染得到的一张 Depth Map。
    
    摄像机深度：
    
    ```
    Camera
    ↓
    Scene
    ↓
    Depth
    ```
    
    阴影图：
    
    ```
    Light
    ↓
    Scene
    ↓
    ShadowMap
    ```
    
    渲染某个像素时，把这个像素转换到 Light Space，然后比较：
    
    ```
    当前Pixel到Light的深度
    VS
    ShadowMap中记录的最近深度
    ```
    
    如果：
    
    ```
    CurrentDepth > ShadowDepth
    ```
    
    说明光到当前像素之间还有别的物体挡住，因此当前像素处于阴影。
    
    继续追问的话通常会问：
    
    ```
    Shadow Acne
    Depth Bias
    Peter Panning
    PCF
    CSM
    ```
    
11. **Velocity Buffer 有什么作用？**  
    Velocity Buffer，也叫 Motion Vector Buffer，用来记录当前像素相对于上一帧的位置变化。
    
    可以理解为：
    
    ```
    上一帧位置
    → 当前帧位置
    ```
    
    得到一个二维运动向量。
    
    主要用于：
    
    ```
    TAA
    TSR
    DLSS
    FSR
    Motion Blur
    Temporal Denoising
    ```
    
    最重要的用途之一是 Temporal Reprojection。
    
    例如：
    
    ```
    PrevUV
    =
    CurrentUV - Velocity
    ```
    
    这样当前像素就可以找到上一帧与自己对应的那个像素。
    
12. **TAA 为什么需要 Velocity 和 History？**  
    TAA 的核心思想是：
    
    > 当前帧的信息不够，就利用之前帧的信息。
    
    History Buffer 保存：
    
    ```
    上一帧结果
    ```
    
    Velocity Buffer 保存：
    
    ```
    当前像素上一帧在哪里
    ```
    
    所以：
    
    ```
    CurrentUV
        ↓
    Velocity
        ↓
    找到PrevUV
        ↓
    Sample History
        ↓
    Current + History
        ↓
    TAA
    ```
    
    如果没有 Velocity，物体移动以后你就不知道上一帧哪个 Pixel 才是当前物体对应的位置。
    
    所以可以直接记：
    
    ```
    History
    = 上一帧是什么
    
    Velocity
    = 上一帧在哪里
    ```
    
    两者结合就是 Temporal Reprojection。
    
13. **Hi-Z 是什么？为什么可以做 Occlusion Culling？**  
    Hi-Z 全称 Hierarchical Z，是对 Depth Buffer 建立的一套分层 Mipmap / Depth Pyramid。
    
    例如：
    
    ```
    1920×1080
    ↓
    960×540
    ↓
    480×270
    ↓
    240×135
    ↓
    ...
    ```
    
    每一级代表更大的屏幕区域。
    
    普通 Depth 判断的是：
    
    ```
    一个Pixel
    ```
    
    Hi-Z 可以快速判断：
    
    ```
    一大片区域
    ```
    
    例如一个物体的屏幕包围盒落在某个 Hi-Z 区域内，如果发现它的最近深度都比当前遮挡物还远，就可以判断：
    
    > 整个物体都被挡住了。
    
    那么就可以直接 Cull，不进入真正绘制。
    
    所以 Hi-Z 的优势在于：
    
    > 用低分辨率层级快速做大范围深度判断。
    
    除了 Occlusion Culling，还常用于 SSR 等屏幕空间 Ray Marching。
    
14. **UE 的 CustomDepth / CustomStencil 有什么作用？**  
    CustomDepth 是 UE 允许指定物体额外写入的一张特殊深度 Buffer。
    
    SceneDepth：
    
    ```
    正常场景最前面的深度
    ```
    
    CustomDepth：
    
    ```
    特定Actor自己的深度
    ```
    
    例如角色被墙挡住：
    
    ```
    SceneDepth
    = 墙
    
    CustomDepth
    = 角色
    ```
    
    如果：
    
    ```
    CustomDepth > SceneDepth
    ```
    
    可以判断角色被挡住，然后做：
    
    ```
    X-Ray
    Outline
    穿墙高亮
    ```
    
    CustomStencil 则是给这些对象额外打标签：
    
    ```
    Enemy = 1
    Friend = 2
    Interactable = 3
    ```
    
    后处理根据 Stencil 值进行不同表现。
    
    所以 UE 中经常：
    
    ```
    CustomDepth
    → 判断位置/遮挡
    
    CustomStencil
    → 判断对象类型
    ```
    
15. **SceneColor 和 BackBuffer 有什么区别？**  
    SceneColor 通常是场景完成主要光照之后得到的 HDR 中间结果。
    
    例如：
    
    ```
    GBuffer
    ↓
    Lighting
    ↓
    SceneColor
    ```
    
    但 SceneColor 通常还不能直接显示，因为之后还有：
    
    ```
    Bloom
    Exposure
    TAA / TSR
    Motion Blur
    Tone Mapping
    Color Grading
    UI
    ```
    
    最后才得到用于 Present 的 BackBuffer。
    
    所以：
    
    ```
    SceneColor
    =
    渲染管线中的HDR场景颜色
    
    BackBuffer
    =
    最终准备提交给显示器的画面
    ```
    
    最简单记忆：
    
    ```
    SceneColor
        ↓
    Post Process
        ↓
    BackBuffer
        ↓
    Present
        ↓
    Monitor
    ```

这 15 道里面，**最值得重点吃透的其实是 1、2、5、7、8、9、10、12、13**。它们很容易从一个问题继续追问成完整的图形学面试链条，比如：

```
Depth
↓
Early-Z
↓
Depth PrePass
↓
Overdraw

GBuffer
↓
Deferred
↓
Position重建
↓
透明物体
↓
Forward vs Deferred

Velocity
↓
History
↓
Reprojection
↓
TAA Ghosting

Depth
↓
Hi-Z
↓
Occlusion Culling
↓
GPU Driven Rendering
```

把这些“追问链”串起来，比单独背 15 个定义更有用。