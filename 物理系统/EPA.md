你这里的 EPA 一般指碰撞检测里的 **Expanding Polytope Algorithm，扩展多面体算法**。它通常和 **GJK（Gilbert–Johnson–Keerthi）** 配套使用：

> **GJK 负责回答：两个凸体是否相交？**  
> **EPA 负责回答：如果相交了，它们“陷进去多深”，以及应该沿哪个方向把它们分开？**

因此在物理引擎中经常是：

 GJK \rightarrow \text{发现碰撞} \rightarrow EPA \rightarrow \begin{cases} \text{Penetration Normal}\\ \text{Penetration Depth} \end{cases} 

理解 EPA 的关键，不是死记算法步骤，而是理解一个核心思想：

> **在 Minkowski Difference（闵可夫斯基差）里，碰撞穿透深度，就等价于“原点到 Minkowski 边界的最短距离”。**

---

## 先从 GJK 留下的问题讲起

假设两个凸物体：

$$
A,\quad B
$$

定义 Minkowski Difference：

$$
 C=A-B 
$$

也就是：

$$
 C=\{a-b\mid a\in A,b\in B\} 
$$

GJK 有一个非常重要的性质：

$$
 A\cap B\neq \varnothing 
$$

等价于：

$$
 0\in A-B 
$$

也就是说，如果两个物体发生重叠，那么 **Minkowski Difference 会把原点包在里面**。

比如二维中，两个矩形发生重叠：

```
World Space

      A
   +------+
   |      |
   |   +--+---+
   +---|--+   |
       |      |
       +------+
          B
```

转换成 Minkowski Difference 后，可以想象得到一个更大的凸多边形：

```
Minkowski Difference

          *
       *     *
     *         *
    *     O     *
     *         *
       *     *
          *
```

其中：

```
O = Origin
```

因为原点在里面，所以 GJK 可以确认：

> A 和 B 相交。

但是光知道：

```
碰撞 = true
```

对于物理引擎是不够的。

物理引擎还要知道：

```
往哪个方向推出去？
推出去多远？
```

例如：

```
      A
   +------+
   |      |
   |      |
   +---+--+
       ↑
       │ penetration depth
   +---+--+
   |      |
   |  B   |
   +------+
```

这就是 EPA 要解决的问题。

---

# EPA 的几何本质

先看 Minkowski Difference：

```
                boundary
                   *
               *       *
            *             *
          *                 *
         *        O          *
          *                 *
            *             *
               *       *
                   *
```

因为发生碰撞：

$$
 O\in C 
$$

现在考虑：

> 从原点出发，向哪个方向移动最短，可以离开这个凸体？

比如：

```
                *
             *     *
           *         *
         *             *
        *      O ------| *
         *             *
           *         *
             *     *
                *
```

这条最短距离就是：

$$
 d=\min_{x\in \partial C}\|x\| 
$$

这个距离恰好对应两个真实物体之间的：

$$
 \boxed{\text{Penetration Depth}} 
$$

而从原点指向最近边界点的方向，则对应：

$$
 \boxed{\text{Penetration Normal}} 
$$

所以 EPA 本质上在干一件事：

$$
 \boxed{ \text{不断逼近 Minkowski Difference 上距离原点最近的边界} } 
$$

---

# 为什么不能直接从 GJK 得到？

这是非常重要的面试点。

GJK 检测碰撞时维护的是一个 **Simplex（单纯形）**。

二维中 simplex 最大是：

$$
 Triangle 
$$

三维中 simplex 最大是：

$$
 Tetrahedron 
$$

例如 2D GJK 最终可能得到：

```
          A
         / \
        /   \
       /  O  \
      /       \
     B---------C
```

原点 O 在 triangle ABC 内部。

这只能说明：

$$
 0\in \triangle ABC 
$$

所以物体碰撞。

但是这个 triangle 并不一定就是 Minkowski Difference 真正的外边界。

真正的 Minkowski Difference 可能是：

```
                *
            *       *
         *             *
       *        A        *
      *        / \        *
     *        / O \        *
      *      B-----C      *
       *                 *
         *             *
            *       *
                *
```

GJK 得到的 ABC 只是 Minkowski Shape 内的一部分。

因此：

```
Origin 到 ABC 的距离
```

并不一定是真正的最小穿透距离。

EPA 所做的就是：

> 从 GJK 得到的 simplex 开始，一点一点扩展这个 simplex，直到它逼近真正的 Minkowski 边界。

这也是为什么它叫：

> **Expanding Polytope Algorithm**

也就是：

> 不断扩展 Polytope（凸多面体）。

---

# 二维 EPA 最容易理解

先理解 2D，再看 3D 会非常简单。

假设 GJK 最终得到 triangle：

```
             A
            / \
           /   \
          /  O  \
         /       \
        B---------C
```

这个 triangle 包围原点。

EPA 第一件事：

> 找 triangle 三条边中，距离原点最近的边。

即：

```
AB
BC
CA
```

分别计算：

$$
 d_{AB}  d_{BC}  d_{CA} 
$$

找：

$$
 d_{min} 
$$

假设最近的是：

```
BC
```

那么：

```
             A
            / \
           /   \
          /  O  \
         /   |   \
        B----|----C
             ↓
          closest edge
```

---

# 怎样计算一条边到原点的距离？

假设边：

$$
 AB 
$$

首先计算边：

$$
 e=B-A 
$$

求垂直于这条边的法线：

$$
 n 
$$

并保证它指向 **Polytope 外侧**。

然后：

$$
 d=n\cdot A 
$$

如果 \(n\) 已归一化：

$$
 \|n\|=1 
$$

那么：

$$
 d=n\cdot A 
$$

就是：

> 原点到这条直线的距离。

二维示意：

```
                 n
                 ↑
                 │
       A---------P---------B
                 │
                 │ d
                 │
                 O
```

因此我们找：

$$
 \min d_i 
$$

就是当前 Polytope 上最接近原点的边。

---

# EPA 最聪明的一步：继续做 Support

假设当前最近的边是：

```
A -------- B
```

它的 outward normal 为：

$$
 n 
$$

EPA 不会立刻认为这就是 Minkowski Shape 的真实边界。

因为它可能只是：

```
当前 approximation
```

于是 EPA 用这个法线方向继续求一次 Minkowski Support：

$$
 P=Support_{A-B}(n) 
$$

而 Minkowski Difference 的 support 可以通过：

$$
 Support_{A-B}(n) = Support_A(n)-Support_B(-n) 
$$

得到。

这一点和 GJK 完全一样。

---

# 为什么 Support(n) 可以检查“还有没有更远的边界”？

假设当前：

```
              true boundary
                   P
                  *
                 / \
                /   \
        A------*-----*------B
               current edge

                  O
```

当前 closest edge 是：

```
AB
```

法线是：

```
n
```

如果真实 Minkowski Shape 在这个方向上还有一个更远的点 P：

$$
 n\cdot P > n\cdot A 
$$

说明：

> 当前 AB 还不是 Minkowski Shape 真正的 support plane。

于是把 P 加进 polytope：

原来：

```
A -------- B
```

变成：

```
A
 \
  \
   P
  /
 /
B
```

原来的 edge AB 被拆成：

```
AP
PB
```

然后重新寻找：

> 所有边中离原点最近的边。

这就是 **Expand**。

---

# 一个完整的二维过程

假设初始 triangle：

```
             A
            / \
           / O \
          B-----C
```

第一轮寻找：

```
closest edge = BC
```

法线：

```
        n
        ↓
B---------------C
        |
        O
```

于是：

$$
 P=Support(n) 
$$

得到：

```
             A
            / \
           / O \
          B-----C
           \   /
            \ /
             P
```

于是 polytope 从：

```
ABC
```

扩展成：

```
ABPC
```

然后重新计算所有 edge。

例如新的最近边变成：

```
BP
```

那么：

$$
 n_2=normal(BP) 
$$

继续：

$$
 Q=Support(n_2) 
$$

可能变成：

```
              A
            /   \
           /  O  \
          B       C
           \     /
            Q---P
```

不断这样扩展：

```
triangle
   ↓
quadrilateral
   ↓
pentagon
   ↓
hexagon
   ↓
...
```

越来越接近真实 Minkowski Shape。

---

# 什么时候停止？

假设当前 closest edge：

$$
 AB 
$$

法线：

$$
 n 
$$

edge 到 origin 的距离：

$$
 d_{edge}=n\cdot A 
$$

然后 support：

$$
 P=Support(n) 
$$

新的极值距离：

$$
 d_{support}=n\cdot P 
$$

计算：

$$
 \Delta=d_{support}-d_{edge} 
$$

如果：

$$
 \Delta<\epsilon 
$$

说明：

> 沿这个方向，已经找不到明显更远的新点了。

于是我们认为：

```
当前 edge ≈ Minkowski Difference 真实边界
```

最终：

$$
 \boxed{PenetrationDepth=d_{edge}}  \boxed{PenetrationNormal=n} 
$$

---

# 为什么这个 Normal 就是碰撞法线？

这是 EPA 最值得深入理解的地方。

设最近边：

```
                 n
                 ↑
                 │
        ---------- boundary
                 │
                 │ d
                 │
                 O
```

如果把 Minkowski Difference 向：

$$
 -n 
$$

平移 \(d\)，那么边界就刚好碰到 Origin：

```
Before

        boundary
-----------

      d

      O
```

移动：

$$
 -dn 
$$

之后：

```
boundary
----O----
```

此时：

$$
 0\in \partial(A-B) 
$$

这意味着两个原始物体：

$$
 A,B 
$$

刚好从：

```
penetrating
```

变成：

```
touching
```

所以：

$$
 dn 
$$

就是 **Minimum Translation Vector**，通常简称：

$$
 \boxed{MTV} 
$$

即：

> 让两个物体脱离重叠所需要的最小平移向量。

通常可以写：

$$
 MTV=n\cdot depth 
$$

不过到底是：

$$
 +n 
$$

还是：

$$
 -n 
$$

要取决于你的 Minkowski Difference 定义以及碰撞法线约定。

例如你定义：

$$
 A-B 
$$

与：

$$
 B-A 
$$

法线会反过来。

---

# EPA 和 SAT 的关系其实非常深

如果你学过 SAT：

> Separating Axis Theorem，分离轴定理。

你会发现 EPA 最终干的事情和 SAT 很像。

SAT 会寻找：

```
minimum overlap axis
```

例如：

```
        A
+--------------+
|              |
|      +-------+------+
|      |       |      |
+------+-------+      |
       |              |
       +--------------+
              B
```

可能存在多个候选轴：

$$
 n_1,n_2,n_3,\cdots 
$$

计算每个轴上的 overlap：

$$
 overlap_i 
$$

然后找：

$$
 \min overlap_i 
$$

这个方向就是：

```
minimum penetration axis
```

EPA 本质上也是在 Minkowski Space 中寻找：

$$
 \boxed{\text{距离 Origin 最近的 support plane}} 
$$

所以从几何上可以理解：

$$
 \text{SAT minimum overlap} 
$$

和：

$$
 \text{EPA closest Minkowski boundary} 
$$

实际上是在描述同一个碰撞几何问题。

---

# 三维 EPA：从 Edge 变成 Face

二维 EPA 操作的是：

```
Polygon
```

三维 EPA 操作的是：

```
Convex Polyhedron
```

GJK 在 3D 确认碰撞时，通常得到一个 tetrahedron：

```
             A
            /|\
           / | \
          /  |  \
         B---|---C
          \  |  /
           \ | /
            \|/
             D
```

Origin 在 tetrahedron 中：

$$
 0\in ABCD 
$$

EPA 就从这个 tetrahedron 开始。

二维：

```
找离 Origin 最近的 Edge
```

三维：

```
找离 Origin 最近的 Triangle Face
```

例如：

$$
 ABC 
$$

它的 outward normal：

$$
 n 
$$

距离：

$$
 d=n\cdot A 
$$

找所有 triangle faces 中：

$$
 d_{min} 
$$

---

# 三维中一次 Expansion

假设当前最近 face：

```
ABC
```

法线：

$$
 n 
$$

执行：

$$
 P=Support(n) 
$$

如果：

$$
 n\cdot P-d<\epsilon 
$$

结束。

否则 P 在这个 face 外面：

```
                   P
                  /|\
                 / | \
                /  |  \
               A---|---B
                \  |  /
                 \ | /
                  \|/
                   C
```

那么 P 会成为新的 polytope vertex。

但是这里比 2D 麻烦很多。

因为你不能简单地：

```
ABC → ABP + BCP + CAP
```

为什么？

因为 P 可能同时位于多个 face 的外面。

例如当前 polytope：

```
       ______
      /     /\
     /_____/  \
     \     \  /
      \_____\/
```

新的 P：

```
           P
          /
         /
       polytope
```

从 P 看过去，有很多 face 是“可见”的。

这些 face 已经变成：

> 新 convex hull 内部的面。

必须全部删除。

---

# 三维 EPA 最核心的数据结构：Visible Faces + Horizon

加入新点 P 时，要找所有：

$$
 VisibleFaces 
$$

对于一个 face：

$$
 ABC 
$$

假设 outward normal：

$$
 n 
$$

如果：

$$
 n\cdot(P-A)>0 
$$

说明 P 位于 face 外侧。

因此这个 face 对 P 可见。

这些 face 全部删除。

假设：

```
        visible faces

          /----\
         / //// \
        / /////  \
       /---------\
```

删除这些 faces 后会形成一个洞。

洞的边界叫：

$$
 \boxed{Horizon} 
$$

可以理解成：

> 从 P 看 polytope 时，轮廓线上那些 edge。

然后将 P 和每个 horizon edge 连接：

```
           P
         / | \
        /  |  \
       /   |   \
      *----*----*
```

创建新的 triangle faces。

于是 convex hull 被正确扩张。

这其实已经是一个小型：

> **Incremental Convex Hull Algorithm**

所以 3D EPA 实现比 GJK 要复杂不少。

---

# Horizon 到底是什么？

这个概念在面试里很容易被问。

假设有：

```
Visible Face
Non-visible Face
```

如果一条 edge：

```
一边属于 visible face
另一边属于 non-visible face
```

那么这条 edge 就是：

$$
 HorizonEdge 
$$

因为：

```
visible faces
```

会被删除，而：

```
non-visible faces
```

会被保留。

它们的交界线正好就是新凸包的边界。

所以更新方式是：

```
删除所有 visible faces
        ↓
找到 horizon edges
        ↓
horizon edge + P
        ↓
生成新的 triangles
```

例如：

```
        P
       /|\
      / | \
     /  |  \
    A---B---C
```

---

# EPA 的伪代码

把核心过程压缩以后，大概就是：

```
polytope = simplexFromGJK;

while (iteration < maxIteration)
{
    Face closestFace = FindClosestFaceToOrigin(polytope);

    Vector3 direction = closestFace.normal;

    SupportPoint p = Support(A, B, direction);

    float supportDistance =
        Dot(p.position, direction);

    if (supportDistance - closestFace.distance < epsilon)
    {
        normal = closestFace.normal;
        depth  = closestFace.distance;

        return CollisionResult(normal, depth);
    }

    ExpandPolytope(polytope, p);
}
```

真正麻烦的是：

```
ExpandPolytope()
```

里面会涉及：

```
visible face
horizon edge
删除旧 face
生成新 face
face winding
normal direction
degenerate triangle
duplicate vertex
```

这些都是 EPA 工程实现中的难点。

---

# EPA 中的 SupportPoint 最好不要只保存 Position

实际写物理引擎时，不建议只保存：

```
Vector3 minkowskiPoint;
```

最好保存：

```
struct SupportPoint
{
    Vector3 pointA;
    Vector3 pointB;

    Vector3 point;
};
```

其中：

$$
 point=pointA-pointB 
$$

因为 EPA 最终如果还想得到：

```
接触点 contact point
```

就需要知道 Minkowski Point 是由：

$$
 a-b 
$$

中的哪个：

```
a ∈ A
b ∈ B
```

构成的。

---

# EPA 是否直接给 Contact Point？

这是一个经常被讲错的地方。

EPA 最天然得到的是：

$$
 \boxed{Normal} 
$$

和：

$$
 \boxed{PenetrationDepth} 
$$

它本身并不是单纯靠“最近 face”就自动给你一个可靠的完整 contact manifold。

但是可以利用最终最近 face 的三个 Minkowski vertex：

$$
 P_0,P_1,P_2 
$$

每个 vertex 都保存：

$$
 P_i=A_i-B_i 
$$

然后求 Origin 投影到这个 triangle 上的位置：

```
         P0
        / \
       / X \
      /     \
    P1-------P2
```

求 X 的重心坐标：

$$
 X=\lambda_0P_0+\lambda_1P_1+\lambda_2P_2 
$$

满足：

$$
 \lambda_0+\lambda_1+\lambda_2=1 
$$

那么对应到 A：

$$
 Contact_A= \lambda_0A_0+ \lambda_1A_1+ \lambda_2A_2 
$$

对应到 B：

$$
 Contact_B= \lambda_0B_0+ \lambda_1B_1+ \lambda_2B_2 
$$

因此可以恢复一对 witness points。

这就是为什么 GJK/EPA 实现里的 SupportPoint 经常保存：

```
struct SupportPoint
{
    Vec3 minkowski;
    Vec3 supportA;
    Vec3 supportB;
};
```

而不是只有：

```
Vec3 minkowski;
```

---

# 为什么要找“离 Origin 最近的 Face”？

你可能会有一个疑问：

> EPA 每次都往最近 face 的 normal 扩张，为什么最后一定能找到最小穿透？

因为对于凸体而言，每一个面都对应一个 support plane：

$$
 n\cdot x=d 
$$

Origin 到 plane 的距离为：

$$
 d 
$$

只要 Origin 在凸体内，穿出凸体至少要穿过某一个 support plane。

最小移动距离自然是：

$$
 \min d_i 
$$

所以我们真正寻找的是：

$$
 \boxed{ d=\min_n h_C(n) } 
$$

这里：

$$
 h_C(n) 
$$

就是凸体的 support function：

$$
 h_C(n)=\max_{x\in C} n\cdot x 
$$

EPA 可以理解为：

> 利用 Support Mapping，不断构造 Minkowski Shape 的局部凸包，并寻找最接近 Origin 的 support plane。

这个理解其实比“不断往外加点”更加本质。

---

# GJK 和 EPA 的关系可以这样记

它们虽然都使用 Support Mapping，但搜索目标完全不同：

|GJK|EPA|
|---|---|
|判断 Origin 是否在 Minkowski Difference 内|Origin 已经确定在里面|
|Simplex 朝 Origin 搜索|Polytope 朝外扩张|
|重点是包含关系|重点是最近边界|
|输出是否碰撞|输出 Normal + Depth|
|通常很快|相对更昂贵|

用一句话：

$$
 \boxed{ GJK：想办法把 Origin 包进去 } 
$$

而：

$$
 \boxed{ EPA：Origin 已经包进去了，再从里面寻找最近出口 } 
$$

这个比喻非常好记。

---

# 一个非常直观的比喻

把 Minkowski Difference 想成一个房间。

你站在：

```
Origin
```

位置：

```
        wall
   +-------------+
   |             |
   |     YOU     |
   |      O      |
   |             |
   +-------------+
```

GJK 的任务只是：

> 判断你是不是在这个房间里。

一旦确认：

```
YES
```

GJK 就结束。

EPA 的任务则是：

> 找离你最近的墙。

例如：

```
   +-------------+
   |             |
   |     O --->  |  1.2m
   |             |
   +-------------+
```

那么：

```
最近墙方向 = penetration normal
最近墙距离 = penetration depth
```

你只需要沿这个方向移动 1.2m，就能走出房间。

这个移动就是：

$$
 MTV 
$$

---

# 实际实现中 EPA 最容易踩的坑

如果你之后准备自己实现 GJK + EPA，这几个地方尤其重要：

1. **Face Normal 必须统一朝外。** 如果 winding 不一致，某些 face 的 normal 朝里，那么：

$$
 d=n\cdot p 
$$

会出现负数，visible face 判断也会直接出错。

2. **必须处理 Duplicate Support Point。** 有时：

$$
 Support(n) 
$$

返回的点已经存在 polytope 中。这通常意味着已经无法继续有效扩展，应当触发收敛或容错。

3. **必须设置 epsilon。** 不要判断：

```
supportDistance == faceDistance
```

而应该：

```
supportDistance - faceDistance < epsilon
```

因为浮点运算几乎不会精确相等。

4. **必须设置 max iteration。** 比如：

```
32
64
128
```

否则极端退化情况可能一直迭代。

5. **退化三角形需要过滤。** 如果：

$$
 |(B-A)\times(C-A)|\approx0 
$$

这个 face 几乎没有面积，normal 不稳定，应当丢弃。

6. **碰撞法线方向要统一。** 比如你的引擎规定：

```
Normal 永远从 B → A
```

那么 EPA 返回后必须检查方向，否则后续 impulse：

$$
 J n 
$$

可能直接把物体推得更深。

---

最后可以把整个 **GJK + EPA** 过程压缩成这一张逻辑图：

```
            Convex A         Convex B
                │                │
                └──────┬─────────┘
                       ↓
               Support Mapping
                       ↓
                Minkowski A-B
                       ↓
                      GJK
                       │
              ┌────────┴─────────┐
              │                  │
         Origin outside      Origin inside
              │                  │
         No Collision        Collision
                                 │
                                 ↓
                          Initial Simplex
                                 │
                                 ↓
                               EPA
                                 │
                 Find closest Edge / Face
                                 │
                                 ↓
                          Get face normal
                                 │
                                 ↓
                       Support(normal)
                                 │
                   ┌─────────────┴────────────┐
                   │                          │
             distance improves          no improvement
                   │                          │
            Expand Polytope                   ↓
                   │                 Penetration Normal
                   └──────────→      Penetration Depth
```

如果放到你现在学习的 **UE / 战斗与物理系统** 语境里，最值得记住的面试级结论就是：

$$
 \boxed{\text{EPA = 在 Minkowski Difference 中寻找 Origin 到凸体边界的最近距离}} 
$$

这个最近距离就是 **穿透深度**，最近边界的外法线就是 **碰撞法线**。而 GJK 提供的是 EPA 的初始 polytope，所以这两套算法才会经常以 **GJK + EPA** 的形式一起出现。