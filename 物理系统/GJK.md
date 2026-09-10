GJK（Gilbert–Johnson–Keerthi）是游戏物理里非常经典的**凸体碰撞检测算法**。如果你刚看完 SAT，可以先记住两者最核心的区别：

> **SAT：寻找一条能把两个物体分开的轴。**  
> **GJK：把两个物体变换到“闵可夫斯基差空间”，然后判断原点是否在这个新凸体内部。**

对于 OBB、Convex Hull、Sphere、Capsule 等凸体，GJK 都可以用一套统一框架处理。

GJK 就是：通过闵可夫斯基差把两个凸体的碰撞问题转化为“原点是否位于某个凸体内”，再用 Support Point + Simplex 迭代判断原点是否被包围。

---

# 1. GJK 到底在解决什么问题

假设空间中有两个凸体：

$$
A,\quad B
$$

我们想知道：

$$
A\cap B \neq \emptyset ?
$$

GJK 不直接判断 A 和 B，而是构造：

$$
C=A-B
$$

这里的减法并不是普通几何尺寸相减，而是：

$$
A-B= \{a-b\mid a\in A,b\in B\}
$$

这个东西叫：

**Minkowski Difference，闵可夫斯基差。**

也可以写成：

$$
A-B=A+(-B)
$$

---

# 2. 为什么闵可夫斯基差能判断碰撞

这是理解 GJK 最关键的一步。

假设 A 和 B 相交。

那么一定存在一个点：

$$
\[ p\in A \]
$$

同时：

$$
\[ p\in B \]
$$

于是我们从 A 选：

$$
\[ a=p \]
$$

从 B 选：

$$
\[ b=p \]
$$

那么：

$$
\[ a-b=p-p=0 \]
$$

所以：

$$
\[ 0\in A-B \]
$$

反过来也一样。

如果：

$$
\[ 0\in A-B \]
$$

那么一定存在：

$$
\[ a\in A,b\in B \]
$$

满足：

$$
\[ a-b=0 \]
$$

因此：

$$
\[ a=b \]
$$

说明两个物体有公共点。

所以最终得到一个极其重要的结论：

$$
\[ \boxed{ A与B碰撞 \iff 原点O位于A-B内部 } \]
$$

于是一个“两物体碰撞问题”，被转化成：

> **原点是否位于某个凸体内部？**

这就是整个 GJK 的理论核心。

---

# 3. 一个二维例子

假设有两个矩形：

```
A                 B

+------+        +------+
|      |        |      |
|      |        |      |
+------+        +------+
```

对 A 的每个点 \(a\)，对 B 的每个点 \(b\)，计算：

$$
\[ a-b \]
$$

这些点形成一个新的凸体：

```
Minkowski Difference

      /--------\
     /          \
    |      O     |
     \          /
      \--------/
```

如果两个矩形没有碰撞，原点可能在外面：

```
              O

      /--------\
     /          \
    |            |
     \          /
      \--------/
```

如果两个矩形碰撞：

```
      /--------\
     /          \
    |     O      |
     \          /
      \--------/
```

所以理论上，我们只需要把整个 Minkowski Difference 构造出来，然后执行一次 Point-In-Convex-Test。

但问题是：

**构造整个 Minkowski Difference 很贵。**

例如两个复杂 Convex Mesh 各有几百个顶点。

GJK 的精妙之处就在这里：

> **根本不构建 Minkowski Difference。**

它只在需要的时候查询某个方向最远的一个点。

这个操作就是 GJK 的第二个核心：

# 4. Support Mapping

假设有一个凸体：

```
          *
       *     *
     *         *
    *           *
     *         *
       *     *
          *
```

现在给一个方向：

$$
\[ d \]
$$

例如：

```
        d →
```

我们希望找到这个凸体在 \(d\) 方向最远的点：

```
                  support point
                       ↓
          *         *  X
       *                 →
     *
```

数学上就是：

$$
\[ Support(A,d) = \arg\max_{a\in A}(a\cdot d) \]
$$

因为点积：

$$
\[ a\cdot d \]
$$

越大，说明点在 \(d\) 方向投影得越远。

---

# 5. Minkowski Difference 的 Support

这一条公式是实现 GJK 最重要的公式。

我们需要：

$$
\[ Support(A-B,d) \]
$$

根据：

$$
\[ A-B=A+(-B) \]
$$

可以推导：

$$
\[ \boxed{ Support(A-B,d) = Support(A,d)-Support(B,-d) } \]
$$

所以根本不用生成 \(A-B\)。

只需要执行：

```
Vector3 Support(Vector3 d)
{
    Vector3 a = A.Support(d);
    Vector3 b = B.Support(-d);

    return a - b;
}
```

这就是 GJK 对形状的唯一要求：

> **这个 Shape 能不能告诉我：沿某个方向最远的点在哪里？**

只要能，就能接入 GJK。

所以 GJK 可以统一处理：

Sphere、Box、OBB、Capsule、Cylinder、Convex Hull……

甚至：

```
OBB vs Capsule
Capsule vs Sphere
ConvexHull vs OBB
ConvexHull vs ConvexHull
```

GJK 核心算法完全不用改。

---

# 6. 例如 OBB 怎么求 Support

这和我们前面讲 SAT 时 OBB 的三个轴有关。

一个 OBB 有：

$$
\[ Center=C \]
$$

三个局部轴：

$$
\[ u_0,u_1,u_2 \]
$$

半尺寸：

$$
\[ e_0,e_1,e_2 \]
$$

对于方向：

$$
\[ d \]
$$

support point 就是：

$$
\[ C+ sign(d\cdot u_0)e_0u_0+ sign(d\cdot u_1)e_1u_1+ sign(d\cdot u_2)e_2u_2 \]
$$

直观理解非常简单。

假设：

```
      +-------+
     /       /|
    +-------+ |
    |       | +
    |       |/
    +-------+

               ↗ d
```

对于每根局部轴：

```
dot(axis,d) > 0
```

就选择正方向那个面。

否则选择负方向那个面。

最终就是 OBB 朝 \(d\) 方向最远的那个角。

这和 SAT 很有意思：

SAT 需要分析 OBB 的候选分离轴；

GJK 根本不关心 OBB 是什么。

它只问：

```
Support(d) 是谁？
```

---

# 7. GJK 为什么需要 Simplex

现在我们已经可以不停查询 Minkowski Difference 上的点了。

问题变成：

> 怎么判断原点 O 在不在 Minkowski Difference 里面？

GJK 会逐渐构造一个小的几何体：

$$
\[ Simplex \]
$$

中文通常叫：

**单纯形。**

在不同维度：

$$
\[ 0D:\ 点 \]\[ 1D:\ 线段 \]\[ 2D:\ 三角形 \]\[ 3D:\ 四面体 \]
$$

所以 3D GJK 最多只需要：

```
4 个 support point
```

并不需要保存几十几百个点。

这背后和凸几何里的 Carathéodory 定理有关：在三维空间，如果原点属于某个凸包，那么最多可以用 4 个点组成的凸包表示它。

所以：

$$
\[ \boxed{ 3D碰撞判断最终只需要判断 原点是否能被某个四面体包围 } \]
$$

---

# 8. GJK 整个过程

假设我们随便选择一个初始方向：

$$
\[ d \]
$$

通常可以使用：

```
d = B.Center - A.Center;
```

如果恰好接近 0，就用：

```
(1,0,0)
```

然后求：

$$
\[ A=Support(d) \]
$$

注意这里为了避免名字冲突，后面 Simplex 中通常把最新 support point 叫 `A`。

接下来：

$$
\[ d=-A \]
$$

因为：

```
原点 O = (0,0,0)

A ●------------------● O
          d →
```

从 A 朝原点搜索。

然后再次：

$$
\[ A=Support(d) \]
$$

这里出现 GJK 极其重要的判断：

$$
\[ A\cdot d < 0 \]
$$

则：

$$
\[ \boxed{没有碰撞} \]
$$

为什么？

---

# 9. 为什么 `dot(A,d) < 0` 就能确定不碰撞

假设：

```
            d →

      Minkowski Difference

      /-------\
     /         \
    /           \
---A-------------|-----------O
```

我们正在朝原点方向 \(d\) 找整个 Minkowski Difference 最远的点。

结果连最远点 A 在方向 \(d\) 上的投影都没有超过原点。

也就是：

$$
\[ A\cdot d<0 \]
$$

那么整个 Minkowski Difference 都在：

```
原点的这一侧
```

因为 A 已经是最远点。

于是存在一个平面：

```
Minkowski
Difference      |        O
                |
****************|        
****************|
****************|
```

把 Minkowski Difference 和原点分开。

所以原点绝对不在里面。

也就是：

$$
\[ A\cap B=\emptyset \]
$$

注意工程实现里通常要考虑 epsilon，因此不会简单机械使用完全精确的浮点：

```
if (dot < epsilon)
```

具体“接触算不算碰撞”也会影响条件。

---

# 10. 如果不能排除怎么办

假设：

$$
\[ A\cdot d>0 \]
$$

说明：

```
这个方向还不能把原点排除掉
```

于是把 A 加进 Simplex：

```
simplex.Add(A);
```

然后问：

> 当前 Simplex 有没有包住原点？

如果没有，那么：

> 原点最可能在哪个方向？

然后产生一个新的：

\[ d \]

再继续：

```
Support(d)
→ 更新 Simplex
→ 得到新 d
→ Support(d)
→ ...
```

直到两个结果之一：

```
support 无法越过原点
        ↓
   No Collision
```

或者：

```
四面体包围原点
        ↓
     Collision
```

---

# 11. 先看二维 GJK，会容易很多

二维情况下 Simplex 最多只有：

```
点
线段
三角形
```

假设最新加入的点永远叫：

\[ A \]

---

# 12. Simplex = 一个点

刚开始：

```
A ●

                         O
```

方向很简单：

$$
\[ d=AO \]
$$

由于：

$$
\[ O=0 \]
$$

所以：

$$
\[ AO=-A \]
$$

于是：

```
d = -A;
```

---

# 13. Simplex = 线段 AB

现在有：

```
B ●-------------● A

                    \
                     \
                      O
```

定义：

$$
\[ AO=O-A=-A \]\[ AB=B-A \]
$$

首先判断原点是否位于 AB 方向：

$$
\[ AB\cdot AO>0 \]
$$

情况一：

```
B ●-------------● A
                  \
                   \
                    O
```

说明原点可能和 AB 有关系。

不能删除 B。

于是：

```
Simplex = AB
```

然后寻找一个：

> 垂直于 AB，并指向 O 的方向。

二维可以想成：

```
B ●-------------● A
                  |
                  |
                  ↓ d
                  O
```

3D GJK 中通常用 Triple Cross：

$$
\[ d=(AB\times AO)\times AB \]
$$

也就是：

```
Cross(Cross(AB, AO), AB)
```

它的作用就是：

**取得 AB 的垂直方向中朝向 AO 的那个方向。**

---

# 14. 为什么 Triple Cross 能做到

利用向量三重积：

$$
\[ (A\times B)\times C \]
$$

可以展开。

对于：

$$
\[ (AB\times AO)\times AB \]
$$

结果一定垂直于：

$$
\[ AB \]
$$

同时它位于：

$$
\[ AB,AO \]
$$

构成的平面里。

于是恰好是：

```
         AO
         ↘
B ●──────● A
         |
         ↓
         d
```

所以 GJK 代码里你会频繁看到：

```
Cross(Cross(AB, AO), AB);
```

---

# 15. 如果 `AB · AO <= 0`

例如：

```
B ●--------● A -------- O
```

注意 AB 是：

```
A → B
```

但 AO 是：

```
A → O
```

两个方向完全相反。

说明：

> B 对找到原点完全没帮助。

因此直接把 B 从 simplex 删除：

```
Simplex：

AB

↓

A
```

然后：

$$
\[ d=AO \]
$$

也就是：

```
simplex = { A };
direction = AO;
```

这里体现了 GJK 一个非常重要的思想：

> **不断丢掉“不可能帮助包围原点”的点。**

因此 Simplex 永远很小。

---

# 16. Simplex = 三角形 ABC

这是理解 GJK 最重要的部分之一。

最新点是：

$$
\[ A \]
$$

另外两个点：

$$
\[ B,C \]
$$

```
                A
               / \
              /   \
             /     \
            C-------B
```

定义：

$$
\[ AB=B-A \]\[ AC=C-A \]\[ AO=-A \]
$$

三角形法线：

$$
\[ ABC=AB\times AC \]
$$

现在空间被三角形的边分成不同区域。

二维时可以想：

```
               A
              / \
      区域1   /   \   区域2
            /  ABC\
           C-------B

              区域3
```

GJK 要判断：

```
原点在哪个 Voronoi Region？
```

这是理解 Simplex Handler 的另一种非常好的方式。

---

# 17. 检查 AB 外侧

需要找一个：

```
垂直 AB
并且朝三角形 ABC 外侧
```

方向：

$$
\[ AB\times ABC \]
$$

例如：

```
       A--------------B
       |
       |
       C

       ↑
       AB 外侧
```

判断：

$$
\[ (AB\times ABC)\cdot AO>0 \]
$$

如果成立：

```
O 在 AB 外侧
```

那么 C 不可能帮助我们包围 O。

因此：

```
ABC
 ↓
AB
```

重新回到线段处理。

---

# 18. 检查 AC 外侧

同理：

$$
\[ ABC\times AC \]
$$

是 AC 的外侧方向。

如果：

$$
\[ (ABC\times AC)\cdot AO>0 \]
$$

那么：

```
B 没用了
```

Simplex 变成：

$$
\[ AC \]
$$

继续处理线段。

---

# 19. 如果原点不在 AB、AC 外侧

那么说明它位于：

```
        A
       / \
      /   \
     C-----B
      \   /
       \ /
        O?
```

在三维中还需要判断原点位于三角形：

```
正面
```

还是：

```
背面
```

使用：

$$
\[ ABC\cdot AO \]
$$

如果：

$$
\[ ABC\cdot AO>0 \]
$$

则：

$$
\[ d=ABC \]
$$

否则：

$$
\[ d=-ABC \]
$$

同时通常交换 B、C，保持 simplex winding 一致：

```
Swap(B, C);
direction = -ABC;
```

所以三角形阶段其实是在做：

> 从一个三角形判断原点位于它哪个 Voronoi 区域，然后把不必要的顶点删除。

---

# 20. 到三维以后：四面体 ABCD

3D GJK 最终会得到：

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

现在问题变成：

> 原点是不是在四面体内部？

一个四面体有 4 个面：

$$
\[ ABC \]\[ ACD \]\[ ADB \]\[ BCD \]
$$

由于：

$$
\[ A \]
$$

是最新加入的 support point，标准 GJK 迭代中通常重点检查包含 A 的三个面：

$$
\[ ABC \]\[ ACD \]\[ ADB \]
$$

---

# 21. 四面体判断本质是什么

例如检查面 ABC：

```
          A
         / \
        /   \
       B-----C

       plane ABC
------------------------

             O
```

如果原点在 ABC 面的外面，那么：

```
D 不可能帮忙包围 O
```

于是：

$$
\[ ABCD \]
$$

缩减成：

$$
\[ ABC \]
$$

继续执行三角形 GJK。

同理如果在：

$$
\[ ACD \]
$$

外面：

```
Simplex = ACD
```

如果在：

$$
\[ ADB \]
$$

外面：

```
Simplex = ADB
```

但是如果：

```
O 不在这三个面的任何外侧
```

由于当前 GJK simplex 的构造条件，可以得到：

$$
\[ \boxed{O位于四面体内部} \]
$$

于是：

$$
\[ \boxed{Collision} \]
$$

这就是 3D GJK 最终成功条件。

---

# 22. 所以 GJK 本质上一直在干什么

很多人第一次看代码，会感觉里面全是：

```
Cross
Dot
Cross
Dot
Remove
Cross
```

很难理解。

其实你只需要牢牢记住一句：

> **GJK 不断构造一个 Simplex，并保留“最可能包住原点”的那部分，然后继续往原点所在区域搜索。**

例如：

```
Point
 ↓
Line
 ↓
Triangle
 ↓
Tetrahedron
 ↓
Origin Inside
 ↓
Collision
```

但过程中也可能退化：

```
Triangle
 ↓
发现 O 在 AB 外侧
 ↓
Line AB
 ↓
重新搜索
 ↓
Triangle
```

所以不是单纯：

```
1 → 2 → 3 → 4
```

而可能：

```
1 → 2 → 3 → 2 → 3 → 3 → 4
```

不断逼近原点。

---

# 23. GJK 的核心伪代码

现在看代码就非常容易了：

```
bool GJK(Shape A, Shape B)
{
    // 初始搜索方向
    Vector3 direction = B.Center - A.Center;

    if (direction.LengthSquared() < EPSILON)
        direction = Vector3(1, 0, 0);

    Simplex simplex;

    // Minkowski Difference 上的 Support Point
    Vector3 point = Support(A, B, direction);

    simplex.Add(point);

    // 下一步朝原点搜索
    direction = -point;

    while (true)
    {
        // 朝 direction 找 Minkowski Difference 最远点
        point = Support(A, B, direction);

        // 连最远点都无法越过原点
        // 那么原点一定不在 Minkowski Difference 内
        if (Dot(point, direction) < 0)
        {
            return false;
        }

        simplex.Add(point);

        // 更新 simplex
        // 同时更新下一次搜索方向 direction
        if (HandleSimplex(simplex, direction))
        {
            // Simplex 已经包围原点
            return true;
        }
    }
}
```

而：

```
Support(A,B,d)
```

只有：

```
Vector3 Support(
    Shape A,
    Shape B,
    Vector3 direction)
{
    return A.Support(direction)
         - B.Support(-direction);
}
```

所以 GJK 的主体其实特别短。

真正复杂的是：

```
HandleSimplex()
```

---

# 24. `HandleSimplex()` 的结构

大概就是：

```
bool HandleSimplex(
    Simplex& simplex,
    Vector3& direction)
{
    switch (simplex.Count)
    {
        case 2:
            return HandleLine(simplex, direction);

        case 3:
            return HandleTriangle(simplex, direction);

        case 4:
            return HandleTetrahedron(simplex, direction);
    }

    return false;
}
```

所以你在看 Box2D、Bullet、PhysX 或其他 GJK 实现时，应该按这个思路理解，而不是逐行死磕 Cross Product。

---

# 25. GJK 和 SAT 的思维差别

你之前问 OBB 为什么 SAT 是 15 根轴。

SAT 对两个 OBB：

$$
\[ A_0,A_1,A_2 \]\[ B_0,B_1,B_2 \]
$$

必须检测：

$$
\[ 3+3+9=15 \]
$$

根候选轴。

因为 SAT 思路是：

> 有没有哪个方向能把两个物体完全分开？

所以：

```
寻找分离轴
```

而 GJK 是：

```
        Shape A
        Shape B
           ↓
    Minkowski Difference
           ↓
      原点在里面吗？
```

因此它完全不需要知道：

```
OBB 有 15 个 SAT 轴
```

它只需要：

```
OBB.Support(direction);
```

例如 Box vs Capsule：

SAT 就没那么漂亮，因为你需要重新推导候选分离轴。

但 GJK：

```
SupportBox(d)
SupportCapsule(-d)
```

就结束了。

这正是 GJK 的最大优势之一。

---

# 26. Sphere 的 Support 更简单

Sphere：

$$
\[ Center=C \]
$$

半径：

$$
\[ r \]
$$

方向：

$$
\[ d \]
$$

那么：

$$
\[ Support(d) = C+r\frac{d}{|d|} \]
$$

因为球在某个方向最远的点，就是球心加：

```
归一化方向 × 半径
```

所以：

```
Vector3 Sphere::Support(Vector3 d)
{
    return center + Normalize(d) * radius;
}
```

---

# 27. Capsule 的 Support

Capsule 可以看成：

```
线段 + Sphere
```

假设中心线端点：

$$
\[ A,B \]
$$

半径：

$$
\[ r \]
$$

先找线段在方向 \(d\) 最远的端点：

$$
\[ P= \begin{cases} A,&A\cdot d>B\cdot d\\ B,&otherwise \end{cases} \]
$$

然后：

$$
\[ Support(d) = P+r\hat d \]
$$

因此 Capsule 接入 GJK 也特别容易。

---

# 28. Convex Hull 的 Support

如果凸包有顶点：

$$
\[ v_1,v_2,\dots,v_n \]
$$

最简单实现：

```
Vector3 Support(Vector3 d)
{
    float maxDot = -INF;
    Vector3 best;

    for (Vector3 v : vertices)
    {
        float value = Dot(v, d);

        if (value > maxDot)
        {
            maxDot = value;
            best = v;
        }
    }

    return best;
}
```

复杂度：

$$
\[ O(N) \]
$$

但真实物理引擎会利用 convex hull 邻接关系做 hill climbing，使 support 搜索更快。

---

# 29. 一个非常重要的问题：GJK 只能处理凸体

因为整个理论依赖凸性。

例如这种凹形状：

```
+-------+
|       |
|   +---+
|   |
|   |
+---+
```

Support Mapping 只能看到：

```
最外层
```

它无法正确表达中间凹进去的结构。

所以游戏物理通常：

```
Concave Mesh
     ↓
Convex Decomposition
     ↓
Convex Hull 1
Convex Hull 2
Convex Hull 3
...
```

然后：

```
Hull vs Hull
```

执行 GJK。

这也是很多引擎为什么碰撞体里会区分：

```
Convex Mesh
```

和：

```
Triangle Mesh
```

---

# 30. GJK 判断出来碰撞以后，还缺什么？

假设 GJK 最后得到：

```
Collision = true
```

它实际上只告诉你：

$$
\[ \boxed{两个物体相交} \]
$$

但游戏物理通常还需要：

```
碰撞法线是多少？
```

```
穿透深度是多少？
```

例如：

```
        Box A
     +--------+
     |        |
     |    +---|----+
     +----|---+    |
          | Box B  |
          +--------+
```

需要得到：

$$
\[ Normal \]
$$

以及：

$$
\[ PenetrationDepth \]
$$

GJK 本身的传统布尔版本并不能完整给出这些。

所以通常是：

$$
\[ \boxed{GJK+EPA} \]
$$

GJK：

```
有没有碰撞？
```

EPA：

```
如果碰撞：
碰撞法线是什么？
穿透多深？
```

---

# 31. EPA 为什么能接在 GJK 后面

GJK 碰撞成功时已经得到一个：

```
包含原点的 Minkowski 四面体
```

例如：

```
              *
             / \
            / O \
           *-----*
            \   /
             \ *
```

EPA，Expanding Polytope Algorithm，会从这个 simplex 开始不断：

```
扩张 Minkowski Polytope
```

寻找：

> 距原点最近的 Minkowski Difference 表面。

假设：

```
                 Minkowski surface
                /
               /
              X
             /
            O
```

原点到这个表面的最短方向：

$$
\[ n \]
$$

就是碰撞法线。

距离：

$$
\[ d \]
$$

就是穿透深度。

所以经典 Narrow Phase：

```
Broad Phase
    ↓
可能碰撞的 Pair
    ↓
GJK
    ↓
是否相交？
    ↓ yes
EPA
    ↓
Normal + Penetration
    ↓
Contact Solver
```

这是非常典型的物理引擎结构。

---

# 32. 但还有另一个非常重要的 GJK：Distance GJK

GJK 不只是能：

```
Collision / No Collision
```

实际上它原本特别擅长解决：

\[ 两个凸体之间的最近距离 \]

例如：

```
Box A                Box B

+------+             +------+
|      |             |      |
+------+             +------+
     ●----------------●
             d
```

可以得到：

```
Closest Point A
Closest Point B
Distance
```

这对：

```
Continuous Collision Detection
Shape Cast
Conservative Advancement
```

非常重要。

现代物理引擎中的 GJK 往往其实更接近：

> **不断寻找 Minkowski Difference 上距离原点最近的 simplex。**

如果这个最近距离最终：

$$
\[ =0 \]
$$

那么就是碰撞。

所以从更深层次理解 GJK：

$$
\[ \boxed{ GJK实际上是在凸体中寻找离原点最近的位置 } \]
$$

而：

```
原点在里面
```

只是最近距离恰好为 0 的特殊情况。

---

# 33. 从“最近点”的视角重新理解整个算法

假设 Minkowski Difference：

```
            ********
         ***        ***
       **              **
      *                  *
     *                    *
      **                **
        ***          ***
           ********

                            O
```

GJK 会找：

```
Minkowski Difference
上离 O 最近的位置
```

开始可能得到一个点：

```
A
```

然后发现：

```
线 AB 更接近
```

然后：

```
Triangle ABC 更接近
```

不断收缩：

```
Point
  ↓
Line
  ↓
Triangle
```

如果最终：

```
最近点 ≠ O
```

没有碰撞。

如果：

```
O 被 simplex 包住
```

最近距离：

\[ 0 \]

碰撞。

这个理解比单纯背：

```
dot < 0 return false
```

更接近 GJK 的本质。

---

# 34. 为什么 GJK 通常很快

你可能会觉得：

```
不断迭代 Support
```

会不会特别慢？

实际上正常情况下迭代次数很少。

因为每次都在做非常强的几何剪枝：

```
Support point
       ↓
判断 Voronoi Region
       ↓
删除不可能相关的点
       ↓
直接朝原点搜索
```

Simplex 永远最多：

\[ 4 \]

个点。

所以即使两个 Convex Hull：

```
100个顶点
vs
100个顶点
```

也不会真的比较：

\[ 100\times100 \]

个顶点组合。

它只是不断问：

```
这个方向最远的点是谁？
```

这也是为什么它非常适合通用 Convex Collision。

---

# 35. 但是工程上的 GJK 比教科书版复杂很多

网上经常能看到几十行的 GJK：

```
while (true)
{
    ...
}
```

理论上没问题，但直接放进商业物理系统会遇到很多问题，主要是数值退化：support point 重复、direction 接近零、三点近乎共线、四点近乎共面、两个表面刚好接触、浮点误差导致 simplex 来回震荡、`dot == 0` 时接触到底算碰撞还是非碰撞等。

所以真正实现通常需要：

$$
\[ \epsilon \]
$$

最大迭代次数：

```
MaxIteration = 20 / 32 / 64
```

以及：

```
重复 Support Point 检测
```

否则理论上的：

```
while(true)
```

可能在极端输入下不收敛。

---

# 36. GJK 和 SAT 到底怎么选

如果你以后做类似《鸣潮》这样的战斗 / 物理系统，可以这样理解。

SAT 对：

```
Box
OBB
Triangle
```

非常直接，而且 SAT 很容易得到：

```
Minimum Penetration Axis
```

因此碰撞法线和 penetration 很自然。

但是增加一种新形状时，经常需要重新推导：

```
候选轴有哪些？
```

GJK 则把形状统一成：

```
Support(direction)
```

例如：

```
Sphere ─────┐
Capsule ────┤
OBB ────────┤
ConvexHull ─┤
Cylinder ───┤
            ↓
          GJK
```

这就是为什么 GJK 特别适合：

**通用凸体 Narrow Phase。**

---

# 37. 最后把整个 GJK 压缩成一张思维图

```
两个凸体
A、B
 │
 │
 ▼
构造概念上的 Minkowski Difference
C = A - B
 │
 │
 ▼
A 与 B 碰撞
⇕
原点 O ∈ C
 │
 │
 ▼
不真正构造 C
 │
 ▼
SupportC(d)
=
SupportA(d)
-
SupportB(-d)
 │
 │
 ▼
取一个 Support Point
 │
 ▼
Simplex
 │
 ├── Point
 │
 ├── Line
 │
 ├── Triangle
 │
 └── Tetrahedron
 │
 │
 ▼
判断 O 所在 Voronoi Region
 │
 ▼
删除无用顶点
 │
 ▼
产生新的 Search Direction
 │
 ▼
再次 Support(d)
 │
 ├─────────────┐
 │             │
 ▼             ▼
最远点         Simplex
都没越过O     包围O
 │             │
 ▼             ▼
No Collision   Collision
```

所以真正值得记住的不是某一段 `Cross(Cross())` 代码，而是这三个核心思想：

$$
\boxed{ 碰撞 \iff O\in A-B } \]\[ \boxed{ Support_{A-B}(d) = Support_A(d)-Support_B(-d) }
$$

以及：

\[ \boxed{ 不断维护距离原点最近、最可能包围原点的Simplex } \]

一旦这三个东西理解透了，GJK 的 Line / Triangle / Tetrahedron Handler 就只是**在判断“原点处于当前 Simplex 的哪个 Voronoi 区域”**。

如果接下来要真正吃透实现，最值得继续的是 **“逐行推导 3D GJK 的 `HandleLine / HandleTriangle / HandleTetrahedron`，尤其解释每一个 Cross 为什么这么叉”**。这部分就是面试和自己写物理引擎时最容易卡住的地方。