# UE5 GC（垃圾回收）机制详解 + 高频面试题

UE 的 GC（Garbage Collection，垃圾回收）是 UE 游戏客户端开发中非常重要的 C++ 面试知识点，尤其涉及 `UObject` 生命周期、`UPROPERTY`、智能指针、Actor 销毁和性能优化时，经常会深入考察。

本文以 UE5 为主，结合实际游戏项目讲解，重点放在原理、源码思路、代码示例和面试经典问题上。

## 一、UE GC 核心知识总览

先给出最重要的几个结论：

1. UE GC 的核心算法是 Mark and Sweep（标记-清除），通过可达性分析判断对象是否仍然需要保留。
    
2. GC 主要管理 UObject 及其派生类，例如 `AActor`、`UActorComponent`、`UDataAsset`、`UAnimInstance` 等。
    
3. UE 不依靠普通的 C++ 引用计数管理 `UObject` 生命周期，而是维护一个对象引用关系图。
    
4. `UPROPERTY()` 配合 `TObjectPtr<T>` 可以让 GC 识别强引用，但前提是持有者本身处于 GC 可达路径中。
    
5. 普通的 `UObject*` 成员如果没有被 GC 追踪，不能阻止对象回收。
    
6. `TWeakObjectPtr` 不阻止 GC，适合保存目标 Actor、技能目标等非拥有型引用。
    
7. `Actor->Destroy()` 不等于立即 `delete`，而是结束 Actor 的游戏生命周期，之后再由 GC 处理内存。
    
8. `Outer` 不等于 GC 强引用，`NewObject<T>(this)` 不能单独保证新对象存活。
    
9. UE5 支持 GC Cluster（对象群集）、并行 GC，以及可选的增量可达性分析等优化。
    
10. 游戏实际开发中最常见的问题不是 GC 无法回收对象，而是对象被错误地提前回收，或者被意外的强引用长期保留。
    

Epic 官方将 UE 的 GC 描述为基于对象引用图的标记-清除系统。

## 二、GC 的底层原理

### 1. Mark and Sweep（标记-清除）是什么？

UE 的 GC 并不简单地判断某个对象有没有指针指向它，而是判断：

这个对象能否从 GC 根集合（Root Set）出发，通过 GC 能识别的引用关系访问到。

例如有以下对象关系：

<img width="739" height="562" alt="image" src="https://github.com/user-attachments/assets/4184ad31-3047-42f6-ae00-78c658e9bb83" />

假设：

- `Player` 能从根集合访问到。
    
- `Player` 强引用 `SkillManager`。
    
- `SkillManager` 强引用 `SkillRuntime`。
    
- `Orphan UObject` 没有任何从根集合可达的引用路径。
    

那么 GC 结果就是：

|对象|是否可达|结果|
|---|---|---|
|Player|是|保留|
|GameInstance|是|保留|
|SkillManager|是|保留|
|SkillRuntime|是|保留|
|Orphan UObject|否|回收|

注意：图中的关系只是用于解释算法，不代表这些类在 UE 内部一定直接连接到 Root Set。

### 2. GC 具体执行流程

在 UE 的传统 GC 流程中，可以概括为以下步骤。

1. 准备阶段（Mark Initialization）
    
    为可回收对象建立本轮可达性标记状态，准备后续遍历。
    
2. 遍历根集合（Root Traversal）
    
    从 Root Set 和引擎其他已注册的 GC 引用来源出发，开始查找对象。
    
3. 可达性分析（Reachability Analysis）
    
    沿 GC 能识别的引用递归或迭代扫描，标记所有可达对象。
    
4. 收集不可达对象（Gather Unreachable）
    
    将未被标记为可达的对象视为垃圾，准备进入销毁流程。
    
5. 清理和销毁（Sweep / Purge）
    
    解除相关引用、执行销毁生命周期，并在允许时释放对象内存。
    

这是便于理解的抽象流程。实际 UE 源码中还有 Cluster、并行扫描、增量收集、对象标记和销毁延迟等细节，不一定全部集中在同一帧完成。

### 3. UE GC 如何判断对象间的引用？

例如：

```
UCLASS()
class UMySkillManager : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY()
    TObjectPtr<UObject> SkillData;
};
```

GC 需要识别：

```
UMySkillManager
        |
        | UPROPERTY 强引用
        v
     SkillData
```

这里涉及 UE 的反射系统。

`UPROPERTY()` 不仅仅是编辑器显示属性的宏，它还能让 UHT（Unreal Header Tool，虚幻头文件工具）生成相关的属性元信息。

GC 可以利用这些元信息识别对象引用。

其基本过程是：

1. UHT 处理 `UCLASS`、`UPROPERTY` 等声明。
    
2. UE 为类建立反射属性描述。
    
3. GC 根据反射信息及其他引用收集机制找到对象引用。
    
4. 通过引用遍历判断对象是否可达。
    

在底层，常见关键概念包括：

|类型或机制|作用|
|---|---|
|`UClass`|描述 UObject 类的类型信息|
|`FProperty`|描述反射属性|
|`FObjectProperty`|描述 UObject 对象引用属性|
|`Reference Token Stream`|用于高效遍历引用的机制|
|`TFastReferenceCollector`|GC 引用遍历的重要实现|
|`AddReferencedObjects`|为 GC 补充无法直接通过普通属性遍历获得的引用|

Epic 的 `TFastReferenceCollector` API 文档明确提到，它通过类引用 Token Stream 和 `AddReferencedObjects` 查找引用。

面试重点：

GC 并不是每次都扫描对象中所有内存，猜测哪些字节可能是指针，而是利用 UE 的对象系统维护的引用元信息和专门的引用收集接口进行精确追踪。

## 三、Root Set（根集合）详解

### 1. 什么是 Root Set？

Root Set（根集合）是 GC 可达性分析的根节点集合。

只要对象本身处于 Root Set，或者通过被 GC 识别的引用链能够从根访问到，就不会仅仅因为普通引用计数为零而被回收。

引擎会维护重要的根对象与其他 GC 引用来源。

普通游戏代码通常不需要手动将对象加入 Root Set。

### 2. AddToRoot 和 RemoveFromRoot

`AddToRoot()` 可以显式将 UObject 加入根集合。

```
UObject* Obj = NewObject<UObject>();

// 加入 Root Set
Obj->AddToRoot();

// 即使没有普通 UPROPERTY 强引用，
// 也不会因为不可达而被 GC 回收。
```

如果不再需要，需要移出：

```
if (Obj && Obj->IsRooted())
{
    Obj->RemoveFromRoot();
}

// 之后当没有其他有效强引用时，
// Obj 才可能被后续 GC 回收。
```

`AddToRoot()` 和 `RemoveFromRoot()` 是明确的生命周期干预接口。

Epic Developer Community+1

要特别注意：

`AddToRoot()` 并不意味着对象永远不会被销毁，它只是阻止普通 GC 可达性规则将其当作不可达对象回收。

同时，如果忘记调用 `RemoveFromRoot()`，可能导致对象长期无法回收，造成类似内存泄漏的问题。

所以在 UE5 项目里，优先选择明确的强引用生命周期管理，而不是大量使用 `AddToRoot()`。

### 3. Outer 是不是根引用？

这是一个经典陷阱。

```
UObject* Obj = NewObject<UObject>(this);
```

这里的 `this` 是 `Outer`。

`Outer` 的主要作用包括对象命名空间、层级关系、路径及部分序列化和对象管理语义。

但它不是自动生成的、由 Outer 指向新对象的 GC 强引用。

也就是说：

```
NewObject<UObject>(this);
```

不能仅凭指定 `this` 为 Outer，就保证返回的 UObject 一直存活。

正确示例：

```
UPROPERTY()
TObjectPtr<UObject> MyObject;

void AMyActor::BeginPlay()
{
    Super::BeginPlay();

    // Outer 确定对象的外部归属关系
    // UPROPERTY 则提供 GC 能追踪的强引用
    MyObject = NewObject<UObject>(this);
}
```

还要注意，`Outer` 与 `Owner` 也不是同一个概念，特别是 Actor 的 `Owner` 具有专门的游戏逻辑和网络复制语义。

## 四、UPROPERTY 与 GC 的关系（最高频考点）

### 1. UPROPERTY 为什么能阻止对象回收？

看下面两种写法。

写法 A：GC 不能通过该成员追踪引用

```
UCLASS()
class UMyManager : public UObject
{
    GENERATED_BODY()

public:
    // 没有 UPROPERTY
    UObject* MyObject = nullptr;
};
```

初始化：

```
MyObject = NewObject<UObject>(this);
```

虽然 `MyObject` 是一个非空指针，但 GC 不会因为这个普通成员指针而将目标对象判断为可达。

结果：

```
Manager 存活
    |
    | 普通指针，未被 GC 识别
    v
 MyObject

MyObject 仍可能被回收
```

写法 B：正确建立强引用

```
UCLASS()
class UMyManager : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY()
    TObjectPtr<UObject> MyObject = nullptr;
};
```

只要 `UMyManager` 本身处于有效的 GC 引用路径中，并且没有特殊的强制销毁情况：

```
Root Set
    |
    v
UMyManager
    |
    | UPROPERTY + TObjectPtr
    v
 MyObject
```

GC 就能够识别该引用，保留 `MyObject`。

### 2. TObjectPtr 是不是天然强引用？

不是。

这是 UE5 面试容易答错的地方。

```
// 形式一：正确的持久强引用
UPROPERTY()
TObjectPtr<UObject> ObjA;
```

```
// 形式二：没有 UPROPERTY 的普通成员
TObjectPtr<UObject> ObjB;
```

区别在于：

- `ObjA`：属于反射属性，GC 可以追踪。
    
- `ObjB`：单纯声明 `TObjectPtr` 并不保证它会被 GC 遍历为强引用。
    

Epic 官方文档特别说明，`TObjectPtr` 的常规 GC 强引用保证依赖其作为被 `UPROPERTY` 标记的成员使用。

所以，不能简单记忆为：

> TObjectPtr 一定是强引用。

更准确的表述是：

> 在被 GC 正确追踪的 UObject/结构体属性中，`UPROPERTY()` 标记的 `TObjectPtr` 是常用强引用形式。

### 3. 旧写法 UPROPERTY UObject* 还能用吗？

传统 UE C++ 写法：

```
UPROPERTY()
UObject* MyObject;
```

在支持这种反射属性写法的引擎版本和配置中，它同样可以参与 GC。

但是 UE5 推荐：

```
UPROPERTY()
TObjectPtr<UObject> MyObject;
```

原因不仅是代码风格。

`TObjectPtr` 支持 UE5 的对象句柄机制、Cook 阶段依赖处理，以及增量 GC 所需要的写屏障（Write Barrier）。

因此面试不能回答“UE5 中只有 TObjectPtr 才能防 GC”。

正确答案应区分传统标记-清除流程和增量 GC 的要求。后面会详细解释。

Epic Developer Community+1

## 五、UE 各种指针与 GC 的关系

这是 GC 面试中最容易和 C++ 智能指针混在一起的部分。

### 1. 常用指针对比

|指针类型|是否阻止 GC|典型场景|
|---|---|---|
|`UObject*`|默认否，反射强引用除外|临时对象访问|
|`TObjectPtr<T>`|配合 GC 可追踪的 `UPROPERTY` 时是|UObject 持久成员|
|`TWeakObjectPtr<T>`|否|锁定目标、Actor 缓存|
|`TSoftObjectPtr<T>`|否|资源路径、按需加载|
|`TStrongObjectPtr<T>`|是|普通 C++ 类保存 UObject|
|`TSharedPtr<T>`|不参与 UObject GC|非 UObject 的共享所有权|
|`TUniquePtr<T>`|不参与 UObject GC|非 UObject 的独占所有权|

以上是常用情形，具体还要考虑注册的引用收集器、反射属性和对象本身的销毁状态。

### 2. TObjectPtr：强引用成员

例如，一个技能管理器需要保存当前技能运行时对象：

```
UCLASS()
class UMySkillManager : public UObject
{
    GENERATED_BODY()

public:
    // GC 可追踪的强引用
    // 管理器存活时，通常也会保留技能运行时对象
    UPROPERTY()
    TObjectPtr<UMySkillRuntime> CurrentSkill;
};
```

适合：

- 角色的技能组件
    
- 技能运行时对象
    
- UI Widget 引用
    
- AnimInstance 中需要长期持有的对象
    

### 3. TWeakObjectPtr：弱引用

例如，角色当前锁定了一个敌人。

```
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // 弱引用：不会为了锁定关系阻止敌人 GC
    TWeakObjectPtr<AActor> LockTarget;

    void Attack()
    {
        // 先检查对象是否仍有效
        if (AActor* Target = LockTarget.Get())
        {
            // 访问有效目标
            // 执行攻击逻辑
        }
    }
};
```

为什么不一定要使用 `TObjectPtr`？

因为角色只是“观察”或“锁定”这个目标，并不负责管理敌人的生命周期。

如果敌人死亡并被销毁，锁定关系应该自动失效，而不是阻止敌人释放。

另外，弱引用通常包含对象索引和序列号等身份信息，而不只是简单保存一个地址。

这样即使之前的 UObject 内存槽位被重新利用，弱引用也不会轻易把新对象错当成旧对象。

### 4. TSoftObjectPtr：软引用

例如，技能配置中包含大量动作、特效、音效资源。

不希望加载技能配置时把所有特效资源都加载到内存。

```
UCLASS(BlueprintType)
class UMySkillData : public UDataAsset
{
    GENERATED_BODY()

public:
    // 软引用：记录资源路径，不强制立即加载资源
    UPROPERTY(EditDefaultsOnly)
    TSoftObjectPtr<UParticleSystem> HitEffect;
};
```

使用时：

```
if (!HitEffect.IsNull())
{
    // 同步加载示例
    // 实际战斗项目应谨慎使用，避免战斗时卡顿
    UParticleSystem* Effect = HitEffect.LoadSynchronous();
}
```

非常重要：`TSoftObjectPtr` 负责定位资产，不负责让已加载资产永久存活。

对于异步加载，应通过 `UAssetManager`、`FStreamableManager` 和相应加载句柄管理加载过程与资源驻留时间。

例如：

```
技能配置
    |
    | TSoftObjectPtr
    v
Niagara 特效资源路径
    |
    | 技能实际使用时
    v
异步加载资源
    |
    v
技能实例持有有效引用 / 加载句柄
    |
    | 使用结束
    v
释放不再需要的引用或句柄
    |
    v
资源满足卸载条件后才能释放
```

这对动作游戏非常重要，因为 GC 优化和资源加载优化是相互关联的，但不是同一个问题。

### 5. TStrongObjectPtr：普通 C++ 类中的强引用

如果管理器不是 `UObject`，而是一个普通 C++ 类：

```
class FMyObjectManager
{
public:
    // 适合在非 UObject 所有者中
    // 建立明确的 GC 强引用
    TStrongObjectPtr<UObject> MyObject;

    void Initialize()
    {
        MyObject =
            TStrongObjectPtr<UObject>(
                NewObject<UObject>()
            );
    }
};
```

在 `FMyObjectManager` 持有该强指针期间，它会保护目标 UObject 不被普通 GC 回收。

但它也有额外的引用注册成本，并且没有 `UPROPERTY` 那样直接的引用关系可见性。

不能把它当作 UObject 内部成员的通用替代品。

### 6. 为什么 TSharedPtr 不能管理 UObject？

错误思路：

```
// 不要用 TSharedPtr 管理 UObject 的所有权
TSharedPtr<UObject> MyObject;
```

因为两套机制不同：

`TSharedPtr`：

```
引用计数增加
    |
引用计数减少
    |
计数归零
    |
触发普通 C++ 对象销毁
```

`UObject GC`：

```
Root Set
    |
可达性分析
    |
发现不可达 UObject
    |
执行 UObject GC 销毁流程
```

`UObject` 的创建、登记、反射、生命周期和释放应由 UE 对象系统管理，而非 `TSharedPtr` 在引用计数归零时直接删除。

面试标准回答：

`TSharedPtr`、`TUniquePtr` 主要用于非 UObject 对象；UObject 应使用 UE 的 GC 感知引用机制管理生命周期。

## 六、UObject 和 Actor 的销毁过程

### 1. UObject 的销毁过程

普通 `UObject` 创建：

```
UObject* Obj = NewObject<UObject>();
```

它不会因为当前函数执行完毕就立即析构。

当没有可达的强引用时，可以进入 GC 回收流程。

简化生命周期：

```
NewObject
    ↓
对象初始化
    ↓
正常使用
    ↓
失去 GC 可达性
    ↓
GC 发现不可达
    ↓
BeginDestroy()
    ↓
IsReadyForFinishDestroy()
    ↓
FinishDestroy()
    ↓
C++ 析构与内存释放
```

其中三个关键回调：

BeginDestroy()

对象开始释放资源的阶段。

```
virtual void BeginDestroy() override
{
    // 清理与对象销毁相关的底层资源
    // 不适合继续执行普通游戏玩法逻辑

    Super::BeginDestroy();
}
```

IsReadyForFinishDestroy()

判断是否能够进入最终销毁阶段。

某些对象可能需要等待渲染线程或其他底层异步资源处理完成。

FinishDestroy()

进行最后的销毁清理，随后对象内存才能最终释放。

不要把这些回调当成普通的游戏业务退出事件。

### 2. Actor->Destroy() 与 GC 的区别

这是极高频问题。

```
Enemy->Destroy();
```

并不是执行：

```
delete Enemy;
```

`Destroy()` 负责让 Actor 离开当前游戏生命周期。

简化过程：

```
Enemy->Destroy()
       |
       v
Actor 结束游戏生命周期
       |
       v
触发 EndPlay 等清理逻辑
       |
       v
Actor 被标记为不再可正常使用
       |
       v
后续 GC 回收对象内存
```

需要注意，`OnDestroyed` 和 `EndPlay` 的具体调用顺序及触发条件应以实际引擎版本为准；上图是概念流程。

### 3. EndPlay、OnDestroyed、BeginDestroy 有什么区别？

|函数|作用|通常用于|
|---|---|---|
|`EndPlay()`|Actor 退出游戏生命周期|定时器、委托、技能和玩法清理|
|`OnDestroyed()`|Actor 通过 Destroy 被销毁时的事件|特定 Actor 销毁通知|
|`BeginDestroy()`|UObject 开始 GC 销毁|底层资源释放|
|`FinishDestroy()`|UObject 最终销毁|最后阶段的底层清理|

例如：

```
void AMyEnemy::EndPlay(
    const EEndPlayReason::Type EndPlayReason)
{
    // 清理定时器
    GetWorldTimerManager().ClearAllTimersForObject(this);

    // 取消技能和逻辑事件注册
    // 清理外部系统对该对象的引用关系

    Super::EndPlay(EndPlayReason);
}
```

实际项目里，普通战斗逻辑优先在 `EndPlay` 做清理。

`EndPlay` 不只在 `Destroy()` 时触发，还可能在关卡卸载、地图切换和 PIE 结束时触发。某些流送场景中，EndPlay 发生后也不意味着对象立刻物理销毁。

Epic Developer Community

### 4. Destroy 之后指针还可能不是 nullptr

例如：

```
AActor* Enemy = GetEnemy();

Enemy->Destroy();
```

这时：

```
Enemy != nullptr
```

仍然可能成立。

因为指针保存的地址不会仅仅因为调用 `Destroy()` 就必然立即变成空指针。

但：

```
IsValid(Enemy)
```

通常已经返回 `false`。

因此，对于可能已经销毁的 Actor：

```
if (IsValid(Enemy))
{
    Enemy->SomeFunction();
}
```

通常比单纯判断：

```
if (Enemy != nullptr)
```

更合理。

这里还要区分两个概念：

- `IsValid()`：判断对象是否仍可正常使用。
    
- `IsValidLowLevel()`：更底层的对象完整性检查，不应作为常规生命周期检查替代 `IsValid()`。
    

尤其不能对可能已经成为悬空指针的普通裸指针盲目调用成员方法。

## 七、引用关系中的几个经典陷阱

### 1. 循环引用是否会导致 GC 无法回收？

假设：

```
UCLASS()
class UObjectA : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY()
    TObjectPtr<UObjectB> B;
};
```

另一个类：

```
UCLASS()
class UObjectB : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY()
    TObjectPtr<UObjectA> A;
};
```

对象引用：

```
    A ──────> B
    ^         |
    |         |
    └─────────┘
```

A 和 B 相互强引用。

它们一定无法回收吗？

不一定。

这是 UE GC 与单纯引用计数的重要区别。

如果：

```
Root Set ───> A ───> B
```

A、B 均可达，因此会保留。

但如果：

```
Root Set

   A <────> B
```

没有任何从 Root Set 到 A 或 B 的有效引用路径，那么两个对象虽然相互引用，仍然是不可达的，可以被 GC 一起回收。

面试重点：UE 的标记-清除算法能够处理这种不可达引用环。

但也不能由此认为 UE 永远没有循环引用导致的生命周期问题。例如 `TStrongObjectPtr`、`AddToRoot()` 或其他根引用机制可能让对象长期可达。

### 2. TArray 里的 UObject 指针会被 GC 识别吗？

正确示例：

```
UCLASS()
class UMySkillManager : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY()
    TArray<TObjectPtr<UMySkillRuntime>> Skills;
};
```

如果管理器可达，那么 GC 可以追踪数组元素中的 UObject 强引用。

错误示例：

```
UCLASS()
class UMySkillManager : public UObject
{
    GENERATED_BODY()

public:
    // 未标记 UPROPERTY
    TArray<UObject*> Skills;
};
```

TArray 本身并不会自动让 GC 识别数组里的引用。

需要数组所在的属性参与反射，或者由其他专门的引用收集机制处理。

同理：

```
UPROPERTY()
TMap<int32, TObjectPtr<UObject>> ObjectMap;
```

通常可以被 GC 正确追踪。

### 3. USTRUCT 中的 TObjectPtr 会被追踪吗？

例如：

```
USTRUCT(BlueprintType)
struct FSkillRuntimeInfo
{
    GENERATED_BODY()

    // 结构体内部的反射属性
    UPROPERTY()
    TObjectPtr<UObject> SkillObject = nullptr;
};
```

然后：

```
UCLASS()
class UMySkillManager : public UObject
{
    GENERATED_BODY()

public:
    // 外层也必须能通过反射被 GC 遍历
    UPROPERTY()
    FSkillRuntimeInfo RuntimeInfo;
};
```

GC 可以沿着：

```
UMySkillManager
      |
      v
 RuntimeInfo
      |
      v
  SkillObject
```

识别引用。

但如果外层的 `RuntimeInfo` 是未反射的普通成员，不能仅依赖结构体内部的 `UPROPERTY` 就认为整条链一定可被追踪。

### 4. 在普通 C++ 类中写 UPROPERTY 可以吗？

普通 C++ 类：

```
class FMyManager
{
public:
    UObject* Obj;
};
```

`FMyManager` 不属于 UObject 反射体系，不能简单给成员加 `UPROPERTY` 来解决 GC 问题。

常见解决方案：

- 使用 `TStrongObjectPtr`。
    
- 实现 `FGCObject` 并注册引用。
    
- 让一个真实的 UObject 所有者通过 `UPROPERTY` 保存引用。
    

例如，非 UObject 管理器通过 `FGCObject` 参与引用收集：

```
#include "UObject/GCObject.h"

class FMyNativeManager : public FGCObject
{
public:
    UObject* ManagedObject = nullptr;

    // GC 扫描时主动报告引用
    virtual void AddReferencedObjects(
        FReferenceCollector& Collector) override
    {
        Collector.AddReferencedObject(ManagedObject);
    }

    virtual FString GetReferencerName() const override
    {
        return TEXT("FMyNativeManager");
    }
};
```

这样就为普通 C++ 类提供了一种向 UObject GC 系统报告强引用的方法。

实际开发需要确保 `FGCObject` 实例本身的注册和析构时机正确，不要在不安全的并发阶段修改它持有的引用。

`FGCObject` 是 UE 支持的 GC 引用注册扩展点。

## 八、GC 性能优化详解

UE 是实时游戏引擎，对 GC 的要求不仅是正确释放内存，还要尽可能减少主线程卡顿。

### 1. 为什么 GC 会造成卡顿？

假设一场 Boss 战中：

- 同时存在大量敌人、技能实例和特效对象。
    
- 战斗过程中不断创建临时 UObject。
    
- Boss 战结束后，大量对象同时失去引用。
    
- GC 开始扫描整个 UObject 引用关系图。
    

传统可达性分析如果集中在单帧执行，就可能形成明显的 CPU 耗时峰值。

<img width="738" height="452" alt="image" src="https://github.com/user-attachments/assets/a136cef8-6c54-43c7-a14c-eb60919a3bbe" />

### 2. GC Cluster（对象群集）

这是 UE GC 的常见优化机制。

传统遍历：

```
Root
  |
  v
Object A
  |
  +---> Object B
  |
  +---> Object C
  |
  +---> Object D
```

如果这些对象属于可以被引擎聚类的一组对象，可以通过 GC Cluster 统一处理其可达性。

```
Root
  |
  v
GC Cluster
  |
  +-- Object A
  +-- Object B
  +-- Object C
  +-- Object D
```

优势：

减少大量对象之间的逐个可达性扫描，提高 GC 标记效率。

缺点：

- Cluster 可能延长其中部分对象的存活时间。
    
- 大群集一起销毁时，可能引起较大的清理耗时。
    
- 不适合随意把生命周期完全不同的对象放进同一群集。
    

这里的 GC Cluster 和 Chaos 物理破碎中的 Geometry Collection Cluster 是不同概念。

Epic 的对象管理文档也说明，GC Cluster 通常可以缩短可达性分析，但过大的群集可能增加集中销毁的卡顿。

### 3. Parallel GC（并行垃圾回收）

并行 GC 通过多个线程参与部分引用扫描等工作，减少主线程承担的计算量。

但它不代表所有 GC 工作都可以安全地放在工作线程执行。

例如：

```
主线程
  |
  +-- 准备 GC 数据
  |
  +-- 调度引用遍历任务
           |
           +-- 工作线程 A
           +-- 工作线程 B
           +-- 工作线程 C
  |
  +-- 汇总结果与后续处理
```

关键点：

- 可达性分析可以利用并行处理。
    
- 一些准备、同步和销毁操作仍可能造成停顿。
    
- UObject 引用的修改必须满足引擎对应的线程安全要求。
    

### 4. Incremental GC（增量 GC）

这是 UE5 比较值得掌握的进阶知识。

传统模式下，可达性分析往往集中在某一帧执行。

增量可达性分析将工作拆分到多帧，每帧执行一部分。

假设有 10000 个对象需要处理：

```
传统模式：

Frame 1 ─── 大量可达性分析
Frame 2 ─── 正常游戏
Frame 3 ─── 正常游戏
```

```
增量模式：

Frame 1 ─── 部分可达性分析
Frame 2 ─── 部分可达性分析
Frame 3 ─── 部分可达性分析
Frame 4 ─── 分析完成
```

关键问题在于：

如果 GC 分成多帧处理，游戏逻辑在中途修改了对象引用怎么办？

例如：

```
UPROPERTY()
TObjectPtr<UObject> CurrentObject;
```

GC 已经扫描了一部分对象，此时：

```
CurrentObject = NewObject<UObject>(this);
```

引用关系发生了变化。

这就需要 Write Barrier（写屏障）。

写屏障用于通知增量标记系统，在扫描过程中发生了新的引用写入，使 GC 能正确处理新对象的可达性。

UE5 中 `TObjectPtr` 的价值之一就是支持这种机制。

Epic 文档提供了以下配置示例：

```
[ConsoleVariables]

; 开启增量可达性分析
gc.AllowIncrementalReachability=1

; 开启增量收集不可达对象
gc.AllowIncrementalGather=1

; 每次迭代的软时间预算：2ms
gc.IncrementalReachabilityTimeLimit=0.002
```

需要强调：这些配置不应未经验证直接用于正式项目。Epic 文档仍将该功能描述为实验性，并提示增量可达性在工作线程访问 UObject 的场景存在安全限制。建议结合具体 UE5 版本、源码和平台进行测试。

Epic Developer Community

还要区分：

- Incremental Reachability：分多次完成可达性分析。
    
- Incremental Purge：分多次处理对象最终销毁与释放。
    

两者不是一回事。

### 5. 动作游戏中的 GC 优化方案

对于大量技能、弹幕、伤害数字、UI 和特效对象的游戏，重点是减少不必要的对象创建和引用扫描。

|优化方案|作用|注意点|
|---|---|---|
|对象池（Object Pool）|减少频繁创建和销毁|池中对象仍占用内存|
|软引用资源|避免技能配置硬引用大量资源|必须管理异步加载与驻留|
|控制 UObject 数量|减少 GC 追踪规模|轻量数据可考虑普通 C++ 结构|
|及时解除强引用|允许无用对象回收|注意异步任务与委托|
|合理安排 GC 时机|降低战斗过程中集中停顿|不能随意延迟导致内存上涨|
|并行与增量 GC|分摊部分 GC 工作|测试性能和线程安全|

例如：

一个技能命中效果仅需要保存几个数值：

```
struct FHitResultData
{
    float Damage = 0.0f;
    float CriticalMultiplier = 1.0f;
    int32 HitCount = 0;
};
```

如果没有必要使用 UObject 的反射、身份、生命周期和多态能力，直接使用普通 C++ 数据结构往往更加轻量。

但这也不代表所有技能对象都应该改成普通结构体。

在 GAS 等系统中，是否选择 UObject、UStruct 或普通 C++ 类型，需要考虑反射、复制、蓝图、实例化策略和整体架构。

## 九、GC 调试与问题定位

面试有时不只问原理，还会问：

“游戏里发现对象一直没有释放，怎么定位？”

常用工具如下：

|命令或工具|用途|
|---|---|
|`stat gc`|查看 GC 相关性能统计|
|`obj list`|查看 UObject 数量与对象信息|
|`obj refs name=对象名`|分析目标对象的引用链|
|`gc.ListClusters`|查看 GC Cluster|
|`memreport -full`|输出内存报告|
|Unreal Insights|分析 GC 事件、耗时与线程活动|

部分命令在不同 UE 版本或构建配置中支持程度不同。Epic 官方命令参考中包含对象列表、引用查询和群集诊断等功能。

### 实际案例：关闭 UI 后内存一直没有下降

假设背包 UI 每次打开都会创建一个 Widget。

正常逻辑：

```
打开背包
    ↓
CreateWidget
    ↓
显示 Widget
    ↓
关闭背包
    ↓
清理并释放相关引用
    ↓
后续 GC 可回收 Widget
```

但如果 PlayerController 中保存了强引用：

```
UPROPERTY()
TObjectPtr<UUserWidget> InventoryWidget;
```

仅仅调用：

```
InventoryWidget->RemoveFromParent();
```

并不等于销毁 Widget。

`RemoveFromParent()` 负责从 UI 层级移除，并不会自动清空 PlayerController 的成员引用。

因此 Widget 仍然可达。

如果设计目标是关闭时释放 Widget，应该同时清理相关委托、定时器和引用：

```
if (IsValid(InventoryWidget))
{
    InventoryWidget->RemoveFromParent();
}

InventoryWidget = nullptr;
```

此时 Widget 也不是立即释放，还需要确保不存在其他强引用，等待适当的 GC 周期。

如果内存依然不下降，就要继续检查引用链，区分真正的活对象泄漏和内存分配器暂未将内存归还操作系统的情况。

# 十、UE GC 经典面试问题与参考答案

下面是面向 UE 游戏客户端岗位的 20 道重点题。

## 第一组：基础原理（必会）

### Q1. UE GC 的核心原理是什么？

参考答案：

UE 使用基于引用图的标记-清除算法。

从 GC 根集合及引擎注册的强引用来源出发，遍历可达的 UObject。

能够到达的对象被视为仍在使用，无法到达的对象在符合回收条件后进入销毁流程。

GC 主要管理 UObject，不管理所有普通 C++ 对象。

### Q2. UE GC 与 C++ 引用计数有什么区别？

|比较|UE GC|C++ SharedPtr|
|---|---|---|
|核心方式|可达性分析|引用计数|
|回收时机|GC 执行阶段|引用计数归零|
|循环引用|可处理不可达环|强引用环通常无法自动解除|
|管理对象|UObject|普通 C++ 对象|
|主要成本|扫描、销毁和引用追踪|引用计数维护|

因此，UE 中两个 UObject 相互强引用，不一定形成内存泄漏。

### Q3. 为什么 UPROPERTY 能保护对象不被 GC？

因为它使成员进入 UE 的反射属性体系，GC 可以沿着该成员建立可达性引用关系。

但需要满足两个重要条件：

- 成员确实属于可以被 GC 遍历到的反射对象路径。
    
- 引用指向的 UObject 没有处于需要强制清理的特殊销毁状态。
    

`UPROPERTY` 也不等于直接把目标 UObject 加入 Root Set。

### Q4. UObject 没有 UPROPERTY 引用，会立即被回收吗？

不会。

失去有效强引用只是使对象可能成为不可达对象。

通常还需要后续 GC 执行，才能完成销毁和释放。

所以不能把“没有强引用”和“已经销毁”理解为同一个事件。

### Q5. Outer 能防止子 UObject 被 GC 吗？

不能仅依靠 Outer。

Outer 不是自动提供的从父对象到子对象的 GC 强引用。

实际项目需要通过 `UPROPERTY`、注册引用收集器或者其他合适的生命周期机制保证对象存活。

### Q6. UObject 为什么不能直接 delete？

`UObject` 需要经过 UE 对象系统维护的生命周期。

使用普通 `delete` 会绕过对象登记、反射、引用清理和垃圾回收等机制，可能破坏 UObject 的内部状态。

正常运行时创建 UObject 使用 `NewObject`，Actor 使用 `SpawnActor`，而不是普通 `new`。

## 第二组：指针与生命周期（必会）

### Q7. TObjectPtr 与 UObject* 有什么区别？

参考答案：

`TObjectPtr` 是 UE5 推荐用于持久 UObject 属性的对象指针封装。

当它作为 `UPROPERTY` 被 GC 追踪时，提供有效的强引用，并支持 UE5 对象句柄及增量 GC 写屏障等功能。

普通裸指针适合局部变量、函数参数和短生命周期访问。

旧版本支持的 `UPROPERTY() UObject*` 同样可以提供反射强引用，但需要考虑现代 UE5 的增量 GC 要求。

### Q8. TWeakObjectPtr 和 TObjectPtr 有什么区别？

`TObjectPtr` 主要用于需要保留对象生命周期的引用。

`TWeakObjectPtr` 是非拥有型引用，不阻止对象被 GC。

例如：

```
// 需要保留的技能运行时对象
UPROPERTY()
TObjectPtr<UMySkillRuntime> SkillRuntime;

// 只需要观察的锁定目标
TWeakObjectPtr<AActor> TargetActor;
```

两者的区别是所有权和生命周期语义，而不是简单的性能高低。

### Q9. TSoftObjectPtr 与 TObjectPtr 有什么区别？

`TObjectPtr` 指向内存中的 UObject，用于访问和持有已存在的对象。

`TSoftObjectPtr` 可以保存对象路径，不要求目标资源立即加载，也不负责阻止目标被 GC。

它特别适合资产配置和按需加载场景。

### Q10. Actor 调用 Destroy 后为什么不能立即释放内存？

因为 Actor 的游戏生命周期和 UObject 的物理内存释放属于不同阶段。

`Destroy()` 结束 Actor 的游戏参与状态，并触发相关清理。

真正释放 UObject 内存由后续 GC 处理。

也不能通过 `UPROPERTY` 或 `TStrongObjectPtr` 强行阻止一个已经 Destroy 的 Actor 退出游戏生命周期。

### Q11. 为什么判断 UObject 是否有效不能只使用 nullptr？

因为指针非空不能证明对象仍然可用。

对象可能已经被标记为垃圾，或者 Actor 已经通过 `Destroy()` 结束生命周期。

通常使用：

```
if (IsValid(MyActor))
{
    // 对象当前可以正常使用
}
```

但 `IsValid` 不能修复已经失效的普通悬空指针。

另外，多线程中检查 UObject 生命周期需要额外同步保障，不能把一次有效性检查当成任意线程的长期生命周期保证。

### Q12. GC 可以解决循环引用吗？

对于只包含普通 UObject GC 强引用的不可达环，可以。

例如 A 引用 B，B 引用 A。

如果它们都无法从 GC 根或注册引用来源访问到，就可以被标记为不可达。

但如果 A 被 `AddToRoot()`，整个引用链就可能持续存活。

如果使用不恰当的 `TStrongObjectPtr`，也可能引入更复杂的生命周期问题。

## 第三组：源码与底层机制（进阶）

### Q13. GC 是如何找到 UObject 的引用的？

主要通过：

- UObject 的反射属性信息。
    
- GC 的引用遍历机制与引用 Token Stream。
    
- `AddReferencedObjects` 等专门的引用收集接口。
    
- 引擎自身维护的根对象与引用来源。
    

因此 GC 不需要逐字节猜测哪个内存位置是指针。

### Q14. AddReferencedObjects 的作用是什么？

用于向 GC 主动报告额外的 UObject 引用。

当引用无法被标准 `UPROPERTY` 遍历直接发现时，可以通过这个机制补充引用关系。

常见于：

- UObject 类的自定义引用收集。
    
- 复杂引擎对象系统。
    
- `FGCObject` 管理的非 UObject 所有者。
    

### Q15. UE 的 GC Cluster 是什么？

GC Cluster 可以将一组关联 UObject 视为统一的可达性管理单元，减少逐个对象的扫描成本。

适合存在相同或高度关联生命周期的对象集合。

主要权衡是扫描效率、对象释放粒度和集中销毁时的开销。

### Q16. 增量 GC 是如何保证引用变化安全的？

通过写屏障等机制处理增量扫描过程中发生的引用修改。

例如，将新对象赋给可被追踪的 `TObjectPtr` 属性时，增量标记系统可以处理这次引用变更。

但并不意味着所有裸指针写入、异步线程修改都天然安全。

## 第四组：项目实战（面试加分）

### Q17. 为什么某个 UObject 一直无法被回收？

常见原因：

1. 仍有 `UPROPERTY` 强引用。
    
2. 被 `AddToRoot()`，但忘记移除。
    
3. `TStrongObjectPtr` 长期持有。
    
4. `FGCObject` 等引用收集器仍在报告它。
    
5. 对象所属的关卡、子系统或其他引擎对象仍保留有效引用。
    
6. GC 尚未执行完整回收流程。
    

排查时优先使用 `obj refs` 或引用链分析工具。

不要简单认为“GC 没有工作”。

### Q18. 一个 UObject 被 GC 回收后，普通裸指针会自动置空吗？

一般不会。

```
UObject* RawObject;
```

如果它是未被反射或其他机制追踪的普通 C++ 成员，GC 不知道这个存储位置需要被更新。

目标被回收后，该裸指针可能悬空。

而被 GC 追踪的 UObject 属性引用，在相关清理流程中可以被置空。

对于不需要拥有生命周期的目标，`TWeakObjectPtr` 是更合适的选择。

### Q19. 如何防止战斗中频繁 GC 导致掉帧？

参考答案：

我会先用 Unreal Insights 确认 GC 是否真的造成卡顿，并定位耗时阶段。

如果确定是 GC 问题，再考虑：

- 减少高频创建的临时 UObject。
    
- 使用合理的技能运行时对象池。
    
- 用轻量结构体替代不必要的 UObject。
    
- 减少不必要的引用关系和 UObject 数量。
    
- 合理规划资源加载与释放时机。
    
- 根据目标平台验证并行 GC、增量 GC 和 Cluster 的效果。
    

不会简单地持续禁用 GC，因为这种方式可能将帧率问题转化为内存峰值问题。

### Q20. CollectGarbage() 和 ForceGarbageCollection() 有什么区别？

概念上：

`CollectGarbage()`：

直接请求执行相应的 GC 收集过程，调用时机必须符合引擎的安全要求。

`GEngine->ForceGarbageCollection()`：

通过引擎提供的接口请求或安排一次 GC，具体执行时机受引擎版本与游戏循环控制。

二者都不应该被当成“立即删除指定 UObject”的接口。

例如：

```
// 可以主动触发一次 GC
// 不建议放在 Tick 或普通技能释放中
CollectGarbage(RF_NoFlags);
```

在性能敏感的游戏中，主动调用 GC 前应先明确它的作用和触发时机，避免产生明显卡顿。

## 十一、经典代码分析题

这类题尤其适合检验是否真正理解 GC，而不是只背概念。

代码分析 1 / 5

以下对象会不会被 GC？

void Test()  
{  
UObject* Obj = NewObject<UObject>();  
}

参考答案：
函数返回后，局部 Obj 消失。如果没有其他 GC 可达引用，新对象就可能在后续 GC 中被回收。并非离开函数就立即析构。

代码分析 2 / 5

给了 Outer，能保证对象存活吗？

void AMyActor::BeginPlay()  
{  
Super::BeginPlay();  
UObject* Obj = NewObject<UObject>(this);  
}

参考答案：
不能。Outer 不自动生成从 Actor 到 Obj 的强引用。Obj 仅作为局部变量时，后续可能被 GC 回收。

代码分析 3 / 5

两个对象互相强引用，会泄漏吗？

// A 和 B 都是 UObject  
A->B = B;  
B->A = A;  
  
// A->B 和 B->A 都是 UPROPERTY 强引用

参考答案：
不一定。只要 A、B 都没有从 GC 根可达的引用路径，它们仍可以被回收。相互引用并不能让一个孤立对象环自动成为根。

代码分析 4 / 5

Actor Destroy 后以下判断是否可靠？

AActor* Target = GetTarget();  
Target->Destroy();  
  
if (Target != nullptr)  
{  
Target->SetActorLocation(FVector::ZeroVector);  
}

参考答案：
不可靠。Destroy 后指针可能非空，但 Actor 已经不可用于正常玩法逻辑。应检查对象有效性，而且通常不应在 Destroy 后继续操作该 Actor

代码分析 5 / 5

这段代码是否可能让对象被提前回收？

UCLASS()  
class UMyOwner : public UObject  
{  
GENERATED_BODY()  
public:  
TObjectPtr<UObject> Data;  
};

参考答案：
可能。仅声明 TObjectPtr 不等于注册 GC 强引用。常规持久成员应添加 UPROPERTY，并保证 UMyOwner 本身可达。

参考答案

不可靠。Destroy 后指针可能非空，但 Actor 已经不可用于正常玩法逻辑。应检查对象有效性，而且通常不应在 Destroy 后继续操作该 Actor。

## 十二、面试前最应该掌握的知识优先级

|优先级|必须掌握的内容|
|---|---|
|P0 极高|Mark-Sweep、Root Set、可达性分析、UPROPERTY|
|P0 极高|TObjectPtr、TWeakObjectPtr、TSoftObjectPtr、UObject 生命周期|
|P0 极高|Actor Destroy、IsValid、Outer、循环引用|
|P1 高|TArray 引用追踪、FGCObject、AddReferencedObjects|
|P1 高|GC 性能分析、对象池、异常引用定位|
|P2 进阶|GC Cluster、并行 GC、增量 GC 和写屏障|
|P2 进阶|引用 Token Stream、TFastReferenceCollector、源码细节|

## 十三、最重要的面试总结

如果面试官让你用一两分钟介绍 UE 的 GC，可以这样组织答案：

> UE 的 GC 主要负责 UObject 及其派生对象的内存管理，核心算法是标记-清除。
> 
> 引擎从 Root Set 和其他 GC 引用来源开始执行可达性分析，通过 UPROPERTY、引用收集接口等方式找到仍被使用的 UObject。
> 
> 对于不可达对象，GC 会在合适的时间执行 BeginDestroy、FinishDestroy 等流程并最终释放内存。
> 
> 开发时我会用 UPROPERTY 配合 TObjectPtr 保存需要长期持有的 UObject，使用 TWeakObjectPtr 保存非拥有型引用，使用 TSoftObjectPtr 管理按需加载的资产。
> 
> Actor Destroy 与 GC 回收是两个阶段。Destroy 先结束 Actor 的游戏生命周期，内存之后由 GC 处理。
> 
> 性能方面，我会关注 UObject 数量、可达性分析和销毁阶段的耗时，并根据实际情况使用对象池、资源按需加载、GC Cluster、并行 GC 或增量 GC 优化，最后使用 Unreal Insights 验证效果。

对于 UE 游戏客户端岗位，我认为最值得深入准备的是三个问题：为什么 UObject 会被意外 GC、为什么对象 Destroy 后还不能释放、以及如何定位战斗过程中 GC 带来的卡顿。 这三个问题能够把底层原理、C++ 引用管理和实际项目经验联系起来。
