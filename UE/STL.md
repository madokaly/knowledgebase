# UE5 容器与 C++ STL 容器及底层原理详解

在 Unreal Engine（UE）游戏客户端开发中，容器是非常重要的 C++ 基础知识，尤其是在战斗系统、技能系统、对象管理、网络同步、资源管理和性能优化中。

这部分不仅要知道如何使用容器，更重要的是理解：

- 底层数据结构：连续数组、哈希表、红黑树、链表、稀疏数组。
    
- 内存管理：容器如何分配内存、扩容、缩容和释放。
    
- 时间复杂度：为什么有些容器查询快，有些插入和删除快。
    
- 性能差异：CPU Cache（缓存局部性）、内存碎片、内存分配次数。
    
- UE 特有机制：`UPROPERTY`、GC（垃圾回收）、反射、序列化、Allocator（内存分配器）。
    
- 工程实践：什么情况下使用 `TArray`、`TMap`、`TSet`、`std::vector`、`std::unordered_map` 等。
    

下面按 常用程度 + 底层实现 + 性能原理 + UE 实际使用 + 面试高频问题 进行讲解。

## 一、UE 容器与 STL 容器总览

### 1.1 常见容器对应关系

|UE 容器|C++ STL 容器|底层结构|主要用途|
|---|---|---|---|
|`TArray`|`std::vector`|动态连续数组|有序数据、连续遍历|
|`TMap`|`std::unordered_map`|哈希表|Key-Value 快速查询|
|`TSet`|`std::unordered_set`|哈希集合|去重、快速判断存在性|
|`TSparseArray`|无直接对应|稀疏数组|频繁增删、复用空洞|
|`TDoubleLinkedList`|`std::list`|双向链表|节点增删|
|`TQueue`|`std::queue`（接口近似）|UE 无锁链表队列|生产者消费者|
|`TStaticArray`|`std::array`|固定大小数组|固定数量的数据|
|`TBitArray`|`std::vector<bool>`（用途近似）|位压缩数组|大量布尔标记|
|`TChunkedArray`|`std::deque`（部分特性近似）|分块存储|大量元素、减少整体搬迁|

这里有三个重要区别：

1. `TMap` 和 `std::unordered_map` 都是哈希容器，但内部存储元素的方式不同。
    
2. `TQueue` 和 `std::queue` 并不等价。`TQueue` 提供特定模式下的线程安全操作，而 `std::queue` 默认只是一个容器适配器，不保证线程安全。
    
3. `TChunkedArray` 和 `std::deque` 都涉及分段存储，但并不是可以直接互换的容器。
    

### 1.2 最常用容器的时间复杂度

下面的复杂度假设使用默认或典型实现，哈希查询为平均情况。

|操作|TArray / vector|TMap / unordered_map|TSet / unordered_set|std::map|
|---|---|---|---|---|
|按下标访问|O(1)|不适用|不适用|不适用|
|按 Key 查找|O(n)|O(1) 平均|O(1) 平均|O(log n)|
|尾部添加|O(1) 均摊|—|—|—|
|插入元素|O(n) 中间插入|O(1) 平均|O(1) 平均|O(log n)|
|删除指定元素|O(n) 查找或移动|O(1) 平均（已知 Key）|O(1) 平均（已知 Key）|O(log n)|
|遍历全部元素|O(n)|O(n) 通常|O(n) 通常|O(n)|

注意：`TArray::RemoveAtSwap` 在不触发缩容时，可以做到 O(1) 删除。`std::map` 底层通常是红黑树（Red-Black Tree，平衡二叉搜索树），而非哈希表。

UE 游戏客户端中，最需要深入掌握的是 `TArray`、`TMap`、`TSet`，其次是 `TSparseArray` 和 `TQueue`。

## 二、TArray 与 std::vector：动态数组的底层原理

这是最重要、最常被面试追问的一类容器。

### 2.1 底层内存结构

`TArray` 和 `std::vector` 都通过连续内存存储元素。

假设有：

```
TArray<int32> Numbers;

Numbers.Add(10);
Numbers.Add(20);
Numbers.Add(30);
```

其逻辑内存布局如下：

<img width="735" height="440" alt="image" src="https://github.com/user-attachments/assets/f7602d94-ab76-4a06-a204-21f9a6910e92" />

可以把 `TArray` 理解为主要管理三个状态：

```
// 原理示意，并非 UE 源码
template<typename T>
struct SimpleArray
{
    T* Data;        // 指向连续存储区
    int32 Num;      // 当前有效元素数量
    int32 Capacity; // 已经分配的元素容量，对应 TArray::Max()
};
```

其中：

- `Data`：指向元素内存的起始位置。
    
- `Num`：已经构造并纳入容器管理的元素数量。
    
- `Max`：当前容量，即不扩容时最多可存储的元素数量。
    
- `Max - Num`：未使用容量，UE 称为 Slack（冗余容量）。
    

对于 `std::vector`，常见标准库实现使用三个指针，分别表示起始、已用结束和已分配结束的位置。但 C++ 标准并未强制规定其内部必须使用三个指针。

### 2.2 为什么动态数组需要扩容？

假设某数组当前：

```
Num = 4
Max = 4
```

此时再添加一个元素，原来的存储空间已经不够。

典型扩容流程：

<img width="739" height="292" alt="image" src="https://github.com/user-attachments/assets/a1a6abcc-f423-4b5e-831b-e295c1b1bac1" />

扩容时可能执行：

1. 计算新容量。
    
2. 申请足够大的连续内存。
    
3. 将已有元素迁移到新内存。
    
4. 在新位置构造新元素。
    
5. 释放旧存储空间。
    

需要特别注意，具体分配器也可能原地扩容，并非每次都一定发生地址变化。

因此，单次扩容成本是 O(n)，但大多数尾部插入不需要扩容。

当容量按几何方式增长时，连续执行 n 次 `Add` 或 `push_back` 的总成本为 O(n)，因此：

$$
\boxed{\text{尾部插入的均摊时间复杂度}=O(1)}
$$

这里的均摊（Amortized）是指把偶尔发生的昂贵扩容成本分摊到多次普通插入操作上。

### 2.3 UE TArray 和 std::vector 扩容时最大的底层区别

两者有一个非常值得关注的差异：元素重定位（Relocation）机制。

`std::vector` 扩容时，通常通过元素的移动构造或复制构造将旧元素放到新存储区。对于满足特定条件的简单类型，标准库实现可以优化成字节搬迁。

而 UE 的 `TArray` 采用以可平凡重定位（Trivially Relocatable）为核心的设计约定。对符合要求的元素，可以直接进行原始字节搬迁，而不逐个调用移动构造函数。Epic 官方文档明确指出了这一设计要求。

Epic Developer Community+1

下面用简化伪代码说明区别：

```
// std::vector 的典型扩容思路（简化）
for (size_t i = 0; i < OldSize; ++i)
{
    // 可能移动构造，也可能复制构造
    new (NewData + i) T(std::move_if_noexcept(OldData[i]));
}

// 清理旧对象
// 释放旧内存
```

```
// UE TArray 对可平凡重定位类型的概念示意
FMemory::Memcpy(
    NewData,
    OldData,
    OldNum * sizeof(T)
);

// 释放旧存储区
// 实际实现会考虑类型特征、分配器和生命周期
```

但不能认为任意复杂 C++ 类都可以随意 `Memcpy`。例如：

```
struct FBadData
{
    int32 Value = 0;
    int32* SelfValue = &Value;
};
```

如果 `SelfValue` 指向自身成员，直接将整个对象按字节迁移到新地址后，指针仍然可能指向旧对象的内存位置。

这种具有地址敏感关系的类型，不应直接假设适合 `TArray` 的重定位约定；需要重新设计类型或采用适当的稳定地址间接存储。

面试重点： `TArray` 和 `std::vector` 虽然都是连续动态数组，但不能简单认为两者的元素生命周期和重定位行为完全一样。

### 2.4 TArray 与 vector 的常用接口对照

|功能|UE TArray|STL vector|
|---|---|---|
|添加元素|`Add()`|`push_back()`|
|原位构造|`Emplace()`|`emplace_back()`|
|元素数量|`Num()`|`size()`|
|容量|`Max()`|`capacity()`|
|预分配容量|`Reserve()`|`reserve()`|
|获取首地址|`GetData()`|`data()`|
|下标访问|`Array[i]`|`Vector[i]`|
|清理元素、保留容量|`Reset()`|`clear()`|
|释放多余容量|`Shrink()`|`shrink_to_fit()`|
|删除指定下标|`RemoveAt()`|`erase()`|
|无序快速删除|`RemoveAtSwap()`|手动 swap + pop|
|排序|`Sort()`|`std::sort()`|

特别注意，`std::vector::shrink_to_fit()` 只是请求缩容，不保证一定发生；`TArray::Shrink()` 的设计目标则是移除多余容量。

#### Reserve 与 SetNum 的区别

```
TArray<int32> A;

// 预留 100 个元素的容量
A.Reserve(100);

// 当前元素数量仍然是 0
check(A.Num() == 0);

// 构造 100 个有效元素
A.SetNum(100);

// 当前有 100 个有效元素
check(A.Num() == 100);
```

`Reserve` 只影响 Capacity（容量）；`SetNum` 会改变有效元素数量，增加元素时还涉及初始化。

高频面试陷阱：

```
TArray<int32> Values;
Values.Reserve(100);

// 错误：Reserve 不代表已有 100 个元素
// Values[99] = 10;

// 正确
Values.Add(10);
```

### 2.5 为什么 TArray 中间删除比较慢？

假设有：

```
[10, 20, 30, 40, 50, 60]
```

执行：

```
Numbers.RemoveAt(1, 1, EAllowShrinking::No);
```

删除下标 1 的元素 `20`。

为了保持顺序：

```
删除前：
[10, 20, 30, 40, 50, 60]

删除后：
[10, 30, 40, 50, 60]
     ←   ←   ←   ←
```

后面的元素需要整体向前移动，因此：

$$
T(n)=O(n)
$$

但游戏中很多集合不要求顺序，比如当前激活的敌人、待处理事件、临时碰撞对象等。

这时应该优先考虑：

```
Numbers.RemoveAtSwap(1, 1, EAllowShrinking::No);
```

实际思路如下：

```
原数组：
[10, 20, 30, 40, 50, 60]

用末尾元素替换删除位置：
[10, 60, 30, 40, 50, 60]

移除末尾位置：
[10, 60, 30, 40, 50]
```

不需要搬移整个后半段。

$$
\boxed{\text{RemoveAtSwap}=O(1)}
$$

这里指删除单个已知位置的元素、且不触发内存缩容时的复杂度。

Epic 文档也明确区分了 `RemoveAt` 和 `RemoveAtSwap` 的性能与顺序保证。

Epic Developer Community

#### UE 战斗系统实例：移除失效目标

```
// 保存锁定目标，使用弱引用，不持有 Actor 生命周期
TArray<TWeakObjectPtr<AActor>> Targets;

// 从后向前遍历，删除已经失效的目标
for (int32 i = Targets.Num() - 1; i >= 0; --i)
{
    if (!Targets[i].IsValid())
    {
        // 不要求顺序，因此使用快速删除
        // 禁止缩容，避免循环中发生不必要的内存调整
        Targets.RemoveAtSwap(
            i,
            1,
            EAllowShrinking::No
        );
    }
}
```

为什么逆序遍历？因为正序删除会改变下标，可能漏掉某些元素；逆序更容易保证待遍历的前半段不受影响。

### 2.6 TArray 和 std::vector 的迭代器失效

这是游戏客户端非常常见的崩溃原因。

```
TArray<int32> Array = {10, 20, 30};

int32* Ptr = &Array[0];

// Reserve 可能导致原来的数据被迁移
Array.Reserve(1000);

// 错误：Ptr 可能已经成为悬空指针
// UE_LOG(LogTemp, Log, TEXT("%d"), *Ptr);
```

关键规律：

|操作|TArray / vector 的风险|
|---|---|
|扩容|所有元素指针、引用、迭代器失效|
|中间插入|插入位置及其后的元素位置发生变化；扩容会影响全部|
|中间删除|被删除位置及之后的元素引用可能不再有效|
|`RemoveAtSwap`|删除位置与末尾元素的位置发生变化|
|`Reset`|元素被销毁，原有元素引用失效|
|`Shrink`|可能重新分配存储，指针可能失效|

特别注意：`TArray` 的下标不是天然稳定的对象 ID。

如果一个技能系统把 `TArray` 下标长期当成技能实例 ID，在删除和交换元素后，就可能访问错误的技能实例。

后面介绍的 `TSparseArray` 和带 Generation（代数）的 Handle，可以解决一部分此类问题。

### 2.7 TInlineAllocator：UE 小数组优化

UE 在动态数组上提供了非常实用的内存分配器：

```
TArray<int32, TInlineAllocator<8>> Numbers;
```

含义是：数组对象内部预留可以存储 8 个元素的空间。

<img width="739" height="260" alt="image" src="https://github.com/user-attachments/assets/ed0d1d9f-f690-4537-9a27-90136b1a4d04" />

优点：

- 小数组通常不需要额外堆分配。
    
- 减少内存分配器调用。
    
- 对元素数量少、创建频繁的临时容器尤其有价值。
    

缺点：数组对象本身会变大，而且超过容量后仍需要申请堆内存。它不是对任何数组都更快。

```
void ProcessNearbyEnemies()
{
    // 假设一次技能检测通常只有 4~8 个目标
    TArray<AActor*, TInlineAllocator<8>> NearbyEnemies;

    // ... 收集并处理目标
}
```

这适合原生 C++ 临时数据。对于要交给 UHT（Unreal Header Tool，反射头文件工具）解析的 `UPROPERTY` 数组，应优先使用受支持的普通 `TArray` 形式，不要假定自定义 Allocator 可以被反射系统识别。

## 三、TMap、std::unordered_map 与 std::map：哈希表和红黑树

这部分非常重要，因为三个容器虽然都能完成 Key-Value（键值对）查找，底层却存在明显区别。

### 3.1 三种容器的核心区别

|属性|UE TMap|std::unordered_map|std::map|
|---|---|---|---|
|主要结构|哈希表 + 稀疏数组|哈希表|平衡搜索树，通常红黑树|
|查询复杂度|O(1) 平均|O(1) 平均|O(log n)|
|是否按 Key 排序|否|否|是|
|哈希冲突|需要处理|需要处理|无哈希冲突|
|查找最坏复杂度|O(n)|O(n)|O(log n)|
|元素地址稳定性|增长时可能改变|rehash 通常保持元素地址|插入通常保持元素地址|
|UE 反射支持|支持符合要求的属性类型|不直接支持|不直接支持|

`std::unordered_map` 在主流标准库里通常是节点式哈希结构；`TMap` 则采用 `TSet` 作为基础，元素存储涉及 `TSparseArray`（稀疏数组）。

Epic Developer Community

### 3.2 哈希表的工作原理

假设需要存储玩家技能：

```
TMap<int32, int32> SkillDamage;

SkillDamage.Add(101, 500);
SkillDamage.Add(102, 800);
SkillDamage.Add(103, 1200);
```

哈希表首先根据 Key 计算哈希值：

$$
H=\operatorname{Hash}(Key)
$$

然后把哈希值映射到某个 Bucket（桶）：

$$
BucketIndex=H\bmod BucketCount
$$

实际实现也可能采用位掩码等映射方式，并不一定直接使用取模。

哈希桶与冲突链示意

概念示意：桶用于定位元素，发生碰撞时通过冲突链等机制继续查找。并非 TMap 的精确内部字段图。

假设 Key `101` 和 `105` 被映射到同一个桶，那么会出现 Hash Collision（哈希冲突）。

哈希冲突不能简单用哈希值再次比较就结束，而是需要检查实际 Key 是否相等。

以 UE 默认 `TMap` 的 Key 规则为例：

```
// 先通过哈希函数定位候选区域
GetTypeHash(Key);

// 再通过相等比较确认是否是同一个 Key
KeyA == KeyB;
```

核心要求：

$$
A==B\implies Hash(A)==Hash(B)
$$

如果两个相等的 Key 被计算出不同的哈希值，容器查询的正确性就可能被破坏。

### 3.3 Hash 冲突为什么会影响性能？

正常情况下，哈希函数分布较均匀，每个桶内只有少量元素：

$$
T_{\text{Find}}=O(1)\text{ 平均}
$$

如果大量不同 Key 落入同一桶，就可能需要连续比较很多元素：

$$
T_{\text{Find}}=O(n)\text{ 最坏}
$$

影响性能的主要因素包括哈希函数质量、Load Factor（负载因子）、桶的数量和 Key 比较成本。

对于经常使用字符串查找的数据结构，还需要考虑计算字符串哈希本身的成本。不要只看容器的理论 O(1) 而忽略 Key 的处理开销。

### 3.4 TMap 的底层为什么不是简单的哈希表？

UE 的 `TMap` 可以理解为下面的组合：

```
TMap<Key, Value>
       |
       v
TSet<TPair<Key, Value>>
       |
       +---- 哈希桶数据
       |       |
       |       +---- 快速定位元素
       |
       +---- TSparseArray
               |
               +---- 存储元素
               +---- 支持删除空洞
               +---- 复用空闲槽位
```

这是一张概念结构图。具体类层次、元素元数据和 Allocator 细节应以相应 UE 版本的源码为准。

#### 与 std::unordered_map 的差异

主流 `std::unordered_map` 实现一般使用动态分配的节点：

```
// 节点结构概念示意
struct Node
{
    KeyType Key;
    ValueType Value;

    Node* Next;
};
```

因此插入新节点时，通常不会移动其他已存在节点对象。

`TMap` 使用的稀疏数组则可以为多个元素提供数组式存储；但当底层存储空间不够时，可能重新分配并重定位元素。

所以：

`std::unordered_map` rehash 后通常会使迭代器失效，但按照标准，其已有元素的引用和指针不会仅仅因为 rehash 失效。`TMap` 的元素地址则不能依靠相同的保证。

这是特别值得记住的底层区别。

### 3.5 TMap 常用接口和陷阱

```
TMap<int32, int32> SkillDamage;

SkillDamage.Add(101, 500);
SkillDamage.Add(102, 800);

// 同样的 Key 再次 Add，会替换对应 Value
SkillDamage.Add(101, 600);

// 查询
if (int32* Damage = SkillDamage.Find(101))
{
    UE_LOG(LogTemp, Log, TEXT("Damage = %d"), *Damage);
}

// 判断 Key 是否存在
bool bExists = SkillDamage.Contains(102);

// 不存在则添加默认值，返回 Value 的引用
int32& Value = SkillDamage.FindOrAdd(103);
Value = 1200;

// 删除
SkillDamage.Remove(102);
```

UE 中很容易混淆的三个接口：

|接口|Key 不存在时|返回|
|---|---|---|
|`Find(Key)`|返回 nullptr|指向 Value 的指针|
|`FindRef(Key)`|返回默认值，不插入|Value 的副本|
|`FindOrAdd(Key)`|插入默认 Value|Value 的引用|

尤其是 `FindRef`，虽然名字带 `Ref`，实际返回的是值，而不是引用。

Epic Developer Community

另一个 UE 与 STL 的重要差异：

```
TMap<int32, int32> Map;

// UE TMap：Key 不存在时通常触发检查失败
// int32 A = Map[100];
```

而 STL：

```
std::unordered_map<int, int> Map;

// Key 不存在时，会插入一个默认构造的 Value
int A = Map[100];
```

因此不要直接把 STL 的 `operator[]` 使用习惯照搬到 UE 的 `TMap` 上。

### 3.6 std::map 为什么使用红黑树？

`std::map` 属于有序关联容器。

主流实现采用红黑树（Red-Black Tree），它是通过颜色约束保持大致平衡的二叉搜索树。

其概念结构为：

```
                 40
               /    \
             20      60
            /  \    /  \
          10   30  50   70
```

寻找 Key `50`：

```
50 > 40 → 向右
50 < 60 → 向左
找到 50
```

红黑树通过旋转和重新着色，保证树的高度为 O(log n)，因此查询、插入、删除都能保持对数复杂度。

如果一棵普通二叉搜索树退化成：

```
10
  \
   20
     \
      30
        \
         40
```

查询就可能退化为 O(n)。

但 `std::map` 的优势不仅是复杂度，还有：

- Key 自动保持排序。
    
- 支持 `lower_bound`、`upper_bound` 和范围查询。
    
- 插入通常不影响其他节点的迭代器、引用和指针。
    

代价是每个元素需要节点元数据、动态分配，并且指针跳转较多，缓存局部性一般不如连续数组。

### 3.7 UE 中有对应 std::map 的容器吗？

有一个值得了解的容器：

```
TSortedMap<int32, FString> SortedMap;
```

但它并不是红黑树。

Epic 文档说明，`TSortedMap` 使用 按 Key 排序的 `TArray<TPair<Key, Value>>` 实现。

Epic Developer Community

因此：

|操作|TSortedMap|
|---|---|
|查找|O(log n)，二分查找|
|添加|O(n)，可能需要移动后续元素|
|删除|O(n)，可能需要填补空位|
|遍历|按 Key 排序，局部性较好|

小型、读多写少的映射表中，`TSortedMap` 可能比 `TMap` 更省内存、更快。元素数量增加或频繁修改时，结果则取决于实际负载。

## 四、TSet 与 std::unordered_set：哈希集合

`TSet` 与 `TMap` 底层机制类似，但是 `TSet` 直接用元素自身作为 Key。

例如：

```
TSet<int32> HitActorIds;

HitActorIds.Add(1001);
HitActorIds.Add(1002);
HitActorIds.Add(1001);

// 默认情况下不允许重复的 Key
check(HitActorIds.Num() == 2);

if (HitActorIds.Contains(1001))
{
    // 已经命中过这个对象
}
```

### 4.1 TSet 与 TArray 查找性能

假设需要检查某个 Actor 是否已经被当前攻击命中过。

使用 `TArray`：

```
TArray<int32> HitIds;

if (!HitIds.Contains(ActorId))
{
    HitIds.Add(ActorId);
}
```

`Contains` 需要逐个比较，复杂度 O(n)。

使用 `TSet`：

```
TSet<int32> HitIds;

if (!HitIds.Contains(ActorId))
{
    HitIds.Add(ActorId);
}
```

平均查找复杂度 O(1)。

但是要注意：如果元素只有 4~8 个，TArray 依然可能更快。

原因不是算法复杂度错了，而是：

1. `TArray` 存储连续，CPU 缓存友好。
    
2. 小规模数据线性遍历只需要极少次比较。
    
3. `TSet` 需要计算哈希、访问桶、进行相等比较，还要维护额外元数据。
    

因此，游戏开发不能简单地理解为“哈希容器一定比数组快”。

### 4.2 自定义结构作为 TSet 或 TMap Key

例如使用角色 ID 和技能 ID 组成唯一标识：

```
USTRUCT()
struct FSkillKey
{
    GENERATED_BODY()

    UPROPERTY()
    int32 CharacterId = 0;

    UPROPERTY()
    int32 SkillId = 0;

    // 判断两个 Key 是否相等
    bool operator==(const FSkillKey& Other) const
    {
        return CharacterId == Other.CharacterId
            && SkillId == Other.SkillId;
    }

    // 自定义哈希函数
    friend uint32 GetTypeHash(const FSkillKey& Key)
    {
        return HashCombine(
            GetTypeHash(Key.CharacterId),
            GetTypeHash(Key.SkillId)
        );
    }
};
```

使用：

```
TMap<FSkillKey, int32> SkillLevels;

FSkillKey Key;
Key.CharacterId = 1001;
Key.SkillId = 2002;

SkillLevels.Add(Key, 5);
```

这个案例考察两个知识点：自定义 `operator==`、自定义 `GetTypeHash`。

特别需要避免把可能变化的字段用于已插入 Key 的哈希计算。否则元素进入哈希表后，原来的桶定位关系会被破坏。

## 五、TSparseArray：UE 特有的高频底层容器

`TSparseArray` 是理解 `TSet`、`TMap` 甚至部分游戏对象管理方案的重要基础。

### 5.1 为什么需要稀疏数组？

普通 `TArray`：

```
下标： 0    1    2    3    4
数据： A    B    C    D    E
```

删除 B：

```
下标： 0    1    2    3
数据： A    C    D    E
```

原来 C、D、E 的下标发生变化。

`TSparseArray` 允许删除后留下空洞：

```
下标： 0    1    2    3    4
数据： A   空    C    D    E
```

下次插入 F，可能直接复用空洞：

```
下标： 0    1    2    3    4
数据： A    F    C    D    E
```

### 5.2 底层怎样知道哪些位置有效？

从概念上看，`TSparseArray` 使用元素槽位存储区和 `TBitArray`（位数组）来标记有效槽位。

<img width="736" height="207" alt="image" src="https://github.com/user-attachments/assets/ae5c08c2-c0b9-433e-a538-a2545d0badf7" />

Epic 的 API 文档明确提到它使用 `TArray` 存储槽位，并用 `TBitArray` 记录哪些元素位置已分配。

基本操作：

```
TSparseArray<int32> Values;

// 返回插入位置的索引
int32 IndexA = Values.Add(100);
int32 IndexB = Values.Add(200);
int32 IndexC = Values.Add(300);

// 删除 B
Values.RemoveAt(IndexB);

// 判断原槽位是否有效
check(!Values.IsValidIndex(IndexB));

// 可能复用空闲的 IndexB
int32 IndexD = Values.Add(400);
```

### 5.3 TSparseArray 的优缺点

|优点|缺点|
|---|---|
|删除指定有效槽位通常 O(1)|不保证所有逻辑元素连续|
|删除一般不改变其他元素的索引|需要额外位图与空闲管理信息|
|可以复用删除产生的空槽|有效元素稀疏时遍历效率下降|
|适合索引式对象管理|扩容可能改变元素地址|

最重要的两个区别：

稳定索引不等于稳定指针。`TSparseArray` 扩容时，元素的地址仍然可能改变。

索引不变也不等于身份永久有效。因为删除的槽位会被复用，旧索引可能指向新对象。

### 5.4 为什么对象管理系统经常使用 Generation Handle？

在复杂的战斗系统中，不能总是简单把整数索引当作永久对象 ID。

例如：

```
第 1 步：Index 5 存储敌人 A
第 2 步：敌人 A 死亡，槽位释放
第 3 步：Index 5 被敌人 B 复用
第 4 步：旧代码仍持有 Index 5
```

此时可能错误地把敌人 B 当成敌人 A。

常见解决方案是把句柄（Handle）设计成：

```
struct FObjectHandle
{
    int32 Index = INDEX_NONE;

    // 槽位版本，每次槽位重新分配时更新
    uint32 Generation = 0;
};
```

访问时同时校验：

```
bool IsValidHandle(
    const FObjectHandle& Handle,
    const TArray<uint32>& Generations)
{
    return Handle.Index >= 0
        && Generations.IsValidIndex(Handle.Index)
        && Generations[Handle.Index] == Handle.Generation;
}
```

这是句柄有效性检查的一部分。实际对象池还需要判断槽位当前是否已被占用、世代值如何更新，以及 Generation 溢出时的策略。

这类技术常见于 Entity（实体）、技能实例、特效实例、碰撞对象和资源句柄管理。它与 GAS 的 Handle 思想有相通之处，但不能因此断定某个 GAS Handle 一定就是这种 Index + Generation 布局。

## 六、UE 其他常用容器与 STL 对比

### 6.1 TDoubleLinkedList 与 std::list：双向链表

典型双向链表结构：

```
nullptr ← [A] ⇄ [B] ⇄ [C] ⇄ [D] → nullptr
```

每个节点保存：

```
// 简化的双向链表节点
template<typename T>
struct FListNode
{
    T Value;

    FListNode* Prev; // 前驱节点
    FListNode* Next; // 后继节点
};
```

如果已经拿到了 B 节点的指针，删除 B 时只需修改相邻节点的链接：

```
删除前：
A ⇄ B ⇄ C

删除后：
A ⇄ C
```

因此已知节点的删除操作是 O(1)，不需要搬运其他数据。

但如果需要先搜索 B，查找过程通常是 O(n)。

|操作|链表|连续数组|
|---|---|---|
|按下标访问|O(n)|O(1)|
|已知节点删除|O(1)|O(n)，无序交换删除可 O(1)|
|查找指定值|O(n)|O(n)|
|顺序遍历|缓存局部性较差|缓存局部性较好|
|额外内存|节点指针和分配开销|预留容量的开销|

为什么链表插入删除快，但实际游戏开发里经常不如 `TArray`？

因为 CPU 执行时，数据访问成本也非常重要。

链表节点可能散落在不同的内存区域，每次访问下一个节点都需要根据指针寻找数据；`TArray` 的连续内存更容易被 CPU 预取。

因此，除非确实需要频繁的已知节点增删或稳定的节点链接关系，否则不要仅仅因为链表删除是 O(1) 就优先选择链表。

### 6.2 std::deque：分段连续数组

`std::deque`（Double-Ended Queue，双端队列）的典型实现使用分段存储。

```
          索引管理区
        [P0][P1][P2]
         |   |   |
         v   v   v
        [A B][C D][E F]
```

它不是一整块连续内存，而是若干连续小块组成的数据结构。

与 `std::vector` 对比：

|特性|vector|deque|
|---|---|---|
|整体连续存储|是|否|
|随机访问|O(1)|O(1)|
|尾部插入|O(1) 均摊|O(1)|
|头部插入|O(n)|O(1)|
|迭代缓存友好性|通常更好|分段存储|
|提供 `data()` 获取整体连续区|是|否|

UE 的 `TChunkedArray` 也采用分块分配的设计，但其主要目的包括避免申请巨大连续内存时受到碎片化影响，它不应被理解成与 `std::deque` 完全相同。

Epic Developer Community

### 6.3 TQueue 与 std::queue：队列

队列遵循 FIFO（First In First Out，先进先出）。

```
入队 → [任务A][任务B][任务C] → 出队
```

STL 的 `std::queue` 是一个容器适配器，默认使用 `std::deque` 管理数据。

```
std::queue<int> Tasks;

Tasks.push(10);
Tasks.push(20);

int First = Tasks.front();
Tasks.pop();
```

UE 的 `TQueue` 则是基于链表实现的无界队列，提供不同并发模式。Epic 的文档明确介绍了 MPSC 和 SPSC 模式，以及用于多生产者入队的原子 CAS（Compare-And-Swap，比较并交换）操作。

Epic Developer Community

|TQueue 模式|含义|适合场景|
|---|---|---|
|`EQueueMode::Spsc`|单生产者、单消费者|一个工作线程向游戏线程发送数据|
|`EQueueMode::Mpsc`|多生产者、单消费者|多个工作线程向游戏线程发送消息|
|`EQueueMode::SingleThreaded`|单线程|普通单线程队列|

代码示例：

```
#include "Containers/Queue.h"

// 多生产者、单消费者队列
TQueue<int32, EQueueMode::Mpsc> PendingEvents;

// 多个生产者线程可以入队
PendingEvents.Enqueue(101);
PendingEvents.Enqueue(102);

// 只能由一个消费者执行出队
int32 EventId = 0;

while (PendingEvents.Dequeue(EventId))
{
    // 处理事件
    // 例如把异步计算结果交给游戏线程
}
```

需要注意：

- 无锁不等于零开销，链表节点仍涉及内存管理。
    
- MPSC 不代表多个线程能同时安全出队。
    
- 队列无限增长可能造成内存压力。
    
- 不要仅因为数据放进线程安全队列，就认为数据里面的 UObject 可以被任何线程随意访问。
    

对于高频、固定容量的数据流，还可以了解 UE 的 `TCircularQueue`（环形队列）。它使用环形数组思想，能够减少链表式队列中频繁的节点分配开销。

### 6.4 TBitArray：位压缩数组

假设需要保存 10,000 个敌人的某个布尔状态。

普通 `TArray<bool>` 的元素通常至少按字节存储。

而 `TBitArray` 使用一个 bit（位）存储一个布尔值。

理论上仅考虑数据位：

$$
10000\text{ bits}=1250\text{ bytes}
$$

与每个元素使用 1 字节相比，数据部分可以节省约 87.5% 的空间。

```
#include "Containers/BitArray.h"

TBitArray<> VisibilityFlags;

// 初始化 1000 个 false
VisibilityFlags.Init(false, 1000);

// 设置下标 10 的标记
VisibilityFlags[10] = true;

if (VisibilityFlags[10])
{
    // 对象可见
}
```

`TBitArray` 的元素访问使用位引用代理（Bit Reference Proxy），并不是普通 `bool&`。它适用于大量状态标记、位掩码和需要进行位运算的场景。

`std::vector<bool>` 也常采用位压缩特化，但它具有不同于普通 `std::vector<T>` 的引用与迭代语义，使用时需要格外注意。

## 七、UE 容器与 STL 最大的工程区别：内存管理和 GC

这是 UE 客户端面试中非常重要的一部分。

### 7.1 UE 的内存分配体系

普通 STL 容器通过 Allocator（分配器）获取内存。

例如：

```
std::vector<int> Numbers;
```

默认使用 `std::allocator<int>`，通常最终会调用标准库和平台对应的分配机制。

UE 使用自己的内存分配抽象：

```
TArray / TMap / TSet
          |
          v
  容器 Allocator
          |
          v
     FMemory
          |
          v
    GMalloc / FMalloc
          |
          v
  引擎选用的底层分配器
          |
          v
      操作系统
```

这是通用的逻辑调用关系，并非每个容器操作都必须经过图中所有层级。

UE 可以使用不同的底层内存分配方案，例如 Binned 系列分配器。具体配置取决于引擎版本、平台和构建设置。

游戏引擎使用统一内存管理抽象的目的包括性能优化、内存统计、平台适配和问题追踪。

这并不意味着所有 `std::vector` 都不该使用，而是 UE 容器通常更容易和 UE 自身的系统配合。

### 7.2 最容易误解的地方：TArray 不会自动保护 UObject

例如：

```
TArray<UObject*> CachedObjects;
```

不能认为只要对象指针放进了 `TArray`，就一定能防止 GC 回收。

GC 需要能够从根对象追踪到这些引用。

在 UE5 中，推荐对需要长期持有 UObject 的成员采用受反射系统管理的形式：

```
UCLASS()
class UMyObjectManager : public UObject
{
    GENERATED_BODY()

public:
    // GC 可以通过反射系统追踪这些强引用
    UPROPERTY()
    TArray<TObjectPtr<UObject>> CachedObjects;

    UPROPERTY()
    TMap<int32, TObjectPtr<UObject>> ObjectMap;
};
```

只要拥有该属性的对象本身能够被 GC 追踪到，这些被正确反射的强引用就能参与 GC 引用遍历。

Epic Developer Community

但下面的情况不同：

```
class FNativeManager
{
public:
    // 普通 C++ 类型，不属于 UE 的反射属性系统
    std::vector<UObject*> Objects;
};
```

UE 不会仅因为某个 UObject 指针存在于 `std::vector` 中，就自动扫描它并将其作为强引用。

实际上，普通 C++ 类里的未注册 `TArray<UObject*>` 也有类似问题。

如果不使用反射属性，但确实需要在原生结构中建立 GC 可识别的强引用，则需要根据场景通过 `FGCObject`、引用收集接口等机制注册引用关系。

### 7.3 TWeakObjectPtr 和容器

对于锁定目标、感知目标、临时缓存对象等场景，可以使用弱引用。

```
UPROPERTY()
TArray<TWeakObjectPtr<AActor>> TargetActors;
```

它有两个重要特点：

1. 不会因为这个弱引用而保持目标 UObject 存活。
    
2. 当目标对象不再有效时，可以通过 `IsValid()` 检查。
    

```
for (const TWeakObjectPtr<AActor>& Target : TargetActors)
{
    if (AActor* Actor = Target.Get())
    {
        // Actor 当前仍有效
        // 进行游戏逻辑处理
    }
}
```

因此选型时要同时考虑两个问题：

容器负责什么？ 例如查找、排序、增删与内存布局。

指针负责什么？ 例如对象所有权、强弱引用和生命周期。

二者是不同维度，不能混为一谈。

### 7.4 容器自身的生命周期和 UObject 生命周期

例如：

```
TArray<FString> Names;
```

`TArray` 销毁时，会销毁其内部的 `FString` 元素。

```
TArray<UObject*> Objects;
```

`TArray` 销毁时，不会因为销毁了指针槽位，就直接调用 `delete` 删除指针所指的 UObject。

同理：

```
TArray<TSharedPtr<FMyNativeData>> Data;
```

容器销毁时会销毁内部的 `TSharedPtr`，引用计数随之变化。如果引用计数归零，普通 C++ 对象可能被销毁。

因此：

|元素类型|容器释放后的效果|
|---|---|
|`int32`|释放存储|
|`FString`|销毁字符串对象|
|`FMyStruct`|销毁结构体对象|
|`UObject*`|指针元素消失，不直接删除 UObject|
|`TObjectPtr<UObject>`|释放引用，不直接 delete UObject|
|`TWeakObjectPtr<UObject>`|弱引用消失|
|`TSharedPtr<FNativeData>`|共享引用计数减少|

## 八、CPU Cache：为什么 TArray 经常是游戏性能优化的首选？

假设要每帧遍历 10,000 个战斗对象，并更新其状态。

有两种内存布局。

<img width="739" height="430" alt="image" src="https://github.com/user-attachments/assets/83eab251-a388-4478-994c-8cb7d20851d4" />

### 8.1 CPU Cache Line（缓存行）

CPU 不一定每次只从内存读取当前访问的一个变量，通常以 Cache Line 为单位搬运数据。

很多现代 CPU 使用 64 字节缓存行，但具体大小与架构有关。

假设：

```
struct FParticleData
{
    FVector3f Position; // 12 字节
    FVector3f Velocity; // 12 字节
};
```

如果多个结构体紧密排列，那么一次缓存行读取可能包含多个相邻对象的数据。

顺序遍历时可以利用：

- Spatial Locality（空间局部性）：相邻数据更可能被访问。
    
- Temporal Locality（时间局部性）：最近使用过的数据更可能再次被访问。
    
- Hardware Prefetching（硬件预取）：CPU 提前加载后续可能访问的数据。
    

如果使用链表，CPU 需要读取当前节点中的下一个地址，再去访问其他内存位置。频繁的指针追踪可能导致更多缓存未命中。

### 8.2 为什么 TArray 不一定永远最快？

假设：

```
struct FEnemy
{
    float HP;
    FVector Position;
    FString Name;
    TArray<int32> BuffIds;
    // 其他大量字段
};
```

如果每帧只需要更新 HP，却每次都访问庞大的 `FEnemy` 数据对象，可能浪费缓存带宽。

这又涉及 AoS 与 SoA。

|布局|全称|结构|
|---|---|---|
|AoS|Array of Structures（结构体数组）|`TArray<FEnemy>`|
|SoA|Structure of Arrays（数组结构）|分别保存 HP、Position 等字段|

SoA 示例：

```
struct FEnemyRuntimeData
{
    TArray<float> Healths;
    TArray<FVector3f> Positions;
    TArray<FVector3f> Velocities;
};
```

只处理生命值时：

```
for (float& HP : RuntimeData.Healths)
{
    HP -= 1.0f;
}
```

CPU 只需要重点读取连续的 HP 数据。

这在大规模粒子、批量碰撞数据、实体批处理和性能敏感的模拟系统中很有价值。

但 SoA 会提高维护成本，尤其在元素增删、字段同步和对象化逻辑方面。因此一般只在性能分析表明有收益时采用。

## 九、UE 战斗系统中如何选择容器？

下面结合实际游戏客户端开发给出推荐。

|实际场景|推荐容器|原因|
|---|---|---|
|当前角色拥有的技能列表|`TArray`|需要遍历、可能需要顺序|
|根据技能 ID 查找技能数据|`TMap`|Key 快速查询|
|已命中的 Actor 集合|`TSet`|快速去重|
|每帧临时收集少量碰撞目标|`TArray` + `TInlineAllocator`|减少小数组堆分配|
|大量技能实例、频繁创建销毁|`TSparseArray` 或专用对象池|复用槽位、减少搬移|
|需要排序的小型配置表|`TSortedMap`|连续存储、按 Key 排序|
|异步线程向 GameThread 发送消息|`TQueue`|明确的生产者消费者模式|
|大量布尔标记|`TBitArray`|位压缩存储|
|小型、固定数量的元素|`TStaticArray` 或 `std::array`|不需要动态扩容|
|一批只读的连续元素传参|`TArrayView`|避免复制容器数据|

`TArrayView` 特别值得了解，它相当于一个不拥有元素存储的数组视图：

```
void ProcessTargets(TArrayView<const int32> TargetIds)
{
    for (int32 TargetId : TargetIds)
    {
        // 只读取目标 ID
    }
}
```

它非常适合函数之间传递连续数组，但要确保底层数据在视图使用期间仍然有效，并且没有发生使其失效的重分配。

### 9.1 战斗系统：技能配置与运行时对象分离

假设角色拥有几十个技能，但同一时间只有少量技能实例在运行。

可以考虑：

```
UCLASS()
class UBattleSkillManager : public UObject
{
    GENERATED_BODY()

public:
    // 技能配置：按 SkillId 查询
    UPROPERTY()
    TMap<int32, TObjectPtr<UDataAsset>> SkillConfigs;

    // 当前正在执行的技能 ID
    UPROPERTY()
    TArray<int32> ActiveSkillIds;

    // 当前攻击已经命中过的目标 ID
    TSet<int32> HitTargetIds;
};
```

这是简化示例。

如果技能配置引用的是不需要立即加载的资源，更应考虑 `TSoftObjectPtr`、软路径和异步资源加载。

```
UPROPERTY()
TMap<int32, TSoftObjectPtr<UAnimMontage>> SkillMontages;
```

因为 容器选择解决的是数据组织问题，而软引用解决的是资源依赖和加载时机问题。

不要把这两个层面的优化混淆。

### 9.2 战斗系统：为何不应该全部使用 TMap？

假设每帧需要执行：

```
for (auto& Pair : AllEnemies)
{
    UpdateEnemy(Pair.Value);
}
```

如果 `AllEnemies` 是一个 `TMap<int32, FEnemyData>`，仅仅为了遍历所有敌人，其哈希查找能力并没有发挥作用。

如果逻辑既需要高速遍历，又需要根据 ID 快速查找，可以考虑：

```
// 连续存储，适合遍历
TArray<FEnemyData> Enemies;

// ID 映射到数组下标，适合查询
TMap<int32, int32> EnemyIdToIndex;
```

但这需要解决删除时下标发生变化的问题，例如使用 Swap Remove 后同步更新被交换元素的索引，或者设计更稳定的 Handle 系统。

这是典型的混合数据结构优化。

单个容器未必能同时满足高效遍历、稳定对象身份、快速查询和低内存开销。

## 十、UE 容器与 STL 的高频面试题

下面这组问题建议重点掌握。

点击题目展开参考答案

1. TArray 与 std::vector 的底层区别？

都是连续动态数组。TArray 使用 UE 的分配器体系，并采用元素可平凡重定位的设计约定；std::vector 遵循标准 C++ 对象移动、复制和生命周期规则。两者都存在扩容导致元素地址变化的问题。

2. 为什么 vector 和 TArray 的尾部插入是 O(1)？

准确地说是均摊 O(1)。通常不需要扩容时直接构造尾部元素；容量不足时进行几何扩容。单次扩容 O(n)，但多次插入的总搬迁成本仍为 O(n)。

3. TArray::RemoveAt 和 RemoveAtSwap 有什么区别？

RemoveAt 保持元素顺序，需要移动后续元素，通常 O(n)；RemoveAtSwap 不保证顺序，删除单个已知位置且不缩容时可以 O(1)。

4. TMap 和 std::unordered_map 的实现区别？

二者都基于哈希查找，但 UE TMap 以 TSet 和稀疏数组存储元素，增长可能重定位元素；主流 std::unordered_map 是节点式实现，rehash 会使迭代器失效，但不会仅因 rehash 使元素指针或引用失效。

5. 哈希表为什么是平均 O(1)，而不是一定 O(1)？

哈希冲突会让多个 Key 映射到同一个桶。查找时必须进一步比较 Key；当碰撞严重时，查找可能退化为 O(n)。

6. TSet 和 TArray，哪个 Contains 更快？

TSet 平均 O(1)，TArray 为 O(n)。但小数据量下，TArray 的连续内存、低元数据开销以及无需哈希计算等特点，可能使它实际更快。

7. std::map 和 std::unordered_map 怎么选？

需要排序、范围查询、O(log n) 的最坏情况保证时用 std::map；只需要按 Key 快速查找，且不要求顺序时，优先考虑 std::unordered_map。

8. TSparseArray 如何做到 O(1) 删除？

删除时不移动后面的元素，而是释放指定槽位并记录为空闲。后续插入可以复用空洞。不过元素扩容仍可能改变地址，槽位索引也可能被复用。

9. TArray<TObjectPtr<UObject>> 就一定能阻止 GC 吗？

不一定。必须让 GC 的引用收集机制能够追踪到这些引用。例如存储在可达 UObject 上的 UPROPERTY 容器中。普通未注册的原生容器不会仅靠其类型自动成为 GC 强引用来源。

10. 为什么实际开发中常常优先使用 TArray，而不是链表？

TArray 的连续存储具有更好的缓存局部性、较少的节点元数据和更简单的遍历方式。链表虽然已知节点增删为 O(1)，但链式访问的缓存未命中与分配成本可能更高。

11. TQueue 一定是线程安全吗？

不是。需要看 EQueueMode。Spsc 支持一个生产者和一个消费者，Mpsc 支持多个生产者与一个消费者，SingleThreaded 不提供并发安全保证。

12. 什么情况下需要 TInlineAllocator？

当容器的元素数量通常很少，并且创建频繁时，内联空间可减少堆分配。但容器对象本身会变大，超过内联容量后仍然需要额外分配。

## 十一、底层源码应该重点看哪些文件？

如果你正在阅读 UE 源码，建议按照下面的顺序阅读:

<img width="751" height="371" alt="局部截取_20261116_222415" src="https://github.com/user-attachments/assets/b59bc481-a8c7-4fb8-9ed0-87363f8e1a0d" />

特别推荐按下面这条关系去理解：

```
                TArray
                   |
          连续内存、扩容、重定位
                   |
                   v
            TSparseArray
                   |
          有效位图、空洞、复用
                   |
                   v
                 TSet
                   |
           哈希、桶、冲突处理
                   |
                   v
                 TMap
                   |
             Key-Value 映射
```

这是按知识依赖组织的阅读顺序，而不是严格的类继承关系。

### 最终总结

对于 UE 游戏客户端开发，容器的理解可以分成三个层次：

第一层：会使用。 掌握 `TArray`、`TMap`、`TSet` 的常见 API、时间复杂度和使用场景。

第二层：理解底层。 能解释连续数组扩容、元素重定位、哈希冲突、红黑树、稀疏数组、CPU Cache、迭代器失效。

第三层：工程实践。 能在战斗高频逻辑中控制内存分配、避免引用失效、正确处理 GC、设计稳定 Handle，并通过 Profiling（性能采样分析）验证数据结构选型。

在实际面试中，最容易区分基础与进阶水平的追问往往不是“`TMap` 怎么使用”，而是：

“如果你有 10,000 个战斗对象，需要每帧遍历更新、按 ID 查询，并且对象频繁创建销毁，你会怎样设计底层数据结构？如何同时保证查询效率、缓存局部性和对象引用安全？”

能把 `TArray + TMap`、`TSparseArray`、Generation Handle、GC 和 CPU 缓存结合起来解释，就已经进入 UE 客户端容器的工程设计层面了。
