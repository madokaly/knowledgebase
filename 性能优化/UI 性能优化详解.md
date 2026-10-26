# 游戏客户端 UI 性能优化详解（Unity + UE5）

游戏 UI 优化是客户端性能优化中非常重要的一部分，尤其是 MMO、开放世界、动作 RPG、射击游戏等 UI 比较复杂的项目。

例如《鸣潮》这类动作 RPG，游戏中往往同时存在血条、技能图标、伤害数字、小地图、任务追踪、背包、角色属性面板等大量 UI。如果处理不好，即使 3D 场景渲染已经优化得很好，UI 仍然可能造成明显的掉帧。

UI 优化的核心可以总结为：减少 CPU 重建、降低 GPU 绘制成本、控制内存、避免无意义更新，以及减少 UI 创建和销毁时的卡顿。

## 一、UI 性能开销主要来自哪里？

首先需要知道 UI 渲染的大致流程。

<img width="738" height="685" alt="image" src="https://github.com/user-attachments/assets/bc471033-7fe5-4ff6-a1bf-999e654d099b" />


UI 优化大致可以分成以下几个方向。

|优化方向|主要开销|核心手段|
|---|---|---|
|Draw Call 优化|CPU / 渲染线程|合批、图集、减少材质切换|
|Overdraw 优化|GPU|减少透明 UI 重叠|
|UI 重建优化|CPU|脏标记、缓存、局部更新|
|Layout 优化|CPU|减少嵌套布局、避免频繁重新排版|
|UI 逻辑优化|CPU|事件驱动、去除无效 Tick|
|列表优化|CPU / 内存|虚拟列表、对象池|
|UI 资源优化|内存 / GPU|纹理压缩、按需加载|
|动画优化|CPU / GPU|Tween、材质动画、更新隔离|
|加载优化|CPU / I/O / 内存|异步加载、预加载、生命周期管理|

对于 60 FPS 游戏，一帧时间预算约为 16.67 ms；120 FPS 则约为 8.33 ms。

这并不意味着 UI 可以独占这些时间。UI 必须与游戏逻辑、物理、动画、场景渲染等系统共享帧预算。

## 二、Draw Call 与 UI 合批优化

这是游戏 UI 性能优化中最经典的面试知识。

### 2.1 为什么 UI 会产生大量 Draw Call？

假设背包界面有 100 个物品图标，每个图标都是一个 UI Image。

最直接的绘制方式是：

```
绘制图标1
    设置纹理A
    提交 Draw Call

绘制图标2
    设置纹理B
    提交 Draw Call

绘制图标3
    设置纹理C
    提交 Draw Call

...

绘制图标100
    提交 Draw Call
```

如果每个 UI 都单独提交 Draw Call，会增加 CPU 和渲染线程的工作量，也可能导致 GPU 状态切换。

但 UI 中大量元素通常具有相似的性质：都是矩形网格，采用相同的 UI Shader、Alpha 混合方式和渲染状态。

因此可以尝试将多个元素进行 Batching（合批）。

### 2.2 UI 合批的基本原理

<img width="739" height="355" alt="image" src="https://github.com/user-attachments/assets/b5d20f9a-f5c5-4147-95dc-9ba00127d149" />


合批的目标是让多个 UI 元素在满足渲染顺序和状态兼容等条件下，共享较少的绘制命令。

例如将 100 个符合条件的 Image 合并为少量绘制批次。

需要注意，Draw Call 数量不是唯一性能指标。在很多移动设备上，大面积透明 UI 的 GPU 像素开销可能比 Draw Call 更严重。

### 2.3 使用 Sprite Atlas（精灵图集）

如果 4 个图标分别使用不同纹理：

```
IconA.png
IconB.png
IconC.png
IconD.png
```

GPU 可能需要在绘制它们时多次切换纹理。

如果将它们放入同一图集：

<img width="736" height="226" alt="image" src="https://github.com/user-attachments/assets/4240f74f-6701-4143-9738-df36d9738684" />


每个 UI Image 可以使用不同的 UV，但采样同一张纹理。

这样能够减少纹理切换，并为合批创造条件。

但有三个容易被忽略的问题。

第一，图集相同不等于一定合批成功。 材质、混合状态、裁剪方式、绘制顺序以及引擎内部批次规则都可能破坏合批。

第二，图集越大不一定越好。 如果主界面只需要 5 个图标，但它们与 200 个其他图标打包进一张大图集，就可能产生不必要的内存驻留。

第三，不应该将所有 UI 放在一张超级图集中。 实际项目可以按照常驻 HUD、背包、角色面板、商城等资源生命周期划分图集。

### 2.4 哪些情况会破坏合批？

|情况|为什么可能增加 Draw Call|
|---|---|
|使用不同纹理|纹理绑定发生变化|
|使用不同材质|Shader 或渲染状态不一致|
|不同 Blend 模式|混合状态切换|
|特殊 Mask / 裁剪|可能产生额外渲染命令|
|绘制顺序被打断|不能随意改变半透明对象的顺序|
|特殊 UI Shader|材质参数或渲染状态可能不兼容|
|不同 Canvas 或独立渲染层|可能无法跨边界合批|

例如 UI 绘制顺序为：

```
Image A：纹理A
Image B：纹理B
Image C：纹理A
```

虽然 Image A 和 C 使用相同纹理，但由于半透明对象的绘制顺序可能影响最终颜色，不能简单地把 C 移动到 B 前面进行合批。

因此 UI 合批同时涉及 材质兼容性与绘制顺序。

## 三、Overdraw（过度绘制）优化

这是 UI GPU 优化的重点，特别是在移动端和 VR 设备上。

### 3.1 什么是 Overdraw？

Overdraw 指同一个屏幕像素被重复绘制多次。

例如一个 UI 界面包含：

```
最底层：游戏场景
第二层：全屏半透明背景
第三层：半透明面板
第四层：半透明装饰
第五层：文字和图标
```

如果多个 UI 元素重叠，同一块像素可能经历多次片元处理和颜色混合。

<img width="738" height="521" alt="image" src="https://github.com/user-attachments/assets/4b17fa64-11b1-43e0-9a9d-4d66141bc921" />


例如：

```
第1次：绘制场景背景
第2次：绘制半透明遮罩
第3次：绘制面板背景
第4次：绘制装饰图片
第5次：绘制文字
```

如果同一个像素有 5 层有效绘制，可以粗略认为其 Overdraw 层数为 5。

不过要注意，Overdraw 层数不等于 GPU 耗时一定增加相同比例。实际成本还取决于 Shader、分辨率、纹理采样、混合方式、Early-Z 以及 GPU 架构。

### 3.2 为什么透明 UI 更容易产生 Overdraw？

绝大部分 UI 使用 Alpha Blending（Alpha 混合）。

常见的非预乘 Alpha 混合公式为：

$$
C_{out}=\alpha C_{src}+(1-\alpha)C_{dst}
$$

其中：

- \(C_{src}\)：当前 UI 像素颜色。
    
- \(C_{dst}\)：目标颜色缓冲中已有的颜色。
    
- \(\alpha\)：当前 UI 的透明度。
    

因为透明对象需要与已有颜色混合，往往不能像完全不透明的 3D 物体那样充分利用深度遮挡来减少片元工作。

此外，即使图片中大量像素的 Alpha 为 0，也不代表这些区域一定没有片元执行成本。是否能提前丢弃取决于 Shader 和 GPU 实现。

### 3.3 Overdraw 优化方法

方法一：减少全屏透明层。

例如打开背包时，不要无意义地叠加多张全屏暗色遮罩。能够复用的遮罩应尽量复用。

方法二：裁剪图片的透明区域。

假设一个发光技能图标实际有效内容只占图片中间 40% 的区域，但使用了一个很大的透明矩形。

如果改成更紧凑的网格或图片区域，就可能减少片元覆盖面积。

Unity 中可以根据素材情况考虑使用 Tight Mesh；但它并不适用于所有图片类型和 UI 用途，需要结合实际网格以及 Mask 行为验证。

方法三：控制叠加的装饰效果。

比如一个按钮由背景图、边框图、渐变图、光效图等多个元素组成，可以考虑将静态视觉效果预合成为一张图片。

但这样会失去部分动态调整能力，也可能增加贴图内存。

方法四：减少大面积高成本 UI 材质。

例如大面积实时模糊、复杂渐变、动态噪声、扭曲和多次纹理采样。

这类效果即使 Draw Call 不多，仍可能造成明显 GPU 耗时。

方法五：处理不可见 UI。

已经被其他全屏页面完全覆盖的 UI，可以根据生命周期隐藏、停止更新，或移除渲染。

单纯将透明度设置为 0，不应该当作通用的渲染剔除方案。

### 3.4 面试常问：Draw Call 少，GPU 就一定快吗？

不一定。

假设两个界面：

|指标|界面 A|界面 B|
|---|---|---|
|Draw Call|80|20|
|透明叠加层|少|非常多|
|Shader|简单|复杂|
|覆盖面积|小|大|

界面 B 可能 GPU 开销更高，因为它虽然 Draw Call 少，但是大量像素被重复执行了较复杂的 Shader。

所以：

Draw Call 主要关注提交与状态管理开销；Overdraw 主要关注像素重复处理和混合开销。

## 四、UI Rebuild（重建）优化

这一部分尤其重要，Unity UGUI 和 UE5 UMG 都需要理解。

### 4.1 什么是 UI 重建？

UI 并不一定每帧重新生成全部布局和几何数据。

例如一个普通的背包界面：

```
背包打开
   ↓
创建 UI
   ↓
计算布局
   ↓
生成绘制数据
   ↓
保持静止
```

理想情况下，如果界面内容没有变化，可以反复复用之前计算好的数据。

但是当 UI 属性发生变化时，某些缓存就可能失效。

例如：

```
血量文本从 100 变成 99
        ↓
文本内容变化
        ↓
旧文本几何数据失效
        ↓
生成新的文本绘制数据
        ↓
更新相关渲染信息
```

如果被修改的 UI 位于一棵复杂的布局树中，还可能触发父布局或相关子节点重新计算。

### 4.2 Dirty Flag（脏标记）机制

脏标记是一种常见的优化机制。

核心思想是：

只有发生变化的数据才需要重新计算。

可以将 UI 的脏状态概念性地分成：

|脏状态|含义|
|---|---|
|Layout Dirty|尺寸、位置、布局需要重新计算|
|Geometry Dirty|顶点、网格或绘制几何需要更新|
|Material Dirty|材质或相关渲染信息需要更新|
|Hierarchy Dirty|UI 层级或子节点发生变化|

这些是帮助理解的通用分类，并不表示 Unity、UE 都使用完全相同的内部状态名称或传播规则。

例如修改血条颜色，通常没有必要重新计算血条尺寸。

但是修改容器的宽度，可能引起子节点换行、位置变化，以及进一步的布局计算。

### 4.3 为什么深层 UI 布局容易造成性能问题？

假设层级结构是：

```
Canvas
 └── CharacterPanel
      └── VerticalLayoutGroup
           └── HorizontalLayoutGroup
                └── ContentSizeFitter
                     └── Text
```

当 Text 内容变化时，可能需要：

```
文字内容变化
      ↓
计算文字尺寸
      ↓
更新 ContentSizeFitter
      ↓
重新计算 HorizontalLayoutGroup
      ↓
重新计算 VerticalLayoutGroup
      ↓
更新相关 UI 几何
```

如果文字每帧变化，整条布局链就可能反复执行。

因此，频繁变化的 UI 要尽量避免处于复杂的自动布局层级中。

### 4.4 Unity UGUI 中的 Canvas 重建

Unity 官方指出，当 Canvas 中的 UI 元素变化时，其批次构建可能需要重新分析整个 Canvas 中的相关绘制元素。

所以，如果将所有界面放在一个大型 Canvas 中，一个小 UI 的变化也可能造成较大的 CPU 成本。


例如：

```
MainCanvas
 ├── 静态背景
 ├── 玩家血条（频繁更新）
 ├── 技能冷却（频繁更新）
 ├── 小地图
 ├── 任务列表
 ├── 背包界面
 └── 商店界面
```

当玩家血条发生变化，可能导致大型 Canvas 内部重新分析批次。

优化方案是根据更新频率和生命周期拆分 Canvas。

```
UIRoot
 │
 ├── StaticCanvas          // 静态UI
 │    ├── 固定边框
 │    └── 静态装饰
 │
 ├── DynamicCanvas         // 动态HUD
 │    ├── 血条
 │    └── 技能冷却
 │
 ├── DamageCanvas          // 伤害飘字
 │
 ├── MapCanvas             // 小地图
 │
 └── PopupCanvas           // 弹窗界面
```

这样做的好处是，某个 Canvas 内部发生变化，不必让其他 Canvas 也承担同样的批次重建工作。

但是 Canvas 不能无限拆分，因为不同 Canvas 之间不能直接合并成同一个 UGUI 绘制批次，而且各自存在管理与提交成本。

拆分 Canvas 的原则不是越多越好，而是让更新频率相近、渲染状态兼容的 UI 尽量处于合适的绘制区域。

## 五、Layout（布局）优化

自动布局很好用，但它并非没有成本。

在 Unity 中，典型的布局组件包括 `HorizontalLayoutGroup`、`VerticalLayoutGroup`、`GridLayoutGroup`、`ContentSizeFitter`。

在 UE 中，常见的布局容器包括 `HorizontalBox`、`VerticalBox`、`GridPanel`、`Overlay`、`CanvasPanel` 等。

### 5.1 为什么自动布局有开销？

例如一个动态聊天列表：

```
聊天消息到来
    ↓
创建新的消息控件
    ↓
计算文字高度
    ↓
更新消息控件尺寸
    ↓
更新父容器尺寸
    ↓
重新排列消息位置
    ↓
刷新 ScrollView
```

如果每秒增加大量消息，同时使用多层自动布局，就可能产生较高的 CPU 开销。

尤其是频繁调用强制布局刷新的接口，容易把原本可以合并执行的布局更新变成多次重复计算。

### 5.2 固定布局与自动布局的区别

假设背包中的物品格子尺寸固定：

```
每行：5个格子
格子宽度：100
格子高度：100
格子间距：10
```

没必要每次都使用复杂的自适应布局计算。

可以直接计算第 `index` 个元素的位置：

$$
column=index\%5
$$

$$
row=\left\lfloor\frac{index}{5}\right\rfloor
$$

$$
x=column\times110
$$

$$
y=row\times110
$$

示例 Unity C#：

```
/// <summary>
/// 计算背包格子的布局位置。
/// 这里假定左上角作为锚点和 Pivot。
/// </summary>
public Vector2 GetItemPosition(int index)
{
    const int columnCount = 5;
    const float itemWidth = 100f;
    const float itemHeight = 100f;
    const float spacing = 10f;

    // 当前元素位于第几列
    int column = index % columnCount;

    // 当前元素位于第几行
    int row = index / columnCount;

    // 计算元素的局部布局坐标
    float x = column * (itemWidth + spacing);

    // Unity UI 的锚点向下通常使用负Y
    float y = -row * (itemHeight + spacing);

    return new Vector2(x, y);
}
```

这样计算的复杂度对于单个元素是 \(O(1)\)。

不过手动布局需要自行处理分辨率适配、动态文本尺寸、本地化，以及新增、删除元素等情况。

### 5.3 布局优化的核心原则

静态界面尽量使用简单结构；动态界面仅在数据变化时重新布局；高频变化的文字和图标尽量脱离复杂的父布局链。

在 Unity 中，如果布局组只用于初始化排列，可以在布局稳定后关闭相应自动布局组件，但后续尺寸变化需要主动处理。

对于 UE UMG，也不要认为仅仅把所有布局容器换成 `CanvasPanel` 就一定更快。更重要的是合理利用期望尺寸缓存、Invalidation（失效缓存机制），以及减少实际布局变化。

## 六、动态列表优化：对象池与虚拟列表

这是背包、任务列表、排行榜、商城等界面的高频优化问题。

### 6.1 普通 ScrollView 的问题

假设游戏里存在 10,000 条聊天记录。

如果一次性创建 10,000 个 UI 元素：

```
ScrollView
 └── Content
      ├── Message 1
      ├── Message 2
      ├── Message 3
      ├── ...
      └── Message 10000
```

会产生大量 UI 控件、对象内存、布局和潜在渲染管理成本。

但是屏幕实际可能只显示 10 条消息。

其他消息虽然不在视野中，却未必没有内存和更新成本。

### 6.2 Virtual List（虚拟列表）的原理

虚拟列表的核心是：

数据可以有 10,000 条，但只创建当前可见区域所需的少量 UI 控件。

例如列表高 600 px，每个条目高 60 px。

那么屏幕最多可以显示约 10 条内容。

考虑滚动过程中的缓冲区，可以创建 12～14 个可复用控件。

<img width="738" height="298" alt="image" src="https://github.com/user-attachments/assets/8de24ee7-d835-45cb-b796-633be4554572" />


### 6.3 虚拟列表的实现过程

以固定高度列表为例。

首先根据滚动偏移量，计算当前第一个可见元素：

$$
firstIndex=\left\lfloor\frac{scrollOffset}{itemHeight}\right\rfloor
$$

然后计算可见条目数量：

$$
visibleCount=\left\lceil\frac{viewportHeight}{itemHeight}\right\rceil
$$

为防止滚动时出现空白，再额外创建少量缓冲元素。

伪代码：

```
// 实际数据总数，例如 10000
int dataCount = 10000;

// 每个条目高度
float itemHeight = 60f;

// 列表可视区域高度
float viewportHeight = 600f;

// 当前滚动偏移量，约定向下为正
float scrollOffset = 1230f;

// 计算第一个可见数据索引
int firstIndex =
    Mathf.FloorToInt(scrollOffset / itemHeight);

// 限制索引不超出数据范围
firstIndex = Mathf.Clamp(
    firstIndex,
    0,
    dataCount - 1
);

// 计算需要多少个可复用UI控件
int visibleCount =
    Mathf.CeilToInt(viewportHeight / itemHeight) + 2;

// 根据 firstIndex 为复用控件绑定对应数据
// 如果滚动到下一段，就重复利用已有控件
```

真正实现时还需要处理快速滚动、控件重定位、滚动位置和数据索引映射。对于高度可变的列表，通常还需要维护条目高度缓存及前缀和等数据结构。

### 6.4 对象池与虚拟列表有什么区别？

|对比|对象池|虚拟列表|
|---|---|---|
|核心目标|避免反复创建和销毁|减少同时存在的 Widget 数|
|主要机制|缓存对象并复用|只实例化可见部分|
|能否减少初始创建数量|不一定|可以|
|适合场景|弹窗、飘字、提示|背包、聊天、排行榜|
|是否能结合使用|可以|可以|

两者经常结合使用。

例如伤害数字使用对象池；上千条背包记录使用虚拟列表，再配合可复用的条目 Widget。

在 UE5 中，`ListView`、`TileView` 等具有条目生成和复用机制的控件通常比手动向普通 `ScrollBox` 添加数千个子 Widget 更适合大规模列表。

## 七、UI 更新逻辑优化：事件驱动代替 Tick

很多 UI 性能问题并不是 GPU 绘制导致的，而是程序员写了大量不必要的每帧逻辑。

### 7.1 错误做法：每帧刷新血条

Unity 示例：

```
void Update()
{
    // 每一帧都读取生命值
    float hp = player.CurrentHP;
    float maxHP = player.MaxHP;

    // 每一帧都设置UI
    hpSlider.value = hp / maxHP;

    // 每一帧都更新文字
    hpText.text = hp.ToString();
}
```

假设角色的生命值在 10 秒内没有发生变化。

60 FPS 下，这段代码仍然执行大约 600 次。

其中频繁生成文字字符串，还可能带来临时内存分配与 GC 压力。

### 7.2 正确做法：事件驱动

游戏逻辑中只在生命值发生变化时发送事件。

```
using System;
using UnityEngine;

public class PlayerHealth : MonoBehaviour
{
    public float CurrentHP { get; private set; } = 100f;
    public float MaxHP { get; private set; } = 100f;

    // 生命值变化事件
    public event Action<float, float> OnHealthChanged;

    public void SetHealth(float newHP)
    {
        // 限制血量范围
        newHP = Mathf.Clamp(newHP, 0, MaxHP);

        // 没有变化就不通知UI
        if (Mathf.Approximately(CurrentHP, newHP))
            return;

        CurrentHP = newHP;

        // 只有血量变化时才更新UI
        OnHealthChanged?.Invoke(CurrentHP, MaxHP);
    }
}
```

UI 监听事件：

```
using TMPro;
using UnityEngine;
using UnityEngine.UI;

public class PlayerHealthUI : MonoBehaviour
{
    [SerializeField] private PlayerHealth player;
    [SerializeField] private Image healthFill;
    [SerializeField] private TMP_Text healthText;

    private void OnEnable()
    {
        // 订阅生命值事件
        player.OnHealthChanged += RefreshHealth;

        // 初始化显示
        RefreshHealth(player.CurrentHP, player.MaxHP);
    }

    private void OnDisable()
    {
        // 取消订阅，防止重复监听和生命周期问题
        player.OnHealthChanged -= RefreshHealth;
    }

    private void RefreshHealth(float current, float max)
    {
        // 更新血条填充比例
        healthFill.fillAmount =
            max > 0f ? current / max : 0f;

        // 使用TextMeshPro的格式化接口
        // 避免普通字符串拼接带来的部分分配
        healthText.SetText("{0:0}/{1:0}", current, max);
    }
}
```

两种方案的区别：

|对比|Update 轮询|事件驱动|
|---|---|---|
|更新频率|每帧|数据变化时|
|逻辑开销|持续产生|按需产生|
|数据同步|轮询检查|主动通知|
|适合场景|确实逐帧变化的值|血量、金币、任务状态等|

UE GAS 中也可以使用相同思路。

例如通过 ASC（Ability System Component）的属性变化回调来刷新血量 UI，而不是每帧通过 Widget Binding 查询 Attribute。

```
// 示意：UE5 GAS 中监听生命值变化
// ASC 为已初始化的 AbilitySystemComponent

ASC->GetGameplayAttributeValueChangeDelegate(
    UMyAttributeSet::GetHealthAttribute()
).AddUObject(
    this,
    &UMyHUDWidget::OnHealthChanged
);
```

```
void UMyHUDWidget::OnHealthChanged(
    const FOnAttributeChangeData& Data)
{
    // 新的生命值
    const float NewHealth = Data.NewValue;

    // 根据生命值更新血条
    // MaxHealth 应由已有的属性缓存或相关数据提供
    const float Percent = MaxHealth > 0.f
        ? FMath::Clamp(NewHealth / MaxHealth, 0.f, 1.f)
        : 0.f;

    HealthBar->SetPercent(Percent);
}
```

上面是核心逻辑示例，实际还需要在 Widget 生命周期结束时按需解绑委托，并处理 ASC 更换等情况。

这也是 Epic 官方推荐的 UMG 优化方式：减少 Tick、OnPaint 逻辑以及复杂的属性 Binding，尽量通过事件响应数据变化。

## 八、UI 动画优化

UI 动画是容易被忽略的性能热点。

例如伤害数字、血条变化、按钮缩放、技能冷却、角色切换过渡等。

### 8.1 动画为什么会影响 UI 性能？

假设技能图标每帧改变大小。

```
技能图标尺寸变化
        ↓
UI 几何或布局数据失效
        ↓
需要重新计算相关数据
        ↓
更新绘制信息
```

如果它还参与自动布局，成本可能进一步放大。

不同动画修改的属性，对 CPU 的影响并不相同。

### 8.2 动画优化优先级

|动画类型|特点|优化思路|
|---|---|---|
|材质 UV 动画|GPU 可以完成主要计算|适合循环流动效果|
|材质发光动画|通常不修改 UI 布局|适合技能图标发光|
|透明度动画|可能触发绘制更新|避免不必要的重建|
|Transform 动画|可能影响几何或缓存|隔离动态节点|
|尺寸 / 布局动画|可能引起父子布局更新|尽量避免逐帧重排|
|大量数字飘字|大量对象同时更新|对象池、数量限制|

UE 官方的 UMG 性能指导中特别提到了材质动画的优势：对于适合在 GPU 上执行的循环视觉效果，材质方案能减少逐帧 UI 动画逻辑开销，但 GPU Shader 本身仍然需要消耗计算资源。

### 8.3 Unity 中使用 DOTween

例如通过 DOTween 实现血条平滑变化：

```
using DG.Tweening;
using UnityEngine.UI;

private Tween healthTween;

public void AnimateHealth(float targetPercent)
{
    // 避免旧动画和新动画同时修改血条
    healthTween?.Kill();

    targetPercent = UnityEngine.Mathf.Clamp01(targetPercent);

    // 在0.2秒内平滑调整血条
    healthTween = healthFill
        .DOFillAmount(targetPercent, 0.2f)
        .SetEase(Ease.OutQuad);
}

private void OnDestroy()
{
    // 避免对象销毁后Tween仍然持有引用
    healthTween?.Kill();
}
```

DOTween 可以简化动画更新管理，但它并不会自动消除 UI 属性变化产生的重建成本。

例如大量 UI 元素同时修改布局尺寸，即使全部使用 DOTween，也依然可能出现 CPU 峰值。

因此，更重要的是减少动画影响范围。

## 九、UE5 UMG 专项优化

这是 UE 客户端开发需要重点掌握的部分。

UE 的 UI 系统主要涉及：

Slate（底层 UI 框架）→ UMG（面向游戏的 UI 开发框架）→ Widget Blueprint / C++。

UMG 很多绘制与布局工作最终通过 Slate 完成。

### 9.1 Invalidation Box（失效缓存容器）

其核心思想是：

如果 UI 没有变化，就复用之前缓存的布局与绘制信息。

例如：

```
Invalidation Box
 ├── 背包背景
 ├── 装饰图案
 ├── 标题
 └── 静态按钮
```

首次渲染时：

```
计算布局
   ↓
生成绘制数据
   ↓
缓存数据
```

后续未变化时：

```
复用缓存
   ↓
跳过不必要的布局和绘制计算
```

当子 Widget 发生变化，缓存可能失效，相应区域需要重新计算。

例如一块静态面板，初次显示需要构建 100 个 Widget 的绘制数据。

如果它之后一直保持不变，缓存机制可以避免持续重复计算这 100 个 Widget 的相关信息。

这里缓存的是 Slate 的 UI 计算和绘制数据，而不等同于把画面渲染成一张纹理。

### 9.2 Global Invalidation（全局失效缓存）

UE 还提供 Global Invalidation。

它相当于在更大范围内统一管理 Widget 的失效与缓存，减少未变化 Widget 的重复工作。

相关控制项：

```
; UE 控制台变量形式
Slate.EnableGlobalInvalidation=1
```

开启后，SWindow 可以作为统一的失效缓存根节点；其内部的普通 Invalidation Box 不再作为独立缓存根工作。

这通常需要项目级测试，不应无条件开启并认为一定能提升性能。

### 9.3 Retainer Box（离屏缓存渲染）

Retainer Box，也称 Retainer Panel，其机制与 Invalidation Box 不一样。

它可以先把子 UI 绘制到 Render Target（渲染目标纹理），再将这张纹理合成到最终屏幕。

<img width="736" height="326" alt="image" src="https://github.com/user-attachments/assets/29b3b344-2846-47fd-9485-48e9d3e03679" />


这种机制有两个重要好处。

首先，多个子 UI 的最终屏幕合成可以简化为更少的绘制操作。

其次，可以通过 Phase 等机制控制更新频率。

例如一个不需要每帧刷新的小地图装饰界面，可能可以降低其绘制更新频率。

但 Retainer Box 不是免费优化。

它需要额外的 Render Target 内存，重新绘制缓存时也会产生额外的 GPU 工作。

如果一个复杂 UI 每帧都发生变化，那么可能每帧都需要重新渲染缓存纹理，然后再合成到屏幕，结果反而更慢。

Epic 官方建议通常优先考虑 Invalidation 缓存，在确实需要进一步降低绘制提交成本时，再评估 Retainer。

### 9.4 Invalidation Box、Retainer Box、Volatile 对比

|技术|主要作用|适合场景|
|---|---|---|
|Invalidation Box|缓存 Widget 布局与绘制数据|静态或低频变化界面|
|Global Invalidation|更大范围的失效缓存管理|复杂 UI 项目|
|Retainer Box|离屏渲染为纹理并控制刷新|复杂但低频变化的 UI|
|Volatile|高频变化元素不缓存其 Paint 数据|持续变化的动画元素|

其中 Volatile（易变 Widget）值得注意。

如果一个 Widget 每帧变化，让它不断生成、丢弃缓存可能没有意义。

可以考虑将它标记为 Volatile，使其不缓存每帧都会改变的绘制数据，同时让周围不变的 UI 继续受益于 Invalidation。

不过 Volatile 本身不是加速开关，大量 Volatile Widget 同样会增加持续更新成本。

### 9.5 UMG 的 Hidden 和 Collapsed

UMG 中常见的 Visibility 设置，也与性能有关。

|Visibility|是否显示|是否占据布局空间|
|---|---|---|
|Visible|是|是|
|Hidden|否|是|
|Collapsed|否|否|
|Hit Test Invisible|是|是，且不参与命中测试|

例如暂时隐藏一个任务提示，但希望其他 UI 的布局保持原样，可以使用 Hidden。

如果希望隐藏后让剩余子控件重新排列，则可以使用 Collapsed。

不过不要将它们看成相同的性能优化选项，因为 Collapsed 可能影响父容器布局。

## 十、UI 输入检测优化

UI 中的大量交互控件也会带来输入处理开销。

### 10.1 Unity Graphic Raycaster

Unity UGUI 通过 Graphic Raycaster 处理 UI 图形命中检测。

假设一个 Canvas 有：

```
100 个 Image
50 个 Text
20 个 Button
```

如果所有图形都启用了 Raycast Target，就可能扩大待检测的候选集合。

实际上只有 Button 等交互控件需要响应输入。

因此可以将纯装饰图片、说明文字等不需要接收输入的 Graphic 的 `Raycast Target` 关闭。

例如：

```
// 纯展示的Image不参与UI射线检测
decorativeImage.raycastTarget = false;

// 纯展示的文字也不参与UI射线检测
descriptionText.raycastTarget = false;
```

同时，不需要输入的 Canvas 没必要挂载 Graphic Raycaster。

Unity 官方也将这两点列为 UGUI 优化建议。

### 10.2 UE UMG 的输入优化

UE 可以通过 Visibility 的命中测试设置减少无意义的 UI 输入检测。

例如在一个大面板中，纯装饰层可以使用 `Not Hit-Testable (Self & All Children)`，但如果子节点有按钮，需要改用允许子节点命中的设置。

核心原则是：只有真正需要交互的 Widget 才参与命中测试。

不要为了减少输入检测开销，而错误关闭了子控件的交互功能。

## 十一、UI 内存与资源加载优化

这部分对大型 RPG、MMO 和移动游戏十分重要。

### 11.1 UI 纹理内存

UI 贴图通常包含大量 RGBA 纹理。

以一张没有压缩、没有 Mipmap 的 RGBA8 图片为例：

$$
Memory=Width\times Height\times4
$$

假设图片为 2048×2048：

$$
2048\times2048\times4=16\text{ MiB}
$$

也就是说，一张 2048×2048 的 RGBA8 UI 图片，仅未压缩的纹理像素数据就约为 16 MiB。

如果项目里有几十张这样的大图，内存占用就可能非常明显。

优化时需要结合目标平台选择纹理压缩格式。例如桌面 GPU 常用 BC 系列，移动端支持的设备可以考虑 ASTC。具体还需要权衡透明度质量、压缩损失和加载成本。

### 11.2 为什么隐藏的 UI 仍然可能占用大量内存？

以 UE UMG 为例，某个完整角色培养界面可能有很多子面板：

```
CharacterWindow
 ├── AttributePanel
 ├── WeaponPanel
 ├── SkillPanel
 ├── EquipmentPanel
 └── UpgradePanel
```

即使只显示 AttributePanel，如果整个 Widget Tree 已经构造完成，其他子 Widget 也可能早已存在于内存中。

如果蓝图还通过硬引用关联大量图片、Widget Class 和游戏资源，加载界面可能引入更大的资源依赖集合。

Epic 官方也明确提到：Widget 的子节点即使不可见，仍可能需要加载和构造。

可以改成按需创建界面：

```
打开角色界面
     ↓
加载角色主界面
     ↓
显示属性面板

玩家点击技能页
     ↓
异步请求技能页资源
     ↓
资源准备完成
     ↓
创建技能页Widget
     ↓
显示技能页
```

UE 中可以根据资源引用关系，使用 Soft Object Reference / Soft Class Reference 与异步资源加载。

Unity 项目则可以结合 Addressables、AssetBundle 或现有资源管理框架。

### 11.3 UI 缓存与销毁策略

不应该所有界面都立即销毁，也不应该所有界面都永久缓存。

例如：

|UI 类型|建议生命周期|
|---|---|
|玩家 HUD|通常常驻|
|高频伤害数字|对象池|
|频繁打开的背包|可考虑缓存|
|不常用的商城大页面|按需加载、适时释放|
|大型活动界面|按活动生命周期管理|
|临时提示弹窗|池化或轻量创建|

UI 缓存能够减少重复创建的 CPU 开销，但会增加常驻内存。

因此需要在打开延迟与内存占用之间取得平衡。

## 十二、Unity 和 UE5 的 UI 性能排查工具

开发中，不能只凭感觉判断是哪部分耗时。

### 12.1 Unity 的工具

|工具|主要用途|
|---|---|
|Unity Profiler|分析主线程、渲染、GC 和 UI 开销|
|Frame Debugger|检查 Draw Call、材质切换与 Batch|
|Memory Profiler|查看 UI 纹理、对象和内存占用|
|RenderDoc|分析 GPU 绘制与资源状态|
|Xcode / Android GPU 工具|真机分析 GPU、带宽与 Overdraw|

UGUI 中尤其值得关注 `Canvas.BuildBatch`、Canvas 更新相关的性能采样，以及布局重建所消耗的时间。

在不同 Unity 版本中，这些采样项的名称和归属可能有差异。

### 12.2 UE5 的工具

|工具或命令|主要用途|
|---|---|
|`stat unit`|查看整体 Game、Draw、GPU 等耗时|
|`stat slate`|查看 Slate 的 UI 相关统计|
|`stat gpu`|分析 GPU 耗时|
|Unreal Insights|定位 CPU 耗时、卡顿及线程行为|
|Slate Insights|查看 Widget 更新、失效和 Paint|
|Widget Reflector|检查 Widget 层级、布局和相关统计|
|RenderDoc|逐绘制事件分析 GPU 渲染|

其中 Slate Insights 特别适合排查哪些 Widget 正在反复 Invalidate，以及为什么触发更新。

### 12.3 如何判断 UI 性能瓶颈？

假设你发现打开背包后帧率明显下降。

可以按照下面的分析流程排查。

<img width="739" height="445" alt="image" src="https://github.com/user-attachments/assets/01a50678-7c2a-460c-a20e-db1357fc9b67" />


还需要特别区分两类卡顿。

打开 UI 的瞬间卡顿 通常优先检查同步加载、Widget 创建、首次文本生成、首次布局与资源上传。

UI 打开后持续掉帧 则优先检查 Tick、属性绑定、Canvas Rebuild、Overdraw、动画和周期性更新。

这两类问题的优化方向经常完全不同。

## 十三、项目案例：类《鸣潮》战斗 HUD 如何优化？

假设一个动作游戏战斗 HUD 包含如下组件：

```
BattleHUD
 │
 ├── PlayerHP
 ├── BossHP
 ├── SkillButtons
 ├── SkillCooldowns
 ├── BuffIcons
 ├── DamageNumbers
 ├── QuestTracker
 ├── MiniMap
 └── BattleTips
```

可以采用以下优化设计。

|UI 模块|更新频率|优化思路|
|---|---|---|
|玩家血条|受到伤害或恢复时|属性变化事件、平滑动画|
|Boss 血条|战斗事件驱动|独立动态层、事件刷新|
|技能图标|偶尔变化|静态缓存、共用图集|
|技能冷却|动态变化|单独隔离、简化倒计时更新|
|Buff 图标|Buff 增减时|对象池、事件驱动|
|伤害数字|高频|对象池、总数上限、批量管理|
|任务追踪|低频|缓存绘制数据|
|小地图|根据玩法更新|控制刷新范围或频率|
|战斗提示|短时显示|对象池、闲置时停止更新|

这里要特别强调几个细节。

技能冷却数字不一定需要每帧刷新。 如果文本只显示整数秒数，完全可以在显示值发生变化时更新文字；而冷却填充动画可以保持必要的平滑更新。

伤害数字不应该无上限生成。 高密度战斗中可以限制同时显示的飘字数量，对低优先级飘字进行合并或省略，避免造成 Widget 数量暴涨。

Boss UI 和普通 HUD 可以按照刷新行为隔离。 这样复杂动画和战斗事件就不容易波及整棵 UI 树。

UI 视觉效果与逻辑数据可以采用不同更新频率。 例如实际技能冷却由战斗逻辑精确管理，而显示文字只在数值发生变化时刷新，不影响技能真实判定。

## 十四、UI 优化常见面试问题

下面这些问题在 Unity / UE 客户端性能优化面试中非常值得重点准备。

1. UI Draw Call 为什么会增加？

纹理、材质、渲染状态、Mask、绘制顺序或批次边界变化，都可能导致合批被打断。

2. 什么是 UI Overdraw？如何优化？

同一个像素被多次绘制。可以减少透明层、缩小无效覆盖区域、合成静态装饰，并简化复杂 Shader。

3. Unity Canvas 为什么需要拆分？

大型 Canvas 中局部更新可能导致较大范围的批次重建分析。按照更新频率拆分可缩小重建影响范围，但也会引入额外批次边界。

4. 为什么嵌套 LayoutGroup 容易卡顿？

子元素尺寸变化可能沿父级布局链传播，触发多层布局重新计算；层级越复杂、更新越频繁，成本通常越高。

5. UI 对象池和虚拟列表有什么区别？

对象池重点降低重复创建和销毁成本；虚拟列表重点减少同时实例化的 Widget 数量。二者可以结合。

6. UE Invalidation Box 与 Retainer Box 有什么区别？

前者缓存布局和绘制信息，主要减少 CPU 重复计算；后者绘制到离屏纹理，可以控制更新频率并减少最终合成的绘制命令，但需要额外显存和渲染开销。

7. 为什么不建议频繁使用 UI Tick 或 Binding？

即使数据没有变化，也可能重复执行查询、格式化和属性设置；通常应尽量用事件通知和按需刷新替代。

8. 为什么减少 Draw Call 后帧率没有提升？

瓶颈可能在 GPU 像素填充、复杂材质、游戏逻辑、UI 重建或内存加载，Draw Call 并非唯一性能因素。

9. 打开背包时瞬间卡顿怎么分析？

重点检查同步资源加载、Widget 实例化、文本生成、布局计算、Canvas BuildBatch、纹理上传及 GC。

10. 大型商业游戏怎样组织 UI 优化？

建立性能预算和监控体系，按更新频率及生命周期分层，使用虚拟列表、资源管理与失效缓存机制，并在目标设备上反复验证。

## 十五、最终总结：UI 优化优先级

从实际项目收益来看，我建议按照以下优先级处理，而不是一开始就全面重构 UI。

|优先级|优化事项|主要解决的问题|
|---|---|---|
|P0|定位 CPU / GPU 瓶颈|避免无效优化|
|P0|移除高频无效 Tick、Binding|持续 CPU 开销|
|P0|虚拟列表、对象池|大量 Widget 和创建卡顿|
|P1|Canvas / Invalidation 管理|UI 重建开销|
|P1|Overdraw 和复杂 Shader|GPU 像素开销|
|P1|自动布局优化|布局反复计算|
|P1|UI 异步加载与缓存策略|首次打开卡顿、内存|
|P2|图集与材质合批|Draw Call、状态切换|
|P2|输入命中检测优化|多余的 UI 交互检查|
|P2|动画更新策略|无效动画计算、缓存失效|

以上优先级是复杂游戏 UI 的一个起点，实际顺序必须由 Profiler 数据决定。

真正需要掌握的核心原理是：UI 性能不是单纯的渲染问题。一个 UI 元素的变化可能引起逻辑更新、布局重算、几何重建、批次调整和 GPU 绘制。优秀的 UI 优化，就是把变化的范围和频率控制到最低，同时避免使用过多内存去换取并不明显的性能收益。

特别是针对 UE 游戏客户端高级岗位，建议深入掌握 Slate 的 Invalidation 机制、UMG 的 Widget 生命周期、Retainer Box 的离屏缓存原理，以及 Slate Insights 的性能定位方法。这些内容比单纯回答“使用图集减少 Draw Call”更有技术深度。
