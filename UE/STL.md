UE 容器与 C++ 标准库容器的底层思想基本相同：**用不同的数据布局，交换访问、查找、插入、删除和内存占用的成本。** UE 容器额外考虑了引擎接口、内存分配，以及反射系统的接入。

最先记住三个对应关系：

- `TArray` 对应 `std::vector`。
- `TMap` 更接近 `std::unordered_map`。
- `TSet` 更接近 `std::unordered_set`。

特别注意，`TMap` 并不对应有序的 `std::map`。下面以 UE5 常见接口和 C++17/20 为主讲解；少数接口细节随引擎版本变化。

**选容器之前，先明确数据要怎样使用。** 例如，需要按下标读取、根据 ID 查找、去重、保持排序，还是从两端不断取放数据？这些需求对应不同容器。

|需求或结构|UE 常见选择|C++ 标准库选择|主要特点|
|---|---|---|---|
|动态数组|`TArray<T>`|`std::vector<T>`|元素连续，按下标访问快|
|编译期固定长度数组|`TStaticArray<T, N>`|`std::array<T, N>`|长度固定，元素存放在容器对象内部|
|哈希集合|`TSet<T>`|`std::unordered_set<T>`|默认去重，查找平均很快|
|哈希映射|`TMap<K, V>`|`std::unordered_map<K, V>`|根据键查找值|
|允许重复键的哈希映射|`TMultiMap<K, V>`|`std::unordered_multimap<K, V>`|一个键对应多个元素|
|始终按键排序的映射|按需求选择有序结构|`std::map<K, V>`|有序遍历、范围查询|
|始终排序的集合|按需求选择有序结构|`std::set<T>`|元素有序且唯一|
|双向链表|`TDoubleLinkedList<T>`|`std::list<T>`|已知节点位置时，插入删除快|
|双端队列|根据 UE 版本和用途选择|`std::deque<T>`|两端插入删除快，支持随机访问|
|先进先出队列|`TQueue<T, Mode>` 等|`std::queue<T>`|UE 队列还区分并发模式|
|稀疏索引存储|`TSparseArray<T>`|无直接对应|删除后可以留下空槽|
|动态位数组|`TBitArray<>`|`std::vector<bool>` 等|压缩存储布尔状态，接口并不等价|
|不拥有数据的数组视图|`TArrayView<T>`|`std::span<T>`，C++20|借用已有连续数据|

表中是用途对应关系，不能据此假定两边的失效规则、所有权和接口完全一致。

**`TArray` 和 `std::vector` 应当作为理解动态数组的起点。** 它们把有效元素连续存放，适合顺序遍历、按下标读取、批量计算，以及与需要连续缓冲区的接口交互。

可以把它们理解为：

```
容器元数据：数据地址、元素数量、容量
                     │
                     ▼
连续存储区域：[元素0][元素1][元素2][空闲空间][空闲空间]
```

这里有两个容易混淆的概念：

|概念|UE|STL|含义|
|---|---|---|---|
|有效元素数量|`Num()`|`size()`|已经存在、可以访问的元素数量|
|容量|`Max()`|`capacity()`|当前存储最多可容纳的元素数量|

`Reserve` 只预留容量，不增加有效元素：

```
TArray<int32> Numbers;
Numbers.Reserve(100);

// Numbers.Num() 仍然是 0。
// 此时不能访问 Numbers[0]。

Numbers.Add(10);
Numbers.Emplace(20);

// 现在 Numbers.Num() 是 2。
```

STL 的行为类似：

```
std::vector<int> Numbers;
Numbers.reserve(100);

Numbers.push_back(10);
Numbers.emplace_back(20);
```

需要实际增加元素数量，使用 UE 的 `SetNum()` 或 STL 的 `resize()`；新增元素按各自规则构造或初始化。`reserve()` 本身不创建元素。

常用操作可以这样对照：

|操作|`TArray`|`std::vector`|
|---|---|---|
|尾部添加|`Add(Value)`|`push_back(Value)`|
|尾部原地构造|`Emplace(Args...)`|`emplace_back(Args...)`|
|插入|`Insert(Value, Index)`|`insert(Position, Value)`|
|获取数量|`Num()`|`size()`|
|判断为空|`IsEmpty()`|`empty()`|
|调整数量|`SetNum(N)`|`resize(N)`|
|预留容量|`Reserve(N)`|`reserve(N)`|
|获取数据指针|`GetData()`|`data()`|
|排序|`Sort()`、`Algo::Sort()`|`std::sort()`|
|按值查找|`Find()`、`Contains()` 等|`std::find()` 等|

`Emplace` 可以直接使用构造参数创建元素，但不意味着所有场景都比 `Add` 快。传入已经存在的对象时，仍然可能发生复制或移动；已有容量不足时，扩容成本也仍然存在。

**动态数组快在访问和遍历，代价主要出现在扩容及中间插入删除。**

假设元素比较、移动等单次操作成本视为常数：

|操作|`TArray` / `std::vector` 的典型复杂度|
|---|---|
|按下标访问|O(1)|
|遍历全部元素|O(n)|
|在未排序数组中查找|O(n)|
|尾部添加|均摊 O(1)|
|一次需要搬迁元素的扩容|O(n)|
|在中间插入|O(n)|
|保持顺序地删除中间元素|O(n)|
|已排序数组二分查找|O(log n)|

“均摊 O(1)”意味着大量尾部添加的平均成本较低，**不保证每一次添加都耗时相同**。某一次扩容可能需要搬迁全部已有元素。

UE 提供了很实用的交换删除：

```
TArray<int32> A = {10, 20, 30, 40};
A.RemoveAt(1);
// A: {10, 30, 40}，保持剩余元素的顺序。

TArray<int32> B = {10, 20, 30, 40};
B.RemoveAtSwap(1, 1, EAllowShrinking::No);
// B: {10, 40, 30}，用尾部元素填补空位。
```

如果已经知道下标，只删除一个元素，并且禁止缩容，`RemoveAtSwap` 可以避免移动整段后续元素，适合顺序不重要的运行时列表。但如果调用 `RemoveSwap(Value)`，它还需要先查找值，整体不能直接按 O(1) 理解。

清空时也要区分目的：

- `Reset()`：清除元素，通常保留现有容量，适合反复使用的临时数组。
- 默认分配器下的 `Empty()`：清除元素，并按指定保留量处理存储；不带参数通常用于释放原有动态存储。
- `std::vector::clear()`：清除元素，保留容量。
- `std::vector::shrink_to_fit()`：请求缩减容量，标准不保证一定执行。

保留容量不等于保留元素；需要析构的元素仍会被析构。

**`TMap` 适合“通过键找值”，它的核心是哈希。** 例如通过物品 ID 找数量、通过角色 ID 找角色数据。

可以概念性地理解为：

```
Key → 计算哈希 → 定位桶 → 比较候选键 → 找到 Value
```

哈希并不能保证直接唯一定位元素。不同键可能产生相同哈希，这叫哈希冲突，仍然需要比较键是否相等。

```
TMap<FName, int32> ItemCounts;

const FName PotionKey(TEXT("Potion"));

ItemCounts.Add(PotionKey, 3);
ItemCounts.Add(PotionKey, 5); // 同键覆盖，现在是 5。

if (const int32* Count = ItemCounts.Find(PotionKey))
{
    UE_LOG(LogTemp, Log, TEXT("Count: %d"), *Count);
}

++ItemCounts.FindOrAdd(FName(TEXT("Arrow")));
// 不存在时创建默认值，再加一。

for (const auto& Pair : ItemCounts)
{
    // Pair.Key
    // Pair.Value
}
```

`TMap` 的键唯一，`TMultiMap` 允许重复键。`TMap::Add` 遇到已有键会替换原来的键值对；`TMultiMap::Add` 会保留多个条目。两者都不保证按键排序或按插入顺序遍历。

与 STL 对照时，最容易出错的是下标访问：

|行为|`TMap`|`std::unordered_map`|
|---|---|---|
|查找，不存在时不插入|`Find(Key)`|`find(Key)`|
|判断存在|`Contains(Key)`|`contains(Key)`，C++20|
|不存在时默认插入|`FindOrAdd(Key)`|`operator[]`|
|插入或覆盖|`Add(Key, Value)`|`insert_or_assign(Key, Value)`，C++17|
|仅在不存在时插入|先明确所需 UE 接口语义|`try_emplace()`，C++17|

```
// UE：要求键已经存在，缺失时触发检查。
int32 Count = ItemCounts[PotionKey];

// STL：不存在时会插入一个默认构造的值。
int Count2 = StdItemCounts["Potion"];
```

因此，在 STL 中使用 `Map[Key]` 做“只读查询”，可能悄悄改变容器内容。

哈希容器的查找、插入、删除通常是**平均 O(1)，最坏 O(n)**。键本身的哈希成本也要算进去：处理一个很长的字符串，不等同于处理一个整数。

**`std::map` 和 `std::set` 的优势是维持顺序。** 它们通常使用平衡搜索树实现，常见实现是红黑树，但 C++ 标准没有强制必须采用红黑树。

|比较项|`std::map` / `std::set`|哈希映射 / 哈希集合|
|---|---|---|
|遍历顺序|按比较器排序|无排序保证|
|查找|O(log n)|平均 O(1)，最坏 O(n)|
|单个元素插入|通常 O(log n)|平均 O(1)|
|范围查询|很适合|通常需要扫描或额外索引|
|键的要求|严格弱序比较关系|哈希及相等关系|

例如查询分数在 `[100, 200]` 的全部记录：

```
std::map<int, PlayerData> PlayersByScore;

auto First = PlayersByScore.lower_bound(100);
auto Last  = PlayersByScore.upper_bound(200);

for (auto It = First; It != Last; ++It)
{
    // 处理区间内的数据。
}
```

定位边界加遍历结果的成本是 O(log n + k)，其中 `k` 是结果数量。

还要注意，`std::map` 判断两个键是否等价，依据的是比较器：

```
!Compare(A, B) && !Compare(B, A)
```

它不一定等同于 `A == B`。

UE 也有 `TSortedMap` 等有序结构，但其底层可以是排序数组，不能把它的插入成本和地址稳定性直接套用成 `std::map`。如果数据很少修改、需要频繁有序遍历，排序后的 `TArray` 也值得考虑。

**`TSet` 适合表达“是否存在”，而不是“第几个”。** 典型用途包括去重、已访问集合、已触发目标集合。

```
TSet<int32> VisitedIds;

VisitedIds.Add(1001);
VisitedIds.Add(1002);
VisitedIds.Add(1001);

// 默认不保留重复元素，Num() 是 2。

if (VisitedIds.Contains(1002))
{
    // 已访问。
}

VisitedIds.Remove(1001);
```

默认情况下，`TSet` 使用 `operator==` 判断相等，使用 `GetTypeHash()` 计算哈希。它的元素没有稳定的排序，也不能把遍历位置理解为业务下标。

自定义类型作为 UE 哈希键，可以这样写：

```
struct FGridKey
{
    int32 X = 0;
    int32 Y = 0;

    bool operator==(const FGridKey& Other) const
    {
        return X == Other.X && Y == Other.Y;
    }

    friend uint32 GetTypeHash(const FGridKey& Key)
    {
        return HashCombine(
            ::GetTypeHash(Key.X),
            ::GetTypeHash(Key.Y));
    }
};

TMap<FGridKey, int32> CellCosts;
TSet<FGridKey> VisitedCells;
```

最关键的约束是：

```
A == B  ⇒  Hash(A) == Hash(B)
```

反方向不成立：哈希相同，不代表键相同。

另外，**元素进入容器后，不要原地修改影响哈希或相等判断的字段**。例如直接修改集合中网格键的 `X`，会使元素实际所在的位置与新哈希不一致。需要改变键时，应删除旧元素，再插入新元素。

**链表、双端队列和适配器解决的是另外几类访问模式。**

|STL 类型|主要能力|使用时的关键限制|
|---|---|---|
|`std::array<T, N>`|固定长度连续存储|不能动态增长；不代表一定在栈上|
|`std::deque<T>`|O(1) 随机访问，两端插入删除快|整体不保证连续，不能当作一块连续缓冲区|
|`std::list<T>`|双向链表；已知位置时单节点插入删除 O(1)|不支持 O(1) 下标访问，查找节点仍是 O(n)|
|`std::forward_list<T>`|单向链表|适合只需向前遍历及在已知位置后插删|
|`std::stack<T>`|后进先出|容器适配器，通常基于 `deque`|
|`std::queue<T>`|先进先出|容器适配器，通常基于 `deque`|
|`std::priority_queue<T>`|优先级最高的元素先出|默认堆顶为最大值；压入弹出 O(log n)，取堆顶 O(1)|

“链表插入删除快”的前提是**已经知道节点位置**。如果每次都要先遍历查找，总成本仍然可能是 O(n)。链表还会增加节点指针、分配和缓存访问成本，所以大量顺序遍历的数据，动态数组经常更合适。

**指针、引用和迭代器失效，是容器使用中最需要认真掌握的规则。**

三者分别可以理解为：

- 指针：指向某个元素的地址。
- 引用：某个元素的别名。
- 迭代器：用于遍历、定位元素的对象，不一定就是裸指针。

以下总结常见插入、删除场景：

|容器|常见失效规则|
|---|---|
|`TArray` / `std::vector`|扩容搬迁时，原元素的指针、引用和迭代器失效；插删引起元素移动时，也会影响对应位置|
|`TMap` / `TSet`|添加可能引起元素存储重新分配；不要假定元素地址具有 STL 节点容器的稳定性|
|`std::map` / `std::set`|插入不使已有元素迭代器和引用失效；删除通常只影响被删元素|
|`std::unordered_map` / `std::unordered_set`|rehash 使迭代器失效，但不使已有元素的指针和引用失效；删除影响被删元素|
|`std::list`|插入不使已有元素迭代器和引用失效；删除影响被删元素|
|`std::deque`|两端插入使迭代器失效，但已有元素引用保持有效；中间插入影响更广|

这也是 `TMap` 和 `std::unordered_map` 不能简单互换的重要原因之一

例如：

```
TArray<int32> Values = {10, 20, 30};

int32* First = &Values[0];

Values.Add(40); // 可能扩容。

// 不能再假定 First 仍然有效。
// 更稳妥的做法是在修改完成后重新取得地址。
```

即使地址没有改变，排序或交换删除后，那个地址上的元素也可能换成了另一个业务对象。**地址稳定、下标稳定、对象身份稳定，是三个不同的问题。**

普通范围 `for` 遍历期间，也不要随意增加或删除当前容器的元素。批量过滤可以使用：

```
// UE：保留不满足条件的元素。
Values.RemoveAll([](int32 Value)
{
    return Value < 0;
});

// STL，C++20。
std::erase_if(StdValues, [](int Value)
{
    return Value < 0;
});

// STL，C++17 常见写法。
StdValues.erase(
    std::remove_if(
        StdValues.begin(), StdValues.end(),
        [](int Value) { return Value < 0; }),
    StdValues.end());
```

`std::remove_if` 本身只重排保留元素并返回新的逻辑结尾，不会改变 `vector::size()`；最后的 `erase` 才真正缩短容器。

**容器管理的是元素；元素指向的对象，由元素类型决定如何管理。**

```
TArray<FString> Names;             // 管理 FString 元素。
TArray<FMyData*> RawPointers;      // 管理指针值，不自动 delete 目标。
TArray<TUniquePtr<FMyData>> Owned;  // 智能指针负责释放目标。
```

复制一个值容器，通常会复制其中的元素；但复制裸指针，只会复制地址：

```
TArray<FMyData*> A;
TArray<FMyData*> B = A;

// A、B 是两个容器。
// 其中对应的指针仍然指向同一个 FMyData。
```

不能笼统地说“复制容器就是深拷贝”。应当继续追问：**元素的复制语义是什么？**

STL 同理。`std::vector<std::unique_ptr<T>>` 不支持普通复制，因为 `unique_ptr` 表达独占所有权；它可以移动。

如果通过智能指针间接存放对象，动态数组扩容通常搬迁的是智能指针元素，目标对象可以留在原地址，但目标对象仍可能因删除元素、重置指针或容器析构而销毁。

**UE 特有的重点，是把容器存储与 `UObject` 的 GC 分开理解。**

下面这行代码本身，不会让 GC 自动追踪数组里的对象：

```
TArray<UObject*> Objects;
```

仅仅把元素改成 `TObjectPtr`，也不等于任何位置上的引用都自动成为 GC 强引用：

```
TArray<TObjectPtr<UObject>> Objects;
```

典型做法是在可达的 `UCLASS` 对象中声明被反射系统追踪的属性：

```
// 以下是 UCLASS 中的成员声明。

UPROPERTY()
TArray<TObjectPtr<UObject>> StrongObjects;

UPROPERTY()
TArray<TWeakObjectPtr<AActor>> ObservedActors;
```

这两种元素表达不同语义：

|元素类型|主要语义|
|---|---|
|被 GC 追踪的 `TObjectPtr<T>`|强引用，参与对象可达性追踪|
|`TWeakObjectPtr<T>`|弱引用，不阻止目标被回收，使用前检查有效性|
|`TSoftObjectPtr<T>`|通过资源路径引用，适合按需加载，本身不保持已加载对象存活|
|`TStrongObjectPtr<T>`|可用于普通 C++ 持有者中的强引用，不能声明为 `UPROPERTY`|

如果外层是 `USTRUCT`，还需要保证它本身位于 GC 能追踪到的属性链中。普通 C++ 类中的 `UPROPERTY` 注解不能凭空建立这条链。

此外，强引用不能阻止 Actor 被显式 `Destroy()`，使用 Actor 引用时仍应考虑 `IsValid()`。不要使用 `TSharedPtr<UObject>` 或 `std::shared_ptr<UObject>` 接管 UObject 的销毁。

`std::vector<UObject*>` 同样不会自动进入 UE 的 GC 引用图。确有需要时，可以通过 `FGCObject::AddReferencedObjects` 等机制手动接入，而不是认为 STL 容器完全不能保存 UObject 指针。

**反射支持是 UE 容器的重要优势，但它有明确边界。**

对于受支持的元素类型，`TArray`、`TMap`、`TSet` 可以接入 `UPROPERTY`，进一步参与编辑器、蓝图、属性序列化等系统。`std::vector`、`std::map` 等不能直接替代这些反射属性。

同时需要区分：

- `UPROPERTY()`：声明反射属性。
- `EditAnywhere`：控制编辑器可编辑性。
- `BlueprintReadWrite`：控制蓝图读写权限。
- 网络复制：还需要复制声明、注册，并确认容器及元素在目标版本中的支持情况。

这几种能力不是一个开关。

另一个常见限制是直接嵌套反射容器：

```
// 普通 C++ 使用可以。
TMap<FName, TArray<int32>> Groups;

// 直接作为 UPROPERTY 通常不受支持。
// UPROPERTY()
// TMap<FName, TArray<int32>> Groups;
```

常用解决方式是包一层 `USTRUCT`：

```
USTRUCT(BlueprintType)
struct FIntGroup
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<int32> Values;
};

// 在 UCLASS 中：
UPROPERTY(EditAnywhere, BlueprintReadWrite)
TMap<FName, FIntGroup> Groups;
```

限制来自反射工具和相关系统，并不是 C++ 模板不能嵌套。

**深入性能优化时，还需要理解分配器、稀疏存储和视图。**

`TInlineAllocator` 可以把少量元素的存储直接放进容器对象内部：

```
TArray<FVector, TInlineAllocator<8>> Points;
```

在内联容量以内，元素存储通常不需要额外堆分配；超过容量后可使用后备分配器。它适合数量通常很少的临时数组，但也会增加容器对象本身的大小。

“内联”不意味着“一定在栈上”：容器对象在哪里，其内联存储就在哪里。此外，非默认分配器的数组一般不能直接作为 `UPROPERTY` 使用。

`TSparseArray` 则允许有效索引之间存在空槽：

```
索引：  0     1     2     3
内容：[ A ][空槽][ C ][ D ]
```

普通删除可以避免移动后续元素，因此其他元素索引可以保留。但空槽仍占空间，旧索引可能被新元素复用，扩容或压缩也会影响稳定性。它并不是自动解决悬空引用的“永久 ID 系统”。

`TArrayView` 和 `std::span` 则是借用已有数据：

```
void ProcessPoints(TArrayView<const FVector> Points);
```

这样可以接收不同分配器的数组，而不必复制数据。视图不拥有元素，也不延长底层数据生命周期；底层数组销毁或重新分配后，视图会失效。

只读元素应使用 `TArrayView<const T>`；`const TArrayView<T>` 并不意味着元素不可修改。

还有一个高级差异：UE 文档明确指出，`TArray` 等容器对元素存在可按字节重定位的要求。**支持 C++ 移动构造，不等于一定适合直接放进所有 UE 容器。** 对于带自指针、依赖自身地址或某些标准库内部布局的类型，应核对目标引擎版本的重定位要求。

**多线程安全需要单独设计，不能根据容器名字判断。**

`TArray`、`TMap`、`TSet` 以及常见 STL 容器，都不能随意由多个线程同时增删。只读并发通常可行，但要保证没有线程修改相关数据，且元素自身的访问也安全。

UE 的部分队列明确提供并发模式，例如：

```
TQueue<FMyMessage, EQueueMode::Mpsc> Messages;
```

- SPSC：单生产者、单消费者。
- MPSC：多生产者、单消费者。
- 单线程模式：不提供相应并发保证。

MPSC 不能直接当作多生产者、多消费者队列。队列操作安全，也不意味着消息里保存的 UObject 可以在任意线程访问。

**落实到游戏开发，可以按访问模式做第一轮选择，再测量真实成本。**

|实际场景|优先考虑|原因|
|---|---|---|
|每帧遍历大量位置、速度、状态|`TArray`|连续访问，利于缓存|
|根据物品 ID 查询配置或数量|`TMap`|按键查找|
|记录已经命中的目标 ID|`TSet`|去重及存在性判断|
|固定数量的装备槽位|固定数组，或按反射需求选 `TArray`|明确表达数量约束|
|保持排序并做范围查询|`std::map` 或合适的有序结构|顺序及区间访问|
|数据很少修改、频繁查询和遍历|排序数组|可二分查找，存储紧凑|
|不关心顺序的运行时对象列表|`TArray` 配合交换删除|避免移动整段后续元素|
|跨线程传递任务结果|匹配并发模型的队列|明确生产者、消费者数量|
|需要编辑器、蓝图访问的属性|支持反射的 UE 容器|能接入引擎属性系统|
|独立算法库、第三方库内部|按需求使用 STL|保持标准接口及可移植性|

实际比较性能时，不要只比较 O(1) 和 O(n)。小数组的顺序查找可能比哈希查找更快，因为它省去了哈希计算、额外存储和间接访问；分界点取决于元素数量、键类型、修改频率和硬件。UE 代码中优先匹配引擎接口，独立 C++ 模块中按标准库需求选型，再用真实数据验证。