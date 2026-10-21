有，而且对于 UE 游戏客户端开发岗位，还有大量比普通 C++ 更重要的面试题。

如果目标是《鸣潮》《原神》这类 3D 动作游戏，或者其他使用 Unreal Engine 的大型商业项目，除了 C++ 基础，面试官往往会重点考察：

- UE 底层架构、反射、GC、对象生命周期。
    
- Actor、Component、CharacterMovement。
    
- 战斗系统、GAS、技能、Buff、碰撞和受击。
    
- 动画系统、Root Motion、Montage、Motion Warping。
    
- 多人战斗、RPC、Replication、客户端预测。
    
- 游戏线程、渲染线程、异步任务、资源管理。
    
- 性能优化、Unreal Insights、引擎源码理解。
    

下面再补充 50 道 UE 游戏客户端专项面试题及参考答案，尽量避开前面已经总结的通用 C++ 内容。以 UE5 的常见机制为主，代码加入中文注释。

# 第一部分：UE 核心架构与对象系统（1～12）

## 1. UE 的反射系统是什么？底层原理是什么？

P0 · 高频

参考答案：

UE 反射（Reflection）是一套在运行时查询和操作类型信息的机制。

标准 C++ 没有提供 UE 所需要的完整反射能力，因此 UE 通过 UHT（Unreal Header Tool，头文件处理工具）扫描特定宏声明，生成辅助代码，并在运行时建立反射元数据。

常用的反射宏包括：

|宏|作用|
|---|---|
|`UCLASS()`|声明参与反射的 UObject 类|
|`USTRUCT()`|声明参与反射的结构体|
|`UENUM()`|声明参与反射的枚举|
|`UPROPERTY()`|标记反射属性|
|`UFUNCTION()`|标记反射函数|
|`GENERATED_BODY()`|接入 UHT 生成的声明|

```
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // 可以在蓝图中读取和修改
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float MoveSpeed = 600.0f;

    // 可以被蓝图调用
    UFUNCTION(BlueprintCallable)
    void Attack();
};
```

<img width="738" height="349" alt="image" src="https://github.com/user-attachments/assets/29bbc337-9765-44e3-8cd1-8884a77c61c9" />


反射系统支撑的主要功能有：蓝图调用、属性编辑、序列化、网络复制、GC 引用遍历等。

面试追问：`UClass` 和 C++ 的 `class` 有什么区别？

C++ 的 class 是语言层面的类型定义；`UClass` 是 UE 中描述某个 UObject 类型的运行时元数据对象，可以查找属性、函数、类型继承关系等。

## 2. 什么是 CDO（Class Default Object）？为什么 UE 需要 CDO？

P0 · 高频

CDO 中文一般称为类默认对象。

对于每个 UObject 类，UE 都会建立一个代表该类默认状态的对象。

CDO 的主要用途是：

1. 保存类的默认属性。
    
2. 为新对象提供默认值来源。
    
3. 支持蓝图类默认属性、对象初始化等机制。
    

```
AMyCharacter::AMyCharacter()
{
    // 设置类默认属性
    MaxHP = 100.0f;

    // 创建默认子组件
    WeaponMesh = CreateDefaultSubobject<UStaticMeshComponent>(
        TEXT("WeaponMesh")
    );
}
```

重点陷阱：为什么不能在 Actor 构造函数中访问游戏世界的运行时数据？

因为构造函数不仅可能在游戏运行中调用，还可能在创建 CDO、编辑器加载、蓝图编译等过程中执行。

构造时不应假设存在已经准备好的游戏世界、玩家控制器或其他运行时 Actor。

需要依赖游戏世界的初始化，通常放到 `BeginPlay()`、`PostInitializeComponents()` 等合适阶段。

## 3. `CreateDefaultSubobject`、`NewObject`、`SpawnActor` 有什么区别？

|方法|用途|典型场景|
|---|---|---|
|`CreateDefaultSubobject`|构造默认子对象|Character 的 Mesh、Camera 组件|
|`NewObject`|动态创建 UObject|数据对象、非 Actor 逻辑对象、运行时组件|
|`SpawnActor`|在 World 中生成 Actor|角色、敌人、子弹、道具|

```
// 1. 在构造函数中创建默认组件
Camera = CreateDefaultSubobject<UCameraComponent>(
    TEXT("Camera")
);

// 2. 运行时创建 UObject
UMyData* Data = NewObject<UMyData>(this);

// 3. 在世界中生成 Actor
AActor* Bullet = GetWorld()->SpawnActor<AActor>(
    BulletClass,
    SpawnLocation,
    SpawnRotation
);
```

面试追问：运行时通过 NewObject 创建 ActorComponent 就能直接使用吗？

通常还需要根据组件类型和用途完成注册、附着、初始化等操作，例如调用 `RegisterComponent()`。

## 4. `Outer` 是什么？Outer 是不是对象所有权？

`Outer` 是 UE UObject 体系中的外部对象关系，用于对象的命名空间、层级路径、组织关系等。

例如：

```
UMyData* Data = NewObject<UMyData>(this);
```

此处 `this` 是新对象的 Outer。

但要特别注意：

Outer 不等于 C++ 内存所有权，也不意味着 Outer 必然会阻止子对象被 GC 回收。

如果希望对象持续存活，需要让它处于 GC 可识别的强引用关系中，或使用相应的资源持有机制。

这是 UE GC 面试中的一个常见陷阱。

## 5. UE 的 GC（垃圾回收）原理是什么？

P0 · 高频

UE 的 GC 主要管理 UObject 体系中的对象。

其核心思想是分析对象之间的引用关系，从根集合及其他引擎认可的引用入口出发，识别仍然可达的对象。

简化为：

<img width="735" height="404" alt="image" src="https://github.com/user-attachments/assets/12082d8e-d8e5-43eb-9973-992903b11dc1" />


例如：

```
UPROPERTY()
TObjectPtr<UMyData> Data;
```

当该属性属于一个被 GC 正确追踪、且仍然存活的 UObject 时，可以形成 GC 可见的强引用。

需要掌握：

- `UPROPERTY` 与引用追踪。
    
- `TObjectPtr`。
    
- `TWeakObjectPtr`。
    
- `TSoftObjectPtr`。
    
- `AddToRoot`。
    
- `FGCObject` / `AddReferencedObjects`。
    
- `IsValid()`。
    
- `BeginDestroy()`。
    

追问：`TObjectPtr` 就一定能防止 GC 吗？

不是。它需要处于被正确追踪的引用位置，例如标记为 `UPROPERTY` 的成员，才能作为 GC 强引用发挥预期作用。

## 6. `TObjectPtr`、`TWeakObjectPtr`、`TSoftObjectPtr` 怎么选？

|类型|适合场景|
|---|---|
|`TObjectPtr`|持久持有 UObject 的 GC 可追踪成员引用|
|`TWeakObjectPtr`|不希望阻止对象销毁的弱引用|
|`TSoftObjectPtr`|按需加载的资源引用|
|`TStrongObjectPtr`|非 UObject 类型的宿主需要强引用 UObject|
|原始 `T*`|局部变量、参数等短生命周期访问|

比如角色技能系统：

```
// 当前目标，目标可能随时死亡或者被销毁
TWeakObjectPtr<AActor> CurrentTarget;

// 当前角色持有的组件
UPROPERTY()
TObjectPtr<UAbilitySystemComponent> ASC;

// 尚未加载的特效资源
UPROPERTY(EditDefaultsOnly)
TSoftObjectPtr<UNiagaraSystem> HitEffect;
```

面试时最好再说明：UObject 的引用关系与 UObject 的 GC 生命周期是两件必须同时考虑的事情。

## 7. Actor 从创建到销毁会经历什么生命周期？

P0 · 高频

对于运行时生成的 Actor，典型流程是：

```
SpawnActor
   ↓
PostActorCreated
   ↓
ExecuteConstruction / OnConstruction
   ↓
PreInitializeComponents
   ↓
组件初始化
   ↓
PostInitializeComponents
   ↓
BeginPlay
   ↓
Tick（如果启用）
   ↓
EndPlay
   ↓
后续销毁及 GC 清理
```

实际流程会因从磁盘加载、PIE、延迟生成、网络生成等途径有所不同。

面试追问：为什么 Destroy 之后不应该继续操作 Actor？

`Destroy()` 不是对 Actor 直接执行 C++ `delete`。

它会使 Actor 进入销毁流程，并执行相应的生命周期清理。对象所占的内存不一定立刻释放，但不能因此认为它还可以作为正常 Actor 使用。

应该通过 `EndPlay()`、组件释放、事件解绑等机制清理运行时状态。

## 8. `UActorComponent`、`USceneComponent`、`UPrimitiveComponent` 的区别？

|类型|功能|
|---|---|
|`UActorComponent`|普通组件，不具备空间变换功能|
|`USceneComponent`|增加位置、旋转、缩放及层级附着|
|`UPrimitiveComponent`|在 SceneComponent 基础上增加几何体、渲染或碰撞相关能力|

例如：

```
ACharacter
 ├─ CapsuleComponent
 ├─ SkeletalMeshComponent
 ├─ CharacterMovementComponent
 ├─ CameraComponent
 └─ AbilitySystemComponent
```

其中：

- `CapsuleComponent`：角色胶囊体和碰撞。
    
- `SkeletalMeshComponent`：骨骼网格体及动画。
    
- `CharacterMovementComponent`：角色移动。
    
- `CameraComponent`：摄像机。
    
- `AbilitySystemComponent`：技能系统。
    

实际面试题：如果要实现一个锁定目标组件，你会继承什么？

如果组件只负责寻找目标、保存当前目标、计算切换目标优先级，一般继承 `UActorComponent` 就够了。

如果需要组件自身具备空间坐标及挂接层级，可以考虑 `USceneComponent`。

## 9. UE 的 Tick 执行顺序是什么？如何优化？

UE 可以按照 Tick Group（Tick 组）组织每帧更新。

常见的有：

- `TG_PrePhysics`：物理模拟前。
    
- `TG_DuringPhysics`：物理模拟期间。
    
- `TG_PostPhysics`：物理模拟完成后。
    
- `TG_PostUpdateWork`：后续更新阶段。
    

Actor 和 Component 还可以设置 Tick 依赖关系，以确保先后执行顺序。


```
AMyActor::AMyActor()
{
    PrimaryActorTick.bCanEverTick = true;

    // 设置更新分组
    PrimaryActorTick.TickGroup = TG_PostPhysics;

    // 每 0.1 秒更新一次，而不是每帧更新
    PrimaryActorTick.TickInterval = 0.1f;
}
```

优化思路：

1. 能事件驱动就不每帧轮询。
    
2. 不需要每帧更新的逻辑降低 Tick 频率。
    
3. 不活跃对象可以关闭 Tick。
    
4. 大量同类对象可以使用集中式更新管理。
    
5. 不要把所有角色逻辑都塞到一个 Tick 中。
    

追问：Timer 一定比 Tick 性能好吗？

不一定。Timer 也有调度开销，仍然依赖引擎时间更新机制。

Timer 适合较低频率、延迟或定时事件，但不是无条件更快的 Tick 替代品。

## 10. GameInstance、GameMode、GameState、PlayerState、PlayerController 的作用是什么？

P0 · 高频

这是 UE 游戏框架中最容易连环追问的题目。

|类|核心职责|
|---|---|
|`UGameInstance`|游戏实例级状态，通常跨普通地图切换保留|
|`AGameModeBase`|服务器上的游戏规则、出生、比赛流程|
|`AGameStateBase`|客户端需要知道的公共对局状态|
|`APlayerState`|玩家身份、得分等玩家级复制状态|
|`APlayerController`|输入和控制玩家 Pawn|
|`APawn`|可被 Controller 控制的 Actor|
|`ACharacter`|具备角色移动等预设能力的 Pawn|

例如在多人 Boss 战中：

- GameMode 决定副本胜利条件。
    
- GameState 同步 Boss 战阶段、对局倒计时。
    
- PlayerState 保存需要同步的玩家信息。
    
- PlayerController 处理本地输入和服务器控制关系。
    
- Character 执行移动、攻击、受击。
    

重点：GameMode 只在服务器存在，GameState 会复制到客户端。

## 11. 什么是 Possess？为什么网络游戏中 Possess 很重要？

`Possess` 是 Controller 获得对 Pawn 控制权的过程。

```
APlayerController* PC = /* 获取玩家控制器 */;
APawn* Pawn = /* 获取目标 Pawn */;

if (PC && Pawn && PC->HasAuthority())
{
    PC->Possess(Pawn);
}
```

在默认的服务器权威多人模式中，正式的控制权分配应由服务器管理。

Possess 还与以下内容密切相关：

- 输入控制。
    
- Pawn 所属关系。
    
- Autonomous Proxy。
    
- Server RPC 调用权限。
    
- GAS 的 OwnerActor / AvatarActor 初始化。
    
- 玩家切换角色或重新出生。
    

面试追问：Actor 的 Owner、Outer、AttachParent、Controller 是不是一回事？

不是。

它们分别描述不同的对象关系。尤其是 `Owner` 与网络连接归属相关，而 `Outer` 不代表网络控制权。

## 12. UE 的 Subsystem 有什么作用？

Subsystem 是 UE 提供的一套子系统生命周期和访问框架。

常用类型：

|类型|典型用途|
|---|---|
|`UEngineSubsystem`|引擎级管理服务|
|`UGameInstanceSubsystem`|游戏实例级服务|
|`UWorldSubsystem`|绑定到特定 World 的服务|
|`ULocalPlayerSubsystem`|本地玩家级服务|

例如：

```
UCLASS()
class UTargetManagerSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    void UpdateTargets();
};
```

```
// 获取当前 World 对应的子系统
UTargetManagerSubsystem* Manager =
    GetWorld()->GetSubsystem<UTargetManagerSubsystem>();
```

实际应用：

- 锁定目标管理。
    
- 战斗事件调度。
    
- 局部世界 AI 管理。
    
- 本地输入配置。
    
- 存档系统、账号服务。
    

面试注意：Subsystem 不是线程安全的保证，也不是万能单例。选择时要考虑服务应该属于哪个生命周期。

# 第二部分：UE C++ 工程与 Gameplay 开发（13～20）

## 13. UE 的 Delegate（委托）有哪些类型？

Delegate 本质上是一套类型安全的回调机制。

常见类型包括：

- 单播委托：绑定一个回调。
    
- 多播委托：绑定多个回调。
    
- 动态委托：支持 UE 反射相关功能。
    
- 动态多播委托：常用于蓝图事件广播。
    

例如：

```
// 声明一个传递 float 参数的多播委托
DECLARE_MULTICAST_DELEGATE_OneParam(
    FOnHPChanged,
    float
);

class FHealthSystem
{
public:
    FOnHPChanged OnHPChanged;

    void SetHP(float NewHP)
    {
        // 通知所有监听者
        OnHPChanged.Broadcast(NewHP);
    }
};
```

面试追问：`AddUObject`、`AddRaw`、`AddLambda` 有什么区别？

- `AddUObject`：绑定 UObject 成员函数，具备 UObject 生命周期相关的安全检查机制。
    
- `AddRaw`：绑定普通 C++ 对象成员函数，需要保证对象生命周期。
    
- `AddLambda`：绑定 Lambda，特别注意引用捕获和生命周期。
    

事件解绑、异步回调和对象销毁，是委托面试经常涉及的重点。

## 14. UInterface 接口与 C++ 普通虚接口有什么区别？

UE 接口一般由 `UINTERFACE` 和对应 `I...` 类型共同组成。

```
UINTERFACE(BlueprintType)
class UDamageable : public UInterface
{
    GENERATED_BODY()
};

class IDamageable
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintNativeEvent)
    void ReceiveDamage(float Damage);
};
```

优势是能够接入 UE 类型系统和蓝图。

需要注意的是：对于允许蓝图实现的接口函数，调用时通常需要通过生成的 `Execute_` 方法，而不只是简单调用 C++ 虚函数。

例如：

```
if (Target &&
    Target->GetClass()->ImplementsInterface(
        UDamageable::StaticClass()))
{
    IDamageable::Execute_ReceiveDamage(Target, 50.0f);
}
```

这类接口经常用于伤害、交互、锁定和通用游戏事件。

## 15. `FName`、`FString`、`FText` 的区别是什么？

|类型|主要用途|
|---|---|
|`FName`|名称标识、查找键、内部标记|
|`FString`|字符串处理、拼接、路径|
|`FText`|面向用户显示、文本本地化|

```
FName SocketName = TEXT("WeaponSocket");

FString PlayerName = TEXT("Player01");

FText DisplayName =
    NSLOCTEXT("UI", "Start", "开始游戏");
```

高频追问：

为什么骨骼 Socket 名称常使用 FName？

因为它适合反复使用的名称标识，并支持高效比较。

为什么 UI 的多语言文本更适合 FText？

因为 `FText` 提供了 UE 本地化体系所需要的语义信息，而 `FString` 主要是普通字符串容器。

## 16. `UDataTable`、`UPrimaryDataAsset` 和配置文件怎么选？

`UDataTable` 常用于结构统一的大量表格数据，例如武器基础参数、角色数值、关卡配置。

`UPrimaryDataAsset` 适合结构复杂、需要引用多个资源的独立数据资产，例如技能配置、武器资产、角色定义。

例如：

```
USTRUCT(BlueprintType)
struct FWeaponRow : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    float Damage = 100.0f;

    UPROPERTY(EditAnywhere)
    float AttackInterval = 0.5f;
};
```

实际项目中还会追问：

- 数据配置什么时候加载？
    
- 如何避免所有技能资源随配置一起加载？
    
- 如何按角色、关卡、玩法分组管理资产？
    
- 热更新后的配置如何校验？
    
- 多人游戏如何保证客户端与服务器配置一致？
    

## 17. `GameplayTag` 和枚举有什么区别？

`GameplayTag` 是层级化的标签系统。

例如：

```
State.Combat
State.Combat.Attacking
State.Combat.Stunned

Ability.Attack.Light
Ability.Attack.Heavy
Ability.Dodge
```

它比普通枚举更适合表达复杂的动态组合状态。

```
// 判断角色是否处于眩晕状态
if (ASC->HasMatchingGameplayTag(StunTag))
{
    // 禁止执行某些操作
}
```

优势是可扩展、支持层级匹配、方便和 GAS 集成。

但也不是所有枚举都应该替换成 Tag。角色移动模式这种固定且需要高频 switch 处理的状态，枚举依然很合适。

## 18. UE 中如何实现技能冷却（CD）？

这是非常典型的游戏客户端业务题。

不使用 GAS，可以保存冷却结束时间：

```
float NextAttackTime = 0.0f;

bool CanAttack() const
{
    return GetWorld()->GetTimeSeconds()
        >= NextAttackTime;
}

void Attack()
{
    if (!CanAttack())
    {
        return;
    }

    // 冷却 2 秒
    NextAttackTime =
        GetWorld()->GetTimeSeconds() + 2.0f;

    // 执行攻击逻辑
}
```

使用 GAS 时，通常可以通过 GameplayEffect 表达技能冷却，并使用 GameplayTag 管理冷却状态。

面试追问：多人游戏中 CD 应以客户端还是服务器为准？

服务器应当是最终权威。

客户端可以预测冷却显示和技能激活反馈，但服务器仍需要校验冷却是否结束。

## 19. UE 如何实现可取消的技能？

例如角色正在攻击时受到重击，需要打断攻击。

典型的技能生命周期：

```
请求释放技能
    ↓
校验释放条件
    ↓
进入技能执行状态
    ↓
播放 Montage
    ↓
产生攻击判定
    ↓
等待技能完成
    ↓
恢复可操作状态
```

收到打断事件时：

```
收到眩晕 / 重击
    ↓
判断当前技能能否被打断
    ↓
取消技能
    ↓
停止或切换动画
    ↓
关闭攻击判定
    ↓
清理临时状态
    ↓
进入受击状态
```

重点是：不能只停止 Montage。

还需要同步处理逻辑上的技能状态、输入锁、碰撞窗口、临时组件、Root Motion 以及可能正在执行的异步任务。

如果使用 GAS，还可以利用 GameplayTag 和 `CancelAbility()` 等机制组织取消关系。

## 20. UE 中如何实现事件驱动的战斗系统？

可以将战斗拆成多个相互协作的模块：

<img width="738" height="415" alt="image" src="https://github.com/user-attachments/assets/e1b5352f-5aaf-4169-bab5-47b1a8c489c1" />


常见事件：

```
OnAttackStarted
OnHitConfirmed
OnDamageApplied
OnCharacterStunned
OnAbilityEnded
OnTargetChanged
```

设计重点在于：

- 逻辑依赖方向明确。
    
- 事件订阅和解绑安全。
    
- 防止事件递归触发。
    
- 明确技能取消时如何清理。
    
- 战斗权威状态与纯表现状态分离。
    

# 第三部分：GAS 技能系统（21～27）

这一部分对于 UE 动作游戏、RPG、多人战斗岗位非常重要。

## 21. GAS 的 ASC 应该放在 Character 还是 PlayerState 上？

ASC（Ability System Component，技能系统组件）既可以放在 Character 上，也可以放在 PlayerState 上。

放在 Character 上：

适合技能系统生命周期与角色基本一致的情况，例如普通怪物、Boss。

放在 PlayerState 上：

适合玩家死亡、重生、切换 Pawn 时，希望技能和属性状态跨角色切换保留的情况。

需要区分两个概念：

- `OwnerActor`：ASC 的逻辑所有者。
    
- `AvatarActor`：ASC 当前代表并操作的角色实体。
    

例如 ASC 存放在 PlayerState：

```
void AMyCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);

    // 服务器初始化
    if (AMyPlayerState* PS = GetPlayerState<AMyPlayerState>())
    {
        UAbilitySystemComponent* ASC =
            PS->GetAbilitySystemComponent();

        // OwnerActor = PlayerState
        // AvatarActor = Character
        ASC->InitAbilityActorInfo(PS, this);
    }
}
```

客户端也需要在适当的复制生命周期中初始化，例如通过 `OnRep_PlayerState()`。

常见 Bug： ASC 已经创建，但没有正确初始化 ActorInfo，导致技能获取 AvatarActor、Controller、动画实例等信息失败。

## 22. GA、GE、ASC、AttributeSet、GameplayCue、AbilityTask 之间是什么关系？

|组件|作用|
|---|---|
|ASC|技能管理中心，管理技能、属性、标签、效果等|
|GameplayAbility（GA）|描述技能执行流程|
|GameplayEffect（GE）|修改属性、管理 Buff 和 Debuff|
|AttributeSet|定义生命、攻击、防御等属性|
|GameplayCue|处理视觉、音频等表现|
|AbilityTask|在技能执行期间管理等待、动画和异步事件|

例如一个火球技能：

```
玩家按下技能键
      ↓
ASC 请求激活 GameplayAbility
      ↓
检查蓝量、CD、状态标签
      ↓
GameplayAbility 执行
      ↓
播放施法 Montage
      ↓
AbilityTask 等待施法事件
      ↓
生成火球 / 检测命中
      ↓
GameplayEffect 计算伤害
      ↓
AttributeSet 修改 HP
      ↓
GameplayCue 播放命中特效
      ↓
EndAbility
```

面试官很可能继续问：为什么 GA 不应该直接负责所有伤害数值和所有特效逻辑？

因为 GA 更适合组织技能流程，而数值计算、状态管理、表现反馈分别交给专门模块，更方便复用、网络同步和维护。

## 23. GameplayEffect 的三种持续时间有什么区别？

主要有：

- `Instant`：即时生效，例如直接扣血。
    
- `HasDuration`：持续一段时间，例如 10 秒攻击力提升。
    
- `Infinite`：持续存在，直到被移除。
    

Attribute 还区分：

BaseValue（基础值） 与 CurrentValue（当前值）。

例如：

```
基础攻击力 BaseValue = 100

获得一个 +20% 的持续 Buff

当前攻击力 CurrentValue = 120

Buff 结束

CurrentValue = 100
```

实际计算可能还要考虑多个 Modifier 的聚合顺序、运算类型和其他效果。

即时 GameplayEffect 通常影响基础值，持续性 Modifier 通常影响当前值。

## 24. GameplayAbility 有哪些实例化策略？

|策略|特点|
|---|---|
|NonInstanced|不为每次使用创建独立技能对象，限制较多|
|InstancedPerActor|每个拥有者保留一个可复用技能实例|
|InstancedPerExecution|每次执行创建独立实例|

选择时考虑：

- 技能是否需要保存执行状态？
    
- 是否需要多个执行实例同时存在？
    
- 是否需要使用异步 AbilityTask？
    
- 是否需要蓝图实现？
    
- 对象创建和复制开销能否接受？
    

对于复杂的普通战斗技能，`InstancedPerActor` 是很常见的选择，但它要求在技能结束和取消时认真清理本次执行状态。

对于可并发执行的复杂技能，可以考虑 `InstancedPerExecution`。

## 25. 如何用 GameplayTag 实现技能互斥、打断和霸体？

例如：

```
State.Attacking
State.Dodging
State.Stunned
State.SuperArmor
State.Dead
```

假设正在释放普通攻击：

```
激活普通攻击
    ↓
添加 State.Attacking
    ↓
如果收到眩晕：
    ├─ 有 State.SuperArmor
    │      └─ 不打断攻击
    │
    └─ 没有 State.SuperArmor
           └─ 取消当前攻击
                  ↓
             进入眩晕状态
```

这可以通过 GAS 的 Ability Tags、Activation Blocked Tags、Cancel Abilities With Tag 等相关机制组织。

面试重点：霸体不一定代表完全不受伤。

霸体通常影响的是受击硬直、击退、技能打断等规则，而是否扣血仍然由伤害系统决定。

## 26. `MMC` 与 `ExecutionCalculation` 有什么区别？

MMC（Modifier Magnitude Calculation）用于计算 GameplayEffect 某个 Modifier 的数值大小。

ExecutionCalculation 则适用于更复杂的执行型数值计算，例如涉及攻击者、目标防御、暴击、护盾和多属性修改的伤害结算。

例如伤害计算：

```
// 伪代码：展示伤害公式，不是完整 GAS Execution 实现

float CalculateDamage(
    float Attack,
    float SkillMultiplier,
    float DefenseRate,
    bool bCritical)
{
    float Damage =
        Attack * SkillMultiplier * (1.0f - DefenseRate);

    if (bCritical)
    {
        Damage *= 1.5f;
    }

    return FMath::Max(0.0f, Damage);
}
```

多人游戏中，涉及真实生命值和资源消耗的最终结算应由服务器权威执行。

## 27. GAS 的客户端预测是怎么工作的？

GAS 支持多种技能网络执行策略，其中 `LocalPredicted`（本地预测）非常重要。

典型过程：

```
客户端按下闪避
     ↓
客户端立即预测释放
     ↓
播放动画、启动局部反馈
     ↓
向服务器提交技能请求
     ↓
服务器校验技能条件
     ↓
校验成功 → 确认预测
校验失败 → 拒绝并修正预测状态
```

其中 Prediction Key（预测键）用于关联某些预测行为和服务器确认。

但要注意：GAS 并不是自动支持任何 Gameplay 逻辑的完整回滚。

例如你自行生成的 Actor、修改的外部数组、播放的特殊表现，不一定会被预测系统自动撤销，需要开发者管理。

# 第四部分：动画与战斗表现（28～34）

## 28. Animation Blueprint 的工作原理是什么？

Animation Blueprint（动画蓝图）主要用于计算角色最终骨骼姿态。

通常包含两个概念：

- Event Graph：维护动画所需状态，例如速度、是否在空中。
    
- Anim Graph：计算动画节点、混合状态及最终 Pose。
    

动画姿态计算可以在符合条件时采用并行更新和求值。

面试追问：为什么不能在动画线程里随意访问 Character 的 UObject 数据？

因为 UObject 及大量 Gameplay 数据主要由游戏线程管理，直接跨线程读写可能导致数据竞争和生命周期问题。

实际项目中，应使用引擎支持的线程安全属性访问方式、动画代理或安全快照，避免动画线程直接操作不安全的 Gameplay 对象。

## 29. State Machine、BlendSpace、Montage 有什么区别？

|系统|主要用途|例子|
|---|---|---|
|State Machine|状态间的动画切换|Idle、Run、Jump|
|BlendSpace|根据参数混合动画|根据速度与方向混合跑步|
|Montage|主动控制动画片段和播放流程|攻击、闪避、受击|

实际游戏中经常组合使用：

```
基础移动 State Machine
          ↓
移动 BlendSpace
          ↓
Slot / Montage 混合
          ↓
受击、攻击等上层动作
          ↓
最终骨骼姿态
```

Montage 支持 Sections（分段）、Slots（插槽）、Notifies（通知）等功能，因此适合需要精确控制执行过程的战斗动作。

## 30. AnimNotify 和 AnimNotifyState 有什么区别？

`AnimNotify` 是单个时间点触发的动画事件。

`AnimNotifyState` 定义一段持续时间内的状态，提供开始、更新和结束等回调。

例如攻击动画：

```
0.0s         0.3s          0.7s        1.0s
 |------------|--------------|-----------|
 攻击前摇      打开命中检测     关闭检测    动画结束
              <----------->
               NotifyState
```

一般可以通过 AnimNotifyState 管理武器的攻击判定窗口。

面试追问：如果 Montage 被强制打断，如何保证攻击判定关闭？

不能只依赖动画正常播放到结束。

还应该在技能取消、Montage 中断、角色死亡等逻辑路径中显式清理攻击状态。

## 31. Root Motion 是什么？有哪些优缺点？

Root Motion（根骨骼运动）是从动画中提取根骨骼位移，并应用到角色移动的机制。

例如一次突进攻击，动画本身就包含角色向前移动 3 米的轨迹。

优点：

- 角色移动与动画容易保持一致。
    
- 动作表现更自然。
    
- 适合冲刺攻击、处决、特殊位移。
    

缺点：

- 动画对位移距离的约束更明显。
    
- 动态调整距离需要额外处理。
    
- 网络预测和同步会更复杂。
    
- 还需要正确处理碰撞、阻挡和角色移动模式。
    

UE 的 CharacterMovement 能够对受支持的 Root Motion 移动进行集成处理。多人游戏通常更倾向使用 `Root Motion from Montages Only`，而不是无差别提取所有动画的 Root Motion。

## 32. Motion Warping 是什么？为什么动作游戏需要它？

Motion Warping（运动扭曲）用于动态修改动画 Root Motion，使角色移动能够匹配指定目标。

例如角色有一套固定距离的突进斩击动画，但敌人可能位于不同距离。

```
角色位置 A
       ↓
技能确定目标位置 B
       ↓
计算目标 Warp Transform
       ↓
播放攻击 Montage
       ↓
Motion Warping 修改部分 Root Motion
       ↓
角色在指定时机接近目标 B
```

常见应用：

- 追击攻击。
    
- 处决动画。
    
- 双人交互。
    
- 攀爬。
    
- Boss 抓取。
    
- 特殊闪避和位移技能。
    

实际面试中还会问：Motion Warping 和简单修改 Actor 位置有什么区别？

Motion Warping 可以在指定动画时间窗口内调整运动轨迹，使角色更自然地到达目标位置，而不是直接瞬间传送。

它不是万能的碰撞处理器，目标非法、路径被阻挡、网络预测不一致等仍然需要 Gameplay 逻辑处理。

## 33. Motion Matching 与传统动画状态机有什么区别？

Motion Matching（运动匹配）是一种基于数据搜索的动画选择技术。

传统状态机通常先决定当前状态，再根据状态转换条件选择动画。

Motion Matching 则根据当前角色状态、运动轨迹、骨骼姿态等特征，从动画数据库中搜索更合适的动画姿态。

例如：

```
角色当前速度
       +
未来运动轨迹
       +
当前骨骼姿态
       ↓
构建查询特征
       ↓
Pose Search 数据库
       ↓
搜索匹配姿态
       ↓
播放 / 混合结果
```

UE 的 Pose Search 系统支持这种工作流。


实际应用中还要考虑：

- 动画数据库的覆盖程度。
    
- 运行时搜索成本。
    
- 姿态切换连续性。
    
- 动画压缩和内存。
    
- 动画选择的稳定性。
    

## 34. 如何实现高速武器的准确命中检测？

这是动作游戏岗位很有价值的一道实战题。

假设武器在一帧内移动了很远，如果只检测当前帧的位置，可能出现穿透目标但没有命中的情况。

一种常见方案是连续轨迹 Sweep 检测。

保存上一帧武器 Socket 的位置，再检测上一帧到当前帧之间的运动路径。

```
// 代码示意：在攻击判定窗口内执行
void AMyCharacter::UpdateWeaponTrace()
{
    if (!HasAuthority() || !bAttackTracing)
    {
        return;
    }

    // 获取当前帧武器检测点位置
    const FVector CurrentPos =
        WeaponMesh->GetSocketLocation(TEXT("WeaponTip"));

    FCollisionShape Shape =
        FCollisionShape::MakeSphere(12.0f);

    TArray<FHitResult> HitResults;

    // 检测上一帧到当前帧的移动轨迹
    GetWorld()->SweepMultiByChannel(
        HitResults,
        PreviousWeaponPos,
        CurrentPos,
        FQuat::Identity,
        ECC_Pawn,
        Shape
    );

    for (const FHitResult& Hit : HitResults)
    {
        AActor* Target = Hit.GetActor();

        if (!IsValid(Target) || Target == this)
        {
            continue;
        }

        // 防止同一次攻击重复命中同一个目标
        if (AlreadyHitActors.Contains(Target))
        {
            continue;
        }

        AlreadyHitActors.Add(Target);

        // 通过伤害系统进行服务器权威结算
        ApplyCombatDamage(Target);
    }

    // 保存当前帧位置
    PreviousWeaponPos = CurrentPos;
}
```

这只是单个检测点的简单实现。

实际项目可能会在武器上设置多个 Socket，使用 Capsule Sweep、多个采样点或其他连续碰撞方案，覆盖武器完整运动体积。

面试追问：如何处理目标一帧内被命中多次？

可以使用攻击实例 ID 和已经命中的 Actor 集合。

多段攻击则需要根据设计在不同攻击段之间清理或更新命中记录。

# 第五部分：多人游戏和网络同步（35～43）

## 35. UE 默认使用什么网络同步模型？

UE 经典多人 Gameplay 框架主要采用 Server Authoritative（服务器权威） 模型。

即服务器负责决定游戏中的权威状态，客户端负责发送输入或动作请求，并渲染本地和同步得到的结果。

```
客户端输入
    ↓
本地立即反馈 / 预测
    ↓
Server RPC
    ↓
服务器校验
    ↓
服务器执行权威逻辑
    ↓
状态复制到客户端
    ↓
客户端插值、预测修正、显示
```

这与纯粹的帧同步不是同一个机制。

对于多人动作游戏，UE 提供的状态复制、角色移动预测和 RPC 是重要基础。

## 36. Server RPC、Client RPC、NetMulticast RPC 的区别？

|类型|通常由谁调用|执行位置|
|---|---|---|
|Server RPC|拥有该 Actor 的客户端|服务器|
|Client RPC|服务器|对应的拥有客户端|
|NetMulticast RPC|服务器|服务器及相关客户端|

示例：

```
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // 客户端请求服务器释放技能
    UFUNCTION(Server, Reliable)
    void ServerUseSkill(int32 SkillID);

    // 服务器通知拥有者
    UFUNCTION(Client, Reliable)
    void ClientSkillRejected(int32 SkillID);

    // 服务器广播瞬时表现事件
    UFUNCTION(NetMulticast, Unreliable)
    void MulticastPlayHitEffect();
};
```

```
void AMyCharacter::ServerUseSkill_Implementation(
    int32 SkillID)
{
    // 服务器不能直接相信客户端请求
    // 必须检查技能是否存在、CD、状态和资源消耗等

    if (!CanUseSkillOnServer(SkillID))
    {
        ClientSkillRejected(SkillID);
        return;
    }

    ExecuteSkillOnServer(SkillID);
}
```

重点陷阱：客户端可以对任意 Actor 发起 Server RPC 吗？

不可以。RPC 的远程执行与拥有连接（Owning Connection）有关，客户端通常只能从自己拥有的 Actor 或相关组件发起有效的 Server RPC。

另外，客户端直接调用 NetMulticast 并不能广播到其他客户端。

## 37. `Replicated` 和 `ReplicatedUsing` 有什么区别？

`Replicated` 用来复制属性。

`ReplicatedUsing` 还会绑定一个 RepNotify 函数，用于在客户端收到相应属性更新后执行逻辑。

例如：

```
// MyCharacter.h

UPROPERTY(ReplicatedUsing = OnRep_HP)
float HP = 100.0f;

UFUNCTION()
void OnRep_HP();
```

```
// MyCharacter.cpp

#include "Net/UnrealNetwork.h"

AMyCharacter::AMyCharacter()
{
    // 允许 Actor 复制
    bReplicates = true;
}

void AMyCharacter::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyCharacter, HP);
}

void AMyCharacter::OnRep_HP()
{
    // 客户端收到更新后刷新血条
    RefreshHPUI();
}
```

注意：

- RepNotify 主要用于处理接收端状态更新。
    
- 服务器直接修改 HP 时，并不意味着本机一定会自动执行 `OnRep_HP()`。
    
- 网络属性复制通常传递状态，而不是保证中间每一次变化都会被观察到。
    

如果血量在两次网络发送之间从 100 变成 90、再变成 80，客户端可能只观察到最终的 80。

## 38. Authority、Autonomous Proxy、Simulated Proxy 有什么区别？

UE 网络角色中需要理解：

Authority

服务器上具有权威性的 Actor 实例。

Autonomous Proxy

通常是客户端本地玩家控制的 Pawn，具备本地移动预测能力。

Simulated Proxy

通常是其他客户端看到的远端角色，通过接收网络状态进行模拟和视觉平滑。

例如：

```
玩家 A 的客户端：

自己的角色 A
    → Autonomous Proxy

其他玩家 B
    → Simulated Proxy

服务器上的角色 A、B
    → Authority
```

实际还要考虑 Listen Server、本地控制和网络角色之间的区别。

不能简单地认为“Authority 一定不是本地玩家”。

## 39. CharacterMovement 的客户端预测和服务器纠正原理是什么？

<img width="739" height="279" alt="image" src="https://github.com/user-attachments/assets/35ee5b5b-48e4-4e03-8e08-bd2847f20e08" />


UE 的 `UCharacterMovementComponent` 包含预测、保存移动、服务器校验和客户端修正等机制。

相关源码重点：

- `UCharacterMovementComponent`
    
- `FNetworkPredictionData_Client_Character`
    
- `FSavedMove_Character`
    
- `ReplicateMoveToServer()`
    
- `ServerMove` 相关处理
    
- `ClientAdjustPosition` 相关处理
    
- `SmoothClientPosition()`
    

如果需要扩展冲刺、滑铲等自定义移动，可以研究 Custom Movement Mode 和 SavedMove 体系。

官方文档对这条链路有较完整的描述。

## 40. 为什么服务器位置和客户端视觉位置有时不同？

这是多人动作游戏的高级面试题。

原因包括：

- 网络延迟。
    
- 客户端预测。
    
- 服务器权威纠正。
    
- 网络插值和平滑。
    
- Mesh 与碰撞胶囊体的视觉偏移。
    

例如某一瞬间：

```
客户端预测的逻辑位置：10.0
服务器确认的位置：  9.5
客户端正在显示的位置：10.0
```

收到修正后，客户端可能先将逻辑位置修正，再通过网络平滑逐步消除视觉偏差。

攻击检测应该使用哪个位置？

服务器权威伤害结算应基于服务器认可的游戏状态；不能直接把客户端视觉 Mesh 位置作为权威碰撞坐标。

如果需要补偿网络延迟，应建立专门的命中回溯系统。

## 41. 什么是 Lag Compensation（延迟补偿）？

延迟补偿是多人射击和动作游戏的重要机制。

例如客户端看到敌人位于位置 A，但攻击指令传到服务器时，敌人已经移动到了 B。

一种常见方案是服务器保存角色历史位置或碰撞状态。

```
服务器记录历史碰撞快照
             ↓
客户端在本地发起攻击
             ↓
服务器收到攻击请求
             ↓
根据允许的攻击时间估算历史时刻
             ↓
查询 / 重建历史碰撞状态
             ↓
执行命中验证
             ↓
恢复当前状态并进行权威结算
```

这里要考虑：

- 时间戳可信度。
    
- 最大回溯时间。
    
- 网络抖动。
    
- 掩体穿透问题。
    
- 高速移动目标。
    
- 防作弊。
    
- 多玩家之间的公平性。
    

动作游戏的近战攻击还需要考虑攻击轨迹是否也需要回溯，而不只是受击者的位置。

## 42. 如何优化大量 Actor 的网络同步？

可能采取以下策略：

- 降低不重要 Actor 的更新频率。
    
- 根据距离和相关性控制复制。
    
- 使用 Net Dormancy（网络休眠）。
    
- 优化属性复制条件。
    
- 避免不必要的 Reliable RPC。
    
- 对经常变化的数组使用增量复制机制。
    
- 根据项目规模选择合适的网络复制架构。
    

Fast Array Replication（快速数组复制） 是重要考点。

它适用于较大的动态结构体数组，例如背包数据、战斗状态列表等，可以只标记变化的元素并进行增量复制。

但需要正确调用相关 Dirty 标记，而且不能依赖客户端与服务器始终具有完全相同的数组顺序。

另一个加分点是区分 Replication Graph 与 Iris。二者不是简单的叠加关系；Iris 有自己的过滤和优先级机制，具体支持情况依赖引擎版本和项目配置。

## 43. 多人游戏中动画需要同步什么？

很多新手会认为要把每一帧角色的全部骨骼变换复制给其他客户端。

通常不需要这么做。

更常见的方案是同步或推导动画需要的状态，例如：

- 角色位置、速度和朝向。
    
- 移动模式。
    
- 攻击技能 ID。
    
- 技能阶段。
    
- 关键时间信息。
    
- 特殊 Root Motion 状态。
    
- 必须保持一致的 Gameplay 事件。
    

远端客户端通过本地动画系统生成表现。

为什么不直接同步所有骨骼？

因为骨骼数量多，逐帧同步带宽开销很大，并且通常没有必要。

但对于特殊双人动作、处决、精确交互等场景，还需要对时间和位置进行额外同步。

# 第六部分：多线程、资源与性能优化（44～50）

## 44. UE 游戏线程、渲染线程、RHI 线程分别负责什么？

|线程|主要职责|
|---|---|
|Game Thread|游戏逻辑、Actor 更新、输入、主要 Gameplay 操作|
|Render Thread|组织渲染场景数据与绘制工作|
|RHI Thread|某些配置下处理底层图形接口命令|
|Worker Threads|执行任务系统中的并行工作|

具体线程结构和任务分配受平台、RHI 以及引擎配置影响。

UE 通常通过场景代理（例如 `FPrimitiveSceneProxy`）隔离游戏线程对象和渲染线程数据。

这意味着：不应该让渲染线程直接随意访问游戏线程正在修改的 UPrimitiveComponent 成员。

## 45. 如何在 UE 中安全使用异步任务？

例如需要在后台线程执行计算，然后回到游戏线程更新角色状态：

```
#include "Async/Async.h"

void AMyCharacter::StartCalculation()
{
    // 使用弱引用，避免异步完成前 Actor 已被销毁
    TWeakObjectPtr<AMyCharacter> WeakThis(this);

    // 提交线程池任务
    Async(EAsyncExecution::ThreadPool, [WeakThis]()
    {
        // 这里执行与 UObject 无关的纯 C++ 计算
        // 不直接访问 WeakThis 指向的 UObject

        const int32 Result = ExpensivePureCalculation();

        // 切换回游戏线程
        AsyncTask(ENamedThreads::GameThread,
            [WeakThis, Result]()
            {
                if (AMyCharacter* Character = WeakThis.Get())
                {
                    // 在游戏线程操作角色对象
                    Character->ApplyCalculationResult(Result);
                }
            });
    });
}
```

高频追问：为什么不直接在后台线程调用 `SetActorLocation()`？

因为大量 UObject 和 Actor 操作要求在游戏线程中使用，跨线程直接调用可能引发数据竞争以及状态不一致。

后台线程更适合执行：

- 纯数学运算。
    
- 已复制数据上的批量计算。
    
- 与 UObject 解耦的寻路辅助计算。
    
- 数据解码与处理。
    

结果再安全地提交回对应线程。

## 46. UE 如何异步加载资源？为什么使用软引用？

大型游戏不能把全部角色、技能、特效、地图资源一次性加载到内存。

UE 提供了 Asset Manager、Streamable Manager 等资源加载机制。

例如：

```
UPROPERTY(EditDefaultsOnly)
TSoftObjectPtr<UNiagaraSystem> HitEffect;
```

软引用保存对资源的路径级引用关系，不要求资源始终处于已加载状态。

典型过程：

```
角色准备释放技能
       ↓
检查对应特效是否已经加载
       ↓
未加载则请求异步加载
       ↓
加载完成回调
       ↓
检查角色 / 技能是否仍然有效
       ↓
播放特效
```

需要注意：

- 不能假设加载回调一定在需要资源的那个时刻完成。
    
- 加载期间角色可能死亡或切换地图。
    
- 加载完成后还需要明确资源的持有关系。
    
- 第一次使用大型资源可能引发额外的初始化开销。
    

官方文档也建议使用软引用避免无意间形成大量硬依赖。

## 47. 如果场景中有几千个怪物，如何优化客户端性能？

这是一道典型的综合设计题。

可以从几个层次回答：

逻辑更新层

- 对远离玩家的 AI 降低更新频率。
    
- 使用 AI LOD。
    
- 减少每个怪物独立执行的 Tick。
    
- 批量处理适合批量计算的逻辑。
    
- 不必每帧都做寻路和感知检测。
    

动画层

- 使用 Animation Update Rate 优化。
    
- 远距离降低动画更新频率。
    
- 合理配置骨骼 LOD。
    
- 不可见且不影响逻辑的角色降低动画计算成本。
    

渲染层

- 使用合适的 LOD 和可见性剔除。
    
- 减少不必要的材质复杂度。
    
- 优化阴影、粒子、透明物体。
    
- 对适用对象使用实例化渲染。
    

内存层

- 减少高频对象分配。
    
- 合理使用对象池。
    
- 优化数据结构的缓存局部性。
    
- 避免大规模重复加载资源。
    

面试重点： 不应直接承诺“几千个怪物都可以使用完整 Character、完整动画蓝图和完整 AI 每帧更新”。

对于极大量单位，可以考虑轻量化实体表示、分层模拟或者 Mass Entity 等数据导向方案。

## 48. Unreal Insights 怎么定位卡顿？

Unreal Insights 是 UE 性能分析的重要工具。

常见模块：

|工具|用途|
|---|---|
|Timing Insights|分析 CPU 时间线、线程及任务耗时|
|Memory Insights|分析内存分配、释放及潜在泄漏|
|Networking Insights|分析网络流量与复制相关问题|
|Slate Insights|分析 UI 更新成本|
|Asset Loading Insights|分析资源加载耗时|

例如游戏突然卡顿：

```
发现某一帧耗时异常
          ↓
查看 Game Thread / Render Thread / GPU
          ↓
定位时间线上的异常区间
          ↓
展开函数调用与事件
          ↓
识别异常工作
          ↓
优化并重新测量
```

还应该熟悉：

```
stat unit
stat game
stat gpu
stat rhi
stat memory
```

这些命令可以帮助初步判断瓶颈位置，但应结合平台、构建配置及更详细的性能采样结果分析。

## 49. 如何判断瓶颈在游戏线程、渲染线程还是 GPU？

需要区分不同阶段的耗时。

例如针对 60 FPS 的目标，单帧预算大约为 16.67 ms。

|观察现象|可能的问题|
|---|---|
|Game Thread 耗时高|Gameplay、AI、物理、蓝图、GC|
|Render Thread 耗时高|场景提交、渲染状态组织、绘制命令相关 CPU 开销|
|GPU 耗时高|阴影、光照、像素着色、透明、后处理|
|资源加载时尖峰|同步加载、初始化、资源创建|
|特定动作首次播放卡顿|资源加载、动画初始化、PSO 等|

面试追问：Draw Call 少就一定性能好吗？

不是。

Draw Call 是 CPU 提交与渲染组织成本的重要影响因素之一，但 GPU 性能还取决于：

- 顶点数量和顶点处理。
    
- 像素覆盖率与 Overdraw。
    
- 材质着色复杂度。
    
- 阴影渲染成本。
    
- 内存带宽。
    
- 光照和后处理。
    
- 资源状态与同步。
    

因此应结合实际瓶颈判断优化方向。

## 50. UE 的 UMG 和 Slate 如何进行性能优化？

UMG 是构建游戏 UI 的常用框架，底层与 Slate 系统密切相关。

常见性能问题包括：

- 大量 Widget 每帧执行 Tick。
    
- 频繁绑定属性造成不必要的查询。
    
- 列表一次性创建大量子项。
    
- UI 树复杂。
    
- 大量布局失效。
    
- 频繁重建 Widget。
    
- 复杂 UI 特效造成渲染开销。
    

常见优化方案：

- 事件驱动刷新 UI。
    
- 使用列表虚拟化，例如 ListView。
    
- 合理使用 Invalidation 机制。
    
- 对适当的复杂 UI 使用 Retainer Box。
    
- 避免反复创建和销毁大量相同 Widget。
    
- 用 Slate Insights 分析实际更新成本。
    

实际面试题：为什么 Boss 血条不建议每帧查询 ASC 的 HP？

如果血量只有发生变化时才需要刷新 UI，可以注册属性变化回调，在 HP 变化时更新血条。

这样减少无意义的轮询，也让数据和表现之间的关系更加清晰。

# 第七部分：真正能拉开差距的 UE 项目实战题

前面的 50 道主要考察引擎知识。到了中高级 UE 游戏客户端岗位，面试官还可能给出开放性场景，让你现场设计技术方案。

以下这些题目建议重点准备。

|实战问题|面试官真正考察的内容|
|---|---|
|如何设计一个类似《鸣潮》的角色连招系统？|GAS、输入缓存、技能状态、Montage、取消窗口|
|如何设计 Boss 的多阶段战斗 AI？|行为树、状态机、技能选择、事件驱动、性能|
|如何实现角色锁定目标及目标切换？|空间筛选、视野、射线检测、评分、镜头|
|如何实现闪避无敌帧与完美闪避？|动画事件、碰撞判定、时间窗口、技能状态|
|如何实现弹反？|状态窗口、命中事件、攻击方向、受击反馈|
|如何实现击飞、浮空与连击？|MovementMode、受力、动作控制、状态管理|
|如何实现多人协作 Boss 战同步？|服务器权威、客户端预测、攻击回溯、动画同步|
|如何解决大量技能特效引起的卡顿？|Niagara、资源加载、池化、GPU/CPU Profiling|
|如何实现技能运行时动态修改？|数据驱动、GAS、Modifier、运行时属性|
|如何降低复杂战斗系统的模块耦合？|组件化、接口、GameplayTag、委托、状态管理|

## 最后的复习优先级

我会把 UE 游戏客户端面试分成以下几档：

|优先级|知识领域|建议|
|---|---|---|
|P0|C++ 对象模型、内存、STL、多线程|扎实掌握底层原理|
|P0|UE 反射、GC、生命周期、Actor/Component|能结合源码解释|
|P0|CharacterMovement、动画、碰撞与命中|能分析真实战斗需求|
|P0|GAS、连招、受击、技能取消、Buff|能独立设计系统|
|P0|RPC、Replication、移动预测|能解释服务器和客户端的完整流程|
|P1|UE 线程架构、异步任务、资源加载|能解决实际线程与卡顿问题|
|P1|Unreal Insights、CPU/GPU 性能优化|会定位瓶颈并提出验证方案|
|P1|Motion Warping、Motion Matching、IK|熟悉高级角色动画技术|
|P2|UBT/UHT/UAT、引擎模块、源码修改|中高级岗位加分|
|P2|Mass Entity、Replication Graph、Iris|大规模场景和网络架构加分|

对于动作类 UE 客户端岗位，我建议下一步重点深入五个专题：

1. UE 战斗系统架构：从输入、技能、GAS 到伤害、受击、连招的完整实现。
    
2. UE 角色移动源码：CharacterMovement 的 `PerformMovement`、`PhysWalking`、`PhysFalling` 和网络预测流程。
    
3. UE 动画与战斗结合：Montage、Root Motion、Motion Warping、攻击检测、取消窗口。
    
4. UE 网络同步源码：RPC、OnRep、SavedMove、服务器纠正、延迟补偿。
    
5. UE 性能优化和引擎底层：Game Thread、Render Thread、GC、资源加载、Unreal Insights。
    

这五个专题比单独记忆零散的 API 更接近商业动作游戏客户端的实际开发工作，也是从“会使用 UE”走向“理解 UE 底层并能设计系统”的关键。
