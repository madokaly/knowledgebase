 SAT 是碰撞检测里的 **Separating Axis Theorem，分离轴定理**，那它是游戏物理系统里非常重要的一套凸体碰撞判定方法，尤其常见于 **OBB、凸多边形、凸多面体** 的碰撞检测。

它最核心的一句话是：

> **两个凸物体如果没有相交，那么一定存在一条轴，使两个物体投影到这条轴后的区间完全分离。**

反过来：

> **如果在所有需要检测的候选轴上，两个物体的投影区间都发生重叠，那么两个凸物体一定相交。**

下面从直觉、数学、2D、3D OBB、碰撞深度一直讲到实际实现。

---

# 1. SAT 到底在解决什么问题

假设有两个二维凸多边形：

```
       A
     /---\
    /     \
   /_______\

                   B
                /-----\
               /       \
               \-------/
```

人眼很容易看出来它们没有碰撞。

但计算机怎么判断？

一种思路是：

- 判断所有边是否相交；
- 判断点是否在多边形内部。

但对于复杂的凸体，这样做并不总是最方便。

SAT 换了一个思路：

> 不直接判断二维/三维空间里的形状是否相交，而是把它们投影到一维直线上。

例如选择一个方向：

```
Axis ------------------------->
```

把 A 投影：

```
       A
      / \
     /___\
       ↓
Axis ----[AAAAA]----------------
```

把 B 投影：

```
                  B
                 / \
                /___\
                  ↓
Axis ----------------[BBBB]-----
```

得到两个一维区间：

```
A = [2, 6]
B = [9, 12]
```

显然：

```
6 < 9
```

没有重叠。

那么就可以直接得出：

```
A 和 B 不相交
```

这条 Axis 就叫：

**Separating Axis，分离轴。**

---

# 2. 为什么只找到“一条”分离轴就够了

想象两个箱子：

```
+-------+
|   A   |
+-------+

            +-------+
            |   B   |
            +-------+
```

只要存在一个方向，使得：

```
A投影      B投影
[------]   [------]
```

中间有间隔，那么这两个物体就不可能接触。

因此 SAT 的判断逻辑是：

```
for (每一个候选轴)
{
    投影A;
    投影B;

    if (A与B在这个轴上不重叠)
    {
        return false; // 一定没有碰撞
    }
}

return true; // 所有轴都重叠，因此发生碰撞
```

这是 SAT 最重要的代码思想。

---

# 3. 投影是怎么计算的

假设我们有一个点：

\[ P=(x,y) \]

以及一个单位方向轴：

\[ L=(l_x,l_y) \]

点 P 投影到 L 上，本质就是做：

\[ projection=P\cdot L \]

也就是点积：

\[ projection=x l_x+y l_y \]

例如：

```
P = (3, 4)

Axis = (1, 0)
```

那么：

\[ P\cdot Axis = 3\times1+4\times0 = 3 \]

所以这个点在 X 方向上的投影值就是：

```
3
```

---

# 4. 一个多边形怎么投影

假设一个矩形有四个顶点：

```
P0
+---------+ P1
|         |
|         |
+---------+
P3        P2
```

对于某个 Axis：

```
float min = +INF;
float max = -INF;

for (Point p : Polygon)
{
    float projection = Dot(p, axis);

    min = Min(min, projection);
    max = Max(max, projection);
}
```

最终得到：

```
[min, max]
```

也就是说：

> 一个二维多边形投影到一条直线上以后，就退化成了一个一维区间。

于是二维碰撞问题变成了一维区间重叠问题。

---

# 5. 两个区间如何判断重叠

假设：

```
A = [Amin, Amax]
B = [Bmin, Bmax]
```

不重叠只有两种情况：

```
AAAA
      BBBB
```

或者：

```
BBBB
      AAAA
```

数学上：

\[ A_{max}<B_{min} \]

或者：

\[ B_{max}<A_{min} \]

因此：

```
if (maxA < minB || maxB < minA)
{
    // 存在分离
    return false;
}
```

也可以写成：

```
bool overlap =
    maxA >= minB &&
    maxB >= minA;
```

---

# 6. SAT 最关键的问题：到底检测哪些轴？

如果理论上检查：

```
无限多个方向
```

SAT 根本没法实际运行。

SAT 真正巧妙的地方就在于：

> 对凸多边形，不需要测试无限多个轴，只需要测试有限的一组候选分离轴。

对于二维凸多边形：

> **只需要检测两个多边形所有边的法线。**

这一点极其重要。

---

# 7. 为什么二维只需要检查边的法线？

假设一个矩形：

```
   P0---------P1
   |           |
   |           |
   |           |
   P3---------P2
```

有一条边：

```
Edge = P1 - P0
```

例如：

```
Edge = (1, 0)
```

它的法线可以写成：

```
Normal = (0, 1)
```

二维中，如果：

```
Edge = (x, y)
```

那么一个垂直方向是：

```
Normal = (-y, x);
```

因此可以遍历：

```
A 的每条边法线
+
B 的每条边法线
```

作为 SAT 候选轴。

---

# 8. 二维 SAT 示例

两个矩形：

```
        A
    +--------+
    |        |
    |        |
    +--------+

             /------/
            /  B   /
           /------/
```

A 的边方向可能是：

```
A.Right
A.Up
```

它们的法线也是对应两个方向。

B 旋转了，因此有：

```
B.Right
B.Up
```

所以只需要测试：

```
Axis 1 = A.Right
Axis 2 = A.Up
Axis 3 = B.Right
Axis 4 = B.Up
```

严格来说边法线和 box 自身局部轴只是互相对应，因此矩形最终只有四个候选方向。

---

# 9. 一个简单的二维 SAT 实现

例如：

```
struct Projection
{
    float Min;
    float Max;
};

Projection ProjectPolygon(
    const std::vector<Vector2>& Vertices,
    const Vector2& Axis)
{
    float Min = Dot(Vertices[0], Axis);
    float Max = Min;

    for (int i = 1; i < Vertices.size(); ++i)
    {
        float P = Dot(Vertices[i], Axis);

        Min = std::min(Min, P);
        Max = std::max(Max, P);
    }

    return { Min, Max };
}
```

然后：

```
bool Overlap(
    const Projection& A,
    const Projection& B)
{
    return
        A.Max >= B.Min &&
        B.Max >= A.Min;
}
```

SAT：

```
bool SAT(
    const std::vector<Vector2>& A,
    const std::vector<Vector2>& B)
{
    // 检测A每条边的法线
    for (int i = 0; i < A.size(); ++i)
    {
        Vector2 P0 = A[i];
        Vector2 P1 = A[(i + 1) % A.size()];

        Vector2 Edge = P1 - P0;

        Vector2 Axis =
            Normalize(Vector2(-Edge.y, Edge.x));

        Projection PA = ProjectPolygon(A, Axis);
        Projection PB = ProjectPolygon(B, Axis);

        if (!Overlap(PA, PB))
            return false;
    }

    // 检测B每条边的法线
    for (int i = 0; i < B.size(); ++i)
    {
        Vector2 P0 = B[i];
        Vector2 P1 = B[(i + 1) % B.size()];

        Vector2 Edge = P1 - P0;

        Vector2 Axis =
            Normalize(Vector2(-Edge.y, Edge.x));

        Projection PA = ProjectPolygon(A, Axis);
        Projection PB = ProjectPolygon(B, Axis);

        if (!Overlap(PA, PB))
            return false;
    }

    return true;
}
```

整个算法实际上就是：

```
候选轴
  ↓
投影
  ↓
检查区间重叠
  ↓
发现一个不重叠
  ↓
没有碰撞
```

---

# 10. 为什么 SAT 要求“凸物体”

这个问题面试很容易问。

假设一个凹多边形：

```
+---------+
|         |
|   +-----+
|   |
|   +-----+
|         |
+---------+
```

投影以后：

```
[================]
```

投影区间内部的信息被压扁了。

也就是说：

```
二维凹陷结构
```

在一维投影中消失。

因此可能出现：

```
二维实际上没有碰撞

但是所有投影都重叠
```

导致误判。

所以标准 SAT 直接处理的是：

> **Convex Shape，凸体。**

如果是凹体，一般先：

```
Concave Mesh
       ↓
Convex Decomposition
       ↓
Convex A
Convex B
Convex C
...
```

然后分别检测。

游戏物理引擎大量使用凸包，本质上和这个原因有关。

---

# 11. SAT 不仅能判断碰撞，还能计算穿透深度

这一点特别重要。

假设某个轴上：

```
A:
    [-------------]

B:
          [-------------]
```

投影重叠区域是：

```
          [-----]
```

那么：

\[ Overlap = \min(A_{max},B_{max}) - \max(A_{min},B_{min}) \]

例如：

```
A = [2, 8]
B = [6, 11]
```

那么：

\[ Overlap = 8-6=2 \]

说明沿这个轴至少移动：

```
2
```

才能把两个物体分开。

---

# 12. MTV：Minimum Translation Vector

SAT 中另一个非常重要的概念：

**MTV，Minimum Translation Vector，最小平移向量。**

对每个 SAT 轴，都会得到一个 penetration depth：

```
Axis1 → 3.2
Axis2 → 0.8
Axis3 → 4.5
Axis4 → 2.1
```

选择最小的：

```
0.8
```

对应 Axis2。

于是：

```
CollisionNormal = Axis2

PenetrationDepth = 0.8
```

最终：

\[ MTV = Normal \times PenetrationDepth \]

也就是说：

> 沿这个方向移动最短的距离，就能让两个物体脱离。

---

# 13. 为什么最小重叠轴通常可以作为碰撞法线

例如：

```
        B
    +-------+
    |       |
----+-------+----
    |   A   |
    +-------+
```

假设 X 投影重叠：

```
4
```

而 Y 投影重叠：

```
0.2
```

那么显然只需要：

```
向Y方向推0.2
```

两个物体就分开了。

因此：

```
最小Overlap Axis
≈ 碰撞法线方向
```

于是 SAT 可以同时得到：

```
是否碰撞
碰撞法线
穿透深度
```

例如：

```
struct SATResult
{
    bool IsColliding;

    Vector2 Normal;

    float Penetration;
};
```

---

# 14. 法线方向还需要修正

SAT 找出来的轴通常有正负两个方向：

```
N

和

-N
```

它们表示同一条分离轴。

但物理解算需要确定：

```
到底从A指向B
还是从B指向A
```

通常：

```
Vector2 AB = CenterB - CenterA;

if (Dot(AB, Normal) < 0)
{
    Normal = -Normal;
}
```

这样保证：

```
Normal
```

始终：

```
A → B
```

---

# 15. 进入三维以后 SAT 会复杂很多

二维里：

```
候选轴 = 边法线
```

但三维不同。

三维凸多面体 SAT 的候选轴来自两类：

```
1. 两个多面体所有面的法线

2. A的边 × B的边
```

第二项非常容易被忽略。

数学上：

\[ Axis = Edge_A \times Edge_B \]

也就是：

**两个边方向的叉积。**

---

# 16. 为什么三维还需要 Edge × Edge

这是 SAT 最经典的面试题之一。

两个三维盒子可能出现：

```
没有任何一个面的法线
可以把两个物体分开
```

但某两条边之间存在分离方向。

例如大概这种空间关系：

```
Box A
─────────────

       ╱
      ╱
     ╱ Box B
```

两条边：

```
EA
EB
```

那么：

\[ EA\times EB \]

得到同时垂直于这两条边的方向：

```
        ↑ Axis
        |
EA -----+-----
       /
      /
     EB
```

这个方向可能正好是分离轴。

所以 3D SAT 不能只检查 Face Normal。

---

# 17. OBB vs OBB 的 SAT：非常经典

游戏开发尤其容易问：

> 两个 OBB 如何使用 SAT 判断相交？

一个 OBB 有三个局部坐标轴：

```
A:
A0
A1
A2
```

另一个：

```
B:
B0
B1
B2
```

候选轴总共：

\[ 3+3+3\times3 \]

也就是：

\[ 15 \]

条。

具体：

```
A0
A1
A2

B0
B1
B2
```

6 个面的法线方向。

以及：

```
A0 × B0
A0 × B1
A0 × B2

A1 × B0
A1 × B1
A1 × B2

A2 × B0
A2 × B1
A2 × B2
```

9 个边叉积方向。

因此：

\[ 6+9=15 \]

这就是著名的：

**OBB-OBB 15 Axis SAT Test。**

---

# 18. 这是非常重要的面试记忆点

如果面试官问：

> 两个三维 OBB 用 SAT 需要检查多少条轴？

直接回答：

> **最多 15 条。**

然后解释：

\[ 3+3+9=15 \]

即：

```
A 的3个局部轴
+
B 的3个局部轴
+
A每个轴 × B每个轴
```

实际上平行边产生的叉积可能为零，因此某些轴可以忽略。

---

# 19. OBB 投影不用真的求 8 个顶点

这是实现 SAT 时非常漂亮的优化。

假设 OBB：

```
Center = C

三个局部单位轴：

U0
U1
U2

HalfExtent：

e0
e1
e2
```

对于一个 SAT Axis：

```
L
```

OBB 中心投影：

\[ C_L=C\cdot L \]

OBB 在 L 方向上的投影半径为：

\[ r = e_0 |U_0\cdot L| + e_1 |U_1\cdot L| + e_2 |U_2\cdot L| \]

于是区间直接是：

\[ [C_L-r,\ C_L+r] \]

完全不用计算：

```
8个顶点
↓
逐顶点投影
```

---

# 20. 这个公式为什么成立

OBB 中任何一点可以写成：

\[ P=C+xU_0+yU_1+zU_2 \]

其中：

\[ |x|\le e_0 \]\[ |y|\le e_1 \]\[ |z|\le e_2 \]

投影到 L：

\[ P\cdot L \]

展开：

\[ C\cdot L + x(U_0\cdot L) + y(U_1\cdot L) + z(U_2\cdot L) \]

最大偏移量显然就是：

\[ e_0|U_0\cdot L| + e_1|U_1\cdot L| + e_2|U_2\cdot L| \]

所以这就是投影半径。

这类推导如果你去面试战斗、物理、引擎相关岗位，非常值得理解。

---

# 21. 甚至不需要真的构造投影区间

进一步优化。

两个 OBB：

```
A center = CA
B center = CB
```

定义：

\[ T=C_B-C_A \]

投影到轴 L：

\[ Distance=|T\cdot L| \]

分别计算两个 OBB 的投影半径：

\[ r_A \]\[ r_B \]

只需要判断：

\[ |T\cdot L|>r_A+r_B \]

如果成立：

```
存在间隔
```

因此：

```
不碰撞
```

否则：

```
该轴上重叠
```

这个公式非常经典：

\[ \boxed{ |T\cdot L| > r_A+r_B } \]

代表：

**发现分离轴。**

---

# 22. 画出来就很好理解

两个 OBB 的中心投影：

```
      CA                CB
       |                 |
-------●-----------------●-------- Axis

       <------ d -------->
```

A 投影半径：

```
      |---rA---|
```

B 投影半径：

```
                        |---rB---|
```

如果：

\[ d>r_A+r_B \]

那么：

```
[----A----]       [----B----]
```

中间一定有空隙。

如果：

\[ d\le r_A+r_B \]

则：

```
[------A------]
        [------B------]
```

存在重叠。

---

# 23. 对比 AABB 为什么简单得多

AABB：

**Axis Aligned Bounding Box，轴对齐包围盒。**

因为两个盒子的轴永远就是：

```
X
Y
Z
```

所以 SAT 实际上只需要检查：

```
X
Y
Z
```

例如：

```
if (A.MaxX < B.MinX || B.MaxX < A.MinX)
    return false;

if (A.MaxY < B.MinY || B.MaxY < A.MinY)
    return false;

if (A.MaxZ < B.MinZ || B.MaxZ < A.MinZ)
    return false;

return true;
```

所以你其实可以把：

**AABB 碰撞检测看成 SAT 的一个极度简化版本。**

这也是理解 SAT 很好的方式。

---

# 24. Sphere 为什么通常不用 SAT

球体：

```
Sphere A
Sphere B
```

直接判断：

$$
|C_B-C_A| \le R_A+R_B
$$

即可。

复杂度比 SAT 小得多。

所以一般：

```
Sphere vs Sphere
```

不会用 SAT。

而：

```
AABB vs AABB
```

直接范围测试。

```
OBB vs OBB
Convex Polygon vs Convex Polygon
Convex Polyhedron vs Convex Polyhedron
```

SAT 就比较适合。

---

# 25. SAT 与 Broad Phase / Narrow Phase 的关系

游戏物理系统一般不是：

```
世界里10000个Collider

任意两个都跑SAT
```

那会变成：

$$
O(N^2)
$$

非常贵。

一般碰撞检测分成：

```
Collision Detection

├── Broad Phase
│   ├── AABB
│   ├── BVH
│   ├── Sweep And Prune
│   └── Spatial Hash
│
└── Narrow Phase
    ├── SAT
    ├── GJK
    └── Specialized Tests
```

Broad Phase：

```
先粗略判断
```

例如：

```
10000 Collider
     ↓
Broad Phase
     ↓
找到20对“可能碰撞”
```

然后：

```
20 Candidate Pairs
     ↓
SAT / GJK
     ↓
精确碰撞检测
```

所以 SAT 通常属于：

**Narrow Phase，窄相碰撞检测。**

---

# 26. SAT 与 GJK 的关系

这是进一步学习物理系统时非常关键的一组算法。

两者都可以检测：

```
凸体 vs 凸体
```

SAT 思路：

```
找分离轴
```

GJK 思路：

```
Minkowski Difference
+
Support Mapping
```

简单比较：

||SAT|GJK|
|---|---|---|
|核心思想|找分离轴|Minkowski 差|
|适用对象|凸体|凸体|
|Box|很合适|可以|
|Polygon|很合适|可以|
|任意复杂凸体|候选轴可能很多|通常更合适|
|实现难度|相对直观|较高|
|距离查询|不突出|很擅长|
|穿透信息|可通过最小投影得到|通常结合 EPA|

因此很多物理系统会出现：

```
GJK
+
EPA
```

而简单凸多边形、OBB 等经常非常适合 SAT。

---

# 27. SAT 有个很重要的 Early Out

SAT 很适合提前退出。

假设要测试：

```
15条轴
```

第一条：

```
Overlap
```

第二条：

```
Overlap
```

第三条：

```
Separated
```

那么马上：

```
return false;
```

后面的：

```
12条轴
```

全部不需要测试。

因此在大量“不发生碰撞”的物体之间，SAT 往往可以很快退出。

通常可以优先检测：

```
最容易出现分离的轴
```

进一步提升效率。

---

# 28. 数值误差问题

实际工程实现里不能永远这么判断：

```
if (distance > radiusA + radiusB)
```

浮点误差可能导致接触状态抖动。

所以常出现：

```
constexpr float Epsilon = 1e-6f;

if (distance > radiusA + radiusB + Epsilon)
{
    return false;
}
```

尤其计算：

```
EdgeA × EdgeB
```

时，如果两条边几乎平行：

$$
|A\times B|\approx0
$$

得到的 Axis 接近零向量。

这种轴应该跳过：

```
Vector3 Axis = Cross(AxisA, AxisB);

if (Axis.LengthSquared() < Epsilon)
{
    continue;
}

Axis.Normalize();
```

否则 Normalize 会带来数值问题。

---

# 29. 一个比较完整的 SAT 心智模型

以后看到 SAT，可以直接在脑中形成这张图：

```
             两个凸体
                │
                ▼
        枚举候选分离轴
                │
                ▼
        ┌────────────────┐
        │投影A到当前Axis │
        │投影B到当前Axis │
        └────────────────┘
                │
                ▼
           区间重叠？
           /        \
         否           是
         │            │
         ▼            ▼
    找到分离轴      下一条Axis
         │
         ▼
      不碰撞

所有Axis均重叠
         │
         ▼
       碰撞
         │
         ▼
寻找最小Overlap Axis
         │
         ├── Collision Normal
         │
         └── Penetration Depth
```

---

# 30. 面试时最好能回答到什么程度

如果是你前面提到的那种 **动作游戏 / 战斗开发 / 物理系统岗位**，面试官问：

> 你了解 SAT 吗？

一个比较完整的回答可以是：

> SAT，也就是分离轴定理，主要用于凸体之间的碰撞检测。其核心思想是，如果两个凸体不相交，那么一定能够找到一个轴，使它们在这个轴上的投影区间不重叠。
> 
> 所以算法会枚举有限的一组候选分离轴，把两个物体投影到轴上。如果任何一条轴的投影不重叠，就可以立即判定没有碰撞；如果所有候选轴都重叠，则发生碰撞。
> 
> 二维凸多边形的候选轴来自两个多边形所有边的法线；三维凸多面体除了面的法线以外，还需要考虑两个物体边方向的叉积。
> 
> 比如 OBB 对 OBB，最多测试 15 个轴，包括两个 OBB 各自的 3 个局部轴，以及 3×3 个边叉积轴。
> 
> 同时，可以记录所有轴上的最小投影重叠量，把对应轴作为碰撞法线，最小重叠作为穿透深度，从而得到 MTV。

如果能把这段说清楚，已经不只是“知道 SAT 是什么”，而是真的理解这个算法。

---

# 31. 最值得你牢牢记住的几个公式

SAT 实际上最核心就三个。

## 点到轴上的投影

$$
\boxed{p=P\cdot L}
$$

## 一维区间是否分离

$$
\boxed{ A_{max}<B_{min} \quad\lor\quad B_{max}<A_{min} }
$$

## OBB 在任意轴 L 上的投影半径

$$
\boxed{ r= e_0|U_0\cdot L| + e_1|U_1\cdot L| + e_2|U_2\cdot L| }
$$

于是两个 OBB 在 L 上存在分离的条件：

$$
\boxed{ |(C_B-C_A)\cdot L| > r_A+r_B }
$$

而 **OBB vs OBB 的候选轴数**：

$$
\boxed{ 3+3+3\times3=15 }
$$

这几条基本就是 SAT 的骨架。

顺着你最近在看的**游戏物理系统面试知识**继续学的话，下一步最有价值的是把 **SAT → GJK → Minkowski Difference → EPA → Contact Manifold → Sequential Impulse** 连起来。这样就会从“怎么判断两个 Collider 撞了”，一直走到“UE/PhysX/Chaos 为什么最终能把两个刚体正确弹开、摩擦和堆叠”，整个 Narrow Phase 和 Solver 的链路会完整很多。