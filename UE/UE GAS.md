# UE GAS（Gameplay Ability System）核心组件详解

GAS（Gameplay Ability System，游戏玩法技能系统）是 Unreal Engine 提供的一套技能、属性、Buff/Debuff、战斗状态管理与网络同步框架。

它不仅可以实现普通的技能释放，还能支撑比较复杂的 ARPG、动作游戏、MOBA、多人联机战斗系统。

例如，使用 GAS 可以实现：

- 技能系统：普通攻击、连招、闪避、冲刺、蓄力、格挡、弹反、终结技。
    
- 属性系统：生命值、攻击力、防御力、体力、能量、暴击率。
    
- Buff/Debuff 系统：攻击力提升、减速、中毒、燃烧、眩晕、无敌。
    
- 战斗状态管理：攻击中、闪避中、硬直中、倒地、霸体、无法移动。
    
- 技能限制：消耗体力、冷却时间、状态互斥、技能打断。
    
- 网络同步：客户端预测、服务器权威校验、技能状态复制、属性同步。
    
- 表现系统：技能动画、Niagara 特效、音效、受击反馈。
    

GAS 最大的价值并不是提供一个技能基类，而是将这些系统连接起来，让它们能够以统一的方式协作。

以下以 UE5 的 GAS 架构为基础，结合动作战斗系统进行讲解。

## 一、GAS 整体架构

### 1.1 核心组件总览

先认识 GAS 中最重要的几个类。

|组件/类|中文名称|主要作用|
|---|---|---|
|`UAbilitySystemComponent`（ASC）|技能系统组件|GAS 的核心管理器|
|`UGameplayAbility`（GA）|游戏玩法技能|定义技能的执行流程|
|`UAbilityTask`|技能任务|处理动画、等待事件等异步操作|
|`UAttributeSet`|属性集|管理角色的各种战斗属性|
|`FGameplayAttributeData`|游戏玩法属性数据|保存属性的基础值和当前值|
|`UGameplayEffect`（GE）|游戏玩法效果|处理伤害、治疗、Buff、Debuff、冷却等|
|`FGameplayTag`|游戏玩法标签|表达角色状态、技能类别及限制条件|
|`GameplayCue`（GC）|游戏玩法表现提示|管理特效、音效等视觉和听觉表现|
|`UGameplayModMagnitudeCalculation`（MMC）|效果幅度计算|计算单个属性修改幅度|
|`UGameplayEffectExecutionCalculation`（ExecCalc）|效果执行计算|计算复杂的伤害、治疗等数值|
|`FGameplayAbilitySpec`|技能授予规格|记录角色被授予的技能及相关信息|
|`FGameplayEffectSpec`|效果运行时规格|保存一次 GE 应用所需的运行时参数|
|`FGameplayEventData`|游戏玩法事件数据|在技能和战斗系统之间传递事件|

其中，最重要的六个概念是：

ASC、GA、AttributeSet、GE、GameplayTag、AbilityTask。

掌握这六个部分，就能够理解 GAS 的大部分工作流程。Epic 官方也将技能、属性集、效果、技能任务和 ASC 作为 GAS 的主要组成部分。

### 1.2 各组件之间的关系

<img width="744" height="521" alt="image" src="https://github.com/user-attachments/assets/3055dbc9-7d54-4a40-96d4-97a96c227a2e" />


可以用一句话理解这些组件：

> ASC 负责管理角色的 GAS 状态；GA 决定技能做什么；AbilityTask 决定技能在什么时机继续执行；GE 决定产生什么数值或状态变化；AttributeSet 保存数值；GameplayTag 判断状态；GameplayCue 负责表现。

### 1.3 举例：角色释放一次火焰斩

假设一个角色使用火焰斩：

```
玩家按下技能键
    │
    ▼
AbilitySystemComponent（ASC）
    │
    ├── 查询角色是否拥有火焰斩
    ├── 查询是否有眩晕、沉默等禁止释放的状态
    └── 尝试激活 GameplayAbility
             │
             ▼
       GA_FireSlash
             │
             ├── 检查体力是否足够
             ├── 扣除体力
             ├── 应用冷却效果
             ├── 添加攻击状态 Tag
             │
             ├── AbilityTask
             │      └── 播放攻击 Montage
             │
             ├── 命中检测
             │      └── 确定目标
             │
             ├── GameplayEffect
             │      ├── 计算伤害
             │      └── 修改目标 Health
             │
             ├── GameplayCue
             │      ├── 播放火焰特效
             │      └── 播放命中音效
             │
             └── EndAbility
                    └── 结束技能并清理状态
```

这里有一个重要的设计思想：

技能执行逻辑、数值计算、状态管理、视觉表现应尽量解耦。

例如，火焰斩造成 100 点伤害，这个具体数值不一定写死在 `GA_FireSlash` 中，而可以交给 GameplayEffect 和它关联的计算器完成。

下面依次解释每个组件的内部机制。

## 二、AbilitySystemComponent（ASC）—— GAS 的核心管理器

核心类：`UAbilitySystemComponent`

ASC 是 GAS 中最重要的组件。

它继承自 `UActorComponent`，可以挂载在角色、PlayerState 或其他 Actor 上。

ASC 不直接实现某个技能的具体行为，而是管理角色拥有的技能、属性、GameplayEffect、GameplayTag 以及网络状态。

### 2.1 ASC 主要管理什么？

可以把 ASC 看成角色的 GAS 运行时管理中心。

① 技能管理

ASC 负责记录角色拥有的技能，以及技能的激活与移除。

```
// 授予技能：通常只在服务器执行
ASC->GiveAbility(
    FGameplayAbilitySpec(AttackAbilityClass, 1)
);

// 尝试通过技能类激活技能
ASC->TryActivateAbilityByClass(AttackAbilityClass);

// 通过技能句柄激活技能
ASC->TryActivateAbility(AbilityHandle);

// 根据 GameplayTag 激活匹配的技能
ASC->TryActivateAbilitiesByTag(AbilityTags);

// 移除指定技能
ASC->ClearAbility(AbilityHandle);
```

例如角色刚进入游戏时，服务器给他授予普通攻击、闪避、跳跃以及三个特殊技能。

当玩家按下攻击键时，输入系统通知 ASC 尝试激活普通攻击。

需要注意：`GiveAbility` 是服务器权威操作，客户端不能通过调用它擅自获得新技能。

② 属性管理

ASC 能访问角色注册的 AttributeSet，读取属性，以及处理属性变更。

例如：

```
// 获取当前生命值
float Health = ASC->GetNumericAttribute(
    UBattleAttributeSet::GetHealthAttribute()
);

// 修改生命值的 BaseValue
ASC->SetNumericAttributeBase(
    UBattleAttributeSet::GetHealthAttribute(),
    100.f
);
```

虽然可以通过接口直接修改属性，但是战斗中的伤害、治疗、体力消耗等，通常更推荐通过 GameplayEffect 完成。

这样可以统一处理效果计算、Buff、预测和网络同步。

③ GameplayEffect 管理

ASC 负责应用和管理 GE。

例如：

```
// 创建 GameplayEffect 的上下文
FGameplayEffectContextHandle Context =
    ASC->MakeEffectContext();

// 根据 GE 类创建运行时规格
FGameplayEffectSpecHandle SpecHandle =
    ASC->MakeOutgoingSpec(
        DamageEffectClass,
        1.0f,
        Context
    );

if (SpecHandle.IsValid())
{
    // 将伤害效果施加到目标 ASC
    ASC->ApplyGameplayEffectSpecToTarget(
        *SpecHandle.Data.Get(),
        TargetASC
    );
}
```

这里的 `DamageEffectClass` 可以是编辑器中配置好的 `GE_Damage`。

ASC 会负责按照 GE 的定义执行伤害计算和属性修改。

④ GameplayTag 管理

ASC 还可以判断角色当前状态。

```
// 查询角色是否拥有眩晕状态
bool bStunned = ASC->HasMatchingGameplayTag(
    StunnedTag
);

// 添加一个松散标签
ASC->AddLooseGameplayTag(StunnedTag);

// 移除一个松散标签
ASC->RemoveLooseGameplayTag(StunnedTag);
```

注意：松散标签（Loose Gameplay Tags）与 GE 授予的标签生命周期不同，而且普通 Loose Tag 操作不会自动处理网络复制。

在多人游戏中，推荐让需要持续同步的状态尽可能通过 GA/GE 管理，或者明确设计标签同步方式。

⑤ 网络同步

ASC 支持 GAS 状态复制，包括技能状态、GameplayEffect、GameplayTag 等，并与预测机制协同工作。

但并不是 ASC 中的所有内容都会无条件同步到所有客户端。

它有专门的 GameplayEffect 复制模式，后面会详细解释。

### 2.2 ASC 应该挂在 Character 还是 PlayerState？

这是 GAS 项目中很重要的架构问题。

方案一：ASC 挂在 Character 上

适合怪物、Boss、普通 NPC，以及不需要跨角色重生保留 GAS 状态的角色。

Character 被销毁后，它的 ASC、技能实例和持续效果也会随之结束。

方案二：ASC 挂在 PlayerState 上

适合需要在 Pawn 死亡、重生或者切换角色时保留某些技能、资源、Buff 状态的玩家。

Pawn 可以通过 `IAbilitySystemInterface` 返回 PlayerState 上的 ASC。

例如：

```
PlayerState
    └── AbilitySystemComponent
            ├── 技能列表
            ├── 属性集
            ├── Buff 列表
            └── 冷却信息

Character_A（当前操作角色）
    └── 使用 PlayerState 的 ASC

切换角色

Character_B（新操作角色）
    └── 继续使用 PlayerState 的 ASC
```

不过，ASC 放在 PlayerState 上，并不意味着所有 GAS 状态都应该跨角色保留。

比如角色 A 的生命值、攻击力、霸体状态不一定适合在切换角色后直接继承给角色 B。

这部分需要在游戏的角色切换架构中明确设计，可能需要独立存储各角色的属性和状态。

Epic 官方明确支持 ASC 放在 PlayerState 上，以解决重生、Pawn 切换等场景的状态持久化问题。

### 2.3 OwnerActor 和 AvatarActor 的区别

ASC 内部通过 `FGameplayAbilityActorInfo` 保存技能执行需要的 Actor 信息。

其中，最重要的是：

OwnerActor：逻辑拥有者

AvatarActor：技能的实际执行载体

例如：

```
ASC->InitAbilityActorInfo(
    OwnerActor,
    AvatarActor
);
```

对于直接挂在 Character 上的 ASC：

```
ASC->InitAbilityActorInfo(
    this,   // OwnerActor
    this    // AvatarActor
);
```

对于挂在 PlayerState 上的 ASC：

```
ASC->InitAbilityActorInfo(
    PlayerState, // OwnerActor：逻辑拥有者
    Character    // AvatarActor：实际战斗角色
);
```

为什么需要区分？

假设技能需要播放攻击动画：

OwnerActor 可能是 `PlayerState`，它没有 SkeletalMesh，也没有实际的战斗位置。

因此，GA 需要通过 AvatarActor 找到当前实际执行技能的角色，取得角色位置、动画实例、控制器等信息。

在切换 Pawn 或重新 Possess 时，需要在服务器和相应客户端的合适生命周期重新初始化或刷新 ActorInfo，不能只在构造函数中初始化一次。

## 三、GameplayAbility（GA）—— 技能的逻辑主体

核心类：`UGameplayAbility`

GA 用来描述一个技能的完整行为流程。

例如：

`GA_Attack`：普通攻击

`GA_Dodge`：闪避

`GA_Parry`：弹反

`GA_Sprint`：冲刺

`GA_Ultimate`：终结技

`GA_HitReaction`：受击反应

GA 不一定是玩家主动按键触发的技能。被动能力、受击反馈、状态驱动的动作，也可以采用 GA 实现。

### 3.1 GA 的生命周期

<img width="742" height="485" alt="image" src="https://github.com/user-attachments/assets/f5c70274-f846-410d-b583-35b8a7c6bd09" />


其中几个函数特别重要。

CanActivateAbility() —— 是否允许激活

例如：

```
bool UGA_Attack::CanActivateAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayTagContainer* SourceTags,
    const FGameplayTagContainer* TargetTags,
    FGameplayTagContainer* OptionalRelevantTags
) const
{
    if (!Super::CanActivateAbility(
        Handle,
        ActorInfo,
        SourceTags,
        TargetTags,
        OptionalRelevantTags))
    {
        return false;
    }

    // 可以增加项目自己的特殊技能条件

    return true;
}
```

实际项目中不一定要为每个条件写 C++。

很多限制可以直接通过 GameplayTag、Cost GameplayEffect 和 Cooldown GameplayEffect 配置实现。

ActivateAbility() —— 开始执行技能

```
void UGA_Attack::ActivateAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    const FGameplayEventData* TriggerEventData
)
{
    Super::ActivateAbility(
        Handle,
        ActorInfo,
        ActivationInfo,
        TriggerEventData
    );

    // 扣除资源并应用冷却
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        // 提交失败，结束技能
        EndAbility(
            Handle,
            ActorInfo,
            ActivationInfo,
            true,
            true
        );
        return;
    }

    // 在这里启动播放动画等 AbilityTask
    // 技能结束时由 Task 回调调用 EndAbility
}
```

`CommitAbility()` 主要用于按照 GA 配置提交资源消耗和冷却。

例如技能需要消耗 20 点体力，并产生 5 秒冷却，便可以通过对应的 GE 实现。

注意，`CommitAbility` 的调用时机是可以设计的。

蓄力技能可能先进入蓄力阶段，确认释放时才 Commit；而闪避通常一开始就 Commit。

EndAbility() —— 结束技能

调用之后，GAS 会清理与该技能执行相关的状态及其受管理的 Task。

需要注意：

EndAbility 不代表此前施加的所有 GE 都会自动移除。

例如技能已经给敌人施加了一个持续 10 秒的燃烧 GE，技能结束后，燃烧效果仍然可以继续存在。

### 3.2 GA 的 Instancing Policy（实例化策略）

实例化策略决定多个技能执行过程如何使用 GA 对象。

|模式|工作原理|常见使用场景|
|---|---|---|
|`NonInstanced`|不创建独立的技能执行实例，共用类默认对象逻辑|简单、无实例状态的技能|
|`InstancedPerActor`|每个拥有技能的 Actor 通常维护一个 GA 实例|常规攻击、闪避、冲刺|
|`InstancedPerExecution`|每次技能执行创建新实例|允许独立保存每次执行状态的技能|

例如一个蓄力技能：

```
// 假设 GA 中定义了这个成员变量
float ChargeTime = 0.f;
```

如果技能执行过程需要修改 `ChargeTime`，一般不适合使用 `NonInstanced`。

对于动作游戏，`InstancedPerActor` 往往是比较实用的默认选择。

但要注意：`InstancedPerActor` 复用同一个实例，并发执行或反复激活时的成员状态需要自己管理。

### 3.3 GA 的 Net Execution Policy（网络执行策略）

这是多人 GAS 中的重要配置。

|策略|核心含义|适用情况|
|---|---|---|
|`LocalPredicted`|客户端先执行预测，服务器再确认|玩家攻击、闪避等响应敏感操作|
|`LocalOnly`|仅本地执行技能逻辑|纯本地操作或表现|
|`ServerOnly`|仅服务器执行技能逻辑|权威战斗结算、某些 AI 行为|
|`ServerInitiated`|由服务器发起，并在客户端执行相应技能逻辑|服务器触发的同步技能行为|

举个例子：玩家网络延迟 100ms。

如果闪避需要等待服务器确认才能开始播放动画，操作手感就可能出现明显延迟。

使用 `LocalPredicted`，客户端可以先进入闪避并播放动画，同时把激活请求发送给服务器。

服务器仍然负责决定实际操作是否有效。

需要区分的是，技能的 Net Execution Policy 与 GA 实例属性复制、GE 复制模式是三个不同概念。

## 四、AbilityTask —— 处理技能中的异步流程

核心类：`UAbilityTask`，继承自 `UGameplayTask`

AbilityTask 可以理解为：

技能执行过程中的异步任务节点。

例如一个技能：

```
开始技能
   │
   ▼
播放攻击动画
   │
   ▼
等待动画进入攻击判定帧
   │
   ▼
启用武器碰撞检测
   │
   ▼
等待动画结束
   │
   ▼
结束技能
```

播放动画通常需要持续多个游戏帧。

而普通 C++ 函数不会自动停在某行代码上等待动画播放完成。

因此 GAS 提供 AbilityTask，让技能可以在等待期间保持执行状态，并在某个条件满足时通过回调继续后面的逻辑。

### 4.1 常见 AbilityTask

|AbilityTask|中文解释|使用场景|
|---|---|---|
|`PlayMontageAndWait`|播放 Montage 并等待|攻击、闪避、受击动画|
|`WaitDelay`|等待指定时间|技能前摇、延迟爆炸|
|`WaitGameplayEvent`|等待玩法事件|动画命中通知、连招窗口|
|`WaitInputPress`|等待输入按下|连招、蓄力|
|`WaitInputRelease`|等待输入松开|蓄力释放|
|`WaitGameplayTagAdded`|等待标签出现|监听特殊战斗状态|
|`WaitGameplayTagRemoved`|等待标签移除|等待眩晕结束|
|`WaitTargetData`|等待目标数据|锁定目标、瞄准确认|

### 4.2 PlayMontageAndWait 为什么重要？

假设角色释放普通攻击：

```
GA_Attack
    │
    ▼
PlayMontageAndWait
    │
    ├── OnCompleted
    │      └── 动画正常完成，结束技能
    │
    ├── OnInterrupted
    │      └── 动画被其他动画中断
    │
    ├── OnCancelled
    │      └── 技能被取消
    │
    └── OnBlendOut
           └── 动画开始混出
```

该 Task 提供了多个不同阶段的委托，可以实现：

攻击动画结束时释放攻击状态；受击打断攻击时取消技能；闪避动画混出后恢复部分输入；连招时切换下一个 Montage Section。

这些回调的时机并不完全相同，例如 `OnBlendOut` 不等于 `OnCompleted`。

### 4.3 AbilityTask 和普通 ActorComponent 有什么区别？

普通 ActorComponent 可以长期挂在角色上，负责一个持续运行的功能。

AbilityTask 通常绑定某一次 GA 执行过程。

当对应技能结束时，AbilityTask 会被管理和清理，不再继续等待之前的事件。

例如一个 `WaitInputRelease` Task 不应该在技能已经取消之后，仍然留下输入回调。

这也是 GAS 相比手写大量 Timer、Delegate 更容易统一管理生命周期的原因。

不过 AbilityTask 并不是每个 Task 都会自动通过网络复制。它通常依赖 GA 和其他 GAS 网络机制协同工作。

## 五、AttributeSet —— 战斗数值管理系统

核心类：`UAttributeSet`、`FGameplayAttributeData`

AttributeSet 用来组织角色的数值属性。

例如动作游戏中可能会有：

```
BattleAttributeSet
    │
    ├── Health         当前生命值
    ├── MaxHealth      最大生命值
    ├── Stamina        当前体力
    ├── MaxStamina     最大体力
    ├── AttackPower    攻击力
    ├── Defense        防御力
    ├── CritRate       暴击率
    ├── CritDamage     暴击伤害倍率
    └── MoveSpeed      移动速度
```

### 5.1 为什么不直接使用 float？

因为普通 `float` 不能直接提供 GAS 的属性聚合、GE 修改、持续 Buff、预测协作等功能。

例如：

```
float AttackPower = 100.f;
```

如果玩家有一个攻击力提升 20% 的 Buff，你可能直接把属性修改成 120。

但当 Buff 消失、多种 Buff 叠加、装备加成改变、角色等级变化之后，如何正确恢复属性？

这些事情会让普通变量的管理逐渐复杂。

GAS 使用 `FGameplayAttributeData` 来区分：

BaseValue：基础值

CurrentValue：当前计算值

### 5.2 BaseValue 和 CurrentValue 的区别

假设角色基础攻击力为 100。

<img width="753" height="513" alt="image" src="https://github.com/user-attachments/assets/4318e10d-2f1d-49b7-ade4-e7b5b1b477f8" />



这里必须理解：

Instant GE（即时效果）一般修改属性的 BaseValue，而具有持续时间的非周期属性修饰通常影响 CurrentValue。

对于周期性 GE，每次 Period 执行产生的数值修改具有即时执行的性质，不是简单地在结束时反向恢复。

例如中毒每秒扣血，不会因为中毒结束就把已经损失的 HP 加回来。

### 5.3 C++ 定义 AttributeSet

下面是一个简单的属性集。

`BattleAttributeSet.h`：

```
#pragma once

#include "CoreMinimal.h"
#include "AttributeSet.h"
#include "AbilitySystemComponent.h"

// .generated.h 必须放在本头文件所有其他 #include 之后
#include "BattleAttributeSet.generated.h"

UCLASS()
class UBattleAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    // 当前生命值
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing=OnRep_Health)
    FGameplayAttributeData Health;

    // 最大生命值
    UPROPERTY(BlueprintReadOnly)
    FGameplayAttributeData MaxHealth;

    // 生成 Health 的常用访问方法
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(UBattleAttributeSet, Health)
    GAMEPLAYATTRIBUTE_VALUE_GETTER(Health)
    GAMEPLAYATTRIBUTE_VALUE_SETTER(Health)
    GAMEPLAYATTRIBUTE_VALUE_INITTER(Health)

    UFUNCTION()
    void OnRep_Health(const FGameplayAttributeData& OldHealth);

    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps
    ) const override;
};
```

`BattleAttributeSet.cpp`：

```
#include "BattleAttributeSet.h"
#include "Net/UnrealNetwork.h"

void UBattleAttributeSet::OnRep_Health(
    const FGameplayAttributeData& OldHealth
)
{
    // 配合 GAS 属性复制与预测更新
    GAMEPLAYATTRIBUTE_REPNOTIFY(
        UBattleAttributeSet,
        Health,
        OldHealth
    );
}

void UBattleAttributeSet::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps
) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 生命值发生变化后同步到客户端
    DOREPLIFETIME_CONDITION_NOTIFY(
        UBattleAttributeSet,
        Health,
        COND_None,
        REPNOTIFY_Always
    );
}
```

此示例仅展示 Health 的复制，其他需要网络同步的属性也应按需求配置。还需要在 ASC 所属 Actor 中创建或注册 AttributeSet，并通过初始 GE 或初始化流程设置初始数值。

### 5.4 AttributeSet 的重要回调

PreAttributeChange()

属性当前值即将发生变化时触发，适合进行数值约束。

例如：

```
void UBattleAttributeSet::PreAttributeChange(
    const FGameplayAttribute& Attribute,
    float& NewValue
)
{
    Super::PreAttributeChange(Attribute, NewValue);

    // 当前生命值限制在 0 ~ 最大生命值之间
    if (Attribute == GetHealthAttribute())
    {
        NewValue = FMath::Clamp(
            NewValue,
            0.f,
            GetMaxHealth()
        );
    }
}
```

PostGameplayEffectExecute()

适合处理 GE 执行之后的结果。

例如：

```
void UBattleAttributeSet::PostGameplayEffectExecute(
    const FGameplayEffectModCallbackData& Data
)
{
    Super::PostGameplayEffectExecute(Data);

    if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        SetHealth(
            FMath::Clamp(GetHealth(), 0.f, GetMaxHealth())
        );

        if (GetHealth() <= 0.f)
        {
            // 服务器侧可进入死亡判定流程
            // 实际项目应注意只触发一次
        }
    }
}
```

此处用 Health 演示最简单的处理方式。

大型战斗项目通常还会使用 `IncomingDamage`、`IncomingHealing` 等中间属性（Meta Attribute，元属性），先接收待结算结果，再统一处理护盾、血量、死亡以及受击事件。

还需要注意：`PreAttributeChange` 并不是所有 BaseValue 修改路径的通用拦截器。对于基础值修改，可以根据需要处理 `PreAttributeBaseChange` 等回调；对于各种入口的最终约束，也需要统一设计。

官方对这些回调及属性复制方式有专门说明。

## 六、GameplayEffect（GE）—— 伤害、Buff、Debuff 的核心

核心类：`UGameplayEffect`

如果说 GA 负责执行技能，那么 GE 主要负责：

当某种战斗行为发生后，目标的属性和状态应该怎样变化。

例如：

```
GA_Attack
    └── GE_Damage
          └── 敌人减少生命值

GA_Dodge
    └── GE_StaminaCost
          └── 自己减少体力

GA_FireAttack
    └── GE_Burning
          ├── 添加燃烧 Tag
          └── 周期性造成伤害

GA_Ultimate
    └── GE_Cooldown
          ├── 持续 20 秒
          └── 限制再次释放终结技
```

这里最值得理解的一点是：

GameplayEffect 并不必须由 GameplayAbility 产生。

例如场景中的毒池、装备、机关、道具、环境区域，也可以直接向角色的 ASC 施加 GameplayEffect。

### 6.1 GE 的三种 Duration 类型

Instant（即时效果）

立即执行一次修改，不作为持续效果保留。

示例：攻击命中后减少 100 HP、回复 50 HP、消耗 20 体力。

Has Duration（有限持续时间）

在指定时间内持续存在，到期后自动结束。

示例：攻击力提升 20%，持续 10 秒；敌人减速 5 秒；技能冷却 8 秒。

Infinite（无限持续时间）

不会因固定持续时长结束，需要通过其他机制移除。

示例：装备提供的属性加成、某个常驻被动状态、永久光环效果。

### 6.2 Period（周期性执行）

Duration 决定 GE 存在多久，Period 则可以决定持续期间多长时间执行一次。

例如一个中毒效果：

```
GE_Poison
    │
    ├── Duration = 5 秒
    │
    ├── Period = 1 秒
    │
    └── Modifier
           ├── Attribute = Health
           ├── Operation = Add
           └── Magnitude = -10
```

这个 GE 可以用来实现持续扣血。

周期中毒示例

假设首次结算发生在施加后第 1 秒，初始 HP = 100

<img width="746" height="569" alt="image" src="https://github.com/user-attachments/assets/0ab5cd95-d0b4-4cce-9ce7-0042ee842f7d" />


示意数值，实际首次结算时机由 Period 及 Execute Periodic Effect On Application 等配置决定。

注意：如果开启了 `Execute Periodic Effect On Application`（施加时立即执行周期效果），第一次伤害结算时机会改变，不能只根据 Duration 和 Period 简单推断结算次数。

### 6.3 Modifiers（属性修饰器）

GE 可以通过 Modifier 修改 Attribute。

常见操作包括：

|类型|含义|示例|
|---|---|---|
|`Additive`|加法修饰|攻击力 +50|
|`Multiplicitive`|乘法修饰|攻击力乘以 1.2|
|`Division`|除法修饰|属性按指定倍率除算|
|`Override`|覆盖|将指定属性覆盖为某个值|

比如攻击力从 100 增加到 150，可以使用一个 Additive 为 +50 的 Modifier。

实际多个 Modifier 同时存在时，GAS 会按照自己的属性聚合规则求值。不能简单认为所有 Modifier 都按 GE 的应用顺序逐项计算。

### 6.4 Stacking（Buff 叠层）

GE 自带持续效果的叠层机制。

假设一个 Buff：

```
Buff：狂暴

每层攻击力 +10%
最大叠层数：5
持续时间：10 秒
```

狂暴 Buff 叠层示例

<img width="739" height="213" alt="image" src="https://github.com/user-attachments/assets/07291a5b-70f9-46b6-82fc-be6cfb16fbb5" />


GAS 还支持配置叠层持续时间刷新、周期重置以及达到叠层上限时的溢出行为。

这些配置非常适合实现中毒、流血、元素积累以及可叠层 Buff。

### 6.5 GameplayEffectComponent（GEComponent）

这是 UE5 GAS 中值得特别关注的机制。

GEComponent 不等于 AbilitySystemComponent。

它是 GE 内部使用的功能扩展对象，负责定义一个 GE 的部分附加行为。

在较新的 UE5 版本中，许多 GE 的标签管理和应用限制等功能可以由 GEComponent 配置。

常见的 GEComponent：

|组件|用途|
|---|---|
|`UTargetTagsGameplayEffectComponent`|给目标授予 GameplayTag|
|`UBlockAbilityTagsGameplayEffectComponent`|阻止目标激活符合标签条件的 GA|
|`UImmunityGameplayEffectComponent`|控制对其他 GE 的免疫|
|`URemoveOtherGameplayEffectComponent`|按条件移除目标其他 GE|
|`UAdditionalEffectsGameplayEffectComponent`|附加其他 GE|
|`UChanceToApplyGameplayEffectComponent`|控制 GE 的施加概率|
|`UTargetTagRequirementsGameplayEffectComponent`|设置目标标签条件|
|`UCustomCanApplyGameplayEffectComponent`|自定义 GE 应用条件|

例如一个眩晕 GE：

```
GE_Stun
    │
    ├── Duration = 3 秒
    │
    ├── TargetTags Component
    │      └── 授予 State.Stunned
    │
    ├── BlockAbilityTags Component
    │      └── 阻止指定类型的技能
    │
    └── GameplayCue
           └── 播放眩晕特效
```

该 GE 结束时，系统会根据对应组件和效果生命周期清理其授予的标签等状态。

非常重要：GEComponent 不是给每一次 Buff 独立创建的普通 ActorComponent。

它属于 GE 定义资产，对多次应用是共享的，因此不要在 GEComponent 对象中随意保存某个具体角色、某一次应用的可变运行时状态。

Epic 官方 GameplayEffect 文档对 GEComponent 的类型与职责有完整说明。
## 七、GameplayTag —— 战斗状态与技能限制的核心

核心类型：`FGameplayTag`、`FGameplayTagContainer`、`FGameplayTagQuery`

GameplayTag 是 GAS 中非常重要的系统。

它可以理解成一种层级化、可查询的状态或类别标识。

例如：

```
State
 ├── Attacking       攻击中
 ├── Dodging         闪避中
 ├── Stunned         眩晕中
 ├── Dead            死亡
 └── Invincible      无敌

Ability
 ├── Attack
 │    ├── Light
 │    └── Heavy
 ├── Dodge
 └── Ultimate

Event
 ├── Attack
 │    └── Hit
 └── Combo
      └── Next

GameplayCue
 ├── Hit
 └── Buff
      └── Fire
```

### 7.1 为什么不用 bool 管理状态？

比如一个角色可以处于：

```
bool bIsAttacking;
bool bIsDodging;
bool bIsStunned;
bool bIsDead;
bool bIsInvincible;
```

这种方式对于简单项目完全可以使用。

但是当技能数量增多，就需要处理大量互斥关系：

攻击时能否闪避？闪避是否可以打断攻击？眩晕能否释放技能？霸体是否免疫打断？死亡后哪些技能自动终止？

如果全部使用 bool，条件很容易散落在各种技能和角色组件之中。

GameplayTag 可以将这些条件集中配置和查询。

例如：

```
// 判断是否拥有眩晕状态
if (ASC->HasMatchingGameplayTag(StunnedTag))
{
    // 禁止某些战斗操作
}
```

注意：GameplayTag 本身只是标识和匹配机制。它不会因为名称叫 `State.Stunned`，就自动让 Character 停止移动或者让所有技能失效。

必须在 GA、GE、移动组件或者其他游戏规则中使用这些 Tag。

### 7.2 GAS 中几种重要的技能标签配置

假设有一个攻击技能 `GA_Attack`。

Activation Required Tags（激活需要的标签）

只有角色满足指定标签条件，技能才允许激活。

例如某个特殊技能需要：

```
State.PowerMode
```

只有角色进入强化状态后，才能释放这个技能。

Activation Blocked Tags（阻止激活的标签）

如果角色拥有某种状态，则禁止技能激活。

例如：

```
State.Stunned
State.Dead
```

表示角色眩晕或死亡时不允许释放攻击。

Activation Owned Tags（激活期间授予的标签）

技能激活后，在技能执行期间给角色添加标签。

例如：

```
State.Attacking
```

技能正常结束或取消后，GAS 会清理该激活过程授予的标签。

Block Abilities With Tag（阻止其他技能）

当当前 GA 正在执行时，阻止具有指定技能标签的其他 GA 激活。

例如重攻击正在执行：

```
Block Abilities With Tag:
    Ability.Attack.Light
```

就可以阻止普通轻攻击技能启动。

Cancel Abilities With Tag（取消其他技能）

激活当前技能时，取消已经运行的、满足指定标签条件的其他 GA。

例如闪避技能设置：

```
Cancel Abilities With Tag:
    Ability.Attack
```

那么激活闪避时可以取消匹配的攻击类 GA，从而形成攻击取消机制。

这些属于 GAS 内置的技能标签交互配置，不需要全部手动调用 CancelAbility。

### 7.3 GameplayTag 非常适合处理动作游戏的技能优先级

例如可以设计以下规则：

|当前状态|新输入|预期结果|
|---|---|---|
|普通攻击中|普通攻击|连招窗口内缓冲下一段|
|普通攻击中|闪避|满足取消条件时中断攻击|
|重攻击蓄力中|闪避|根据蓄力阶段决定能否取消|
|眩晕中|普通攻击|阻止|
|霸体中|被普通攻击命中|扣血但不产生普通硬直|
|无敌中|被攻击命中|不进行伤害结算|
|死亡中|任意战斗技能|阻止|

但这里要特别区分：

Tag 负责描述状态和表达规则，而战斗状态机、动画窗口、取消优先级仍然需要项目自己的逻辑。

比如“攻击的第 10～20 帧可以闪避取消”，这个时间窗口需要动画通知、AbilityTask 或自定义战斗状态逻辑配合。

GameplayTag 自身不会提供帧级窗口管理。

### 7.4 标签的层级匹配

例如：

```
Ability.Attack
Ability.Attack.Light
Ability.Attack.Heavy
```

GameplayTag 可以按照层级进行匹配。

查询 `Ability.Attack`，在使用支持父子层级匹配的接口时，可以匹配更具体的攻击标签。

如果只需要严格相等的标签，则应使用 Exact 类型的匹配接口。

大型战斗项目应统一约定标签命名规则，避免到处使用任意字符串。

官方对 GameplayTag 的层级结构和匹配机制有专门介绍。

## 八、GameplayCue —— 战斗特效与音效管理

常见类：`UGameplayCueNotify_Static`、`AGameplayCueNotify_Actor`

GameplayCue 负责 GAS 中的表现层。

例如角色被火属性攻击命中：

```
GE_FireDamage
    │
    └── GameplayCue.Hit.Fire
             │
             ├── 播放 Niagara 火花
             ├── 播放命中音效
             ├── 播放局部特效
             └── 其他非权威视觉反馈
```

### 8.1 GameplayCue 有哪些执行事件？

|事件|作用|举例|
|---|---|---|
|`OnActive`|持续 Cue 开始激活时|Buff 开始时播放启动特效|
|`WhileActive`|Cue 进入或处于激活状态时的表现入口|持续 Buff 的视觉状态|
|`Executed`|一次性执行表现|攻击命中特效、爆炸|
|`OnRemove` / `Removed`|持续 Cue 被移除时|Buff 结束，移除持续粒子|

注意 `WhileActive` 不是每帧调用的 Tick 函数。

### 8.2 GameplayCueNotify_Static 和 GameplayCueNotify_Actor

GameplayCueNotify_Static

适合没有复杂持续实例状态的一次性表现。

例如：

一次普通攻击命中，播放一个火花特效和命中音效。

GameplayCueNotify_Actor

适合需要 Actor 生命周期管理的持续效果。

例如：

角色开启持续 10 秒的火焰护盾，需要在角色身上持续挂载 Niagara 特效，并在 Buff 结束时停止。

### 8.3 为什么 GameplayCue 不建议处理实际伤害？

因为 GameplayCue 的设计目标是网络高效的表现通知。

它不应当作为权威战斗事件的唯一依据。

例如，下面这种设计不合理：

```
GameplayCue 播放爆炸
    └── 在 Cue 内计算爆炸伤害
```

因为远端 Cue 的表现通知可能丢失，或者在不同客户端以不同时间执行。

更合适的设计是：

```
服务器确认爆炸命中
    │
    ├── GameplayEffect
    │      └── 结算实际伤害
    │
    └── GameplayCue
           └── 播放爆炸特效与音效
```

也就是说：

真正影响战斗结果的逻辑交给服务器与 GameplayEffect，GameplayCue 主要负责表现。

Epic 官方也强调，GameplayCue 使用的通知不应被当作可靠的玩法同步机制。

## 九、GameplayEffectCalculation —— GAS 的数值计算机制

GAS 提供两类特别重要的数值计算扩展。

### 9.1 MMC：GameplayModMagnitudeCalculation

`UGameplayModMagnitudeCalculation`

MMC 主要用来计算某一个 Modifier 的数值幅度（Magnitude）。

例如：

```
GE_AttackBuff
    │
    └── Modifier：AttackPower
            │
            └── MMC_AttackBuff
                    │
                    └── 根据角色属性计算加成
```

假设攻击力加成公式是：

$$
\text{额外攻击力}=\text{基础攻击力}\times 0.2 
$$

角色基础攻击力为 200，那么 MMC 可以计算出一个 +40 的修饰幅度。

随后 GE 将这个计算结果应用到对应属性。

MMC 适合属性增益、装备属性换算、等级成长和其他可以表达成单个 Modifier 幅度的问题。

它可以参与预测，但具体能否安全预测，还取决于计算依赖的数据是否能在预测端正确获取。

### 9.2 ExecCalc：GameplayEffectExecutionCalculation

`UGameplayEffectExecutionCalculation`

ExecCalc 更适合复杂的、一次结算中涉及多个属性或多个输出的计算。

例如：

```
攻击者 ASC
    │
    ├── AttackPower
    ├── CritRate
    └── CritDamage
             │
             ▼
       ExecCalc_Damage
             ▲
             │
目标 ASC
    │
    ├── Defense
    ├── DamageReduction
    └── ElementResistance
             │
             ▼
       得到最终伤害
             │
             ▼
       输出 IncomingDamage
             │
             ▼
       目标 AttributeSet
             │
             ├── 扣除护盾
             ├── 扣除生命值
             └── 处理死亡判定
```

例如设计一个简化伤害公式：

$$
D=\max(0,A\times M-Def) 
$$

其中：

\(A\)：攻击力

\(M\)：技能倍率

\(Def\)：目标防御减伤值

<img width="740" height="403" alt="image" src="https://github.com/user-attachments/assets/6a60e4b7-3dc1-44e8-847f-e67d3012b656" />

### 9.3 MMC 和 ExecCalc 的区别

|对比项|MMC|ExecCalc|
|---|---|---|
|核心职责|计算单个 Modifier 的幅度|执行复杂结算|
|多属性输入|支持|支持|
|一次输出多个属性修改|不直接负责|支持|
|典型使用场景|Buff 加成、属性换算|伤害、治疗、护盾结算|
|客户端预测|可以参与符合条件的预测流程|通常按服务器权威执行设计|

对于复杂动作游戏，ExecCalc 很适合统一处理伤害计算规则，避免将大量伤害公式分散在不同 GA 中。

但 ExecCalc 本身不是一个可随意在客户端预测并自动回滚的伤害系统。对真实生命值等权威结算，通常仍应采用服务器判定。

## 十、FGameplayAbilitySpec 和 FGameplayEffectSpec —— 运行时数据

这两个结构虽然不是 ActorComponent，但在 GAS 的运行过程中非常重要。

### 10.1 FGameplayAbilitySpec：角色实际拥有的技能记录

需要区分：

`UGameplayAbility`：定义技能的行为与配置。

`FGameplayAbilitySpec`：描述某个 ASC 被授予的技能。

例如：

```
FGameplayAbilitySpec Spec(
    AttackAbilityClass, // 技能类
    1,                  // 技能等级
    INDEX_NONE          // 输入 ID（可选）
);

FGameplayAbilitySpecHandle Handle =
    ASC->GiveAbility(Spec);
```

同一个 `GA_Attack` 类可以被很多角色拥有。

但是，每个角色的 ASC 中都可以拥有自己的技能记录，比如不同等级、不同输入绑定或者动态标签。

`FGameplayAbilitySpecHandle` 则是定位这条技能记录的句柄。

因此：

技能类是定义，AbilitySpec 是授予记录，AbilitySpecHandle 是查找该技能记录的标识。

### 10.2 FGameplayEffectSpec：一次 GE 应用的参数

前面提到，`UGameplayEffect` 主要是配置定义。

但一次攻击需要携带动态参数。

例如同一个 `GE_Damage`：

```
玩家 A 释放普通攻击
    └── Damage = 100

玩家 A 释放重攻击
    └── Damage = 350

玩家 B 释放普通攻击
    └── Damage = 180
```

三个攻击可以使用同一个 GE 类，只是应用时的数据不同。

这时就需要 `FGameplayEffectSpec`。

它可以记录效果等级、运行时传入的数值、上下文、标签信息和属性捕获相关数据等。

SetByCaller（调用方指定数值）

例如技能在命中时计算出一个动态伤害值：

```
// 为本次 GE Spec 设置动态数值
SpecHandle.Data->SetSetByCallerMagnitude(
    DamageDataTag,
    350.f
);
```

其中 `DamageDataTag` 可以配置为：

```
Data.Damage
```

对应的 GE Modifier 可以读取这个 SetByCaller 值作为数值来源。

不过 SetByCaller 本身不会自动执行伤害逻辑，需要 GE 的 Modifier 或相应计算逻辑实际使用它。

### 10.3 GameplayEffectContext：效果上下文

常见类型：

`FGameplayEffectContext`

`FGameplayEffectContextHandle`

Context 用来记录一次效果发生的上下文信息，例如：

```
EffectContext
    │
    ├── Instigator       发起者
    ├── EffectCauser     实际效果造成者
    ├── SourceObject     来源对象
    └── HitResult        命中信息（可选）
```

例如一个角色用弓箭射中敌人：

Instigator 可以是射箭的角色。

EffectCauser 可以是箭矢 Actor。

HitResult 可以包含箭矢命中的位置、法线、碰撞组件等数据。

这样伤害计算、击杀归属、受击反馈以及表现层就可以共享这一次命中的相关信息。

在动作游戏里，这对武器攻击、投射物、AOE 技能和伤害来源追踪非常有用。

## 十一、GameplayEvent 和 TargetData —— 事件通信及目标传递

### 11.1 GameplayEvent：技能间的事件通信

GameplayEvent 通常通过 `FGameplayTag` 标识事件类型，并使用 `FGameplayEventData` 携带相关数据。

例如：

```
Event.Attack.Hit
Event.Combo.Next
Event.Dodge.Perfect
Event.Parry.Success
Event.Character.Dead
```

在攻击动画播放过程中，动画通知触发：

```
AnimNotify
    │
    ▼
发送 Event.Attack.Hit
    │
    ▼
AbilityTask_WaitGameplayEvent
    │
    ▼
GA_Attack 接收到事件
    │
    ▼
启用攻击判定或结算命中
```

C++ 发送事件的示例：

```
#include "AbilitySystemBlueprintLibrary.h"

// 创建事件数据
FGameplayEventData EventData;

EventData.EventTag = AttackHitTag;

// 设置攻击者
EventData.Instigator = Attacker;

// 设置目标
EventData.Target = Defender;

// 向目标 Actor 的 ASC 发送 GameplayEvent
UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(
    Defender,
    AttackHitTag,
    EventData
);
```

注意，这个例子是向 `Defender` 发送事件。如果是攻击者自己运行的 GA 正在等待动画命中通知，事件通常应该发给攻击者自己的 ASC。

GameplayEvent 既可以用来触发配置了相应 Trigger 的技能，也可以被已经运行的 GA 通过 `WaitGameplayEvent` 监听。

另外，SendGameplayEventToActor 是玩法事件接口，不代表自动向服务器发送可靠 RPC。在多人项目中，必须明确事件应该在客户端、服务器还是预测窗口内生成，并使用合适的同步方式。

### 11.2 TargetData：技能的目标信息

常见结构：

`FGameplayAbilityTargetDataHandle`

TargetData 用于承载目标选择和命中相关数据。

例如：

```
GA_Fireball
    │
    ├── 玩家进入瞄准
    │
    ├── 确定目标位置
    │
    ├── 生成 TargetData
    │
    ├── 通过 GAS 相关机制传递目标数据
    │
    └── 服务器验证后执行技能
```

它可以保存目标 Actor、HitResult 或其他自定义目标数据。

对于需要服务器验证的技能，目标数据非常重要。

比如客户端预测自己的射线命中了敌人，并不意味着服务器一定接受。

服务器仍需要根据游戏的命中规则校验数据。

## 十二、GAS 的多人同步与客户端预测

这一部分建议重点掌握，因为 GAS 在网络战斗中的优势，很大程度上来自其预测和复制机制。

### 12.1 ASC 的 GameplayEffect Replication Mode

`UAbilitySystemComponent` 支持几种 GameplayEffect 复制模式。

|模式|效果复制行为|常见选择|
|---|---|---|
|`Full`|向相关客户端完整复制活动 GE 信息|单机、调试或确有完整效果数据需求|
|`Mixed`|自己的拥有端获取完整 GE，其他模拟客户端获取精简信息|玩家角色|
|`Minimal`|客户端只接收精简的 GE 相关信息|AI、NPC、Boss 等|

例如：

```
ASC->SetReplicationMode(
    EGameplayEffectReplicationMode::Mixed
);
```

这里特别容易混淆：

Replication Mode 主要控制 GameplayEffect 的复制，而不是控制所有 GAS 信息是否同步。

AttributeSet 中的属性仍按照自己的属性复制规则处理。

GameplayTag、GameplayCue 以及技能激活，也分别涉及相应的网络机制。

此外，`Mixed` 模式需要正确的网络 Owner 关系，否则拥有端识别可能不符合预期。

### 12.2 LocalPredicted 的完整思路

假设玩家的技能有 100ms 往返延迟。

<img width="737" height="546" alt="image" src="https://github.com/user-attachments/assets/0b48596c-c6f3-41fb-8c81-0794345ea4fc" />


GAS 内部通过 Prediction Key（预测键）等机制关联客户端的预测操作与服务器确认。

但需要明确：

GAS 的预测机制不等于完整的战斗系统预测。

例如：

客户端可以提前预测技能动画、部分 GameplayEffect 及 GameplayTag 变化。

但攻击是否命中敌人、造成多少实际伤害，通常仍需要服务器进行权威结算。

尤其是 Instant GE 导致的实际伤害等变化，并不能直接当作普通可回滚持续 GE 一样处理。

Epic 官方特别说明，GAS 的预测回滚能力存在边界，并非所有瞬时属性变化都能通用地预测与回滚。

### 12.3 GAS 不会自动解决哪些战斗网络问题？

GAS 已经提供技能激活预测、GameplayEffect 复制、部分属性预测协调以及网络任务机制。

但复杂多人战斗仍需要项目自己实现：

角色移动预测与校正、Root Motion 的网络协调、攻击命中延迟补偿、碰撞查询的服务器权威校验、敌人状态同步、受击表现纠偏、远端角色插值，以及不同技能之间的精确时序同步。

例如，你设计一个高速闪避加反击技能，GAS 可以管理这个技能的激活、体力消耗、无敌 Tag 和冷却，但闪避轨迹的同步、服务器如何认定闪避成功，依然需要与 UE 的 CharacterMovement 等系统协作。

## 十三、C++ 项目如何接入 GAS？

这里给一个基础的接入方式，方便把前面介绍的组件联系到实际项目。

### 13.1 启用模块

首先在 UE 插件管理中启用 Gameplay Ability System 插件。

随后在 `.Build.cs` 中添加依赖：

```
PublicDependencyModuleNames.AddRange(
    new string[]
    {
        "GameplayAbilities",
        "GameplayTags",
        "GameplayTasks"
    }
);
```

### 13.2 让 Character 拥有 ASC

`BattleCharacter.h`（节选）：

```
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "AbilitySystemInterface.h"

// 必须最后包含 generated.h
#include "BattleCharacter.generated.h"

class UAbilitySystemComponent;
class UBattleAttributeSet;
class UGameplayAbility;

UCLASS()
class ABattleCharacter :
    public ACharacter,
    public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    ABattleCharacter();

    // 实现 GAS 接口
    virtual UAbilitySystemComponent*
        GetAbilitySystemComponent() const override;

protected:
    // GAS 核心组件
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
    TObjectPtr<UAbilitySystemComponent> ASC;

    // 角色战斗属性集
    UPROPERTY()
    TObjectPtr<UBattleAttributeSet> AttributeSet;

    // 初始攻击技能
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UGameplayAbility> AttackAbilityClass;

    virtual void BeginPlay() override;
};
```

`BattleCharacter.cpp`（节选）：

```
#include "BattleCharacter.h"
#include "AbilitySystemComponent.h"
#include "BattleAttributeSet.h"
#include "Abilities/GameplayAbility.h"

ABattleCharacter::ABattleCharacter()
{
    // 创建 ASC
    ASC = CreateDefaultSubobject<UAbilitySystemComponent>(
        TEXT("AbilitySystem")
    );

    // 开启网络复制
    ASC->SetIsReplicated(true);

    // 玩家角色常用 Mixed 复制模式
    ASC->SetReplicationMode(
        EGameplayEffectReplicationMode::Mixed
    );

    // 创建属性集
    AttributeSet = CreateDefaultSubobject<UBattleAttributeSet>(
        TEXT("AttributeSet")
    );
}

UAbilitySystemComponent*
ABattleCharacter::GetAbilitySystemComponent() const
{
    return ASC;
}

void ABattleCharacter::BeginPlay()
{
    Super::BeginPlay();

    // 示例：ASC 与角色自身关联
    ASC->InitAbilityActorInfo(this, this);

    // 只有服务器能授予技能
    if (HasAuthority() && AttackAbilityClass)
    {
        ASC->GiveAbility(
            FGameplayAbilitySpec(
                AttackAbilityClass,
                1
            )
        );
    }
}
```

这是一个用于理解接入关系的简化示例。

实际多人玩家项目还应处理 PossessedBy、OnRep_PlayerState 或其他对应生命周期，确保服务器和拥有端的 ActorInfo、网络 Owner 关系正确，且初始化与技能授予不会因重复调用而出现问题。

然后就可以在 Enhanced Input 的输入处理函数中，通过 ASC 尝试激活技能。

```
// 玩家按下攻击键
void ABattleCharacter::OnAttackPressed()
{
    if (ASC && AttackAbilityClass)
    {
        ASC->TryActivateAbilityByClass(
            AttackAbilityClass
        );
    }
}
```

这样，输入不再直接管理攻击动画、冷却以及资源消耗，而是触发一个被 ASC 管理的 GameplayAbility。

在后续复杂项目中，可以再进一步建立输入 Tag 与 GA 的映射系统，让 Enhanced Input 和具体 GA 类解耦。

## 十四、动作战斗项目应该怎样划分 GAS 的职责？

实际开发中，建议不要把所有战斗逻辑都放进一个巨大 GA，也不要试图把所有系统强行改造成 GAS。

以下是一种比较清晰的架构划分。

|战斗模块|推荐主要承担者|GAS 的参与方式|
|---|---|---|
|技能释放|GameplayAbility|执行流程、冷却、消耗|
|连招系统|GA + 连招管理逻辑|GA 执行每段动作，配合 Tag 与事件|
|动画播放|AnimInstance + AbilityTask|Task 管理技能动画生命周期|
|攻击命中|自定义 HitDetection|将命中结果传给 GE/GA|
|伤害计算|GE + ExecCalc|计算并施加伤害|
|属性管理|AttributeSet|保存生命、体力、攻防等|
|Buff/Debuff|GameplayEffect|持续时间、叠层与状态|
|无敌、霸体、眩晕|GameplayTag + 战斗规则|描述与判断状态|
|受击反应|GA + 动画系统|硬直、击退、倒地等|
|战斗表现|GameplayCue|特效、音效、视觉反馈|
|AI 决策|BehaviorTree / StateTree 等|AI 决定何时激活 GA|
|网络同步|ASC + UE 网络框架|技能预测、效果复制及权威状态|

这里尤其需要注意三个问题。

第一，GAS 并不代替所有战斗状态机。

对于强动作游戏，攻击阶段、前后摇、取消窗口、霸体窗口、连招输入缓存仍然需要精细控制。GA 和 GameplayTag 是这些系统的重要组成部分，但并不是全部。

第二，GAS 不负责替你完成所有攻击命中检测。

例如刀剑攻击可能采用武器 Socket Trace、碰撞体检测、多段 Sweep 或自定义 Hitbox 系统。这部分通常需要自己实现，然后通过 GE 进行伤害结算。

第三，GAS 的技能生命周期与动画生命周期要明确区分。

例如技能动画结束时，是否代表伤害判定结束？是否解除移动锁定？是否允许下一次攻击？是否结束无敌？这些可能是不同时间点，不能全部简单绑定在 Montage Completed 上。

## 十五、一个完整的战斗功能在 GAS 中如何组织？

假设需要实现一个“闪避成功触发完美闪避，再释放强化反击”的技能。

可以这样拆分：

```
GA_Dodge（闪避技能）
    │
    ├── Cost GE
    │      └── 扣除体力
    │
    ├── Cooldown GE
    │      └── 产生闪避冷却
    │
    ├── AbilityTask
    │      └── 播放闪避 Montage
    │
    ├── GameplayTag
    │      ├── State.Dodging
    │      └── State.Invincible
    │          （仅在无敌窗口有效）
    │
    ├── 自定义闪避判定
    │      └── 判断是否满足完美闪避条件
    │
    └── 完美闪避成立
           │
           ├── GameplayEvent
           │      └── Event.Dodge.Perfect
           │
           ├── GameplayEffect
           │      └── GE_PerfectDodgeBuff
           │
           ├── GameplayTag
           │      └── State.CanCounterAttack
           │
           ├── GameplayCue
           │      └── 慢动作视觉反馈 / 特效
           │
           └── GA_CounterAttack
                  │
                  ├── 检查反击条件
                  ├── 播放反击动画
                  ├── 进行命中检测
                  └── GE_CounterDamage
                         └── ExecCalc 计算伤害
```

这种设计的优势是，每个模块负责自己的职责。

以后想修改闪避的体力消耗，主要调整 Cost GE；想修改无敌窗口，调整技能阶段和 Tag 控制；想增加完美闪避 Buff，修改对应 GE；想增加反击伤害，修改伤害 GE 或计算参数。

不需要把所有功能都耦合在同一个 Character 类里面。

## 十六、总结：学习 GAS 的推荐顺序

理解 GAS 不建议一开始就研究所有源码，而是按依赖关系逐层掌握。

<img width="746" height="569" alt="image" src="https://github.com/user-attachments/assets/cfb4881a-a8fd-4c89-9403-27c8353a0ff6" />


对于动作战斗开发，我认为最值得深入研究的是以下四组关系：

GA + AbilityTask：决定技能执行的具体阶段、动画时序、取消与中断机制。

GameplayTag + GA：决定战斗状态、技能互斥、受击硬直与霸体逻辑。

GameplayEffect + AttributeSet + ExecCalc：构成统一的战斗数值与伤害结算系统。

ASC + Prediction + Replication：决定技能在多人联机环境中的预测、权威判定和状态同步。

这四组机制组合起来，才真正形成一套能够支撑复杂动作战斗的 GAS 架构。

官方资料可以继续参考 [Gameplay Ability System Overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/understanding-the-unreal-engine-gameplay-ability-system?utm_source=chatgpt.com) 和 [Gameplay Effects](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-effects-for-the-gameplay-ability-system-in-unreal-engine?utm_source=chatgpt.com)。

后续如果要深入到类似高动作性 ARPG 的具体实现，最值得展开的专题是：如何通过 GAS 实现完整的普通攻击连招、闪避取消、受击硬直、霸体和弹反系统。这部分能够进一步解释各个组件在真实战斗中的执行时序，以及哪些功能应该由 GAS 管理、哪些需要自定义战斗框架。
