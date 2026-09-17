Broad Phase（粗检测 / 宽相位）阶段的目标不是判断两个物体“真的碰撞了没有”，而是**快速排除绝大多数不可能碰撞的物体对**，生成候选 Pair，交给 Narrow Phase 做精确检测。

实际物理引擎里，最常见的 Broad Phase 算法主要是下面几类。对游戏开发和面试来说，重点掌握 **SAP、Uniform Grid / Spatial Hash、BVH / Dynamic AABB Tree**。

|算法|核心思想|适合场景|重点程度|
|---|---|---|---|
|Brute Force|所有物体两两检测|物体极少|★★|
|Sweep and Prune（SAP）|在坐标轴上排序 AABB 区间|大量动态物体、位置连续变化|★★★★★|
|Uniform Grid|空间划分成规则格子|物体大小接近、分布均匀|★★★★★|
|Spatial Hash|Grid + 哈希表|大空间/无限空间|★★★★★|
|BVH / AABB Tree|用树组织包围盒|通用复杂场景|★★★★★|
|Dynamic AABB Tree|动态维护 AABB 树|动态物体较多|★★★★★|
|Octree / Quadtree|递归空间划分|大型空间、静态物体|★★★|
|KD-Tree / BSP|按空间平面递归切分|偏静态场景|★★|

下面重点讲几个真正高频的。

---

## 1. 暴力检测 Brute Force

最简单：

```
for i = 0 ~ n
    for j = i+1 ~ n
        if AABB[i] overlaps AABB[j]
            CandidatePair(i,j)
```

复杂度：

 O(N^2) 

例如：

```
100 个物体
→ 4950 对

10000 个物体
→ 约 5000 万对
```

所以通常只适用于：

- 物体数量非常少
- 调试
- 作为算法正确性的 baseline

它本身严格来说也可以算 Broad Phase，但大型物理引擎基本不会直接这样干。

---

# 2. Sweep and Prune：SAP

这是 Broad Phase 最经典的算法之一，也叫：

> Sort and Sweep

核心思想是利用 **AABB 在某个坐标轴上的投影区间**。

假设有三个物体：

```
A: [1 ------ 5]
B:       [3 ------ 7]
C:                  [8 --- 10]
```

先只看 X 轴。

A 和 B：

```
[1,5]
[3,7]
```

区间重叠。

所以：

```
A B 可能碰撞
```

A 和 C：

```
[1,5]

        [8,10]
```

完全没有重叠。

因此：

```
A C 一定不碰撞
```

甚至不需要检查 Y、Z。

### SAP 的过程

每个 AABB：

```
minX
maxX
```

例如：

```
A.minX
B.minX
A.maxX
B.maxX
C.minX
C.maxX
```

对这些端点排序：

```
---------- X ---------->

Amin
   Bmin
       Amax
          Bmax
                Cmin
                     Cmax
```

从左向右扫描。

维护一个 Active List：

```
遇到 min
→ 加入 Active

遇到 max
→ 从 Active 移除
```

当 B.min 被扫描到时：

```
Active = { A }

因此：
A-B 是潜在碰撞对
```

然后继续检查 Y、Z：

```
X overlap
&&
Y overlap
&&
Z overlap
```

才成为真正的 Broad Phase Candidate Pair。

---

## SAP 为什么特别适合游戏？

因为游戏具有一个非常重要的特征：

> Temporal Coherence，时间连续性。

这一帧：

```
A B C D E
```

下一帧通常不会突然变成：

```
E C A D B
```

而更可能是：

```
A B D C E
```

只发生少量交换。

因此排序通常可以使用：

```
Insertion Sort
```

在基本有序的数据上非常快。

理想情况下可以接近：

$$
 O(N) 
$$

所以 SAP 很适合：

- 刚体
- 角色
- 箱子
- 动态场景
- 连续运动物体

---

# 3. Uniform Grid

另一个非常经典的 Broad Phase。

直接把世界切成格子：

```
+----+----+----+----+
|    | A  |    |    |
+----+----+----+----+
|    | A  | B  |    |
+----+----+----+----+
|    |    | B  | C  |
+----+----+----+----+
```

每个物体根据 AABB 放入对应 Cell。

例如：

```
Cell(2,3)
{
    Player
    Enemy1
    Enemy2
}
```

那么：

```
Player
```

根本没必要检测：

```
几百米外的 Enemy100
```

只检查：

```
同 Cell

以及必要的相邻 Cell
```

---

## Grid 为什么快？

假设：

```
世界里 10000 个物体
```

但平均一个 Cell 只有：

```
10 个物体
```

那么每个物体只需要和附近这几个对象进行检测。

理想情况下可以接近：

$$
 O(N) 
$$

---

## Grid 最大的问题

就是：

> Cell Size 很难选。

比如：

```
Cell = 1m
```

但是有一个：

```
100m × 100m Boss
```

它会跨越非常多 Cell。

反过来：

```
Cell = 100m
```

但是大量物体只有：

```
0.5m
```

一个 Cell 里就会塞大量物体。

于是 Broad Phase 退化。

所以 Uniform Grid 特别适合：

```
物体尺寸接近
+
物体分布相对均匀
```

例如：

- 粒子
- 子弹
- RTS 单位
- 大量小型实体

---

# 4. Spatial Hash

Spatial Hash 本质上就是：

> 无限 / 巨大空间版本的 Uniform Grid。

普通 Grid 可能直接创建：

```
Cell grid[1000][1000][1000];
```

空间非常浪费。

Spatial Hash 则：

```
hash(x, y, z)
```

例如：

```
int hash(int x, int y, int z)
{
    return
        x * 73856093 ^
        y * 19349663 ^
        z * 83492791;
}
```

然后：

```
HashTable
   ↓
Bucket
   ↓
Objects
```

只有真正存在物体的位置才需要 Cell。

所以特别适合：

```
巨大世界
开放世界
稀疏空间
```

以及：

```
粒子
布料
流体
大量单位
```

---

# 5. BVH

这个在现代碰撞检测里非常重要。

BVH：

> Bounding Volume Hierarchy  
> 包围体层次结构

通常 Broad Phase 使用：

> AABB Tree

比如场景有：

```
A B C D E F
```

建立：

```
                Root AABB
               /         \
          AABB1             AABB2
         /    \            /    \
       A+B    C          D+E     F
```

父节点包围子节点。

查询一个物体 Q：

```
              Root
               |
           overlap?
          /         \
        yes          no
        ↓             X
      Node1
     /     \
   yes     no
    ↓       X
  Object
```

如果某个高层节点都不重叠：

```
整个子树全部排除
```

这就是它高效的原因。

---

# 6. Dynamic AABB Tree

游戏物理中尤其重要。

普通 BVH 更适合：

```
Static Geometry
```

但是游戏中的物体一直在移动：

```
Character
Enemy
RigidBody
Vehicle
```

如果每帧重新构建 BVH：

```
Build BVH
Build BVH
Build BVH
...
```

代价很大。

于是出现：

> Dynamic AABB Tree

核心思想：

```
物体移动
↓
更新 Leaf
↓
必要时重新插入 / 调整 Tree
```

而不是整棵树重建。

---

## Fat AABB

Dynamic AABB Tree 还有一个非常经典的优化：

> Fat AABB

比如真实 AABB：

```
+------+
| Obj  |
+------+
```

实际树里保存稍微扩大的：

```
+------------+
|            |
|   +----+   |
|   |Obj |   |
|   +----+   |
|            |
+------------+
```

只要真实 AABB 还在 Fat AABB 内：

```
不用更新树
```

物体稍微移动：

```
1 cm
2 cm
5 cm
```

都不需要重新插入节点。

这可以显著降低：

```
Tree Update Cost
```

这是 Dynamic AABB Tree 一个非常经典的面试点。

---

# 7. Octree / Quadtree

Quadtree：

```
2D 空间
```

把区域递归分为四份：

```
+-------+-------+
|       |       |
|   1   |   2   |
|       |       |
+-------+-------+
|       |       |
|   3   |   4   |
|       |       |
+-------+-------+
```

Octree：

```
3D 空间
```

每个区域分成：

$$
 8 
$$

个子区域。

不断递归：

```
World
 ↓
8 regions
 ↓
8 regions
 ↓
8 regions
```

优点是非常直观，并且适合：

```
大型空间
静态物体
空间分布不均匀
```

但对于大量动态刚体：

```
Object 移动
→ 跨 Octree Node
→ 删除
→ 重新插入
```

维护成本可能比较高。

所以：

> Octree 非常著名，但不一定是现代刚体物理 Broad Phase 的首选。

---

# 最重要的三个

如果你是准备**游戏开发 / UE / Unity / 物理引擎面试**，Broad Phase 最值得重点掌握的是：

```
                Broad Phase
                     │
        ┌────────────┼────────────┐
        │            │            │
       SAP          Grid         BVH
        │            │            │
 Sort & Sweep   Spatial Hash   AABB Tree
                                  │
                           Dynamic AABB Tree
```

它们各自解决不同类型的问题：

|场景|更适合|
|---|---|
|大量动态刚体，并且运动连续|SAP|
|大量尺寸相似小物体|Uniform Grid|
|巨大、稀疏空间|Spatial Hash|
|通用刚体系统|Dynamic AABB Tree|
|大量静态几何|BVH|
|大型空间层级划分|Octree|

你可以把整个碰撞检测流程记成：

```
                所有 Collision Objects
                         ↓
                 ┌─────────────┐
                 │ Broad Phase │
                 └─────────────┘
                         ↓
                   Candidate Pairs
                  A-B  A-F  C-D
                         ↓
                 ┌─────────────┐
                 │Narrow Phase │
                 └─────────────┘
                         ↓
                SAT / GJK / EPA
                         ↓
              Contact / Penetration
                         ↓
                Collision Solver
```

所以你前面学的 **GJK / EPA** 属于 Narrow Phase，而 **SAP、Spatial Hash、Dynamic AABB Tree** 就属于它们之前的 Broad Phase。

如果按面试重要程度继续往下学，我会建议下一步重点理解 **Dynamic AABB Tree 的插入、删除、查询和 Fat AABB**，因为它能把 Broad Phase 和你已经学过的 GJK/EPA 整个碰撞检测链串起来。