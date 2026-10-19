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

![[局部截取_20261020_012944.png]]

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

![[局部截取_20261020_013130.png]]

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

![[局部截取_20261020_013213.png]]


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

![Chart](data:image/png;base64,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AQhAoNAEYn08hG2sh4fGQQACEIAABCAAAQh0lADCtqOkKAcBCOSPADVDAAIQgAAEckAAYZsDiFQBAQhAAAIQgAAE8kmAujtGAGHbMU6UggAEIAABCEAAAhCIOQGEbcwHiOZBIH8EqBkCEIAABCBQWgQQtqU1nvQGAhCAAAQgAIFcEaCexBFA2CZuyGgwBCAAAQhAAAIQgEBLBBC2LVHBB4H8EaBmCEAAAhCAAATyRGA1Ybt06VK76qqr7KabbrLPPvsse9jGxkZ79NFHbcyYMXb22Wfb5MmT7aOPPspub76iesaPH2/Tp09vvskWLFhgY8eOtblz52a3qdzFF19sPl1xxRV266232sKFC03HzhZkBQIQgAAEIACBEiZA1yDQdQJNhK0E5L333uuE57Jly5oIyjlz5thdd91l++67r40ePdpqa2uduP3kk09aPHpDQ4OpjhUrVqy2Xft+/PHHtmrVquw2lZP/oIMOsqOPPtoGDBhg8+bNswkTJtjUqVObtCW7EysQgAAEIAABCEAAAhAICTQRti+88IK9/PLLtsMOO4SbM2blypX28MMP28CBA+3www+3fv362UknnWTLly9vEnXNlO76srq62gYNGmRDhgyxESNG2Lhx42zo0KE2e/ZsW7JkSdcrZs+yJwAACEAAAhCAAARKn0BW2Cryes8999jOO+9sffv2bdLzd955xxYvXmy77rqrpVIpt23DDTe0TTfd1J577rm8RVNTqZT179/fJKwRtg47CwhAAAIQgEA+CFAnBEqCgBO2moLw0EMPuQ4pIptKZcSrcwQLiUqVWXvttYNc5r+iosI222wz05SC6FzczNbcLSVqKysrrWfPnsYfBCAAAQhAAAIQgAAEWiPghO2rr75qjz/+uB133HEWFa9+p/r6etM0gT59+niXs+l02jQ3tqamxuVbWsycOTN7QZi/MGzKlCnW1j6+nvnz59u0adNcBHmjjTby7natIsw+RQt7nyz+dyzvHIJIP5zhrPPMJ84Hzgd/LshyPnA+6DzwifOB88GfC7IdOR+iZfx6WvNk//KXv9huu+1m22+/vfd32NbV1ZkuFGttB10EpovBokkXoCkK23wfTYe48sornRAeNWqUTZw40dZdd107/fTTrUePHs2Lk4cABCAAAQgkmgCNhwAEcksgrSkIH374oW2wwQYuavvYY4/Z22+/bZ9++qk9+eSTpm06pO5YIBGs9WhShLetaQKah6uLwaJJF6cp2hutR+uqZ9iwYe6uCOeee65NmjTJ3V5M4lbbO5o22WQT8ym6j/fJ4s8wggMc9HrwifOB88GfC7KcD5wPOg984nzgfPDngmwczodoG/x6WvebTaVS9uCDD7rbaunWWpqaoLmzDzzwgL355pu23nrruYvGFFH1O2rOrQTwGmus4aYpeH93rKY7+LsiKNLbu3fv7lRXpvvSbQhAAAIQgAAEIFCeBNKnnHKKXX311U3S/vvv7+a16iEJAwcONM1v7dWrlz3//PNZShLEErYSoLqQLLuBFQhAAAIQgECcCdA2CECgZAm4i8fa690666xju+++uz3xxBP20ksvuTm1M2bMcBeASfi2tz/bIQABCEAAAhCAAAQgkG8CHRK2aoRuA7bjjju6p42NHDnSZs2aZcOHD3f3stX2Mk90HwIQgAAEIAABCECgyARaFLbHHHOMXXLJJabpB759uouB7k5w3XXXucfcXnvtte4pYX57c6t5ubqrgepqvm2bbbZxAnnw4MHZTSqn8tov62QFAhCAAARKhADdgAAEIJB/Ai0K27YOK7GruxRI6LZVLqnbdOsy3cIs2n5dQPfKK69EXawnnIAuftSdPqLd0B1A/vvf/5ru2xz1s55sAno963Xte6Hx1ThrvL0Pm3wCvKaTP4Yd7YHeuzXevvyqVavs2WeftWXLlnkXtgQI6H1b79/RrnREj3Va2EYPUOz1XBxfLw5dCKdbm+nBERdddJHddNNNpheKr/+pp56yhx9+2GexCSWg29XNnTvXfve737l7JY8fP9409r47r732mt1///1u7rj3YZNHQG+EurB1+vTpdtVVV9n555/v7vrie6KHw2icNd7eh00mAV7TyRy3zrZar9kFCxbYvffea+PGjTN9Ts8N3st9PToPdD/+d99917uwCSSQKz1WcsJWYB599FF3/9uzzz7bTXn46KOPWhxilZXI0d0f9EGnD0N9CJ533nk8EKJFYvFyavzee+89++1vf+u+jLT1aOe//e1vdtlll9k999zj7u5x6qmn2tixY6350/Ti1UNaozHWUxE1VprbP2bMGPvHP/7hLmBtiY5e6z/+8Y/d+aBbGG644Yamh74ccsghLRUvB19i+qhbTOq1rPdfvXfri+cLL7zQavt5TbeKJjEb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)

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

![[局部截取_20261020_013335.png]]

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

![[局部截取_20261020_013513.png]]
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

![[局部截取_20261020_013718.png]]

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

![[局部截取_20261020_013559.png]]

对于动作战斗开发，我认为最值得深入研究的是以下四组关系：

GA + AbilityTask：决定技能执行的具体阶段、动画时序、取消与中断机制。

GameplayTag + GA：决定战斗状态、技能互斥、受击硬直与霸体逻辑。

GameplayEffect + AttributeSet + ExecCalc：构成统一的战斗数值与伤害结算系统。

ASC + Prediction + Replication：决定技能在多人联机环境中的预测、权威判定和状态同步。

这四组机制组合起来，才真正形成一套能够支撑复杂动作战斗的 GAS 架构。

官方资料可以继续参考 [Gameplay Ability System Overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/understanding-the-unreal-engine-gameplay-ability-system?utm_source=chatgpt.com) 和 [Gameplay Effects](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-effects-for-the-gameplay-ability-system-in-unreal-engine?utm_source=chatgpt.com)。

后续如果要深入到类似高动作性 ARPG 的具体实现，最值得展开的专题是：如何通过 GAS 实现完整的普通攻击连招、闪避取消、受击硬直、霸体和弹反系统。这部分能够进一步解释各个组件在真实战斗中的执行时序，以及哪些功能应该由 GAS 管理、哪些需要自定义战斗框架。