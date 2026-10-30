# BVH（Bounding Volume Hierarchy，层次包围体）原理详解

BVH（Bounding Volume Hierarchy，层次包围体）是游戏引擎、物理引擎和光线追踪中非常重要的一种空间加速数据结构。

它的主要作用是：通过树形结构组织物体的包围盒，在碰撞检测、射线检测等操作中，快速排除大量不可能命中的物体，从而减少计算量。

BVH 和 KD-Tree、八叉树、Dynamic AABB Tree 都属于空间查询中经常使用的数据结构，但它们的空间组织方式有所不同。

下面重点讲解 BVH 的底层原理、构建算法、遍历过程、SAH 优化、动态更新、C++ 实现，以及游戏开发中的应用。

## 一、BVH 的核心原理

### 1.1 为什么需要 BVH？

假设游戏场景中有 10000 个物体。

现在有一条射线，需要判断它击中了哪个物体。

最直接的方法是：

```
// 暴力遍历场景中的所有物体
for (int i = 0; i < 10000; i++)
{
    // 判断射线是否与当前物体相交
    if (RayIntersectObject(ray, objects[i]))
    {
        // 记录命中结果
    }
}
```

这种方法的问题是：每次射线查询都可能需要测试 10000 个物体。

对于 N 个物体，单次暴力查询的时间复杂度为：

$$
O(N)
$$

如果场景每帧存在大量射线检测，例如：

- 射击游戏的子弹命中检测
    
- AI 的视线检测
    
- 角色与环境的碰撞检测
    
- 光线追踪的光线与三角形相交检测
    
- 鼠标点击场景物体

暴力遍历就会产生很大的性能开销。

BVH 的解决思路是：先检测大包围盒，再逐步缩小检测范围。

### 1.2 BVH 的树形结构

假设场景中存在 8 个物体：

```
                场景中的所有物体
                      │
                 Root AABB
                 /       \
                /         \
            AABB A       AABB B
            /   \         /   \
           /     \       /     \
         AABB C AABB D AABB E AABB F
          / \    / \    / \    / \
         物1 物2 物3 物4 物5 物6 物7 物8
```

这里：

根节点 Root AABB

包含整个 BVH 管理的所有物体。

内部节点 Internal Node

包含它下面所有子节点的包围盒。

叶子节点 Leaf Node

真正保存物体、三角形或者指向它们的索引。

需要注意，叶子节点不一定只有一个物体，也可以保存多个物体。

在常见的二叉 BVH 中，每个内部节点有两个子节点，但也存在 BVH4、BVH8 等多叉结构。

### 1.3 BVH 的空间组织方式

![[局部截取_20261031_005612.png]]

如果射线完全没有与左子树 A 的包围盒相交，那么：

左子树中的所有物体都不需要检测。

假设左子树包含 5000 个物体，那么一次 AABB 检测就可能排除这 5000 个物体。

这就是 BVH 加速查询的核心：

利用包围盒的包含关系，批量排除不可能满足查询条件的物体。

## 二、BVH 为什么通常使用 AABB？

BVH 的全称是 Bounding Volume Hierarchy。

其中 Bounding Volume（包围体）并不一定是 AABB。

常见包围体包括：

|包围体|特点|检测开销|
|---|---|---|
|AABB（轴对齐包围盒）|与坐标轴对齐|低|
|OBB（有向包围盒）|可以旋转，更贴合物体|较高|
|Bounding Sphere（包围球）|使用球心和半径|通常很低|
|k-DOP（离散方向多面体）|可以更紧密地包围物体|较高|

在游戏引擎中，AABB BVH 非常常见。

### 2.1 AABB 的结构

AABB（Axis-Aligned Bounding Box，轴对齐包围盒）只需要两个三维坐标：

```
struct AABB
{
    FVector Min; // 包围盒最小坐标
    FVector Max; // 包围盒最大坐标
};
```

例如：

```
Min = FVector(0, 0, 0);
Max = FVector(10, 10, 10);
```

表示一个从 `(0,0,0)` 到 `(10,10,10)` 的立方体。

### 2.2 AABB 重叠检测

判断两个 AABB 是否重叠非常简单。

只需要判断它们在 X、Y、Z 三个轴上的投影是否发生重叠。

```
bool IntersectAABB(const AABB& A, const AABB& B)
{
    // X 轴没有重叠
    if (A.Max.X < B.Min.X || A.Min.X > B.Max.X)
        return false;

    // Y 轴没有重叠
    if (A.Max.Y < B.Min.Y || A.Min.Y > B.Max.Y)
        return false;

    // Z 轴没有重叠
    if (A.Max.Z < B.Min.Z || A.Min.Z > B.Max.Z)
        return false;

    // 三个轴都重叠，说明两个 AABB 相交
    return true;
}
```

对于三维 AABB，这个操作只需要固定数量的比较。

因此时间复杂度为：

$$
O(1)
$$

但需要注意：

AABB 重叠，不代表真实物体一定发生碰撞。

它仅仅说明这两个物体有发生碰撞的可能。

所以在物理引擎中，BVH 通常用于 Broad Phase（粗检测），之后还需要 Narrow Phase（精确检测）。

## 三、BVH 是如何构建的？

BVH 的构建是理解其原理的重点。

最常见的构建方式之一是：

自顶向下构建（Top-Down Construction）。

基本过程是将所有物体包围在一个大 AABB 中，然后递归地将物体划分为两组。

### 3.1 构建流程

假设有 8 个物体。

![[局部截取_20261031_005634.png]]

算法步骤如下。

第一步：计算整个场景的 AABB

假设每个物体都有一个 AABB。

那么父节点的 AABB 就是所有子物体包围盒的并集。

公式：

$$
Min_P=\min(Min_1,Min_2,\ldots,Min_N)
$$

$$
Max_P=\max(Max_1,Max_2,\ldots,Max_N)
$$

这里的最小值和最大值需要分别在 X、Y、Z 三个轴上计算。

例如：

```
AABB MergeAABB(const AABB& A, const AABB& B)
{
    AABB Result;

    // 分别计算三个轴的最小坐标
    Result.Min.X = FMath::Min(A.Min.X, B.Min.X);
    Result.Min.Y = FMath::Min(A.Min.Y, B.Min.Y);
    Result.Min.Z = FMath::Min(A.Min.Z, B.Min.Z);

    // 分别计算三个轴的最大坐标
    Result.Max.X = FMath::Max(A.Max.X, B.Max.X);
    Result.Max.Y = FMath::Max(A.Max.Y, B.Max.Y);
    Result.Max.Z = FMath::Max(A.Max.Z, B.Max.Z);

    return Result;
}
```

第二步：选择划分轴

BVH 通常会沿 X、Y、Z 中的某个轴对物体进行分组。

常见的简单策略是选择物体中心分布范围最大的轴。

例如：

```
X 方向物体中心分布范围：100
Y 方向物体中心分布范围：20
Z 方向物体中心分布范围：10
```

可以优先选择 X 轴划分。

原因是物体主要沿 X 轴分布，沿这个方向划分通常更容易形成空间上分离的两组物体。

但是，最长轴划分不一定是最优划分，后面会讲 SAH 优化算法。

第三步：将物体划分为两组

例如沿 X 轴排序：

```
X 轴方向 →

物1  物2  物3  物4  |  物5  物6  物7  物8

        Left       |         Right
```

中位数划分可以得到：

```
Left  = {物1, 物2, 物3, 物4}

Right = {物5, 物6, 物7, 物8}
```

然后分别计算 Left AABB 和 Right AABB。

第四步：递归划分

对左右两组继续执行相同的操作。

```
BuildBVH(objects)
{
    // 1. 计算当前物体集合的包围盒
    AABB bounds = CalculateBounds(objects);

    // 2. 如果物体数量足够少，创建叶节点
    if (objects.size() <= LeafSize)
    {
        return CreateLeafNode(bounds, objects);
    }

    // 3. 选择最佳划分轴
    int axis = ChooseSplitAxis(objects);

    // 4. 将物体分为左右两组
    auto [leftObjects, rightObjects] = Split(objects, axis);

    // 5. 递归构建左右子树
    Node* left = BuildBVH(leftObjects);
    Node* right = BuildBVH(rightObjects);

    // 6. 创建父节点
    return CreateInternalNode(bounds, left, right);
}
```

### 3.2 什么时候停止划分？

通常有几个停止条件。

最常见的是叶节点包含的图元数量小于阈值。

例如：

```
const int MaxLeafObjects = 4;
```

当节点中只有 4 个或更少物体时，停止划分。

也可以结合最大树深、划分是否有收益、物体中心是否重合等条件。

不能无限递归，否则构建成本和节点内存开销都会增加。

### 3.3 BVH 的构建复杂度

假设：

- 总共有 N 个物体。
    
- 每次都将物体平均分成两组。
    
- 使用 `nth_element` 等方法寻找中位划分。

那么每层的划分总工作量通常是：

$$
O(N)
$$

树的深度约为：

$$
O(\log N)
$$

所以典型构建复杂度为：

$$
\boxed{O(N\log N)}
$$

但这取决于具体构建算法。

例如，若每次递归都对当前物体集合完整排序，平均平衡划分时的总复杂度可以达到 

$$
O(N\log^2N)
$$

## 四、BVH 的查询过程

BVH 最重要的能力是快速查询。

先以射线检测为例。

### 4.1 射线遍历 BVH

假设有下面这棵树：

```
                    Root
                   /    \
                  A      B
                 / \    / \
                C   D  E   F
```

现在有一条射线。

首先判断射线是否与 Root AABB 相交。

如果没有相交：

```
Ray → Root AABB

没有相交

直接结束查询
```

因为根节点包含所有物体，射线不与根节点相交，就不可能击中其中的任何物体。

如果与 Root 相交，继续检查 A 和 B。

假设：

```
Root：相交
  |
  ├── A：不相交
  |
  └── B：相交
         |
         ├── E：相交
         |
         └── F：不相交
```

那么整棵 A 子树和 F 子树都被排除。

只需要进入 E 节点。

![[局部截取_20261031_005733.png]]

这里被剔除的 A 子树中，无论有 10 个、100 个还是 10000 个物体，都不需要继续检测。

### 4.2 射线与 AABB 的检测

BVH 中经常使用 Slab Method（平板法）检测射线与 AABB 是否相交。

射线方程：

$$
P(t)=O+tD
$$

其中：

- \(O\) 是射线起点。
    
- \(D\) 是射线方向。
    
- \(t\) 是沿射线方向的参数。

对于 X 轴：

$$
t_{x1}=\frac{Min_x-O_x}{D_x}
$$

$$
t_{x2}=\frac{Max_x-O_x}{D_x}
$$

同理可以得到 Y、Z 轴对应的区间。

射线与 AABB 相交的条件是：

射线位于 X、Y、Z 三个轴包围范围内的 t 区间存在公共部分，并且这个公共部分与射线的有效参数范围相交。

代码如下：

```
// 射线数据
struct Ray
{
    FVector Origin;     // 射线起点
    FVector Direction;  // 射线方向
};

// 返回射线是否与 AABB 相交
bool RayIntersectAABB(
    const Ray& RayData,
    const AABB& Box,
    float MaxDistance,
    float& OutNearT)
{
    // 射线的有效参数区间
    float TMin = 0.0f;
    float TMax = MaxDistance;

    // 分别处理 X、Y、Z 三个轴
    for (int Axis = 0; Axis < 3; Axis++)
    {
        const float Origin = RayData.Origin[Axis];
        const float Dir = RayData.Direction[Axis];

        // 射线在当前轴上几乎没有方向分量
        if (FMath::Abs(Dir) < 1e-8f)
        {
            // 如果起点在该轴包围范围外
            // 则射线永远不可能进入包围盒
            if (Origin < Box.Min[Axis] ||
                Origin > Box.Max[Axis])
            {
                return false;
            }

            continue;
        }

        // 计算与两个平板的交点参数
        float T1 = (Box.Min[Axis] - Origin) / Dir;
        float T2 = (Box.Max[Axis] - Origin) / Dir;

        // 保证 T1 是较近交点，T2 是较远交点
        if (T1 > T2)
        {
            Swap(T1, T2);
        }

        // 收缩公共区间
        TMin = FMath::Max(TMin, T1);
        TMax = FMath::Min(TMax, T2);

        // 三轴公共区间为空
        if (TMin > TMax)
        {
            return false;
        }
    }

    // 返回射线进入包围盒的位置参数
    OutNearT = TMin;

    return true;
}
```

这里的 `MaxDistance` 实际是射线参数 t 的上限；只有当 `Direction` 是单位向量时，它才直接代表世界空间距离。

### 4.3 BVH 射线遍历算法

可以使用递归或者显式栈进行遍历。

下面是简化伪代码：

```
void TraverseBVH(Node* Node, const Ray& RayData)
{
    if (Node == nullptr)
        return;

    float NearT;

    // 1. 检测射线与当前节点 AABB 是否相交
    if (!RayIntersectAABB(
            RayData,
            Node->Bounds,
            MaxDistance,
            NearT))
    {
        // 当前节点没有相交
        // 可以直接剔除整个子树
        return;
    }

    // 2. 判断是否是叶节点
    if (Node->IsLeaf())
    {
        // 叶节点才真正检测物体或三角形
        for (Primitive& P : Node->Primitives)
        {
            // 精确求交，例如射线与三角形检测
            TestRayPrimitive(RayData, P);
        }

        return;
    }

    // 3. 递归检测左右子树
    TraverseBVH(Node->Left, RayData);
    TraverseBVH(Node->Right, RayData);
}
```

这是最基础的遍历。

实际工程中通常还会进一步优化：

近端优先遍历（Near-First Traversal）

先访问射线距离较近的子节点。

如果已经找到一个距离为 10 的命中点，那么后面包围盒进入距离超过 10 的节点就可以直接剔除。

这是最近命中查询非常重要的优化。

提前终止（Early Exit）

如果只是判断射线是否击中任何物体，而不需要最近命中点，那么找到一个有效命中后即可结束。

不过，如果需要找最近命中点，就不能在找到第一个命中后直接结束。

### 4.4 BVH 查询复杂度

在树相对平衡、空间组织较好的情况下，一次查询可能只需要访问少量节点。

对于 N 个物体，树的高度通常约为：

$$
h \approx \log_2 N
$$

例如：

$$
N=1024
$$

那么：

$$
h\approx10
$$

但是需要特别注意：

树的高度是 O(log N)，不代表所有 BVH 查询都保证 O(log N)。

如果射线穿过大量子树，或者节点包围盒重叠严重，就需要访问很多节点。

因此查询复杂度大致如下：

|场景|复杂度|
|---|---|
|理想情况下，剪枝非常有效|可能接近 \(O(\log N)\)|
|一般情况下|取决于访问的节点与图元数量|
|最坏情况|\(O(N)\)|

例如，一条射线恰好穿过所有物体的包围盒，那么 BVH 就可能接近暴力遍历。

## 五、SAH：BVH 最重要的构建优化算法

SAH（Surface Area Heuristic，表面积启发式）是 BVH 的经典优化算法。

它要解决的问题是：如何划分物体，才能让之后的射线查询更快？

### 5.1 为什么简单的中位数划分不一定好？

假设场景中的物体分布如下：

```
X 轴方向 →

物1 物2 物3 物4 物5 物6 物7             物8
```

如果使用中位数划分：

```
Left                   Right

物1 物2 物3 物4  |  物5 物6 物7             物8
```

虽然左右物体数量一样，但是右侧 AABB 非常大。

这可能使射线频繁进入右子树。

反过来，如果划分为：

```
Left                              Right

物1 物2 物3 物4 物5 物6 物7    |    物8
```

虽然左侧有 7 个物体，右侧只有 1 个物体，但是空间分布更集中。

在某些查询分布下，这种不平均的划分反而更快。

所以：

BVH 不一定要追求左右物体数量相等，而应该追求查询成本更低。

### 5.2 SAH 核心公式

常见的简化 SAH 划分代价为：

$$
\boxed{ C = C_T+ \frac{S_L}{S_P}N_LC_I+ \frac{S_R}{S_P}N_RC_I }
$$

其中：

| 符号      | 含义             |
| ------- | -------------- |
| \(C_T\) | 遍历当前内部节点的成本    |
| \(C_I\) | 检测一个图元的成本      |
| \(S_P\) | 父节点 AABB 的表面积  |
| \(S_L\) | 左子节点 AABB 的表面积 |
| \(S_R\) | 右子节点 AABB 的表面积 |
| \(N_L\) | 左子节点中的图元数量     |
| \(N_R\) | 右子节点中的图元数量     |

这个公式的核心思想是：

在简化的随机射线假设下，子节点包围盒的表面积与父节点表面积之比，可以近似反映射线进入该子节点的概率。

因此：

$$
\frac{S_L}{S_P}
$$

可以看成进入左子树的近似概率权重。

$$
N_LC_I
$$

表示检测左侧全部图元的成本。

左右两边分别加权，就得到了这个候选划分的估计成本。

这不是所有实际射线分布下都精确成立的概率公式，而是一种非常有效的启发式估计。

### 5.3 AABB 表面积计算

假设 AABB 的尺寸为：

$$
X=Max_x-Min_x
$$

$$
Y=Max_y-Min_y
$$

$$
Z=Max_z-Min_z
$$

那么表面积：

$$
\boxed{S=2(XY+XZ+YZ)}
$$

代码：

```
float GetSurfaceArea(const AABB& Box)
{
    FVector Size = Box.Max - Box.Min;

    return 2.0f *
    (
        Size.X * Size.Y +
        Size.X * Size.Z +
        Size.Y * Size.Z
    );
}
```

### 5.4 SAH 为什么可以优化性能？

假设有两种划分方案：

第一种：

```
              Parent
             /      \
            /        \
      Left AABB     Right AABB

      两个包围盒重叠较大
```

第二种：

```
              Parent
             /      \
            /        \
      Left AABB     Right AABB

      两个包围盒更加紧凑、分离
```

如果第一种划分产生两个非常大的 AABB，那么射线可能经常同时进入左右子树。

如果第二种划分让 AABB 更紧凑，那么射线可能只需要遍历其中一棵子树。

这时第二种划分通常更好。

SAH 就是通过评估候选划分的成本，选择较优的划分。

### 5.5 SAH 的具体实现流程

最直接的方式是枚举很多划分位置，逐个计算 SAH Cost。

但是如果对所有位置都进行完整计算，构建成本可能很高。

所以工程中经常使用：

Binned SAH（分桶 SAH）

基本过程是将选定轴上的物体中心分成固定数量的桶。

例如 8 个桶：

```
X 轴方向 →

| B0 | B1 | B2 | B3 | B4 | B5 | B6 | B7 |
```

每个桶保存：

```
struct Bucket
{
    int Count;       // 桶中的图元数量
    AABB Bounds;     // 桶中所有图元的合并包围盒
};
```

然后遍历 7 个桶边界，尝试以下划分：

```
B0 | B1 B2 B3 B4 B5 B6 B7

B0 B1 | B2 B3 B4 B5 B6 B7

B0 B1 B2 | B3 B4 B5 B6 B7

...
```

每种划分计算一次 SAH Cost。

最终选择成本最小的位置。

实际中通常也会比较 X、Y、Z 三个轴的候选划分。

通过前缀和后缀形式累计包围盒和图元数量，可以高效计算各个候选成本。

相比完整枚举所有几何划分位置，分桶 SAH 能显著降低构建开销。

### 5.6 SAH 和中位数划分的区别

|对比|中位数划分|SAH 划分|
|---|---|---|
|优化目标|左右数量相对平衡|降低预计查询成本|
|构建速度|通常较快|通常较慢|
|树结构|比较平衡|不一定平衡|
|包围盒质量|不一定理想|通常更好|
|光线追踪性能|一般|通常更好|
|适用场景|简单或频繁重建的 BVH|静态场景、高质量 BVH|

需要注意，SAH 的优势取决于查询类型和数据分布，不保证在所有情况下都优于中位数划分。

## 六、BVH 如何用于碰撞检测？

游戏物理引擎中的碰撞检测通常包含两个阶段。

```
                  场景中所有物体
                         │
                         ▼
                  Broad Phase
                    碰撞粗检测
                         │
                    BVH / AABB
                         │
                         ▼
                 筛选候选碰撞对
                         │
                         ▼
                  Narrow Phase
                    碰撞精确检测
                         │
                  GJK / SAT / EPA
                         │
                         ▼
                实际碰撞结果与接触信息
```

### 6.1 Broad Phase：粗检测

假设场景中有 1000 个物体。

如果直接两两检测：

$$
\frac{N(N-1)}{2}
$$

那么：

$$
\frac{1000\times999}{2}=499500
$$

需要检查 499500 对物体。

这只是候选物体对的数量，还没有考虑每次几何检测的成本。

BVH 可以通过包围盒层级，大量剔除不可能碰撞的物体对。

例如：

```
BVH Root
   |
   ├── 左边区域
   |      |
   |      ├── A
   |      └── B
   |
   └── 右边区域
          |
          ├── C
          └── D
```

如果左边区域和右边区域的包围盒完全不相交：

那么左边和右边的物体之间不可能发生碰撞。

可以批量排除左右子树之间的物体对。

需要注意，如果要找出整棵树内部所有碰撞对，可以使用树与树的节点对遍历算法；也可以逐物体执行重叠查询并去重。

### 6.2 Narrow Phase：精确检测

经过 BVH 粗检测之后，可能得到：

```
候选碰撞对：

A-B
A-C
D-E
F-G
```

然后才针对这些候选物体执行精确检测。

例如：

```
for (const auto& Pair : CandidatePairs)
{
    // Broad Phase 只判断包围盒是否可能碰撞

    // Narrow Phase 进一步判断真实形状是否相交
    // 例如使用 GJK、SAT 等算法
    if (NarrowPhaseCollision(Pair.A, Pair.B))
    {
        // 生成接触点、法线、穿透深度等信息
    }
}
```

GJK、SAT、EPA 适用于不同的凸形状检测和接触信息求解任务，并不是所有场景都需要按这三个算法依次执行。

BVH 的核心职责通常不是计算碰撞点，而是减少需要进行精确检测的物体数量。

## 七、BVH 的动态更新

前面的构建主要适用于静态数据。

但是游戏中的物体经常移动。

例如角色奔跑、怪物移动、物理物体掉落等。

这时 BVH 中保存的 AABB 会发生变化。

处理动态 BVH 常见有三种策略。

### 7.1 Rebuild：重新构建

最直接的方法是每次场景变化后重新构建 BVH。

```
void UpdateBVH()
{
    // 获取所有物体最新的 AABB
    UpdateAllObjectBounds();

    // 删除旧 BVH 并重新构建
    RebuildBVH();
}
```

优点是实现简单，可以重新优化整棵树的空间组织。

缺点是：

构建成本较高。

如果每帧都对大量物体重建高质量 BVH，可能产生较大的 CPU 开销。

所以这种方式更适合静态数据、更新频率较低的数据，或者使用高效快速构建算法的特定场景。

### 7.2 Refit：重新计算包围盒

Refit（包围盒重拟合）是动态 BVH 常见的优化方式。

其核心思想是：

不改变树结构，只更新节点的 AABB。

例如：

```
更新前：

             Parent AABB
              /      \
             A        B

物体 A 移动后：

             Parent AABB（旧）
              /      \
        A（新位置）    B
```

这时需要重新计算：

```
A 的 AABB
   ↓
Parent AABB
   ↓
Parent 的父节点 AABB
   ↓
一直更新到 Root
```

代码示意：

```
AABB RefitBVH(Node* Node)
{
    // 如果是叶节点
    if (Node->IsLeaf())
    {
        // 根据叶节点的所有物体
        // 重新计算当前包围盒
        Node->Bounds = CalculateLeafBounds(Node);

        return Node->Bounds;
    }

    // 更新左右子树
    AABB LeftBounds = RefitBVH(Node->Left);
    AABB RightBounds = RefitBVH(Node->Right);

    // 父节点 AABB 等于左右子节点 AABB 的并集
    Node->Bounds = MergeAABB(LeftBounds, RightBounds);

    return Node->Bounds;
}
```

对于完整的自底向上 Refit：

$$
O(N)
$$

如果只更新一个叶节点到根节点的路径，且维护了父节点关系，那么典型更新成本取决于树高：

$$
O(h)
$$

对于相对平衡的树：

$$
O(\log N)
$$

但这并不包含维护所有移动物体的总成本。

Refit 最大的问题是：只更新包围盒，不改善树的拓扑结构。

### 7.3 为什么 Refit 会退化？

假设最初 BVH 将物体按照左右两组划分。

![[局部截取_20261031_010021.png]]

如果左子树的物体移动到右边，右子树的物体移动到左边：

左右子树的 AABB 可能会严重重叠。

此时射线可能需要同时遍历两棵子树，查询性能下降。

所以 Refit 通常会结合局部优化或者周期性重建。

### 7.4 Dynamic BVH：动态 BVH

对于大量移动物体，可以使用支持插入、删除、移动和重新平衡的动态包围体树。

Dynamic AABB Tree（动态 AABB 树）就是一种典型的动态 BVH 实现。

常见技术包括：

Fat AABB（扩张包围盒）

给真实 AABB 增加一定余量。

```
// 根据实际 AABB 创建扩张 AABB
FatMin = ActualMin - Margin;
FatMax = ActualMax + Margin;
```

如果物体移动后仍位于原来的 Fat AABB 内，就可以避免立即重新插入树。

查询得到候选结果后，仍应使用真实包围盒或者真实形状进行后续检测。

Remove + Reinsert（删除并重新插入）

如果物体移动超出原来的 Fat AABB，可以将对应叶节点从树中移除，然后根据新位置重新插入。

Tree Rotation（树旋转）

通过局部调整父子节点关系，降低包围盒重叠程度或者减小某种树质量代价。

这和 AVL Tree 等平衡树中的旋转有形式上的相似性，但优化目标不完全一样。

AVL 主要维护树高平衡。

动态 BVH 还需要考虑包围盒面积、重叠程度及查询成本。

## 八、C++ 实现一个简单的 BVH

下面给出一个可用于理解核心原理的 C++17 示例。

这个版本支持：

- 根据物体 AABB 创建二叉 BVH。
    
- 通过物体中心的最长轴进行中位数划分。
    
- 查询与指定 AABB 重叠的物体。
    
- 找到可能发生碰撞的物体对。

为了简化代码，没有实现 SAH 和动态更新。

### 8.1 核心代码

```
#include <algorithm>
#include <array>
#include <cmath>
#include <iostream>
#include <limits>
#include <numeric>
#include <utility>
#include <vector>

// 三维向量
using Vec3 = std::array<float, 3>;

// -----------------------------
// AABB：轴对齐包围盒
// -----------------------------
struct AABB
{
    Vec3 Min;
    Vec3 Max;

    AABB()
    {
        // 初始化为空包围盒
        const float Inf = std::numeric_limits<float>::infinity();

        Min = { Inf, Inf, Inf };
        Max = { -Inf, -Inf, -Inf };
    }

    // 合并另一个包围盒
    void Expand(const AABB& Other)
    {
        for (int Axis = 0; Axis < 3; ++Axis)
        {
            Min[Axis] = std::min(Min[Axis], Other.Min[Axis]);
            Max[Axis] = std::max(Max[Axis], Other.Max[Axis]);
        }
    }

    // 判断两个 AABB 是否重叠
    bool Overlaps(const AABB& Other) const
    {
        for (int Axis = 0; Axis < 3; ++Axis)
        {
            // 某个轴上没有重叠，整体就没有重叠
            if (Max[Axis] < Other.Min[Axis] ||
                Min[Axis] > Other.Max[Axis])
            {
                return false;
            }
        }

        return true;
    }

    // 获取指定轴的包围盒中心
    float Center(int Axis) const
    {
        return (Min[Axis] + Max[Axis]) * 0.5f;
    }
};

// -----------------------------
// 场景物体
// -----------------------------
struct Object
{
    int ID;
    AABB Bounds;
};

// -----------------------------
// BVH 节点
// -----------------------------
struct BVHNode
{
    AABB Bounds;

    // 左右子节点索引
    int Left = -1;
    int Right = -1;

    // 叶节点对应的物体索引范围
    int Begin = 0;
    int Count = 0;

    bool IsLeaf() const
    {
        return Left == -1;
    }
};

// -----------------------------
// BVH 数据结构
// -----------------------------
class BVH
{
public:

    explicit BVH(std::vector<Object> InObjects, int InLeafSize = 2)
        : Objects(std::move(InObjects)),
          LeafSize(std::max(1, InLeafSize))
    {
        // 为物体创建索引数组
        Indices.resize(Objects.size());

        std::iota(Indices.begin(), Indices.end(), 0);

        if (!Objects.empty())
        {
            // 对 N 个物体构建 BVH
            // 二叉 BVH 的节点数量最多为 2N-1
            Nodes.reserve(Objects.size() * 2);

            Root = BuildNode(0, static_cast<int>(Objects.size()));
        }
    }

    // ----------------------------------
    // 查询与指定 AABB 重叠的物体
    // ----------------------------------
    template<typename Func>
    void Query(const AABB& QueryBounds, Func&& Callback) const
    {
        if (Root == -1)
            return;

        // 使用显式栈代替递归
        std::vector<int> Stack;
        Stack.push_back(Root);

        while (!Stack.empty())
        {
            int NodeIndex = Stack.back();
            Stack.pop_back();

            const BVHNode& Node = Nodes[NodeIndex];

            // 当前节点包围盒不重叠
            // 剔除整个子树
            if (!Node.Bounds.Overlaps(QueryBounds))
                continue;

            // 到达叶节点
            if (Node.IsLeaf())
            {
                // 检查叶节点中的每个物体
                for (int i = 0; i < Node.Count; ++i)
                {
                    int ObjectIndex = Indices[Node.Begin + i];

                    const Object& Obj = Objects[ObjectIndex];

                    // 叶节点 AABB 重叠不代表
                    // 叶节点里的每个物体都重叠
                    if (Obj.Bounds.Overlaps(QueryBounds))
                    {
                        Callback(Obj);
                    }
                }
            }
            else
            {
                // 内部节点继续访问左右子树
                Stack.push_back(Node.Left);
                Stack.push_back(Node.Right);
            }
        }
    }

    // ----------------------------------
    // 查询所有可能碰撞的物体对
    // 假设每个物体的 ID 唯一
    // ----------------------------------
    std::vector<std::pair<int, int>> FindCandidatePairs() const
    {
        std::vector<std::pair<int, int>> Result;

        for (const Object& A : Objects)
        {
            Query(A.Bounds, [&](const Object& B)
            {
                // ID 比较避免自碰撞以及重复碰撞对
                if (A.ID < B.ID)
                {
                    Result.emplace_back(A.ID, B.ID);
                }
            });
        }

        return Result;
    }

private:

    // ----------------------------------
    // 递归构建 BVH
    // 处理 Indices 中 [Begin, End) 范围
    // ----------------------------------
    int BuildNode(int Begin, int End)
    {
        const int Count = End - Begin;

        // 1. 计算当前节点的总 AABB
        AABB Bounds;

        // 计算物体中心的分布范围
        Vec3 MinCenter;
        Vec3 MaxCenter;

        float Inf = std::numeric_limits<float>::infinity();

        MinCenter = { Inf, Inf, Inf };
        MaxCenter = { -Inf, -Inf, -Inf };

        for (int i = Begin; i < End; ++i)
        {
            const Object& Obj = Objects[Indices[i]];

            Bounds.Expand(Obj.Bounds);

            for (int Axis = 0; Axis < 3; ++Axis)
            {
                float C = Obj.Bounds.Center(Axis);

                MinCenter[Axis] = std::min(MinCenter[Axis], C);
                MaxCenter[Axis] = std::max(MaxCenter[Axis], C);
            }
        }

        // 2. 创建当前节点
        const int NodeIndex = static_cast<int>(Nodes.size());

        Nodes.emplace_back();

        Nodes[NodeIndex].Bounds = Bounds;
        Nodes[NodeIndex].Begin = Begin;
        Nodes[NodeIndex].Count = Count;

        // 3. 物体数量足够少，创建叶节点
        if (Count <= LeafSize)
        {
            return NodeIndex;
        }

        // 4. 计算物体中心在三个轴上的分布范围
        Vec3 Extent =
        {
            MaxCenter[0] - MinCenter[0],
            MaxCenter[1] - MinCenter[1],
            MaxCenter[2] - MinCenter[2]
        };

        // 5. 选择分布范围最大的轴
        int SplitAxis = 0;

        if (Extent[1] > Extent[SplitAxis])
            SplitAxis = 1;

        if (Extent[2] > Extent[SplitAxis])
            SplitAxis = 2;

        // 所有物体中心几乎重合
        // 无法继续有效划分，创建叶节点
        if (Extent[SplitAxis] < 1e-6f)
        {
            return NodeIndex;
        }

        // 6. 选择中位数
        int Mid = Begin + Count / 2;

        // nth_element 不需要将所有物体完整排序
        // 只需要将中位位置上的元素放到正确的位置
        std::nth_element(
            Indices.begin() + Begin,
            Indices.begin() + Mid,
            Indices.begin() + End,
            [&](int A, int B)
            {
                return Objects[A].Bounds.Center(SplitAxis) <
                       Objects[B].Bounds.Center(SplitAxis);
            }
        );

        // 7. 递归构建左右子节点
        int LeftIndex = BuildNode(Begin, Mid);
        int RightIndex = BuildNode(Mid, End);

        // 8. 当前节点变成内部节点
        Nodes[NodeIndex].Left = LeftIndex;
        Nodes[NodeIndex].Right = RightIndex;
        Nodes[NodeIndex].Count = 0;

        return NodeIndex;
    }

private:

    std::vector<Object> Objects;

    // 物体索引数组：划分时仅调整索引
    std::vector<int> Indices;

    // 使用连续数组保存节点
    std::vector<BVHNode> Nodes;

    int Root = -1;
    int LeafSize = 2;
};
```

### 8.2 测试代码

```
AABB MakeBox(
    float MinX, float MinY, float MinZ,
    float MaxX, float MaxY, float MaxZ)
{
    AABB Box;

    Box.Min = { MinX, MinY, MinZ };
    Box.Max = { MaxX, MaxY, MaxZ };

    return Box;
}

int main()
{
    // 创建 4 个场景物体
    std::vector<Object> Objects =
    {
        {1, MakeBox(0, 0, 0, 2, 2, 2)},
        {2, MakeBox(1, 1, 1, 3, 3, 3)},
        {3, MakeBox(8, 8, 8, 9, 9, 9)},
        {4, MakeBox(10, 10, 10, 11, 11, 11)}
    };

    // 创建 BVH
    BVH Tree(Objects);

    // 查找所有可能发生碰撞的物体对
    auto CandidatePairs = Tree.FindCandidatePairs();

    for (const auto& Pair : CandidatePairs)
    {
        std::cout
            << "Candidate Pair: "
            << Pair.first
            << " - "
            << Pair.second
            << std::endl;
    }

    return 0;
}
```

输出：

```
Candidate Pair: 1 - 2
```

因为物体 1 和物体 2 的 AABB 发生重叠。

其他物体的 AABB 不发生重叠，所以不需要执行后续精确碰撞检测。

这个示例采用逐物体查询方式，便于理解和验证。大型物理系统还可以使用更高效的节点对遍历来生成碰撞候选对。

### 8.3 为什么使用索引数组？

上面的实现没有直接在构建过程中移动 `Object`。

而是使用：

```
std::vector<int> Indices;
```

存储对象索引。

在构建 BVH 时，只需要改变索引顺序。

这样做有几个好处：

对象自身的数据不需要频繁移动；BVH 节点可以连续存储；方便管理三角形索引、物理对象标识和其他外部数据。

同时还可以改善遍历时的内存布局。

在高性能的渲染器或者物理引擎中，数据布局也是影响 BVH 性能的重要因素。

## 九、BVH 和 KD-Tree、八叉树、Dynamic AABB Tree 的区别

这一部分也是游戏客户端开发和图形学面试中经常被问到的内容。

### 9.1 BVH 和 KD-Tree

两者最本质的区别是：

![[局部截取_20261031_010050.png]]

|对比|BVH|KD-Tree|
|---|---|---|
|划分对象|物体或图元集合|空间区域|
|内部节点|子包围盒|划分轴与划分位置|
|子节点包围区域|可以重叠|通常是互不重叠的空间分区|
|几何体跨区域|通常由一个叶节点保存|可能需要重复引用或裁剪图元|
|动态更新|有较成熟的动态维护方案|通常较复杂|
|光线追踪|广泛使用|也适用|
|碰撞检测|广泛使用|可使用，但不一定是首选|

需要注意，标准的 BVH 是按照图元分组的，但也存在允许空间划分和图元引用重复的变种，例如 Spatial Split BVH。

KD-Tree 也有按数据点划分、按空间划分等不同实现，需要结合具体场景理解。

### 9.2 BVH 和 Octree（八叉树）

八叉树每次将三维空间划分成 8 个子区域。

```
BVH：

                Root
               /    \
              A      B

Octree：

                Root
           / / / / \ \ \ \
          8 个子区域
```

八叉树更强调规则的空间划分。

BVH 则根据物体的空间分布构建包围盒层级。

因此：

如果场景物体分布非常不均匀，BVH 往往能更灵活地贴合实际物体分布。

而八叉树的规则空间结构在一些范围查询、空间管理任务中具有实现和组织上的优势。

### 9.3 BVH 和 Dynamic AABB Tree

实际上：

$$
\boxed{\text{Dynamic AABB Tree 是动态 BVH 的一种常见形式}}
$$

它不是与 BVH 完全并列、互不相关的数据结构。

典型的区别如下：

|对比|静态 BVH|Dynamic AABB Tree|
|---|---|---|
|主要目标|高质量静态空间查询|高效维护移动物体|
|构建方式|自顶向下划分、SAH 等|常采用动态插入|
|移动物体|Refit 或重建|更新、删除、重新插入|
|节点优化|构建时重点优化|运行时局部优化|
|典型用途|静态三角形、Ray Tracing|物理 Broad Phase|

对于游戏物理系统，动态 AABB 树特别值得掌握。

## 十、BVH 在游戏引擎中的实际应用

### 10.1 物理引擎

BVH 可以用于粗检测加速。

例如场景中有 5000 个动态或静态碰撞对象。

物理引擎可以利用空间加速结构快速找出 AABB 可能重叠的对象，然后只对候选对象执行精确碰撞检测。

在 UE 的 Chaos 物理系统中，空间加速和碰撞检测是重要的底层环节；实际使用的加速结构和维护策略会随对象类型、引擎版本及具体子系统而变化，不能简单理解为所有场景都只使用同一种 BVH。

### 10.2 光线追踪

光线追踪是 BVH 非常典型的应用。

例如一帧画面发射大量光线，需要判断每条光线与哪些三角形相交。

如果一个模型拥有 100 万个三角形，暴力求交的开销非常高。

BVH 可以先检测较大的三角形群包围盒，然后逐层深入，只对少量候选三角形执行精确求交。

现代硬件光线追踪通常使用加速结构组织几何体。

例如在常见的硬件光线追踪 API 中：

BLAS（Bottom-Level Acceleration Structure，底层加速结构）

用于管理模型自身的几何图元。

TLAS（Top-Level Acceleration Structure，顶层加速结构）

用于管理场景中的几何实例。

它们通常采用 BVH 或类似的层次加速思想，但底层节点布局、构建方式和遍历细节可能由驱动与硬件实现决定。

```
                   TLAS
               场景实例层级
                      │
         ┌────────────┼────────────┐
         │            │            │
     实例 A         实例 B        实例 C
         │            │            │
        BLAS         BLAS         BLAS
         │            │            │
     三角形集合    三角形集合    三角形集合
```

这种两层结构可以更方便地管理模型实例及其变换。

### 10.3 场景查询

在游戏客户端中，BVH 还可以加速：

角色攻击的射线检测、鼠标拾取、AI 视线检测、静态场景几何查询等。

对于三角形 Mesh，还可以为模型单独构建 BVH，从而快速查询射线与 Mesh 的交点。

### 10.4 距离和邻近查询

BVH 不仅可以处理射线和碰撞。

还可以用于：

最近点查询、范围查询、邻近物体搜索等。

例如查询某个点到三角网格的最近点，可以先计算点到各节点 AABB 的距离下界。

如果某个节点 AABB 的最小可能距离已经超过当前找到的最近距离，就可以剪枝。

这是 BVH 能用于更广泛空间查询的重要原因。

## 十一、BVH 的实际性能优化

实际工程中的 BVH 性能不只取决于构建算法。

### 11.1 减少子节点包围盒重叠

如果兄弟节点 AABB 大量重叠，那么一次查询可能需要同时访问多个子树。

因此，减少重叠通常有助于提升查询效率。

不过需要注意，不能只优化重叠面积。

节点深度、节点表面积、图元数量、查询分布等也会影响最终性能。

SAH 就是综合考虑部分因素的一种方法。

### 11.2 调整叶节点图元数量

如果每个叶节点只保存 1 个图元：

BVH 节点较多，遍历内部节点的成本可能较大。

如果每个叶节点保存 100 个图元：

树变浅了，但进入叶节点后需要检测大量图元。

所以必须平衡：

$$
\text{总成本} = \text{节点遍历成本} + \text{叶节点图元检测成本}
$$

最佳叶节点大小取决于图元检测成本、查询分布、缓存结构和硬件。

### 11.3 使用连续内存布局

如果每个节点都动态分配：

```
struct Node
{
    Node* Left;
    Node* Right;
};
```

节点可能分散在内存的不同位置。

遍历时容易产生大量随机内存访问。

所以更高性能的 BVH 通常会使用连续数组：

```
std::vector<BVHNode> Nodes;
```

通过节点索引访问：

```
BVHNode& LeftNode = Nodes[Node.Left];
```

这样有利于改善数据局部性，并减少大量小对象分配的开销。

### 11.4 多叉 BVH

基础 BVH 常使用二叉树。

但是实际还可以使用：

BVH4：每个内部节点最多 4 个子节点。

BVH8：每个内部节点最多 8 个子节点。

多叉 BVH 可以减少树深度，也更容易配合 SIMD 等方式批量检测多个子节点包围盒。

不过分支更多，并不代表一定更快，因为一次节点访问需要测试的子包围盒也更多。

需要结合 CPU/GPU 硬件、数据布局以及遍历算法进行优化。

### 11.5 动态树的周期性优化

对于频繁移动的场景：

可以优先使用 Refit 或局部更新降低维护成本。

当包围盒重叠程度、查询节点访问数量等指标明显恶化时，再执行局部重组或重建。

这是在构建成本与查询成本之间进行权衡。

## 十二、BVH 常见面试题

### Q1：BVH 为什么能够加速碰撞检测？

因为 BVH 将物体按照空间位置组织为层次包围盒。

检测过程中，如果一个节点的包围盒不满足相交条件，就可以一次性排除整个子树的物体，避免逐个进行精确检测。

### Q2：BVH 的时间复杂度是多少？

对于使用中位划分的常见平衡二叉 BVH：

构建通常可以达到：

$$
O(N\log N)
$$

查询在剪枝效果好的情况下可能接近：

$$
O(\log N)
$$

但是最坏情况下：

$$
O(N)
$$

因此不能认为 BVH 所有查询都保证 O(log N)。

### Q3：为什么 BVH 通常使用 SAH？

因为普通中位数划分主要追求左右数量平衡，而 SAH 通过估计射线进入子节点的成本，尝试减少查询过程中需要访问的节点和图元数量。

### Q4：BVH 和 KD-Tree 的核心区别是什么？

BVH 主要对物体或图元集合构建层次包围盒；KD-Tree 则主要利用超平面递归划分空间或数据集合。

BVH 的兄弟节点包围盒可以重叠，而空间划分式 KD-Tree 的左右空间区域通常互不重叠。

### Q5：BVH 如何处理动态物体？

常见方式包括 Refit、重建，以及动态树的删除重新插入和局部旋转。

Refit 的优点是更新快，但不会改变树拓扑，物体长期运动可能导致包围盒组织退化。

### Q6：BVH 是否只能用于碰撞检测？

不是。

它还可以用于射线追踪、范围查询、三角形拾取、最近点搜索等。

### Q7：BVH 节点包围盒越小，查询就一定越快吗？

不一定。

因为查询效率还与节点数量、树深度、重叠程度、图元分布和内存访问有关。

需要综合考虑整个树的遍历成本。

## 十三、核心总结

|核心问题|答案|
|---|---|
|BVH 是什么？|一种基于层次包围盒的空间加速数据结构|
|最重要的思想是什么？|通过父包围盒快速剔除整棵子树|
|怎么构建？|递归划分物体，创建左右子树|
|怎么查询？|检测节点包围盒，剪枝并深入相交子树|
|为什么使用 SAH？|降低预计遍历与图元检测成本|
|如何处理动态物体？|Refit、重新插入、局部旋转或重建|
|和 KD-Tree 最大的区别？|BVH 按物体组织层级，KD-Tree 通常按空间划分|
|和 Dynamic AABB Tree 的关系？|Dynamic AABB Tree 是动态 BVH 的常见实现形式|
|主要应用？|游戏物理碰撞、光线追踪、射线检测、空间查询|

对于 UE 游戏客户端和图形学开发，我认为最值得深入掌握的是三个部分：

第一，BVH 构建和 SAH。 要能够解释为什么中位数划分不一定最优，以及 SAH 为什么能提升射线查询效率。

第二，BVH 遍历和剪枝。 要理解为什么仅一次父节点 AABB 检测，就能排除大量物体，以及最近命中查询为什么应该优先遍历较近节点。

第三，动态 BVH 和 Dynamic AABB Tree。 要理解 Refit、Fat AABB、删除重新插入和树旋转分别解决什么问题，以及为什么动态场景不能简单地只做一次静态 BVH 构建。

这三个部分基本覆盖了 BVH 在游戏物理、渲染和空间查询中的主要技术原理。