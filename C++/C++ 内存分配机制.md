# C++ 内存分配机制详解（原理 + 源码 + UE 游戏开发 + 面试）

C++ 内存管理是游戏客户端开发中非常重要的知识点，尤其是 UE5、游戏引擎底层、战斗系统、网络同步等高性能场景。

理解 C++ 内存分配，不能只停留在“栈上分配快，堆上分配慢”，还需要理解：

1. C++ 内存区域：栈、堆、全局区、静态区、常量区分别是什么。
    
2. 内存分配方式：`malloc/free`、`new/delete`、`new[]/delete[]` 的实现原理。
    
3. 对象生命周期：内存分配、对象构造、析构、内存释放之间的关系。
    
4. 底层分配机制：内存分配器如何向操作系统申请内存。
    
5. 内存对齐：为什么结构体存在内存填充。
    
6. 内存碎片：内部碎片、外部碎片如何产生。
    
7. 内存池：为什么游戏引擎大量使用内存池。
    
8. 智能指针与 RAII：如何避免内存泄漏。
    
9. UE5 内存管理：`FMemory`、`FMalloc`、`UObject` GC 的关系。
    
10. 高频面试题：内存分配机制的常见考点。

# 一、C++ 内存区域

首先，需要区分两个概念：

- 内存区域：程序运行时内存通常如何组织。
    
- 存储期：C++ 语言规定一个对象的存储空间能够存在多久。

平时提到的栈、堆、全局区等，主要是典型操作系统和编译器实现下的内存布局。

## 1.1 典型的进程虚拟内存布局

![[局部截取_20261026_235501.png]]

图中为常见概念模型，实际地址排列、栈与堆的增长方向、ASLR 布局均由平台与实现决定。

|内存区域|主要用途|生命周期|
|---|---|---|
|栈 Stack|函数调用帧、典型局部变量|随作用域或调用结束|
|堆 Heap|动态申请的内存|显式释放或所有者管理|
|Data|已初始化的全局、静态变量|通常整个程序运行期间|
|BSS|零初始化的全局、静态变量|通常整个程序运行期间|
|只读数据区|字符串字面量等|通常整个程序运行期间|
|Text|程序机器指令|程序映射期间|

注意：C++ 标准没有要求编译器必须按照上图组织内存。例如局部变量可能被优化到寄存器中，某些常量可能被合并。

## 1.2 栈内存（Stack）

```
void TestStack()
{
    int A = 10;          // 自动存储期变量，通常放在栈上
    float B = 3.14f;

    int Array[100];      // 固定大小的局部数组，通常放在栈上

} // 离开作用域，局部对象的生命周期结束
```

栈的特点：

- 分配速度通常非常快。
    
- 函数返回时，栈帧通常整体回收。
    
- 由编译器生成对应的管理代码。
    
- 线程一般具有独立的调用栈。
    
- 栈空间通常有大小限制。

例如：

```
void Test()
{
    int A[10000000];  // 大约 40 MB

    A[0] = 1;
}
```

这样的局部数组可能造成栈溢出（Stack Overflow）。

### 为什么栈分配快？

栈通常只需要调整栈指针。

下面是简化示意：

```
sub rsp, 64    ; 为局部数据预留栈空间
...
add rsp, 64    ; 恢复栈指针
```

实际编译器还会考虑寄存器保存、栈对齐、优化和调用约定等。

重点：栈快，是因为其典型分配模式简单，而不是因为栈使用了一种比堆更快的物理内存。

## 1.3 堆内存（Heap）

```
void TestHeap()
{
    int* A = new int(10);

    // A 本身通常是栈上的局部指针变量
    // *A 指向的 int 对象具有动态存储期

    delete A;
}
```

堆的特点：

- 可以按运行时需要动态申请。
    
- 生命周期不依赖创建它的函数作用域。
    
- 支持不同大小的分配和释放。
    
- 需要分配器管理可用内存。
    
- 可能产生内存碎片。

堆内存并不一定每次申请都要进入操作系统内核。

例如：

```
int* A = new int(10);
```

典型情况下：

```
new
 ↓
operator new
 ↓
C++ / C 运行时内存分配器
 ↓
检查已有空闲内存块
 ├── 有合适内存：直接分配
 └── 没有合适内存：向操作系统申请更多内存
```

所以不能简单认为 `new` 每次都执行系统调用。

## 1.4 全局变量与静态变量

```
int GlobalA = 100;   // 静态存储期，通常放在 Data 段
int GlobalB;         // 零初始化，通常放在 BSS 段

void Test()
{
    static int Count = 0;

    Count++;
}
```

`Count` 与一般局部变量不同。

虽然它在函数内部声明，但是：

- 不会随着函数退出而销毁。
    
- 只进行一次初始化。
    
- 通常持续存在到程序结束。

这里有个重要细节：

`static` 局部变量具有静态存储期，但这不意味着所有静态变量都在编译期完成初始化。

例如：

```
int GetValue();

void Test()
{
    static int Value = GetValue();
}
```

这里的 `Value` 通常在程序第一次执行到声明位置时初始化。

C++11 起，标准保证函数局部静态变量的初始化具有线程安全性。

## 1.5 四种存储期

从 C++ 语言层面看，分类如下：

|存储期|示例|
|---|---|
|自动存储期|普通局部变量|
|静态存储期|全局变量、static 变量|
|线程存储期|thread_local 变量|
|动态存储期|new 创建的对象|

例如：

```
thread_local int ThreadCounter = 0;
```

每个线程拥有自己的 `ThreadCounter` 实例。

这一点在游戏引擎的线程局部缓存、性能分析器等场景中很常见。

# 二、malloc/free 与 new/delete 的区别

这是 C++ 面试最常见的问题之一。

## 2.1 malloc/free

`malloc` 来自 C 语言标准库。

```
#include <cstdlib>

void Test()
{
    // 申请能够容纳 10 个 int 的原始内存
    int* Data = static_cast<int*>(
        std::malloc(sizeof(int) * 10)
    );

    if (Data == nullptr)
    {
        return;
    }

    // 对于 int 这样的隐式生命周期类型，
    // 在满足相关语言规则时可以通过这种存储使用对象
    Data[0] = 100;

    std::free(Data);
}
```

`malloc` 的作用是申请内存空间。

它不会调用 C++ 类的构造函数。

`free` 用于释放相应的分配，不会调用 C++ 类的析构函数。

## 2.2 new/delete

```
class Player
{
public:

    Player()
    {
        // 构造函数
    }

    ~Player()
    {
        // 析构函数
    }
};

void Test()
{
    Player* P = new Player();

    delete P;
}
```

`new Player()` 通常包含两个核心步骤：

1. 申请能够容纳 `Player` 的内存。
    
2. 在这块内存上构造 `Player` 对象。

`delete P` 则会：

1. 调用对象析构函数。
    
2. 释放对应内存。

## 2.3 两者完整对比

|对比项|malloc/free|new/delete|
|---|---|---|
|所属体系|C 标准库|C++ 语言运算符|
|申请单位|字节大小|对象类型|
|构造函数|不调用|调用|
|析构函数|不调用|调用|
|返回值|void*|对应类型指针|
|类型转换|C++ 中通常需要|不需要|
|失败行为|返回 nullptr|普通 new 抛出异常|
|自定义行为|通常使用分配器包装|可重载 operator new/delete|
|数组支持|计算总字节数|new[] / delete[]|

例如：

```
Player* P1 = static_cast<Player*>(
    std::malloc(sizeof(Player))
);

// P1 得到的是一块内存的地址
// malloc 没有调用 Player 的构造函数
// 因此不能简单将其视为已经完成正常构造的 Player 对象

std::free(P1);
```

对于具有非平凡构造函数的类，直接使用 `malloc` 并不会完成所需的对象构造。

### 为什么 new 通常比 malloc 更适合 C++ 对象？

因为 C++ 不只是管理内存，还需要管理对象的生命周期。

对象内部可能存在：

- `std::string`
    
- `std::vector`
    
- 智能指针
    
- 互斥锁
    
- 文件句柄
    
- 网络连接
    
- GPU 资源句柄

这些成员通常依赖构造和析构建立、维护和清理状态。

# 三、new/delete 底层原理

这一部分非常重要。

要区分：

- `new`：C++ 表达式。
    
- `operator new`：负责分配内存的函数。
    
- 构造函数：负责初始化对象。
    
- `delete`：C++ 表达式。
    
- `operator delete`：负责回收内存的函数。

## 3.1 new 的执行流程

```
Player* P = new Player();
```

概念上可以拆成：

```
new Player()
     │
     ▼
计算 sizeof(Player)
     │
     ▼
调用合适的 operator new
     │
     ▼
取得满足大小及对齐要求的内存
     │
     ▼
调用 Player 构造函数
     │
     ▼
返回 Player*
```

例如下面是概念性伪代码：

```
void* Memory = operator new(sizeof(Player));

// 概念上的下一步：
// 在 Memory 指向的存储空间中构造 Player 对象

Player* P = new (Memory) Player();
```

上述代码展示了底层原理，但并不等同于普通 `new` 表达式的所有细节，特别是构造失败时的异常处理。

## 3.2 delete 的执行流程

```
delete P;
```

概念上：

```
delete P
    │
    ▼
检查空指针
    │
    ▼
调用 Player 析构函数
    │
    ▼
调用匹配的 operator delete
    │
    ▼
归还内存给分配器
```

例如，理解层面可以近似看成：

```
P->~Player();

operator delete(P);
```

不过实际的 `delete` 表达式还涉及析构函数选择、类专用释放函数、数组信息、对齐和大小信息等，不能机械地用这两行替代所有情况。

## 3.3 重载 operator new

C++ 允许为某个类提供专用的分配逻辑。

```
#include <cstddef>
#include <new>
#include <cstdio>

class Enemy
{
public:

    int HP = 100;

    // 自定义对象内存分配
    static void* operator new(std::size_t Size)
    {
        std::printf("Enemy 分配内存：%zu\n", Size);

        // 调用全局分配函数
        return ::operator new(Size);
    }

    // 自定义对象内存释放
    static void operator delete(void* Ptr) noexcept
    {
        std::printf("Enemy 释放内存\n");

        ::operator delete(Ptr);
    }

    Enemy()
    {
        std::printf("Enemy 构造函数\n");
    }

    ~Enemy()
    {
        std::printf("Enemy 析构函数\n");
    }
};

int main()
{
    Enemy* E = new Enemy();

    delete E;
}
```

典型输出：

```
Enemy 分配内存：4
Enemy 构造函数
Enemy 析构函数
Enemy 释放内存
```

假设 `Enemy` 仅有一个 4 字节的 `int` 成员，并且该平台的 `int` 为 4 字节。

由此可以看出：

内存分配和对象构造是两个不同的过程。

# 四、new[] 与 delete[] 的实现原理

数组分配比单对象分配复杂一些。

```
Enemy* Enemies = new Enemy[5];

delete[] Enemies;
```

通常需要进行：

```
new Enemy[5]
       │
       ▼
申请数组所需内存
       │
       ▼
连续构造 5 个 Enemy
       │
       ▼
返回第一个元素地址
```

释放时：

```
delete[] Enemies
       │
       ▼
依次析构数组元素
       │
       ▼
释放数组存储空间
```

析构顺序与构造顺序相反。

## 4.1 Array Cookie 是什么？

编译器有时需要记录数组元素数量，以便 `delete[]` 时知道应该调用多少次析构函数。

一种常见实现是：

![[局部截取_20261026_235544.png]]

有些 ABI 会在数组内存前面保存一个 Cookie：

```
[元素个数][额外填充][对象0][对象1][对象2]...
```

这样 `delete[]` 可以知道需要析构多少个对象。

但是要注意：

Array Cookie 不是 C++ 标准强制要求的实现。

当元素析构是平凡的，或者编译器有其他方式取得数量时，可能不需要这种 Cookie。

## 4.2 为什么不能混用 new[] 和 delete？

错误示例：

```
Enemy* Enemies = new Enemy[5];

delete Enemies;  // 未定义行为
```

原因是单对象删除与数组删除的语义不同。

例如：

- 需要析构的对象数量不同。
    
- 释放函数的匹配方式可能不同。
    
- 数组可能存在额外的管理信息。

正确：

```
Enemy* Enemies = new Enemy[5];

delete[] Enemies;
```

还要特别注意：

```
int* Data = new int[10];

delete Data;  // 即使 int 没有非平凡析构函数，也仍然是未定义行为
```

不要因为简单类型在测试中没有崩溃，就认为这种写法安全。

# 五、Placement New：在指定内存中构造对象

这是理解 C++ 内存池的核心知识。

普通 `new`：

```
Enemy* E = new Enemy();
```

负责申请内存并构造对象。

而 Placement New 可以在已经存在的内存空间上构造对象。

```
#include <new>
#include <cstddef>

class Enemy
{
public:
    int HP;

    Enemy(int InHP)
        : HP(InHP)
    {
    }

    ~Enemy()
    {
    }
};

void Test()
{
    // 申请原始内存
    void* Memory = ::operator new(sizeof(Enemy));

    // 在指定的内存地址构造 Enemy
    Enemy* E = new (Memory) Enemy(100);

    // 手动调用析构函数
    E->~Enemy();

    // 释放原始内存
    ::operator delete(Memory);
}
```

调用过程：

```
申请内存
  ↓
获得地址 0x1000
  ↓
Placement New
  ↓
在 0x1000 执行构造函数
  ↓
使用 Enemy 对象
  ↓
调用析构函数
  ↓
释放内存
```

Placement New 本身通常不申请新的存储空间。

它只是利用指定的存储空间完成对象构造。

因此可以把内存管理拆成两个独立模块：

内存分配器：负责提供一块存储空间。

对象构造逻辑：负责在存储空间中创建对象。

这也是自定义内存池的基础。

在 C++20 中，也可以使用 `std::construct_at` 和 `std::destroy_at` 来显式管理对象的构造与销毁。

# 六、C++ 内存对齐

内存对齐是底层开发中非常常见的问题。

例如：

```
struct Test
{
    char A;
    int B;
    char C;
};
```

可能有人会认为：

```
char = 1 字节
int  = 4 字节
char = 1 字节

总大小 = 6 字节
```

但在常见 ABI 下：

```
sizeof(Test) == 12
```

原因是内存对齐和填充（Padding）。

## 6.1 对齐示意

![[局部截取_20261026_235608.png]]

其典型分布如下：

|偏移|内容|
|---|---|
|0|char A|
|1~3|填充|
|4~7|int B|
|8|char C|
|9~11|尾部填充|

填充是为了满足成员对齐要求，并保证数组中每个结构体元素都能够正确对齐。

## 6.2 如何减少结构体大小？

改变成员排列：

```
struct TestOptimized
{
    int B;
    char A;
    char C;
};
```

在常见平台下：

```
sizeof(TestOptimized) == 8
```

因为两个 `char` 可以排列在 `int` 之后。

对于百万级的游戏对象，这种优化可能明显减少内存占用。

例如一百万个对象：

|单对象大小|总占用|
|---|---|
|12 字节|约 12 MB|
|8 字节|约 8 MB|

仅修改成员排列就可能节约约 4 MB。

不过结构体成员顺序还可能受到序列化兼容性、二进制 ABI 和缓存访问模式等因素限制。

## 6.3 alignas

C++ 提供显式对齐要求：

```
struct alignas(16) VectorData
{
    float X;
    float Y;
    float Z;
    float W;
};
```

查看对齐：

```
sizeof(VectorData);
alignof(VectorData);
```

常见结果：

```
sizeof(VectorData) = 16
alignof(VectorData) = 16
```

C++17 起，普通 `new` 对超对齐类型（over-aligned type）还可以选择支持对齐的 `operator new` 重载。

这在 SIMD、数学库和引擎底层中非常重要。

# 七、底层内存分配器如何工作？

前面已经知道：

```
Enemy* E = new Enemy();
```

通常需要经过某种内存分配器。

但操作系统并不是直接为每一个 C++ 对象建立独立内存区域。

## 7.1 分配器的典型工作过程

```
程序申请 64 字节
       │
       ▼
内存分配器
       │
       ▼
查找空闲块
       │
       ├── 找到合适的块
       │       │
       │       ▼
       │    返回地址
       │
       └── 没找到
               │
               ▼
        获取更多虚拟内存
               │
               ▼
        拆分或管理新内存
               │
               ▼
            返回地址
```

例如一个分配器预先管理 4 KB 内存块：

```
4096 字节内存块
    ↓
划分成 64 个 64 字节槽位
    ↓
每个槽位可以分配给一个小对象
```

这就是很多小对象分配器的基本思路之一。

实际分配器还要解决：

- 不同大小的分配请求。
    
- 多线程竞争。
    
- 内存对齐。
    
- 空闲块合并。
    
- 内存碎片。
    
- 内存归还操作系统。
    
- 分配失败处理。

## 7.2 malloc 底层会调用什么？

这依赖操作系统及使用的内存分配器。

以部分 Linux 分配器实现为例，可以通过：

- `brk` / `sbrk` 相关机制扩展堆区域。
    
- `mmap` 创建新的虚拟内存映射。

Windows 下，分配器可能使用：

- Windows Heap API。
    
- `VirtualAlloc` 等虚拟内存 API。

因此下面这种理解并不准确：

```
malloc → 一定调用 mmap
```

也不准确：

```
new → 一定调用 malloc
```

标准并没有要求全局 `operator new` 必须通过 `malloc` 实现。

实际过程取决于运行时库、自定义分配器以及引擎的内存管理系统。

# 八、内存碎片详解

内存碎片是游戏长期运行时值得关注的问题。

## 8.1 外部碎片

假设分配器管理一段连续内存。

![[局部截取_20261026_235631.png]]

假设剩余两个独立空闲块：

```
空闲块1 = 128 字节
空闲块2 = 128 字节
```

总空闲空间为 256 字节。

但是现在申请：

```
malloc(200);
```

如果分配器无法找到连续的 200 字节可用空间，可能需要扩展内存区域，即使现有空闲空间总量已经超过 200 字节。

这就是外部碎片。

## 8.2 内部碎片

假设分配器按 64 字节的槽位管理内存。

```
malloc(50);
```

分配器可能实际占用：

```
64 字节
```

其中：

```
50 字节 → 用户请求的数据空间
14 字节 → 该槽位未使用的空间
```

这叫内部碎片。

## 8.3 游戏中为什么容易出现内存碎片？

例如战斗系统反复进行：

```
生成怪物
  ↓
创建技能
  ↓
分配伤害事件对象
  ↓
生成子弹
  ↓
删除怪物
  ↓
删除技能
  ↓
删除子弹
  ↓
重复
```

不同对象：

- 大小不同。
    
- 生命周期不同。
    
- 分配时间不同。
    
- 释放顺序不同。

容易给通用分配器带来压力。

所以游戏引擎经常结合对象池、内存池、帧分配器和固定大小分配器来优化。

# 九、内存池（Memory Pool）

对于游戏客户端开发，这是最值得深入理解的部分之一。

传统动态分配：

```
for (int i = 0; i < 1000; i++)
{
    Enemy* E = new Enemy();
}
```

可能涉及多次分配器调用。

如果这些对象都很小，并且需要频繁创建销毁，那么通用分配器未必是最合适的方案。

## 9.1 内存池原理

预先申请一大块内存，然后划分为若干固定大小的槽位。

```
预先申请一大块内存
          │
          ▼
   ┌──────┬──────┬──────┬──────┐
   │ Slot │ Slot │ Slot │ Slot │
   └──────┴──────┴──────┴──────┘
          │
          ▼
  将空闲槽位组织成 Free List
          │
          ▼
  创建对象时取一个空闲槽位
          │
          ▼
  使用 Placement New 构造对象
          │
          ▼
  销毁对象时析构并归还槽位
```

## 9.2 简化版固定大小内存池

下面的示例展示核心机制，使用固定容量，避免扩容带来的复杂性。

```

#include <cstddef>
#include <new>
#include <utility>
#include <type_traits>
#include <cassert>

// 固定容量的单类型对象池
// 为了突出分配原理，这里不处理多线程同步
template<typename T, std::size_t Capacity>
class ObjectPool
{
private:

    // 每个槽位既可以存放对象，也可以存放空闲链表指针
    union Slot
    {
        alignas(T) std::byte Storage[sizeof(T)];

        Slot* Next;

        Slot() {}
        ~Slot() {}
    };

    // 固定数量的槽位
    Slot Slots[Capacity];

    // 空闲链表头
    Slot* FreeHead = nullptr;

public:

    ObjectPool()
    {
        static_assert(Capacity > 0);

        // 将所有槽位串成空闲链表
        for (std::size_t i = 0; i + 1 < Capacity; ++i)
        {
            Slots[i].Next = &Slots[i + 1];
        }

        Slots[Capacity - 1].Next = nullptr;

        FreeHead = &Slots[0];
    }

    // 创建对象
    template<typename... Args>
    T* Create(Args&&... args)
    {
        if (FreeHead == nullptr)
        {
            return nullptr; // 内存池已满
        }

        // 取出一个空闲槽位
        Slot* Current = FreeHead;
        FreeHead = Current->Next;

        try
        {
            // 在指定存储地址上构造对象
            return new (Current->Storage)
                T(std::forward<Args>(args)...);
        }
        catch (...)
        {
            // 如果构造函数抛出异常，归还槽位
            Current->Next = FreeHead;
            FreeHead = Current;
            throw;
        }
    }

    // 销毁对象
    void Destroy(T* Object)
    {
        if (Object == nullptr)
        {
            return;
        }

        // 调用析构函数
        Object->~T();

        // 本例要求传入的指针必须来自此内存池，
        // 并且每个对象只能销毁一次
        Slot* Current =
            reinterpret_cast<Slot*>(Object);

        // 归还槽位到空闲链表
        Current->Next = FreeHead;
        FreeHead = Current;
    }
};
```

使用方法：

```
struct Enemy
{
    int HP;

    explicit Enemy(int InHP)
        : HP(InHP)
    {
    }
};

int main()
{
    ObjectPool<Enemy, 1000> Pool;

    // 从空闲链表中取出槽位
    Enemy* A = Pool.Create(100);
    Enemy* B = Pool.Create(200);

    // 归还槽位
    Pool.Destroy(A);
    Pool.Destroy(B);
}
```

这里使用 `union` 让同一块槽位空间在空闲时保存链表指针，在对象存活时保存对象。

该示例仅是教学版本，生产环境还需要考虑内存池析构时残留的活动对象、指针所有权验证、并发安全和内存统计等问题。尤其要注意，池销毁之前必须销毁所有存活对象。

## 9.3 为什么内存池快？

假设空闲链表已经初始化完成。

一次分配的核心逻辑：

```
Slot* Current = FreeHead;

FreeHead = Current->Next;
```

一次归还的核心逻辑：

```
Current->Next = FreeHead;

FreeHead = Current;
```

这部分操作通常具有 O(1) 的时间复杂度。

相对于通用分配器，它可能减少：

- 内存块搜索。
    
- 锁竞争。
    
- 分配元数据维护。
    
- 大量不同大小请求带来的碎片。

不过它仍然有容量限制、空闲槽位占用和对象构造析构成本，不代表在所有情况下都比成熟的通用分配器快。

# 十、几种常用的游戏内存分配器

游戏引擎往往不会只采用一种分配器。

|分配器|基本原理|典型用途|
|---|---|---|
|Linear Allocator|指针线性递增|一次性加载、临时数据|
|Stack Allocator|按后进先出释放|嵌套临时分配|
|Pool Allocator|固定大小槽位复用|固定大小的小对象|
|Free List Allocator|空闲块链表|可变大小动态分配|
|Buddy Allocator|2 的幂次块拆分合并|页、资源块管理|
|Arena Allocator|按区域批量管理和回收|关卡、帧、任务生命周期数据|

## 10.1 Linear Allocator：线性分配器

最简单的分配器之一。

```
[已使用内存][已使用内存][剩余空间......]
                         ↑
                       Offset
```

申请一块内存时，通常直接增加 Offset。

```
void* Allocate(std::size_t Size)
{
    if (Offset > Capacity ||
        Size > Capacity - Offset)
    {
        return nullptr;
    }

    void* Result = Buffer + Offset;

    Offset += Size;

    return Result;
}
```

这是简化原理代码，未处理内存对齐、对象生命周期等细节。

特点：

- 分配速度非常快。
    
- 不支持随意释放单个对象。
    
- 通常通过 `Reset()` 一次性回收全部空间。
    
- 适合生命周期一致的临时数据。

### 适用于什么场景？

例如一帧内临时生成的：

```
可见性计算临时数组
粒子排序临时数组
碰撞检测临时数据
AI 查询临时结果
```

如果这些数据全部在同一帧结束时失效，就适合按照统一生命周期管理。

但如果数据会跨帧使用，就不能直接重置其所在的分配区域。

## 10.2 Stack Allocator：栈式分配器

基本思想：

```
分配 A
分配 B
分配 C

必须先释放 C
然后释放 B
最后释放 A
```

它并不一定使用线程调用栈，而是自定义的 LIFO 内存分配器。

## 10.3 Pool Allocator：固定块分配器

适合：

- 战斗事件节点。
    
- 固定尺寸的消息对象。
    
- AI 任务节点。
    
- 粒子数据节点。
    
- 一些小型临时对象。

但大型游戏通常需要处理不同大小的对象，因此还会使用多个 Size Class（大小类别）来管理内存。

例如：

```
大小类别
16 字节
32 字节
64 字节
128 字节
256 字节
512 字节
...
```

这也是现代高性能通用分配器常见的设计思路。

# 十一、智能指针与 RAII

C++ 中大量内存问题来源于对象所有权不明确。

例如：

```
void Test()
{
    Enemy* E = new Enemy();

    if (SomeCondition())
    {
        return; // 内存泄漏
    }

    delete E;
}
```

当函数提前返回时，分配的对象没有释放。

## 11.1 unique_ptr

```
#include <memory>

void Test()
{
    std::unique_ptr<Enemy> E =
        std::make_unique<Enemy>();

    // 使用对象
    E->HP = 100;

    // 离开作用域自动析构
}
```

`unique_ptr` 表示独占所有权。

其主要特点：

- 不支持复制所有权。
    
- 可以移动所有权。
    
- 离开作用域时自动释放。
    
- 本身通常不需要引用计数控制块。

## 11.2 shared_ptr

```
#include <memory>

void Test()
{
    std::shared_ptr<Enemy> A =
        std::make_shared<Enemy>();

    std::shared_ptr<Enemy> B = A;

    // A 与 B 共享对象所有权
}
```

`shared_ptr` 使用共享所有权机制。

当最后一个拥有该对象的 `shared_ptr` 被销毁或重置时，对象通常随之销毁。

### shared_ptr 的典型内部结构

```
shared_ptr A ──┐
               │
shared_ptr B ──┼──> 控制块
               │       │
shared_ptr C ──┘       ├── 强引用计数
                       ├── 弱引用计数相关状态
                       └── 删除器等信息

多个 shared_ptr 共享所有权
```

通常还涉及一个被管理对象的地址。

`std::make_shared` 常能够把对象与控制块安排在一次内存分配中，减少分配次数并改善局部性。但控制块因为 `weak_ptr` 仍然存活时，对应的整体存储可能无法立即完全释放。

## 11.3 weak_ptr

`weak_ptr` 用于弱引用，不增加对象的共享所有权计数。

例如：

```
struct Node
{
    std::shared_ptr<Node> Next;

    std::weak_ptr<Node> Parent;
};
```

它可以帮助避免因 `shared_ptr` 互相持有而形成的引用循环。

注意：`weak_ptr` 不保证对象还活着，需要先通过 `lock()` 尝试获得 `shared_ptr`。

# 十二、UE5 中的内存分配

对于 UE 游戏客户端开发，理解标准 C++ 的内存分配之后，还需要了解 UE 自己的内存管理体系。

## 12.1 FMemory

UE 提供了自己的内存相关接口。

典型用法：

```
// 申请内存
void* Memory = FMemory::Malloc(1024);

// 释放内存
FMemory::Free(Memory);
```

`FMemory` 把内存分配能力统一封装起来。

底层可以通过 `GMalloc` 对应的 `FMalloc` 实现处理分配请求。

概念上的关系：

```
UE 代码
   │
   ▼
FMemory::Malloc
   │
   ▼
GMalloc（FMalloc 接口）
   │
   ▼
具体分配器实现
   │
   ├── Binned 系列分配器
   ├── 平台分配器
   └── 其他配置的分配器
```

具体采用哪一种分配器，与 UE 版本、平台和构建配置有关。

## 12.2 FMallocBinned 系列

UE 的 Binned 系列分配器主要针对不同大小的内存申请进行分类管理。

例如：

```
内存申请
   │
   ▼
判断请求大小
   │
   ├── 小型分配
   │     ↓
   │   从对应大小类别的池中取块
   │
   └── 大型分配
         ↓
       使用对应的大块分配路径
```

它的主要设计目标包括：

- 提高常见分配大小的处理效率。
    
- 减少内存碎片。
    
- 改善多线程内存分配性能。
    
- 减少不必要的系统级分配操作。

不能简单把 Binned 分配器理解为一个只支持固定大小对象的 Pool Allocator；它通常管理多个分配大小类别。

## 12.3 UObject 内存管理

UE 中非常重要的一点：

`UObject` 的生命周期不等同于普通 C++ 对象。

例如：

```
UObject* Obj = NewObject<UObject>();
```

这通常涉及：

- UObject 创建机制。
    
- UObject 注册和管理。
    
- 反射相关信息。
    
- 垃圾回收系统（GC）。

不应该使用：

```
delete Obj; // 错误：不要手动 delete UObject
```

对于 `AActor`，也通常通过 `Destroy()` 请求销毁，而不是直接 `delete`。

对于继承自 `UObject` 的对象，应遵循 UE 的对象创建及 GC 规则。

## 12.4 UE 智能指针与 UObject GC 的区别

|对象类型|常见管理方式|
|---|---|
|普通 C++ 对象|new/delete、RAII、TUniquePtr|
|共享所有权普通对象|TSharedPtr、TSharedRef|
|弱引用普通共享对象|TWeakPtr|
|UObject|UE GC、UPROPERTY 等引用追踪机制|
|UObject 弱引用|TWeakObjectPtr|

例如：

```
// 普通 C++ 对象：使用 UE 智能指针
TUniquePtr<FMyData> Data =
    MakeUnique<FMyData>();
```

而 UObject：

```
UPROPERTY()
TObjectPtr<UObject> MyObject;
```

这两套机制不能混为一谈。

在支持 `TObjectPtr` 的 UE5 版本中，通常优先使用对应推荐的 UObject 引用表示方式，并通过反射系统正确声明需要 GC 追踪的成员。

# 十三、常见内存错误

## 13.1 内存泄漏（Memory Leak）

```
void Test()
{
    int* A = new int(10);

    // 没有 delete
}
```

如果失去了最后一个能够释放该分配的所有权路径，就可能形成内存泄漏。

## 13.2 悬空指针（Dangling Pointer）

```
int* A = new int(10);

delete A;

// A 仍然保存原来的地址
// 但该地址上的对象已经不再存活

// *A = 20; // 未定义行为
```

建议在需要继续保存该指针变量的场景中：

```
delete A;
A = nullptr;
```

注意，设置一个指针为 `nullptr` 并不会自动清理其他指向相同地址的指针。

## 13.3 重复释放（Double Free）

```
int* A = new int(10);

delete A;
delete A; // 未定义行为
```

## 13.4 越界访问

```
int* A = new int[10];

A[10] = 100; // 越界，未定义行为

delete[] A;
```

有效索引：

```
0 ~ 9
```

## 13.5 分配与释放不匹配

```
void* A = std::malloc(100);

delete A; // 错误
```

正确：

```
std::free(A);
```

同理：

```
int* A = new int(10);

std::free(A); // 错误
```

## 13.6 使用已经释放的对象（Use After Free）

在游戏客户端中，异步操作尤其容易触发此问题。

例如：

```
// 某个异步任务保存了对象的裸指针
Enemy* Target = ...;

// 主线程随后销毁了 Enemy
// 但异步任务仍然使用 Target
```

这类问题单纯使用 `nullptr` 检查无法解决，因为悬空指针也可能不是空指针。

需要从对象所有权、任务取消、生命周期保证和线程同步等方面设计。

# 十四、内存优化：游戏客户端实战

仅仅减少 `new` 的次数，不一定意味着整体性能最优。

还要考虑缓存命中率、对象布局、内存访问模式和分配生命周期。

## 14.1 连续内存与 CPU Cache

对比两种形式：

```
std::vector<Enemy> Enemies;
```

以及：

```
std::vector<Enemy*> Enemies;
```

第一种：

```
Enemy0 Enemy1 Enemy2 Enemy3 Enemy4
连续存储
```

第二种：

```
Pointer0 Pointer1 Pointer2 Pointer3
   │         │        │        │
   ▼         ▼        ▼        ▼
Enemy0    Enemy1   Enemy2   Enemy3
可能散布在不同内存地址
```

第一种如果按顺序遍历大量 `Enemy`，通常更有利于 CPU 缓存预取。

第二种则提供了更灵活的对象地址稳定性和多态支持，但可能增加指针间接访问和缓存未命中。

对于战斗系统中的大量同类型实体，数据布局可能比单次分配速度更影响性能。

## 14.2 AoS 和 SoA

AoS（Array of Structures，结构体数组）：

```
struct Enemy
{
    float HP;
    float Speed;
    float PositionX;
    float PositionY;
};

Enemy Enemies[1000];
```

SoA（Structure of Arrays，数组式结构）：

```
struct EnemiesData
{
    float HP[1000];
    float Speed[1000];
    float PositionX[1000];
    float PositionY[1000];
};
```

当游戏逻辑只需要遍历 HP 时：

```
for (int i = 0; i < 1000; ++i)
{
    EnemiesData.HP[i] -= 10.0f;
}
```

SoA 能让需要读取的 HP 数据更加连续，通常更有利于缓存和 SIMD 优化。

但如果每次处理都需要同一对象的大部分成员，AoS 也可能具有优势。

这类设计属于数据导向设计（Data-Oriented Design）。

## 14.3 减少频繁申请释放

比如子弹：

```
// 每次发射
Bullet* B = new Bullet();

// 子弹命中
delete B;
```

如果每秒创建销毁大量子弹，就可能给分配器和对象管理增加负担。

可以考虑：

- 预分配子弹池。
    
- 复用子弹对象。
    
- 采用连续数组管理。
    
- 批量处理更新。
    
- 根据场景动态控制对象池容量。

但如果子弹是 `AActor` 或 `UObject`，复用也必须遵守 UE 对象生命周期与组件管理规则，不能简单把普通 C++ 内存池代码直接套用在它们上面。

# 十五、C++ 内存分配高频面试题

下面这些问题适合 UE 游戏客户端岗位准备。
1. 栈和堆有什么区别？

栈上的自动存储期对象通常随作用域自动销毁，分配模式简单；堆用于动态分配，生命周期可独立于作用域。堆分配通常需要分配器维护元数据及空闲块，成本与行为受实现影响。

2. new 和 malloc 的区别？

new 是 C++ 表达式，负责分配空间并初始化对象；malloc 是 C 库函数，取得原始存储空间，不执行类构造函数。普通 new 失败通常抛 std::bad_alloc，而 malloc 失败返回空指针。

3. new 一定调用 malloc 吗？

不一定。全局 operator new 常通过运行时分配器实现，但 C++ 标准不要求必须调用 malloc。类也可以定义自己的 operator new。

4. 为什么 new[] 必须匹配 delete[]？

因为数组删除需要正确销毁所有元素，并使用匹配的内存释放机制。有些实现还需要数组 Cookie。new[] 与 delete 混用属于未定义行为。

5. 什么是 Placement New？

在已有且满足大小、对齐要求的存储空间中构造对象，不负责申请新的存储空间。常用于容器、对象池和手动管理对象生命周期。

6. 内存池为什么能优化性能？

固定大小池可以利用空闲链表实现通常 O(1) 的槽位获取和归还，降低通用分配器管理成本。但要权衡预留内存、容量限制和并发等问题。

7. 什么是内存碎片？

内部碎片是分配块中未被利用的空间；外部碎片是空闲块不连续而影响后续分配的现象。

8. 什么是内存对齐？

对象地址和成员偏移需要满足类型的对齐要求。编译器通常插入 Padding，并可能提高结构体的大小。

9. vector 扩容会发生什么？

当容量不足时通常申请更大的连续存储空间，将原有元素移动或复制到新存储空间，销毁旧元素并释放旧存储。扩容通常使原来的元素指针、引用和迭代器失效。

10. shared_ptr 为什么可能发生内存泄漏？

如果多个对象通过 shared_ptr 形成强引用环，即使外部不再持有它们，引用计数也可能无法降为零。通常用 weak_ptr 打破不必要的共享所有权循环。

11. malloc 和 operator new 的内存能混用释放吗？

不能依赖它们底层可能采用相同分配器这一实现细节。必须使用匹配的分配/释放接口，避免未定义行为。

12. UE 中为什么不能随便 delete UObject？

UObject 由 UE 对象系统和垃圾回收机制管理。直接 delete 绕过了引擎对象生命周期规则，应按 UObject、Actor 的对应 API 管理。

# 十六、知识总结

从游戏引擎角度理解 C++ 内存分配，我建议把它分为四层：

![[局部截取_20261026_235809.png]]

对 UE 游戏客户端来说，优先掌握的重点是：

- 基础必会：栈与堆、`new/delete`、`malloc/free`、内存对齐、对象生命周期。
    
- 进阶必会：Placement New、内存池、碎片管理、智能指针、RAII。
    
- 引擎底层：`FMalloc`、Binned 分配器、`UObject` GC、`TArray` 扩容和内存管理。
    
- 性能优化：缓存局部性、连续存储、AoS/SoA、减少热路径动态分配、多线程分配器竞争。

其中，最能体现引擎底层能力的，不是会不会调用 `new`，而是能否设计一种满足对象大小、生命周期、内存对齐、线程安全和性能约束的分配策略。

特别是动作游戏中频繁发生的战斗事件、技能计算、物理查询、动画任务和网络消息处理，了解如何按对象的生命周期管理内存，往往比简单地将所有分配替换为内存池更重要。