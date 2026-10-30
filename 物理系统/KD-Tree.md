# KD-Tree（K维树）原理详解

KD-Tree（K-Dimensional Tree，K维树）是一种用于多维空间划分、最近邻搜索和空间范围查询的数据结构。

在游戏开发中，KD-Tree 经常用于空间查询，例如：

- 查找距离角色最近的敌人。
    
- 查询角色周围一定半径内的对象。
    
- 查找距离某个位置最近的导航节点。
    
- 点云、空间采样点的最近邻查询。
    
- 静态场景中的空间索引与查询加速。
    

它的核心思想是：

通过不断沿不同坐标轴划分空间，将大量空间点组织成一棵二叉树，查询时利用空间位置关系排除不可能满足条件的节点，从而减少距离计算。

下面重点讲解 KD-Tree 的建树原理、最近邻查询、回溯剪枝、时间复杂度、C++ 实现，以及它在 UE 游戏开发中的使用场景。

## 一、KD-Tree 的基本原理

### 1.1 为什么需要 KD-Tree？

假设游戏场景中有 100 万个敌人，我们希望找到距离玩家最近的敌人。

最直接的方法就是暴力遍历：

```
float MinDistSq = FLT_MAX;
AActor* NearestEnemy = nullptr;

for (AActor* Enemy : Enemies)
{
    // 计算玩家与敌人之间的距离平方
    float DistSq = FVector::DistSquared(
        PlayerLocation,
        Enemy->GetActorLocation()
    );

    if (DistSq < MinDistSq)
    {
        MinDistSq = DistSq;
        NearestEnemy = Enemy;
    }
}
```

这种方式每次需要检查所有敌人。

时间复杂度：

$$
O(N)
$$

当 N=1000000 时，一次查询最多需要计算 100 万次距离。

如果每帧有大量 AI 都在执行最近邻查询，开销就会非常明显。

KD-Tree 的目的就是：提前建立空间索引，使查询时无需遍历所有对象。

对于平衡、低维且分布合理的 KD-Tree，最近邻查询通常能显著快于暴力遍历；理想情况下可以接近O(log N)，但最坏情况下仍然是O(N)。

### 1.2 KD-Tree 是什么？

KD-Tree 本质上是一棵二叉树，不过它与普通二叉搜索树的区别在于：

普通二叉搜索树依据一个值划分，例如：

```
           10
          /  \
         5    15
        / \   / \
       2   7 12  20
```

左子树数值较小，右子树数值较大。

KD-Tree 依据的是多维空间中的坐标。

例如二维空间中的一个点：

$$
P=(x,y)
$$

三维空间中的一个点：

$$
P=(x,y,z)
$$

对于三维 KD-Tree，可以依次选择：

|树的深度|划分轴|空间划分方式|
|---|---|---|
|0|X轴|根据 X 坐标划分|
|1|Y轴|根据 Y 坐标划分|
|2|Z轴|根据 Z 坐标划分|
|3|X轴|再次根据 X 坐标划分|
|4|Y轴|再次根据 Y 坐标划分|
|5|Z轴|再次根据 Z 坐标划分|

常见的划分方式为：

$$
Axis=Depth\bmod K
$$

其中：

- \(Depth\)：当前节点的深度，根节点为0。
    
- \(K\)：空间维度，不是 K 个最近邻中的 K。
    
- \(Axis\)：当前节点使用的划分轴。
    

例如：

```
// 三维空间，交替使用 X、Y、Z 轴
int Axis = Depth % 3;
```

一个 KD-Tree 节点通常保存一个空间点、该节点的划分轴，以及左右子节点。

### 1.3 空间是怎么被划分的？

![[局部截取_20261031_003348.png]]

这里要注意一个容易混淆的概念：

按照 X 轴划分，指的是比较 X 坐标，但在二维平面中形成的是垂直于 X 轴的直线。

类似地：

- 按照 X 轴划分：二维中是一条竖线，三维中是 YZ 平面。
    
- 按照 Y 轴划分：二维中是一条横线，三维中是 XZ 平面。
    
- 按照 Z 轴划分：三维中是 XY 平面。
    

因此，KD-Tree 本质上是在递归地使用超平面划分 K 维空间。

## 二、KD-Tree 的建树过程

使用一个二维案例，来详细解释建树算法。

假设有 6 个点：

|节点|X|Y|
|---|---|---|
|A|2|3|
|B|5|4|
|C|9|6|
|D|4|7|
|E|8|1|
|F|7|2|

### 2.1 第一次划分：选择 X 轴

因为根节点的深度是 0：

$$
Axis=0\bmod2=0
$$

所以使用 X 轴。

按照 X 坐标排序：

```
A(2,3)
D(4,7)
B(5,4)
F(7,2)  <-- 选择中位点
E(8,1)
C(9,6)
```

这里使用下标 `N/2` 的元素作为中位点。对于偶数个点，取偏右的中位点是一种常见约定。

因此根节点为：

$$
F=(7,2)
$$

划分规则：

```
                 F(7,2)
                   |
          按照 X = 7 划分
                   |
         ┌─────────┴─────────┐
         │                   │
     X < 7 的点          X > 7 的点
         │                   │
      A、B、D              C、E
```

这一步完成后，根节点左侧的点都位于 X 较小的空间，右侧的点位于 X 较大的空间。

### 2.2 第二次划分：选择 Y 轴

深度变成 1：

$$
Axis=1\bmod2=1
$$

因此改用 Y 轴划分。

对于左子树：

```
A(2,3)  Y=3
B(5,4)  Y=4  <-- 中位点
D(4,7)  Y=7
```

选择 B 作为左子树根节点。

对于右子树：

```
E(8,1)  Y=1
C(9,6)  Y=6  <-- 中位点
```

选择 C 作为右子树根节点。

此时：

```
                  F(7,2)
                  [X轴]
                 /     \
                /       \
           B(5,4)       C(9,6)
           [Y轴]         [Y轴]
```

### 2.3 第三次划分：继续选择 X 轴

深度为 2：

$$
Axis=2\bmod2=0
$$

重新使用 X 轴。

将剩余节点放入对应子树，得到最终结构：

具体结构为：

![[局部截取_20261031_003451.png]]

这里最重要的是：KD-Tree 的左右子树关系取决于当前节点的划分轴，而不是所有层都比较 X 坐标。

例如 B(5,4) 的右子节点 D(4,7)：

虽然 D 的 X 坐标 4 小于 B 的 X 坐标 5，但 D 依然在 B 的右子树。

因为 B 节点使用的是 Y 轴：

$$
D_y=7>B_y=4
$$

这也是 KD-Tree 与普通二叉搜索树最明显的区别。

### 2.4 为什么选择中位点？

选择中位点是为了尽量让左右子树节点数量均衡。

如果总是选择第一个点作为划分节点，可能出现：

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
```

这种退化结构的树高可以达到：

$$
O(N)
$$

查询效率随之下降。

使用中位点划分，左右节点数量通常接近相等，树高约为：

$$
H=O(\log N)
$$

但是注意：

中位点保证的是节点数量上的平衡，不一定代表空间体积均匀，也不意味着最近邻搜索一定只访问 O(log N) 个节点。

## 三、KD-Tree 最近邻搜索（最重要）

KD-Tree 最经典的应用是 Nearest Neighbor Search（最近邻搜索）。

也就是给定一个查询点：

$$
Q=(x,y,z)
$$

找到 KD-Tree 中与 Q 欧氏距离最小的点。

这里有两个关键操作：

向下搜索（Downward Traversal）：根据划分轴选择更可能包含最近点的子树。

回溯剪枝（Backtracking & Pruning）：检查是否有必要搜索另一侧子树。

其中，回溯剪枝是理解 KD-Tree 的关键。

### 3.1 第一步：计算点之间的距离

对于二维点：

$$
P=(x_1,y_1),\quad Q=(x_2,y_2)
$$

欧氏距离：

$$
d(P,Q)=\sqrt{(x_1-x_2)^2+(y_1-y_2)^2}
$$

三维则为：

$$
d(P,Q)=\sqrt{(x_1-x_2)^2+(y_1-y_2)^2+(z_1-z_2)^2}
$$

实际编程中，一般使用距离平方：

$$
d^2(P,Q)=(x_1-x_2)^2+(y_1-y_2)^2+(z_1-z_2)^2
$$

这样可以避免每次计算平方根。

```
// UE5 示例：计算两个空间点的距离平方
float DistSq = FVector::DistSquared(PointA, PointB);
```

### 3.2 第二步：优先搜索更近的子树

继续使用之前构建的 KD-Tree：

```
                  F(7,2)  X
                 /         \
                /           \
           B(5,4) Y       C(9,6) Y
           /     \          /
       A(2,3)  D(4,7)   E(8,1)
```

假设查询点：

$$
Q=(8.2,2.1)
$$

我们的目标是找到距离 Q 最近的点。

① 从根节点 F 开始

F 的坐标为：

$$
F=(7,2)
$$

计算距离平方：

$$
d^2(Q,F)=(8.2-7)^2+(2.1-2)^2
$$

$$
=1.44+0.01=1.45
$$

当前最近点：

```
BestPoint = F
BestDistSq = 1.45
```

由于 F 按 X 轴划分：

$$
Q_x=8.2>F_x=7
$$

所以优先搜索右子树。

② 到达节点 C

$$
C=(9,6)
$$

计算：

$$
d^2(Q,C)=(8.2-9)^2+(2.1-6)^2=15.85
$$

由于：

$$
15.85>1.45
$$

最近点保持 F 不变。

C 使用 Y 轴划分，而：

$$
Q_y=2.1<C_y=6
$$

因此优先搜索左子树 E。

③ 到达节点 E

$$
E=(8,1)
$$

计算：

$$
d^2(Q,E)=(8.2-8)^2+(2.1-1)^2
$$

$$
=0.04+1.21=1.25
$$

由于：

$$
1.25<1.45
$$

更新最近点：

```
BestPoint = E
BestDistSq = 1.25
```

E 是叶子节点，没有更深的子节点。

到这里，我们沿着优先路径访问了：

```
F → C → E
```

但是此时还不能直接结束。

因为另一侧子树中仍然可能存在距离 Q 更近的点。

这就需要回溯。

## 四、KD-Tree 回溯与剪枝原理

### 4.1 为什么需要回溯？

假设有一个查询点 Q，按照划分轴判断，它位于右子树一侧。

于是我们优先搜索右子树。

但是：

查询点位于右侧，并不代表最近的空间点一定属于右子树。

例如下面的场景：

![[局部截取_20261031_004422.png]]

Q 位于划分线右侧，但它距离左侧的 A 比距离右侧的 B 更近。

如果只搜索右子树，就可能漏掉真正的最近邻。

因此，KD-Tree 的查询不是简单地沿某条路径走到叶子节点，而是必须进行回溯判断。

### 4.2 回溯时，怎么判断是否搜索另一侧？

核心是：

比较查询点到划分平面的距离，以及当前已经找到的最近点距离。

假设当前节点的划分轴是 X 轴。

当前节点：

$$
P=(x_p,y_p)
$$

查询点：

$$
Q=(x_q,y_q)
$$

查询点到划分线的距离为：

$$
d_{plane}=|x_q-x_p|
$$

在三维中，如果按 X 轴划分，其划分平面就是：

$$
x=x_p
$$

查询点到该平面的最短距离同样是：

$$
d_{plane}=|x_q-x_p|
$$

注意这里只需要比较当前划分轴对应的坐标。

假设我们当前找到的最近距离是：

$$
d_{best}
$$

那么：

情况一：可以剪枝

$$
d_{plane}>d_{best}
$$

代表另一侧子树位于当前最近邻搜索半径之外，可以安全地跳过。

情况二：不能直接剪枝

$$
d_{plane}\le d_{best}
$$

![[局部截取_20261031_004649.png]]
搜索范围已经碰到或跨过划分平面，另一侧可能存在更近的点，需要继续检查。

实际算法中使用平方距离，避免开平方：

```
// 查询点与划分平面的距离平方
float PlaneDistSq = Delta * Delta;

// 当前找到的最小距离平方
if (PlaneDistSq <= BestDistSq)
{
    // 划分平面与当前搜索半径相交
    // 另一侧子树可能有更近的点，需要检查
    Search(FarChild);
}
else
{
    // 另一侧不可能存在更近的点
    // 直接剪枝，跳过整个子树
}
```

这段代码就是 KD-Tree 查询算法的核心。

需要注意：`PlaneDistSq <= BestDistSq` 只是说明另一侧可能存在更近的点，不是说一定存在。

### 4.3 用之前的例子完整演示剪枝

之前的搜索路径是：

```
F(7,2)
  \
   C(9,6)
   /
  E(8,1)
```

我们已经搜索到了 E：

$$
BestDistSq=1.25
$$

现在开始回溯。

第一步：从 E 回溯到 C

C 使用 Y 轴进行划分。

因此：

$$
\Delta Y=Q_y-C_y=2.1-6=-3.9
$$

$$
PlaneDistSq=(-3.9)^2=15.21
$$

比较：

$$
15.21>1.25
$$

因此 C 的另一侧空间可以直接剪枝，不需要继续搜索。

第二步：继续回溯到 F

F 使用 X 轴划分。

$$
\Delta X=Q_x-F_x=8.2-7=1.2
$$

$$
PlaneDistSq=1.2^2=1.44
$$

比较：

$$
1.44>1.25
$$

这意味着左子树中的任何点，其到 Q 的距离至少为 1.2。

而已经找到的 E 到 Q 的距离为：

$$
\sqrt{1.25}\approx1.118
$$

因此，F 的整个左子树都可以剪掉。

```
                    F(7,2)
                   /      \
                  /        \
              [剪枝]       C(9,6)
               B          /
              / \        E(8,1)
             A   D       ↑
                       最近点
```

最终：

$$
\boxed{NearestPoint=E(8,1)}
$$

$$
\boxed{Distance=\sqrt{1.25}\approx1.118}
$$

原本有 6 个点，但我们只需要对 F、C、E 这 3 个点计算完整距离。

需要强调的是，这只是一个小规模示例。实际加速效果会受到点的分布、维度、查询位置以及剪枝效果影响。

## 五、KD-Tree 最近邻搜索算法总结

现在把整个流程整理成算法。

![[局部截取_20261031_005056.png]]


可以把这个过程理解成：

先按照坐标不断深入，找到一个比较近的候选点，再利用这个点的距离作为搜索半径，不断排除不可能存在更近点的子树。

其伪代码如下：

```
void NearestSearch(Node* Node, Point Query)
{
    if (Node == nullptr)
        return;

    // 1. 计算查询点与当前节点的距离
    float DistSq = DistanceSquared(Query, Node->Point);

    // 2. 更新当前找到的最近点
    if (DistSq < BestDistSq)
    {
        BestDistSq = DistSq;
        BestPoint = Node->Point;
    }

    // 3. 获取当前节点的划分轴
    int Axis = Node->Axis;

    // 4. 查询点与划分平面之间的有符号距离
    float Delta = Query[Axis] - Node->Point[Axis];

    // 5. 根据查询点的位置，判断优先搜索哪棵子树
    Node* NearChild = Delta < 0 ? Node->Left  : Node->Right;
    Node* FarChild  = Delta < 0 ? Node->Right : Node->Left;

    // 6. 优先递归搜索较近的一侧
    NearestSearch(NearChild, Query);

    // 7. 回溯时检查是否需要搜索另一侧
    // 注意 BestDistSq 可能已被前面的递归更新
    if (Delta * Delta <= BestDistSq)
    {
        NearestSearch(FarChild, Query);
    }
}
```

这里有一个面试中非常值得关注的细节：

为什么判断 `Delta * Delta <= BestDistSq` 必须放在搜索 `NearChild` 之后？

因为优先搜索较近的子树，很可能找到一个更近的点，从而缩小 `BestDistSq`。

最近距离越小，能够剪枝的空间通常就越大。

所以先搜索近侧，可以提高后续回溯时的剪枝效率。

## 六、KD-Tree 的其他两种重要查询

KD-Tree 不只可以查询最近的一个点，还可以执行 KNN 和半径查询。

### 6.1 KNN：K Nearest Neighbors（K个最近邻）

假设一个角色希望找到距离自己最近的 5 个敌人。

这时就可以使用 KNN。

与普通最近邻搜索相比，主要区别是：

普通最近邻只需要维护：

```
float BestDistSq;
Point BestPoint;
```

而 KNN 需要保存多个候选点。

一种常用的方法是最大堆（Max Heap）。

当需要搜索最近的 K 个点时，让最大堆保存当前找到的 K 个最近点，并且把这 K 个点中距离最远的放在堆顶。

例如 K=3，当前堆中保存：

```
敌人A：距离 2米
敌人B：距离 5米
敌人C：距离 7米  <-- 堆顶
```

发现一个距离为 4 米的新敌人：

```
新敌人距离 4米 < 当前堆顶 7米
```

于是移除 7 米的敌人，插入 4 米的敌人。

新的 K 个最近点就是：

```
敌人A：2米
新敌人：4米
敌人B：5米
```

此时最大堆的堆顶距离为 5 米。

这个值也是回溯剪枝的依据：

$$
d_{plane}^2 > d_{worst}^2
$$

如果堆中已经有 K 个点，并且划分平面的距离大于当前第 K 近点距离，就可以剪掉另一侧子树。

如果堆中还不足 K 个点，则不能使用当前堆顶距离排除另一侧，因为可能还需要更多候选点。

这一点是 KNN 与普通最近邻查询最重要的区别。

### 6.2 Radius Search（半径范围查询）

假设角色要对周围半径 10 米内的所有敌人造成伤害。

查询目标为：

$$
d(Q,P)\le R
$$

其中：

- Q 是技能释放中心。
    
- P 是被检测的敌人。
    
- R 是技能半径。
    

KD-Tree 的做法是：遍历节点时判断其是否落入半径内，并使用划分平面进行剪枝。

```
void RadiusSearch(Node* Node, Point Query, float RadiusSq)
{
    if (Node == nullptr)
        return;

    // 计算目标点到当前节点的距离平方
    float DistSq = DistanceSquared(Query, Node->Point);

    // 当前节点在技能范围内
    if (DistSq <= RadiusSq)
    {
        Results.push_back(Node->Point);
    }

    // 按照划分轴寻找优先访问的一侧
    int Axis = Node->Axis;

    float Delta = Query[Axis] - Node->Point[Axis];

    Node* NearChild = Delta < 0 ? Node->Left : Node->Right;
    Node* FarChild = Delta < 0 ? Node->Right : Node->Left;

    // 优先查询距离技能中心较近的一侧
    RadiusSearch(NearChild, Query, RadiusSq);

    // 如果技能范围碰到了划分平面
    // 另一侧也可能存在受到技能影响的目标
    if (Delta * Delta <= RadiusSq)
    {
        RadiusSearch(FarChild, Query, RadiusSq);
    }
}
```

这里与最近邻搜索不同的是：

最近邻搜索的判断依据是动态更新的 `BestDistSq`。

半径查询的判断依据是固定的 `RadiusSq`。

此外，半径查询可能返回多个甚至全部点，因此查询开销还与结果数量有关。

## 七、C++ 完整实现：三维 KD-Tree

下面使用 C++17 实现一个可以直接运行的三维 KD-Tree，包含建树、最近邻查询和半径查询。

代码大量使用中文注释，并且不依赖 UE，方便理解核心算法。

```

#include <iostream>
#include <vector>
#include <array>
#include <memory>
#include <algorithm>
#include <limits>
#include <cmath>
#include <utility>

class KDTree3D
{
public:
    // 三维空间点：X、Y、Z
    using Point = std::array<float, 3>;

private:
    // KD-Tree 节点
    struct Node
    {
        // 当前节点在 Points 数组中的下标
        size_t PointIndex = 0;

        // 当前节点的划分轴：0=X，1=Y，2=Z
        int Axis = 0;

        // 左右子节点
        std::unique_ptr<Node> Left;
        std::unique_ptr<Node> Right;
    };

    // 存储所有空间点
    std::vector<Point> Points;

    // KD-Tree 根节点
    std::unique_ptr<Node> Root;

public:
    explicit KDTree3D(std::vector<Point> InPoints)
        : Points(std::move(InPoints))
    {
        // 根据传入的点一次性构建 KD-Tree
        Root = Build(0, Points.size(), 0);
    }

    // 查询距离目标点最近的节点
    // 返回 false 表示树为空
    bool FindNearest(
        const Point& Query,
        Point& OutPoint,
        float& OutDistance) const
    {
        if (!Root)
        {
            return false;
        }

        // 初始最近距离设置为无穷大
        float BestDistSq =
            std::numeric_limits<float>::infinity();

        const Node* BestNode = nullptr;

        // 递归执行最近邻查询
        SearchNearest(
            Root.get(),
            Query,
            BestDistSq,
            BestNode
        );

        // 查询结束，输出最近点
        OutPoint = Points[BestNode->PointIndex];

        // 只有最终输出真实距离时才开平方
        OutDistance = std::sqrt(BestDistSq);

        return true;
    }

    // 查找指定半径内的所有点
    std::vector<Point> FindWithinRadius(
        const Point& Query,
        float Radius) const
    {
        std::vector<Point> Result;

        if (Radius < 0)
        {
            return Result;
        }

        // 只计算一次半径平方
        const float RadiusSq = Radius * Radius;

        SearchRadius(
            Root.get(),
            Query,
            RadiusSq,
            Result
        );

        return Result;
    }

private:
    // 计算两个三维点之间的距离平方
    static float DistanceSquared(
        const Point& A,
        const Point& B)
    {
        float DX = A[0] - B[0];
        float DY = A[1] - B[1];
        float DZ = A[2] - B[2];

        return DX * DX + DY * DY + DZ * DZ;
    }

    // 构建 KD-Tree
    // 使用半开区间 [Begin, End)
    std::unique_ptr<Node> Build(
        size_t Begin,
        size_t End,
        int Depth)
    {
        // 当前范围没有数据
        if (Begin >= End)
        {
            return nullptr;
        }

        // 根据当前深度，循环选择 X、Y、Z 轴
        int Axis = Depth % 3;

        // 找到中位点的下标
        size_t Mid = Begin + (End - Begin) / 2;

        // 将当前范围按照指定坐标轴进行中位数划分
        // nth_element 不会将所有点完全排序
        // 只确保中位点处在正确的划分位置
        std::nth_element(
            Points.begin() + Begin,
            Points.begin() + Mid,
            Points.begin() + End,
            [Axis](const Point& A, const Point& B)
            {
                return A[Axis] < B[Axis];
            }
        );

        // 创建当前节点
        auto Current = std::make_unique<Node>();

        Current->PointIndex = Mid;
        Current->Axis = Axis;

        // 递归构建左子树
        Current->Left = Build(
            Begin,
            Mid,
            Depth + 1
        );

        // 递归构建右子树
        Current->Right = Build(
            Mid + 1,
            End,
            Depth + 1
        );

        return Current;
    }

    // 最近邻递归搜索
    void SearchNearest(
        const Node* Current,
        const Point& Query,
        float& BestDistSq,
        const Node*& BestNode) const
    {
        if (!Current)
        {
            return;
        }

        const Point& CurrentPoint =
            Points[Current->PointIndex];

        // 1. 计算当前节点到查询点的距离平方
        float DistSq =
            DistanceSquared(CurrentPoint, Query);

        // 2. 更新当前找到的最近点
        if (DistSq < BestDistSq)
        {
            BestDistSq = DistSq;
            BestNode = Current;
        }

        // 3. 获取当前节点的划分轴
        int Axis = Current->Axis;

        // 4. 查询点到当前划分平面的有符号距离
        float Delta =
            Query[Axis] - CurrentPoint[Axis];

        // 5. 确定更近与更远的子树
        const Node* NearChild;
        const Node* FarChild;

        if (Delta < 0)
        {
            NearChild = Current->Left.get();
            FarChild = Current->Right.get();
        }
        else
        {
            NearChild = Current->Right.get();
            FarChild = Current->Left.get();
        }

        // 6. 优先搜索查询点所在的一侧
        SearchNearest(
            NearChild,
            Query,
            BestDistSq,
            BestNode
        );

        // 7. 回溯剪枝
        // 当前最近距离可能已在 NearChild 中被更新
        float PlaneDistSq = Delta * Delta;

        if (PlaneDistSq <= BestDistSq)
        {
            // 另一侧可能存在更近的点
            SearchNearest(
                FarChild,
                Query,
                BestDistSq,
                BestNode
            );
        }

        // 否则整个 FarChild 子树都被剪掉
    }

    // 半径范围查询
    void SearchRadius(
        const Node* Current,
        const Point& Query,
        float RadiusSq,
        std::vector<Point>& Result) const
    {
        if (!Current)
        {
            return;
        }

        const Point& CurrentPoint =
            Points[Current->PointIndex];

        // 1. 判断当前点是否在查询半径内
        float DistSq =
            DistanceSquared(CurrentPoint, Query);

        if (DistSq <= RadiusSq)
        {
            Result.push_back(CurrentPoint);
        }

        // 2. 获取当前划分轴
        int Axis = Current->Axis;

        float Delta =
            Query[Axis] - CurrentPoint[Axis];

        // 3. 确定优先搜索方向
        const Node* NearChild =
            Delta < 0
            ? Current->Left.get()
            : Current->Right.get();

        const Node* FarChild =
            Delta < 0
            ? Current->Right.get()
            : Current->Left.get();

        // 4. 搜索较近的一侧
        SearchRadius(
            NearChild,
            Query,
            RadiusSq,
            Result
        );

        // 5. 判断查询半径是否跨越划分平面
        if (Delta * Delta <= RadiusSq)
        {
            SearchRadius(
                FarChild,
                Query,
                RadiusSq,
                Result
            );
        }
    }
};

int main()
{
    // 构建一组三维空间点
    // 为了对应前面的二维案例，Z 统一设置为 0
    std::vector<KDTree3D::Point> Points =
    {
        {2, 3, 0},
        {5, 4, 0},
        {9, 6, 0},
        {4, 7, 0},
        {8, 1, 0},
        {7, 2, 0}
    };

    // 创建 KD-Tree
    KDTree3D Tree(std::move(Points));

    // 查询位置
    KDTree3D::Point Query = {8.2f, 2.1f, 0.0f};

    // 最近邻查询
    KDTree3D::Point Nearest;
    float Distance = 0.0f;

    if (Tree.FindNearest(Query, Nearest, Distance))
    {
        std::cout
            << "Nearest: "
            << Nearest[0] << ", "
            << Nearest[1] << ", "
            << Nearest[2] << "\n";

        std::cout
            << "Distance: "
            << Distance << "\n";
    }

    // 半径查询
    auto NearbyPoints =
        Tree.FindWithinRadius(Query, 3.0f);

    std::cout
        << "Points within radius 3: "
        << NearbyPoints.size()
        << "\n";

    return 0;
}
```

该实现采用中位点建树，支持精确最近邻查询和半径查询。

需要注意，它是静态数据版本。建立索引后，不提供点位置修改接口。如果点的位置发生变化，需要更新或重建数据结构，不能仅修改外部 Actor 的位置而让 KD-Tree 自动同步。

### 7.1 为什么使用 `std::nth_element`，而不是 `std::sort`？

建树时，我们其实没有必要把每个节点范围内的数据全部排序。

例如：

```
原始数据：

9  2  8  4  7  1  6
```

我们真正关心的是中位数：

```
小于中位点的元素 | 中位点 | 大于中位点的元素
```

不要求左右两边的元素完全有序。

`std::nth_element` 恰好适合这个需求。

对于 N 个元素：

|算法|平均比较复杂度|是否完全排序|
|---|---|---|
|`std::sort`|O(N log N)|是|
|`std::nth_element`|O(N)|否，只保证中位点的划分性质|

如果每一层都对当前子数组完整排序，平衡建树的复杂度通常为：

$$
O(N\log^2N)
$$

使用 `std::nth_element`，在平均线性划分的条件下：

$$
O(N\log N)
$$

因此 `std::nth_element` 是常用的 KD-Tree 建树优化方式。

## 八、KD-Tree 时间复杂度分析

这是 KD-Tree 高频面试内容。

|操作|常见情况|最坏情况|
|---|---|---|
|建树（中位点划分）|O(N log N)|取决于中位数选择实现|
|最近邻查询|低维、分布合理时常接近 O(log N)|O(N)|
|半径查询|与半径、分布、结果数量相关|O(N)|
|插入一个点|平衡时 O(log N)|O(N)|
|删除一个点|依赖删除和重平衡策略|O(N) 或需要重建|
|空间存储|O(N)|O(N)|

### 8.1 为什么最近邻不是一定 O(log N)？

因为树高为 
$$
O(log N)
$$
不代表实际只访问一条从根到叶子的路径。

最近邻搜索可能发生回溯。

考虑每个节点的判断：

$$
d_{plane}^2\le BestDistSq
$$

如果这个条件频繁满足，就可能不断进入另一侧子树。

最坏情况下，需要访问全部 N 个节点。

因此：

$$
T_{worst}=O(N)
$$

例如，点分布非常不均匀、查询点距离大量划分平面很近，或者空间维度较高时，KD-Tree 的剪枝效果都可能下降。

### 8.2 为什么高维空间会降低 KD-Tree 的效率？

这个现象叫作：

Curse of Dimensionality（维度灾难）。

在二维或三维空间中，通过一条线或一个平面划分，很容易把大量不相关的点排除。

但是在高维空间中，距离的区分能力往往下降，搜索球可能跨越大量空间分区，导致频繁回溯。

例如：

- 2D：查找地图上的附近敌人，比较适合。
    
- 3D：查找空间中最近的导航点，比较适合。
    
- 128维：如果用于高维特征向量的精确最近邻查询，传统 KD-Tree 往往不再理想。
    

实际效果并没有一个固定维度阈值，需要结合数据分布、点数量和查询模式测试。

## 九、KD-Tree 的常见优化

### 9.1 优化一：按照坐标方差选择划分轴

之前我们使用：

```
int Axis = Depth % 3;
```

这种策略的优点是简单，划分轴均匀轮换。

但是实际游戏场景可能具有特殊分布。

例如一个大地图：

```
X 方向：10000米
Y 方向：10000米
Z 方向：100米
```

空间主要分布在水平面上。

若机械地按照 X、Y、Z 轮换，可能经常沿变化范围很小的 Z 轴切分。

一种优化方案是：

每次选择当前节点集合中坐标跨度最大或方差最大的轴。

例如：

$$
Variance_X=E[(X-E[X])^2]
$$

$$
Variance_Y=E[(Y-E[Y])^2]
$$

$$
Variance_Z=E[(Z-E[Z])^2]
$$

选择：

$$
Axis=\arg\max(Variance_X,Variance_Y,Variance_Z)
$$

这样可能得到空间区分能力更好的划分。

但是额外计算方差也会增加建树成本，因此并不是所有情况都优于循环划分。

### 9.2 优化二：为子树记录 AABB

基础 KD-Tree 查询时，使用节点划分平面进行剪枝。

但划分平面给出的只是一个粗略的空间下界。

如果每个节点额外保存整棵子树的 AABB（Axis-Aligned Bounding Box，轴对齐包围盒），就能进行更强的剪枝。

![[局部截取_20261031_005254.png]]

对于查询点 Q 和 AABB，可以计算：

$$
d^2(Q,AABB)=\sum_{i=1}^{K} \left[ \max(Min_i-Q_i,\ 0,\ Q_i-Max_i) \right]^2
$$

如果：

$$
d^2(Q,AABB)>BestDistSq
$$

那么可以直接跳过该 AABB 对应的整棵子树。

这往往比单纯检查划分平面更有效，因为 AABB 同时考虑了多个维度的空间边界。

不过代价是节点需要保存更多数据，建树和动态更新也会更加复杂。

### 9.3 优化三：改善内存布局

前面的代码使用：

```
std::unique_ptr<Node> Left;
std::unique_ptr<Node> Right;
```

这种实现易读、便于理解，但每个节点独立分配内存，会产生额外开销。

对于几十万甚至上百万个节点，可能出现：

- 节点内存不连续。
    
- 访问子节点时产生较多 Cache Miss（缓存未命中）。
    
- 频繁的小块内存分配带来额外开销。
    

更高效的实现可以将节点保存在连续数组中：

```
struct KDNode
{
    FVector Position;

    // 使用下标代替左右节点指针
    int LeftIndex = -1;
    int RightIndex = -1;

    uint8 Axis = 0;
};

TArray<KDNode> Nodes;
```

这样可以减少独立内存分配，也更方便进行内存池管理。

不过是否能进一步提升 CPU Cache 命中率，还取决于节点的排列顺序及查询访问模式。

## 十、KD-Tree 与八叉树、四叉树、BVH 的区别

这是游戏引擎岗位面试中特别容易一起考察的内容。

KD-Tree、QuadTree、Octree、BVH 都用于空间组织，但是划分方式不同。

KD-Tree
按照选定维度递归二分空间。
![[Pasted image 20261031005335.png]]

QuadTree（四叉树）
将二维区域递归分为四个子区域。

![[Pasted image 20261031005353.png]]

Octree（八叉树）
将三维空间递归分为八个子区域。

![[Pasted image 20261031005358.png]]

BVH（包围体层次结构）
使用层次包围体组织空间对象。
![[Pasted image 20261031005405.png]]

|数据结构|划分方式|主要适用场景|
|---|---|---|
|KD-Tree|沿坐标轴二分|点数据、最近邻、KNN|
|QuadTree|二维空间四分|2D 地图、区域索引|
|Octree|三维空间八分|3D 场景、空间对象管理|
|BVH|递归组织包围体|射线检测、几何查询、碰撞加速|
|Spatial Hash|映射到离散网格单元|大量动态对象邻域查询|
|Dynamic AABB Tree|动态更新层次包围盒|动态碰撞 Broad Phase|

### 10.1 KD-Tree 和 Octree 最重要的区别

KD-Tree：

每次只按一个维度二分，一个节点最多有两个子节点。

```
                  Root
                 /    \
                A      B
               / \    / \
              C   D  E   F
```

Octree：

每次把三维空间分为 8 个区域，一个节点最多有 8 个子节点。

```
                   Root
             / / / / \ \ \ \
            0 1 2 3   4 5 6 7
```

两者的关键区别不是简单地说谁更快。

KD-Tree 的二分位置可以根据点的实际分布调整，而传统规则八叉树通常按照空间区域的几何中点进行划分。

因此，KD-Tree 对静态点集的最近邻查询通常很有竞争力；八叉树更适合按空间区域层次管理三维对象。

### 10.2 KD-Tree 和 BVH 的区别

KD-Tree 更典型的用途是索引空间点。

BVH 更典型的用途是索引具有空间体积的对象，例如三角形、Mesh 或包围盒。

KD-Tree 使用空间划分平面。

BVH 使用 Bounding Volume（包围体）构建层次结构。

例如场景中有大量 Mesh，希望执行 Raycast（射线检测）：

BVH 可以先检测射线与较大的包围盒是否相交，不相交就直接跳过整组几何体。

KD-Tree 也有用于射线追踪和几何索引的变体，但它们通常会储存空间单元和几何引用，与这里演示的点 KD-Tree 实现不完全相同。

因此不能认为 KD-Tree 一定比 BVH 更适合碰撞检测。

## 十一、KD-Tree 在 UE5 游戏开发中的应用

假设我们正在开发一款类似大型开放世界 ARPG 的游戏，场景中有大量 AI、战斗对象和导航节点。

### 11.1 场景一：查找最近的敌人

角色具有自动索敌功能，需要查找距离玩家最近的敌人。

如果场景有 5000 个敌人，每个角色都使用：

```
for (AActor* Enemy : AllEnemies)
{
    // 检查每个敌人的距离
}
```

那么查询量可能很大。

可以考虑把敌人的位置加入空间索引，通过 KD-Tree 加速最近邻查询。

不过这里存在一个问题：

敌人是动态移动的。

如果敌人的位置每帧都发生变化，那么静态 KD-Tree 就需要更新。

这也是 KD-Tree 在游戏运行时使用的一大限制。

如果对象数量很大、移动频繁，而且主要需求是附近对象查询，那么 Spatial Hash 或动态 AABB Tree 可能更合适。

### 11.2 场景二：最近导航点查询

这个场景通常更适合 KD-Tree。

假设地图预先生成了大量导航采样点：

```
TArray<FVector> NavigationPoints;
```

这些点大多不会频繁变化。

游戏运行时，角色需要查找距离自己最近的导航采样点。

此时可以预先建树：

```
// 伪代码：在关卡加载或导航数据准备完成后
KDTree.Build(NavigationPoints);
```

在查询时：

```
// 伪代码：查找角色附近最近的导航采样点
FVector NearestPoint = KDTree.FindNearest(
    Character->GetActorLocation()
);
```

由于点集较稳定，通常不需要频繁重建。

当然，UE 自带的导航系统本身已经提供 NavMesh 查询功能。实际项目中如果需要的是可行走表面上的有效位置，不能单纯依赖三维欧氏距离最近的采样点来替代导航系统查询。

### 11.3 场景三：范围技能搜索敌人

例如角色释放半径为 500cm 的 AOE 技能。

KD-Tree 可以用于：

```
// 伪代码
TArray<FVector> NearbyPoints =
    KDTree.FindWithinRadius(
        CharacterLocation,
        500.0f
    );
```

然后对返回的对象进行阵营、死亡状态、伤害规则等检查。

不过在 UE 中，如果目标是实际的碰撞组件，通常会优先评估引擎提供的物理空间查询，例如：

```
TArray<FOverlapResult> OverlapResults;

// 使用 UE 内置的球形重叠查询
bool bHasOverlap = GetWorld()->OverlapMultiByObjectType(
    OverlapResults,
    SkillCenter,
    FQuat::Identity,
    ObjectQueryParams,
    FCollisionShape::MakeSphere(500.0f)
);
```

这样可以直接使用引擎已有的碰撞加速结构和过滤机制。

KD-Tree 不应该简单地理解为 UE 内置物理查询的通用替代品。

## 十二、KD-Tree 的优点和缺点

|优点|说明|
|---|---|
|空间查询高效|可以跳过大量不相关的空间节点|
|适合最近邻查询|经典的精确 NN、KNN 数据结构|
|适合静态点数据|不需要频繁调整树结构|
|支持多维空间|2D、3D 乃至更高维度|
|平衡树高度较低|通过中位数划分控制树高|

|缺点|说明|
|---|---|
|动态更新不方便|点大量移动后索引可能失效，需要更新或重建|
|高维空间性能下降|容易出现维度灾难|
|最坏查询复杂度 O(N)|回溯可能访问大量节点|
|空间体积对象处理较复杂|基础点 KD-Tree 不直接表达对象尺寸|
|内存布局需要优化|指针式节点可能导致 Cache Miss|

## 十三、KD-Tree 常见面试问题

Q1：KD-Tree 为什么能够加速最近邻搜索？

因为它提前按照多维坐标划分空间。查询时可以根据查询点与划分平面的距离，判断另一侧子树是否有可能存在更近的点，如果不可能，就直接剪枝，避免遍历所有点。

Q2：为什么 KD-Tree 建树时通常选择中位点？

因为中位点可以让左右子树的节点数量接近，从而控制树高，避免树退化为链表。平衡树的高度通常为 \(O(\log N)\)。

Q3：KD-Tree 为什么查询时需要回溯？

因为查询点所在的一侧未必存在全局最近邻。另一侧子树中的点可能更近，因此需要通过当前最近距离与划分平面的距离进行判断，决定是否搜索另一侧。

Q4：KD-Tree 为什么不适合大量高频移动的对象？

因为 KD-Tree 的划分结构依赖点的坐标。对象移动后，如果不更新节点关系，就无法保证空间划分性质仍然成立。大量更新或者频繁重建可能抵消查询加速带来的收益。

Q5：KD-Tree 和 BVH 哪个更适合碰撞检测？

取决于处理的数据和查询类型。如果主要处理静态空间点的最近邻，KD-Tree 往往合适。如果需要对具有体积的碰撞对象执行射线、包围盒、重叠等空间查询，BVH 或其他专用物理加速结构通常更自然。

Q6：KD-Tree 最近邻查询的时间复杂度是不是 O(log N)？

不能保证。平衡 KD-Tree 的树高是 \(O(\log N)\)，但最近邻搜索可能回溯多个子树。低维、分布合理时通常可以获得很好的查询性能，最坏情况下仍为 \(O(N)\)。

## 十四、面试重点总结

理解 KD-Tree，最重要的是区分三个阶段：

建树（Build）

根据划分轴选择中位点，递归构建相对平衡的二叉树。

查询（Search）

根据查询点的位置，优先递归搜索更近的一侧。

回溯剪枝（Backtracking & Pruning）

如果查询点到划分平面的距离大于当前最近距离，就可以排除另一侧子树。

其中最重要的公式是：

$$
\boxed{\Delta^2=(Q[Axis]-P[Axis])^2}
$$

$$
\boxed{\Delta^2>BestDistSq\ \Rightarrow\ 可以剪枝}
$$

对于 UE 游戏客户端开发岗位，建议优先掌握 KD-Tree 的中位数建树、最近邻回溯剪枝、KNN、半径查询，以及它与八叉树、BVH、Spatial Hash 在静态场景和动态战斗对象中的适用性区别。 这些知识既能用于算法面试，也能帮助判断实际项目应该选择哪种空间加速结构。