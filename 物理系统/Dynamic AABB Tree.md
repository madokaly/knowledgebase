# Dynamic AABB Tree 原理详解（动态包围盒树）

Dynamic AABB Tree（动态轴对齐包围盒树）是游戏物理引擎中非常重要的一种空间加速数据结构，主要用于碰撞检测的 Broad Phase（粗检测阶段）。

它可以看成一种专门为动态物体优化的 BVH（Bounding Volume Hierarchy，层次包围体结构）。

它的核心思想是：

把场景中大量物体的 AABB 组织成一棵二叉树，利用父节点包围子节点的空间层次关系，快速排除不可能发生碰撞的物体，同时允许物体移动后高效更新树结构。

它与普通静态 BVH 最大的区别，不是树节点的数据结构，而是它对频繁插入、删除和移动物体的优化。

例如，Box2D 就使用 Dynamic AABB Tree 组织碰撞形状，支持 AABB 查询、射线检测等操作。

下面以 游戏物理碰撞检测为主线，结合 C++ 和 UE 开发场景详细解释。

## 一、为什么需要 Dynamic AABB Tree？

### 1. 普通碰撞检测的问题

假设游戏场景中有 10000 个物体：

- 100 个玩家和 NPC
    
- 300 个可以移动的物理物体
    
- 9600 个静态场景碰撞体
    

如果要判断所有物体之间是否发生碰撞，最直接的方法是两两遍历：

```
for (int i = 0; i < Objects.size(); ++i)
{
    for (int j = i + 1; j < Objects.size(); ++j)
    {
        // 判断两个物体是否可能发生碰撞
        if (CheckCollision(Objects[i], Objects[j]))
        {
            // 处理碰撞
        }
    }
}
```

复杂度为：

$$
O(N^2)
$$

10000 个物体需要进行：

$$
\frac{10000\times9999}{2}=49,995,000
$$

即约 5000 万次物体对检查。

但实际上，大多数物体相隔很远，根本没有发生碰撞的可能。

因此，物理引擎一般将碰撞检测分为两个阶段：

Broad Phase（粗检测）

通过 Dynamic AABB Tree 等空间结构，快速找到可能碰撞的物体对

Potential Collision Pairs（候选碰撞对）

例如：A-B、A-C、D-E

Narrow Phase（精检测）

使用 GJK、SAT、接触生成等算法判断实际碰撞情况

Dynamic AABB Tree 主要解决第一阶段的问题。

需要注意：

Dynamic AABB Tree 不负责精确判断两个物体是否碰撞，它只负责快速生成可能发生碰撞的候选物体对。

## 二、Dynamic AABB Tree 的基本结构

### 1. 先理解 AABB

AABB（Axis-Aligned Bounding Box，轴对齐包围盒）是一个始终与世界坐标轴平行的包围盒。

二维 AABB 可以由两个坐标确定：

```
struct AABB
{
    Vector2 Min; // 包围盒左下角
    Vector2 Max; // 包围盒右上角
};
```

三维 AABB 则包含：

```
struct AABB
{
    FVector Min; // X/Y/Z 最小值
    FVector Max; // X/Y/Z 最大值
};
```

二维下，两个 AABB 是否重叠可以直接通过坐标判断：

```
bool Overlap(const AABB& A, const AABB& B)
{
    // X轴上没有重叠
    if (A.Max.X < B.Min.X || A.Min.X > B.Max.X)
        return false;

    // Y轴上没有重叠
    if (A.Max.Y < B.Min.Y || A.Min.Y > B.Max.Y)
        return false;

    return true;
}
```

三维情况下再增加 Z 轴判断即可。

AABB 的优势是判断简单、计算成本低。

### 2. Dynamic AABB Tree 是二叉树

一个典型的 Dynamic AABB Tree 如下：

<img width="737" height="498" alt="image" src="https://github.com/user-attachments/assets/a2ee6547-22f2-4f78-bdab-97bf6fd4a2d6" />


假设 A、B、C、D 是四个物体：

- 叶子节点 A：保存物体 A 的 AABB。
    
- 叶子节点 B：保存物体 B 的 AABB。
    
- 内部节点 AB：保存能同时包围 A 和 B 的最小 AABB。
    
- 根节点：保存能包围全部四个物体的 AABB。
    

这里最关键的关系是：

$$
AABB_{Parent}=Union(AABB_{Left},AABB_{Right})
$$

Union（合并包围盒）不是简单相加，而是寻找能够同时包含两个包围盒的最小 AABB。

```
AABB Union(const AABB& A, const AABB& B)
{
    AABB Result;

    // 最小边界取两个包围盒中更小的坐标
    Result.Min.X = std::min(A.Min.X, B.Min.X);
    Result.Min.Y = std::min(A.Min.Y, B.Min.Y);

    // 最大边界取两个包围盒中更大的坐标
    Result.Max.X = std::max(A.Max.X, B.Max.X);
    Result.Max.Y = std::max(A.Max.Y, B.Max.Y);

    return Result;
}
```

### 3. 内部节点与叶子节点的区别

|节点类型|保存内容|作用|
|---|---|---|
|根节点 Root|整棵树的 AABB|快速排除整棵树|
|内部节点 Internal|两个子节点 AABB 的并集|加速分支剪枝|
|叶子节点 Leaf|单个物体的代理 AABB、物体 ID|产生候选碰撞对象|

这里还有一个非常重要的特点：

Dynamic AABB Tree 不要求不同子树的 AABB 完全不重叠。

因为 AABB 是物体空间的近似包围结构，不是严格的空间分区。

这也意味着：

- 它不需要像 KD-Tree 那样使用切分平面。
    
- 一个物体通常只需要一个叶子节点代理（Proxy）。
    
- 各层 AABB 可以重叠。
    
- 查询一个空间区域时，可能需要同时遍历多个子树。
    

## 三、Dynamic AABB Tree 最关键的优化：Fat AABB

这是理解 Dynamic AABB Tree 动态性能的关键。

### 1. 普通 AABB 的问题

假设一个物体不断移动：

<img width="737" height="296" alt="image" src="https://github.com/user-attachments/assets/880cc9af-9fc4-48af-9066-67665dc380fb" />

如果叶子节点保存的是完全紧贴物体的 AABB，那么物体每次移动：

1. 叶子节点的 AABB 发生改变。
    
2. 父节点 AABB 可能需要扩大或缩小。
    
3. 祖先节点可能都要更新。
    
4. 当前树结构可能不再适合新位置。
    

假设游戏运行在 60 FPS，有数百个物体持续运动，树结构的维护成本就可能非常高。

### 2. Fat AABB 的解决方案

Fat AABB（膨胀包围盒）是在真实 AABB 外面额外扩展一定空间。

<img width="740" height="403" alt="image" src="https://github.com/user-attachments/assets/e18e620f-4a05-4b1a-81dd-b60caeaec7fc" />


假设：

- 真实 AABB 的 X 范围为 [10, 12]。
    
- 真实 AABB 的 Y 范围为 [5, 7]。
    
- 向四周额外扩展 1 个单位。
    

那么：

```
// 真实包围盒
RealAABB.Min = {10, 5};
RealAABB.Max = {12, 7};

// Fat AABB
FatAABB.Min = {9, 4};
FatAABB.Max = {13, 8};
```

只要移动后的真实 AABB 仍然完全包含在原 Fat AABB 内，就不需要更新树节点，更不需要修改树结构。

```
bool Contains(const AABB& Outer, const AABB& Inner)
{
    return
        Outer.Min.X <= Inner.Min.X &&
        Outer.Min.Y <= Inner.Min.Y &&
        Outer.Max.X >= Inner.Max.X &&
        Outer.Max.Y >= Inner.Max.Y;
}
```

这样，一个物体连续移动很多次，都可能不触发树结构更新。

### 3. 还可以利用速度预测膨胀方向

如果物体正在向右移动，可以让 Fat AABB 向右扩大更多。

常见思路为：

$$
Displacement=Velocity\times\Delta t
$$

再按照移动方向扩展包围盒。

```
AABB CreateFatAABB(
    const AABB& RealBox,
    const Vector2& Displacement)
{
    constexpr float Margin = 0.1f;
    constexpr float PredictFactor = 2.0f;

    AABB Fat = RealBox;

    // 第一阶段：各方向固定膨胀
    Fat.Min.X -= Margin;
    Fat.Min.Y -= Margin;
    Fat.Max.X += Margin;
    Fat.Max.Y += Margin;

    // 第二阶段：沿运动方向预测膨胀
    Vector2 D = Displacement * PredictFactor;

    if (D.X < 0)
        Fat.Min.X += D.X;
    else
        Fat.Max.X += D.X;

    if (D.Y < 0)
        Fat.Min.Y += D.Y;
    else
        Fat.Max.Y += D.Y;

    return Fat;
}
```

注意：这是经典动态树的一种常见实现策略，不代表所有引擎都使用相同的膨胀系数。

### 4. Fat AABB 的代价

Fat AABB 不是越大越好。

|Fat AABB 大小|优点|缺点|
|---|---|---|
|较小|检测更加精确，误报少|更新频繁|
|较大|更新次数少|候选碰撞对增加|
|适中|平衡更新成本与查询成本|需要根据场景调整|

例如两个实际相隔较远的物体，如果 Fat AABB 发生重叠，Broad Phase 也可能认为它们是候选碰撞对。

但这不会直接导致错误的物理碰撞结果，因为后续仍然需要准确的碰撞检测。

Fat AABB 的本质是以一定的空间精度换取动态更新性能。 Box2D 2.4 的动态树文档也明确描述了通过膨胀代理 AABB 避免微小移动引起重新插入的机制。

## 四、Dynamic AABB Tree 如何插入新物体？

Dynamic AABB Tree 的插入算法是面试中比较重要的知识点。

它与普通二叉搜索树不同：

- 二叉搜索树根据 Key 大小决定插入左子树还是右子树。
    
- Dynamic AABB Tree 根据 插入后包围盒的空间代价，决定新物体应该放在哪里。
    

### 1. 插入过程

假设当前树中有 A、B、C、D 四个物体，现在需要插入 E。

<img width="739" height="499" alt="image" src="https://github.com/user-attachments/assets/62eef397-8765-43bd-b602-87cc8c779af0" />


插入的核心过程分为四步：

1. 从根节点开始，寻找最合适的兄弟节点（Sibling）。
    
2. 创建一个新的内部父节点，包含新叶子 E 和找到的兄弟节点。
    
3. 将新父节点连接到原树中。
    
4. 从插入位置向根节点回溯，更新 AABB 并进行必要的树旋转。
    

### 2. 如何确定插入位置？

核心策略通常称为：

SAH（Surface Area Heuristic，表面积启发式）或基于包围盒周长、面积的代价启发式。

它希望让树中包围盒的总体空间代价尽可能小，从而减少未来碰撞查询需要访问的节点数量。

假设有两个 AABB：

$$
AABB_A,\ AABB_B
$$

合并后：

$$
AABB_C=Union(AABB_A,AABB_B)
$$

二维情况下，可以使用周长衡量合并代价：

$$
P(AABB)=2(W+H)
$$

其中 W、H 分别表示包围盒的宽和高。

例如：

原来 AABB 的尺寸是 \(2\times2\)，周长为 8。

与新 AABB 合并后尺寸变为 \(3\times2\)，周长为 10。

那么：

$$
\Delta P=10-8=2
$$

相比之下，如果另一分支合并后周长增加 20，那么通常优先考虑前一个分支。

为什么使用周长或表面积？

因为一个包围盒越大，它与随机查询对象相交的可能性通常也越高。通过控制树中包围盒的面积或周长，可以降低查询时访问无关节点的概率。

不过需要注意：

- 二维经典 Box2D 动态树中常用 AABB 周长作为代价指标。
    
- 三维 BVH 中常用表面积作为 SAH 代价指标。
    
- 不能简单认为所有实现都只比较当前节点扩大的面积。
    

实际插入算法还会考虑新建父节点的代价，以及向下遍历引起的祖先包围盒扩大成本。

### 3. 插入代价的详细计算

经典 Box2D 风格的贪心插入，假设：

- N 是当前内部节点。
    
- A、B 是 N 的两个子节点。
    
- E 是需要插入的新叶子节点。
    

需要比较三个选择：

<img width="738" height="285" alt="image" src="https://github.com/user-attachments/assets/83bd0f33-f3d6-4af6-a804-c8773a73da9c" />


设：

$$
U=Union(AABB_N,AABB_E)
$$

那么一种经典代价计算方式是：

方案一：直接在当前节点处创建新父节点

$$
Cost_{Stop}=2P(U)
$$

方案二：继续向 A 子树下降

$$
Cost_A= \begin{cases} P(Union(A,E))+I,& A\text{为叶子}\\ P(Union(A,E))-P(A)+I,& A\text{为内部节点} \end{cases}
$$

方案三：继续向 B 子树下降

$$
Cost_B= \begin{cases} P(Union(B,E))+I,& B\text{为叶子}\\ P(Union(B,E))-P(B)+I,& B\text{为内部节点} \end{cases}
$$

其中：

$$
I=2(P(U)-P(N))
$$

表示继续向下插入时，对上层节点产生的继承代价（Inheritance Cost）。

这些公式是经典 Box2D 版本的一种启发式代价模型，并不是所有 Dynamic AABB Tree 唯一的插入公式。

如果：

$$
Cost_{Stop}<Cost_A
$$

且：

$$
Cost_{Stop}<Cost_B
$$

就停止搜索，把当前节点作为新叶子的兄弟节点。

否则选择代价更小的子树继续下降。

这是一种局部贪心优化，并不保证最终获得全局最优 BVH。

### 4. C++ 插入核心代码

下面采用二维 AABB 演示，实际三维代码只需要调整包围盒计算及对应代价函数。

```
// 节点结构
struct TreeNode
{
    AABB Box;

    int Parent = -1;
    int Left = -1;
    int Right = -1;

    int Height = 0;

    // 只有叶子节点使用此字段
    int ObjectId = -1;

    bool IsLeaf() const
    {
        return Left == -1;
    }
};

// 以下为核心伪代码：
// Nodes 是节点池，Root 为根节点索引。
// AllocateNode / Balance / Perimeter / Union 为辅助函数。

void InsertLeaf(int Leaf)
{
    // 1. 空树：直接将新叶子设为根节点
    if (Root == -1)
    {
        Root = Leaf;
        Nodes[Leaf].Parent = -1;
        return;
    }

    // 2. 寻找代价最小的兄弟节点
    int Index = Root;
    AABB LeafBox = Nodes[Leaf].Box;

    while (!Nodes[Index].IsLeaf())
    {
        int Left  = Nodes[Index].Left;
        int Right = Nodes[Index].Right;

        float OldCost = Perimeter(Nodes[Index].Box);

        AABB MergedBox =
            Union(Nodes[Index].Box, LeafBox);

        float NewCost = Perimeter(MergedBox);

        // 当前节点与新叶子成为兄弟
        float StopCost = 2.0f * NewCost;

        // 向下插入将导致上层 AABB 扩大的额外代价
        float InheritanceCost =
            2.0f * (NewCost - OldCost);

        // 计算继续进入某个子节点的代价
        auto ChildCost = [&](int Child)
        {
            AABB Combined =
                Union(Nodes[Child].Box, LeafBox);

            if (Nodes[Child].IsLeaf())
            {
                // 叶子节点需要增加一个新的父节点
                return Perimeter(Combined)
                     + InheritanceCost;
            }

            // 内部节点只计算 AABB 扩大带来的增量
            return Perimeter(Combined)
                 - Perimeter(Nodes[Child].Box)
                 + InheritanceCost;
        };

        float LeftCost = ChildCost(Left);
        float RightCost = ChildCost(Right);

        // 在当前节点插入更便宜
        if (StopCost < LeftCost &&
            StopCost < RightCost)
        {
            break;
        }

        // 否则继续进入代价更小的子树
        Index = (LeftCost < RightCost)
              ? Left : Right;
    }

    // 3. 已找到合适的兄弟节点
    int Sibling = Index;

    // 记录原父节点
    int OldParent = Nodes[Sibling].Parent;

    // 分配新的内部父节点
    int NewParent = AllocateNode();

    Nodes[NewParent].Parent = OldParent;
    Nodes[NewParent].Left = Sibling;
    Nodes[NewParent].Right = Leaf;

    Nodes[NewParent].Box =
        Union(Nodes[Sibling].Box, LeafBox);

    Nodes[NewParent].Height =
        Nodes[Sibling].Height + 1;

    // 将兄弟节点和新叶子挂到新父节点下面
    Nodes[Sibling].Parent = NewParent;
    Nodes[Leaf].Parent = NewParent;

    // 4. 把新父节点接回原树
    if (OldParent == -1)
    {
        Root = NewParent;
    }
    else
    {
        if (Nodes[OldParent].Left == Sibling)
            Nodes[OldParent].Left = NewParent;
        else
            Nodes[OldParent].Right = NewParent;
    }

    // 5. 自底向上更新并平衡
    Index = NewParent;

    while (Index != -1)
    {
        // 旋转操作可能会改变本子树的根节点
        Index = Balance(Index);

        int L = Nodes[Index].Left;
        int R = Nodes[Index].Right;

        // 更新父节点包围盒
        Nodes[Index].Box =
            Union(Nodes[L].Box, Nodes[R].Box);

        // 更新树高
        Nodes[Index].Height =
            1 + std::max(
                Nodes[L].Height,
                Nodes[R].Height
            );

        Index = Nodes[Index].Parent;
    }
}
```

这个函数省略了节点内存池和 `Balance()` 的具体实现，但保留了完整的核心插入流程。

## 五、Dynamic AABB Tree 如何删除物体？

删除过程比插入简单。

假设树是：

```
          Root
         /    \
        P      C
       / \
      A   B
```

现在要删除 A。

### 1. 删除步骤

首先找到 A 的父节点 P，以及 A 的兄弟节点 B。

然后：

1. 删除叶子节点 A。
    
2. 删除原父节点 P。
    
3. 将兄弟节点 B 直接连接到 P 的父节点 Root。
    
4. 从 Root 开始向上更新 AABB 和树高。
    
5. 根据需要进行旋转平衡。
    

删除后：

```
          Root
         /    \
        B      C
```

为什么需要同时删除 P？

因为 Dynamic AABB Tree 通常要求每个内部节点都有两个子节点。

如果只删除 A，P 将只剩 B 一个子节点，此时 P 就变得多余。

### 2. C++ 删除核心代码

```
void RemoveLeaf(int Leaf)
{
    // 如果删除的是唯一的根节点
    if (Leaf == Root)
    {
        Root = -1;
        Nodes[Leaf].Parent = -1;
        return;
    }

    int Parent = Nodes[Leaf].Parent;
    int GrandParent = Nodes[Parent].Parent;

    // 找到当前叶子的兄弟节点
    int Sibling =
        (Nodes[Parent].Left == Leaf)
        ? Nodes[Parent].Right
        : Nodes[Parent].Left;

    if (GrandParent != -1)
    {
        // 将兄弟节点直接挂到祖父节点下
        if (Nodes[GrandParent].Left == Parent)
            Nodes[GrandParent].Left = Sibling;
        else
            Nodes[GrandParent].Right = Sibling;

        Nodes[Sibling].Parent = GrandParent;

        // 删除已经不需要的内部父节点
        FreeNode(Parent);

        // 从祖父节点开始，向上更新
        int Index = GrandParent;

        while (Index != -1)
        {
            Index = Balance(Index);

            int L = Nodes[Index].Left;
            int R = Nodes[Index].Right;

            Nodes[Index].Box =
                Union(Nodes[L].Box, Nodes[R].Box);

            Nodes[Index].Height =
                1 + std::max(
                    Nodes[L].Height,
                    Nodes[R].Height
                );

            Index = Nodes[Index].Parent;
        }
    }
    else
    {
        // 原父节点就是根
        // 兄弟节点直接成为新的根节点
        Root = Sibling;
        Nodes[Sibling].Parent = -1;

        FreeNode(Parent);
    }

    Nodes[Leaf].Parent = -1;
}
```

这里的 `RemoveLeaf` 只是把叶子从树结构中摘下来，不一定会释放叶子自身。

这是刻意设计的，因为后续物体移动时，还需要重新把同一个叶子插回树中。

真正销毁物体时，通常还需要另外释放叶子节点对应的 Proxy。

## 六、物体移动时 Dynamic AABB Tree 如何更新？

这是动态树最重要的实际功能之一。

### 1. 物体移动并不意味着立即修改树

假设原来的 Fat AABB 为：

```
X: [0, 10]
Y: [0, 10]
```

物体真实 AABB 原本为：

```
X: [3, 5]
Y: [3, 5]
```

移动后变成：

```
X: [4, 6]
Y: [3, 5]
```

因为移动后的真实 AABB 仍然被 Fat AABB 包含，所以不需要修改树。

但是如果移动后变为：

```
X: [9, 11]
Y: [3, 5]
```

新的真实 AABB 超出了 Fat AABB 的右边界。

这时就需要更新。

### 2. 超出 Fat AABB 后怎么办？

经典策略并不是将所有祖先节点的包围盒简单扩大。

因为物体可能已经从原来的空间区域移动到了另一个很远的位置。

继续保留原来的树结构，可能导致大量内部包围盒被异常拉大。

因此，通常采用：

<img width="741" height="254" alt="image" src="https://github.com/user-attachments/assets/87639b44-72ee-4626-8a1a-7e994f5678a6" />


核心代码：

```
bool MoveProxy(
    int ProxyId,
    const AABB& NewRealAABB,
    const Vector2& Displacement)
{
    TreeNode& Node = Nodes[ProxyId];

    // 仍在原来的 Fat AABB 内
    // 不需要重新插入
    if (Contains(Node.Box, NewRealAABB))
    {
        return false;
    }

    // 1. 从树中移除叶子
    RemoveLeaf(ProxyId);

    // 2. 计算新的 Fat AABB
    Nodes[ProxyId].Box =
        CreateFatAABB(
            NewRealAABB,
            Displacement
        );

    // 3. 在新位置重新寻找最优插入位置
    InsertLeaf(ProxyId);

    return true;
}
```

这类机制通常叫作：

Reinsertion（重新插入）

注意，重新插入的是叶子节点，不一定重新分配物体或叶子内存。

另外，具体 API 会因版本不同而变化。例如经典 Box2D 2.4 的 `MoveProxy` 会检查是否需要重新插入，而当前 Box2D 3.2 的 `b2DynamicTree_MoveProxy` 文档描述的是直接执行移除和重新插入；调用方可以在外部组织更新策略。

## 七、Dynamic AABB Tree 如何保持平衡？

如果只考虑插入和删除，而不进行平衡操作，树可能逐渐退化。

### 1. 为什么树的高度很重要？

理想情况下，具有 N 个叶子节点的树高度接近：

$$
H=O(\log_2 N)
$$

例如 N=1024：

$$
\log_2 1024=10
$$

如果树结构平衡，查询通常可以沿较短的树路径找到对应区域。

但如果树退化：

```
       A
        \
         B
          \
           C
            \
             D
              \
               E
                \
                 F
```

树高可能接近 N。

此时搜索一条深层路径就可能需要访问大量节点。

不过这里需要区分：

树平衡能改善查询效率，但即使高度为 O(log N)，也不能保证每次碰撞区域查询都是 O(log N)。

因为查询可能同时命中多个分支，访问数量还取决于包围盒的重叠程度。

### 2. 使用 Rotation（树旋转）

Dynamic AABB Tree 可以通过局部树旋转改善平衡。

例如：

<img width="738" height="365" alt="image" src="https://github.com/user-attachments/assets/25b37382-be51-40fa-a2d1-e16d9047d6cb" />


旋转前：

```
        A
       / \
      B   C
         / \
        F   G
```

旋转后：

```
        C
       / \
      A   G
     / \
    B   F
```

两棵树包含的叶子节点完全相同。

改变的是：

- 内部节点之间的父子关系。
    
- 节点的树高。
    
- 内部节点对应的 AABB。
    
- 后续查询需要访问的节点数量。
    

### 3. 旋转后为什么需要更新 AABB？

旋转前：

$$
AABB_A=Union(AABB_B,AABB_C)
$$

旋转后：

$$
AABB_A'=Union(AABB_B,AABB_F)
$$

同时：

$$
AABB_C'=Union(AABB_A',AABB_G)
$$

所以每次旋转后，都需要重新计算受到影响的内部节点 AABB。

否则父节点就无法准确包围其子节点。

### 4. 旋转判断条件

一种经典做法是计算两个子树的高度差：

$$
Balance=Height(Right)-Height(Left)
$$

例如：

```
int Balance = RightHeight - LeftHeight;

if (Balance > 1)
{
    // 右子树明显更高
    // 尝试左旋
}
else if (Balance < -1)
{
    // 左子树明显更高
    // 尝试右旋
}
```

但这里有一个重要区别：

普通 AVL 树主要考虑高度。

而 Dynamic AABB Tree 的旋转，除了高度，还可能考虑旋转前后包围盒的面积或周长。

因为两个高度相同的树，其空间查询效率可能完全不同。

例如：

- 树 A：左右子树的 AABB 空间重叠很少。
    
- 树 B：左右子树的 AABB 大量重叠。
    

即使两棵树高度一样，树 A 在很多查询情况下仍然更高效。

因此更进一步的空间优化策略是比较旋转前后的空间代价：

$$
Cost_{\text{before}} \quad \text{与} \quad Cost_{\text{after}}
$$

只执行有利于所选优化目标的局部变换。

经典高度平衡策略和面积优化策略是可以结合使用的不同手段，不应将它们理解为所有动态树实现都完全相同。

## 八、Dynamic AABB Tree 如何进行碰撞查询？

这是整个算法在 Broad Phase 中最核心的应用。

假设我们已经构建了一棵包含 A、B、C、D 的树：

```
               Root
              /    \
            Node1  Node2
            /  \    /  \
           A    B  C    D
```

现在物体 E 需要查找可能碰撞的物体。

### 1. 从根节点开始遍历

首先检查：

$$
Overlap(AABB_E,AABB_{Root})
$$

如果没有重叠：

整棵树都不需要继续搜索。

因为 Root 包含了树中全部叶子的 AABB。

如果重叠，则继续检查 Node1、Node2。

例如：

- E 与 Node1 不重叠：A、B 可以全部排除。
    
- E 与 Node2 重叠：继续检查 C、D。
    
- E 与 C 重叠：把 C 加入候选碰撞结果。
    
- E 与 D 不重叠：直接跳过 D。
    

最终只需要进一步检测 E 与 C。

这就是 BVH 树最重要的剪枝机制：

父节点不相交，则整个子树一定不相交。

反过来则不成立：

父节点相交，并不意味着它的叶子节点一定相交。

### 2. C++ 查询实现

实际中一般使用显式栈进行遍历，避免深层递归开销。

```
template<typename Callback>
void Query(
    const AABB& QueryBox,
    Callback&& OnHit)
{
    if (Root == -1)
        return;

    // 使用栈保存待检查的节点
    std::vector<int> Stack;
    Stack.push_back(Root);

    while (!Stack.empty())
    {
        int Index = Stack.back();
        Stack.pop_back();

        const TreeNode& Node = Nodes[Index];

        // 如果查询区域与当前节点 AABB 不重叠
        // 那么整个子树都可以跳过
        if (!Overlap(QueryBox, Node.Box))
        {
            continue;
        }

        if (Node.IsLeaf())
        {
            // 找到一个候选碰撞物体
            // 注意：不是已经确定发生真实碰撞
            OnHit(Node.ObjectId);
        }
        else
        {
            // 当前是内部节点
            // 继续检查两个子节点
            Stack.push_back(Node.Left);
            Stack.push_back(Node.Right);
        }
    }
}
```

这段代码非常重要，它体现了 Dynamic AABB Tree 的主要加速原理。

对于一个空间分布比较合理的场景，查询通常只需要访问一小部分节点。

不过返回的候选结果仍然可能包括 Fat AABB 重叠、真实碰撞体不重叠的情况。

## 九、如何利用 Dynamic AABB Tree 生成碰撞对？

前面介绍了对单个 AABB 进行查询，那么物理引擎每帧如何利用它找出所有潜在的碰撞对？

### 1. Move Buffer（移动缓冲区）

一种经典设计是维护一个发生了有效代理更新的物体列表。

例如：

```
std::vector<int> MoveBuffer;
```

当物体新建、或运动超出了原来的 Fat AABB，需要重新插入时，可以把代理 ID 加入 MoveBuffer：

```
void UpdateObject(
    int ProxyId,
    const AABB& NewBox,
    const Vector2& Displacement)
{
    bool Reinserted =
        MoveProxy(ProxyId, NewBox, Displacement);

    if (Reinserted)
    {
        // 记录需要重新搜索候选碰撞对的代理
        MoveBuffer.push_back(ProxyId);
    }
}
```

每次物理更新，可以对这些代理执行空间查询。

```
void UpdatePairs()
{
    std::vector<std::pair<int, int>> PairBuffer;

    // 查询本次需要更新的代理
    for (int ProxyId : MoveBuffer)
    {
        AABB QueryBox = Nodes[ProxyId].Box;

        Query(QueryBox, [&](int OtherId)
        {
            // 跳过自己
            if (OtherId == ProxyId)
                return;

            // 对物体 ID 排序，统一碰撞对表示
            int A = std::min(ProxyId, OtherId);
            int B = std::max(ProxyId, OtherId);

            PairBuffer.emplace_back(A, B);
        });
    }

    // 对候选碰撞对排序
    std::sort(PairBuffer.begin(), PairBuffer.end());

    // 去除重复的碰撞对
    PairBuffer.erase(
        std::unique(PairBuffer.begin(), PairBuffer.end()),
        PairBuffer.end()
    );

    // 将候选对交给后续碰撞管理系统
    for (const auto& Pair : PairBuffer)
    {
        ProcessPotentialPair(Pair.first, Pair.second);
    }

    MoveBuffer.clear();
}
```

以上是 Broad Phase 的简化示例，完整物理系统还需要碰撞过滤、代理有效性检查、持续接触管理等功能。

### 2. 为什么可以不对所有物体重新进行两两查询？

因为 Dynamic AABB Tree 中保存的是膨胀后的包围盒。

例如 A、B 的 Fat AABB 已经重叠，但真实碰撞体还没有相交。

此时系统就可以提前建立或保留一个潜在碰撞关系。

之后即使 A、B 在各自的 Fat AABB 内移动，也可以通过持续维护的碰撞关系检查实际接触，而不一定每帧重新从树中发现这一对物体。

这里必须注意一个容易混淆的地方：

Move Buffer 是增量发现候选碰撞对的优化手段，并不意味着 Narrow Phase 只检查 Move Buffer 中的物体。

实际系统还需要维护 Persistent Contact（持续接触）或者 Pair Cache（碰撞对缓存）。

否则仅依靠重新插入时的查询，可能导致接触状态无法正确更新。

### 3. 实际物理引擎中的完整流程

```
                  每帧物理更新
                       |
                       v
                更新物体 Transform
                       |
                       v
                计算最新真实 AABB
                       |
                       v
              更新 Dynamic AABB Tree
                       |
                       v
              查询需要更新的代理
                       |
                       v
                生成候选碰撞对
                       |
                       v
                 碰撞过滤处理
          （Layer、Mask、忽略关系等）
                       |
                       v
               更新碰撞对/接触缓存
                       |
                       v
                   Narrow Phase
              （SAT / GJK 等算法）
                       |
                       v
                  生成接触信息
                       |
                       v
                  物理约束求解
```

这里的重点是 Dynamic AABB Tree 只是碰撞检测流水线中的一环。

它既不需要知道复杂碰撞体的具体几何结构，也不直接负责碰撞响应。

## 十、Dynamic AABB Tree 如何加速 Raycast？

Dynamic AABB Tree 不仅可以做物理粗检测，也适合空间查询。

例如：

- 武器射线检测。
    
- 鼠标点击选中物体。
    
- AI 视线检测。
    
- 查询某个区域内的物体。
    
- Shape Cast（形状扫掠查询）。
    

### 1. 射线检测为什么能加速？

假设场景中有 10000 个物体。

普通做法：

```
for (const auto& Object : Objects)
{
    Raycast(Object);
}
```

需要考虑所有物体。

使用 Dynamic AABB Tree 后：

1. 从根节点开始判断射线是否与 AABB 相交。
    
2. 如果不相交，直接跳过整个子树。
    
3. 如果相交，继续检查子节点。
    
4. 当射线与叶子 AABB 相交时，再执行真正的几何射线检测。
    

这样可以大幅减少对实际碰撞形状进行精确检测的次数。

Box2D 也提供了对应的树级射线查询和扫掠盒查询接口。

### 2. 射线与 AABB 相交：Slab Method

通常使用 Slab Method（平板法）判断射线是否与 AABB 相交。

射线公式：

$$
P(t)=Origin+t\cdot Direction
$$

对每个坐标轴分别计算射线进入和离开包围盒的参数。

以 X 轴为例：

$$
t_1=\frac{MinX-OriginX}{DirectionX}
$$

$$
t_2=\frac{MaxX-OriginX}{DirectionX}
$$

然后对 X、Y、Z 三个轴得到的时间区间求交集。

如果：

$$
t_{\text{Enter}}\le t_{\text{Exit}}
$$

且相交区间与有效射线参数区间有交集，就认为射线与 AABB 相交。

对于接近零的方向分量，需要额外处理平行情况，避免除以零。

### 3. 进一步优化

如果目标是寻找距离最近的命中物体，就可以维护目前最近的碰撞距离：

```
float ClosestDistance = MaxDistance;
```

当找到一个更近的命中点后，缩短允许的射线长度。

这样，后续那些只与射线更远部分相交的 AABB 就可以被跳过。

如果对子节点按预计进入距离从近到远进行遍历，还能提高这种剪枝的效果。

## 十一、Dynamic AABB Tree 的时间复杂度

这里要特别注意：树的高度复杂度不等于查询的严格时间复杂度。

|操作|平衡良好时的典型情况|最坏情况|
|---|---|---|
|插入物体|O(log N)|O(N)|
|删除物体|O(log N)|O(N)|
|更新物体（仍在 Fat AABB 内）|O(1)|O(1)|
|更新物体（需要重新插入）|O(log N)|O(N)|
|单次 AABB 查询|与访问节点数及结果数有关|O(N)|
|单次 Raycast|与访问节点数及命中数有关|O(N)|

其中 N 是物体代理数量。

查询速度主要取决于：

- 树的高度。
    
- 各个内部节点 AABB 的重叠程度。
    
- 查询区域大小。
    
- 实际返回的候选物体数量。
    

例如查询区域覆盖整棵树，即使树非常平衡，仍然需要访问全部叶子节点。

另外，若查询产生 K 个候选结果，那么仅输出结果本身就至少需要 O(K) 的工作。

因此，不能简单说 Dynamic AABB Tree 查询必然为 O(log N)。

### 节点的空间复杂度

对于每个内部节点恰好有两个子节点的二叉树：

假设有 N 个叶子：

$$
InternalNodes=N-1
$$

所以总节点数量为：

$$
TotalNodes=2N-1
$$

整个树结构的空间复杂度为：

$$
O(N)
$$

除了节点存储外，通常还需要少量额外空间保存空闲节点链表、查询栈、移动缓冲区等。

## 十二、Dynamic AABB Tree 与其他空间加速结构有什么区别？

这是游戏引擎和物理引擎面试中非常常见的对比题。

|算法|核心思路|动态更新|典型适用场景|
|---|---|---|---|
|普通静态 BVH|层次包围盒|更新代价可能较高|静态模型、静态几何查询|
|Dynamic AABB Tree|动态层次 AABB、局部更新及旋转|较好|动态物理物体粗检测|
|KD-Tree|使用轴对齐平面递归切分|通常不擅长频繁大范围变化|静态场景空间查询|
|Octree（八叉树）|递归八分空间|中等，取决于实现|大范围三维空间管理|
|Uniform Grid（均匀网格）|固定大小空间单元|通常较好|密度较均匀、物体尺寸接近的场景|
|Sweep and Prune（扫描与裁剪）|轴向区间排序与重叠检测|对运动连续的场景较好|物理 Broad Phase|

### 1. 与普通 BVH 的区别

Dynamic AABB Tree 本身就是 BVH 的一种。

不能理解为两种完全独立的数据结构。

区别主要在构建及维护方式：

静态 BVH

通常侧重：

- 构建高质量树结构。
    
- 尽量减少节点 AABB 的重叠。
    
- 提高长期静态查询效率。
    

Dynamic AABB Tree

通常侧重：

- 支持实时插入与删除。
    
- 利用 Fat AABB 降低更新频率。
    
- 通过局部旋转维护树结构质量。
    
- 避免每次移动都全量重建树。
    

所以 Dynamic AABB Tree 可以理解为针对高动态性而设计的 BVH。

### 2. 与 KD-Tree 的区别

KD-Tree 是空间划分结构，它使用分割平面划分空间。

例如：

```
空间
 |
 |--- X轴切分
 |      |
 |      |--- Y轴切分
 |      |
 |      |--- Y轴切分
```

而 Dynamic AABB Tree 不依靠固定切分平面，而是根据物体的 AABB 进行分组。

两者最关键的区别：

KD-Tree 更偏向空间划分，而 Dynamic AABB Tree 更偏向物体分组。

对于经常移动的游戏物理对象，Dynamic AABB Tree 可以通过重新插入一个叶子来调整空间组织方式。

KD-Tree 如果采用传统静态构建方案，频繁移动可能需要更新大量分组关系，甚至重新构建局部或整个树。

但这也不意味着所有动态 KD-Tree 都不能处理移动物体，而是经典静态 KD-Tree 通常不是这类物理 Broad Phase 的首选。

### 3. 与 Sweep and Prune 的区别

SAP（Sweep and Prune）通过对物体在某条坐标轴上的投影区间进行排序，寻找重叠区间。

例如：

```
X轴：

A:  [------]
B:      [------]
C:                   [------]
D:                      [------]
```

可以快速知道：

- A 和 B 的 X 轴区间重叠。
    
- C 和 D 的 X 轴区间重叠。
    
- A 与 C、D 不重叠。
    

随后再检查其他坐标轴。

SAP 对物体运动连续、排序变化比较小的场景通常很有效。

而 Dynamic AABB Tree 更依赖空间包围盒层次，适用于需要频繁执行局部空间查询、物体动态插入删除的情况。

二者没有绝对的优劣，需要根据物体数量、大小差异、运动特征及空间分布进行选择。

## 十三、实际游戏项目中的优化方向

Dynamic AABB Tree 虽然适合动态场景，但仍然有几个值得注意的优化点。

### 1. 静态与动态物体分开管理

在有大量静态场景物体和少量动态角色的游戏中，可以将它们分别组织。

例如：

```
             Broad Phase
                  |
         -------------------
         |                 |
     Static Tree       Dynamic Tree
         |                 |
    静态环境物体          玩家、怪物
    地形、墙壁            动态物理物体
    静态障碍物            投射物
```

这样可以减少动态对象更新对静态场景数据的影响。

但这只是常见架构选择，并不代表所有引擎都会严格使用两棵 Dynamic AABB Tree。

一些实现可能对静态碰撞、运动学物体和动态物理体采用不同的加速结构。

### 2. 合理选择 Fat AABB 膨胀系数

Fat AABB 较大，更新次数会减少，但误报可能增加。

Fat AABB 较小，误报减少，但重插入可能增加。

因此膨胀系数通常需要结合以下因素调节：

- 物体尺寸。
    
- 最大运动速度。
    
- 物理步长。
    
- 物体是否具有较高的运动连续性。
    

例如，对于高速飞行的子弹，单纯使用很大的 Fat AABB 并不是理想方案。

更重要的是结合 Swept AABB（扫掠包围盒）或 CCD（Continuous Collision Detection，连续碰撞检测），避免物体穿透。

### 3. 避免过大的内部节点 AABB

假设某个内部节点原本只包含一小片区域内的物体。

如果新插入一个特别远的物体，就可能导致这个节点的 AABB 跨越整个场景。

这样的包围盒在查询时很容易与不相关区域发生重叠，降低剪枝效果。

因此插入策略不能只考虑：

- 树是否平衡。
    
- 左右节点数量是否接近。
    

还要考虑空间面积、周长和整体重叠。

### 4. 使用节点内存池

游戏物理中频繁创建和销毁物体是很常见的。

如果每次插入节点都调用：

```
new TreeNode();
```

每次删除都调用：

```
delete Node;
```

可能带来频繁的堆分配和内存碎片问题。

因此通常可以使用：

```
std::vector<TreeNode> Nodes;
```

配合：

```
std::vector<int> FreeNodes;
```

维护节点池。

查询和树旋转使用节点索引，而不是依赖不稳定的指针。

这样也有利于实现更紧凑的内存布局和提高缓存命中率。

Box2D 的动态树同样使用节点池和索引管理节点。

### 5. 监控树结构质量

实际调优时，可以关注三个指标：

Tree Height（树高）

用来判断树是否明显退化。

Area Ratio（面积比）

常见定义是：

$$
AreaRatio= \frac{\sum Area(Node_i)} {Area(Root)}
$$

它可以用于观察树中包围盒面积的累计开销。具体统计节点范围和面积定义应以实际实现为准。

数值越高通常说明空间组织成本越高，但不是唯一的性能判断标准。

Node Visits（查询访问节点数）

统计一次查询到底访问了多少内部节点。

这个指标对判断树查询是否高效通常非常直观。

与之配合，还可以统计每帧：

- 代理重新插入数量。
    
- 候选碰撞对数量。
    
- 被 Narrow Phase 排除的候选对数量。
    
- Broad Phase 总耗时。
    

## 十四、结合 UE 游戏客户端开发理解

如果把这些原理放到 UE 类动作游戏的开发场景中，可以用以下示例理解。

场景中存在：

- 一个玩家角色。
    
- 50 个 AI 怪物。
    
- 200 个动态物理道具。
    
- 数千个静态场景碰撞体。
    

玩家释放范围技能时，需要获取附近可能受到攻击的物体。

一个基于 AABB 树的简化处理流程可以是：

```
// 1. 技能范围，例如球形攻击范围
FVector SkillCenter = PlayerLocation;
float SkillRadius = 500.0f;

// 2. 用球形攻击范围生成查询 AABB
FVector Extent(SkillRadius);

FBox QueryBounds(
    SkillCenter - Extent,
    SkillCenter + Extent
);

// 3. 使用空间树查询候选对象
DynamicTree.Query(QueryBounds, [&](int ObjectId)
{
    // 4. 获取真正的目标信息
    auto* Target = FindObject(ObjectId);

    if (!Target)
        return;

    // 5. 判断目标是否在球形攻击范围内
    float DistanceSquared =
        FVector::DistSquared(
            Target->GetLocation(),
            SkillCenter
        );

    if (DistanceSquared > SkillRadius * SkillRadius)
        return;

    // 6. 继续判断阵营、碰撞形状等
    // 然后交给技能战斗逻辑
    ProcessSkillTarget(Target);
});
```

这里是为了说明原理而写的伪代码，`DynamicTree.Query` 不是 UE 原生 API。

在 UE5 中，通常应优先使用引擎提供的物理查询接口，例如：

- `UWorld::OverlapMultiByChannel`
    
- `UWorld::SweepMultiByChannel`
    
- `UWorld::LineTraceMultiByChannel`
    

UE5 的 Chaos 物理系统内部有自己的空间加速与碰撞检测实现，具体使用何种加速结构、如何组合静态和动态数据，取决于对应系统及引擎版本。

不要直接认为 UE5 所有碰撞查询都通过同一棵经典 Box2D 风格的 Dynamic AABB Tree 完成。

## 十五、面试重点总结

如果面试官问：

“Dynamic AABB Tree 的原理是什么？为什么它适合动态场景？”

可以这样回答：

Dynamic AABB Tree 是一种动态 BVH，主要用于碰撞检测 Broad Phase。

它通过二叉树组织物体的 AABB，其中叶子节点代表物体，内部节点保存子节点 AABB 的并集。

它的核心优化有四点：

1. 空间层次剪枝：如果查询 AABB 与某个内部节点不相交，就可以直接排除整个子树，避免大量无意义的碰撞检测。
    
2. Fat AABB：对真实物体包围盒进行适当膨胀，让物体在一定范围内移动时不需要更新树结构。
    
3. 动态重新插入：如果物体超出 Fat AABB，就从原位置移除叶子，根据包围盒合并代价重新选择插入位置。
    
4. 树旋转与局部优化：维护树的高度与包围盒空间质量，避免大量动态操作导致查询效率下降。
    

它的优点是动态插入、删除、更新相对高效，支持碰撞区域查询与 Raycast，并且不需要每帧全量重建。

缺点是需要额外维护树结构；大量高速移动物体会增加重新插入开销；Fat AABB 和内部节点重叠又会带来一定的候选碰撞误报。

最值得记住的一句话：Dynamic AABB Tree 的优化目标不是让树单纯地保持平衡，而是在动态维护成本和空间查询效率之间取得平衡。

如果继续深入源码实现，最值得重点研究的三个函数是 `InsertLeaf()`、`Balance()` 和 `MoveProxy()`。理解了它们，基本就掌握了经典 Dynamic AABB Tree 的核心算法。
