这一章先完整实现 **Combat Window System（战斗窗口系统）**。后面做 Combo（连招）、Dodge（闪避）、Parry（弹刀）、Motion Warping（攻击追踪）都会直接建立在它上面。

UE 当前官方文档中，`Gameplay Tag` 仍然是层级化标签体系，`UAnimNotifyState` 仍提供 `NotifyBegin / NotifyTick / NotifyEnd` 三个生命周期入口，而 GAS 的 `Gameplay Ability` 与 `Ability Task` 本身也就是为这种跨多帧、依赖动画时机的 Gameplay 行为设计的。[Epic Games Developers](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-gameplay-tags-in-unreal-engine?utm_source=chatgpt.com)

---

# 第一章：Combat Window System 战斗窗口系统

## 1. Combat Window 到底是什么？

先看一个实际的普通攻击 `Attack01`：

Attack01 动画  
0.00 ───────────────────────────────────────── 0.80s  
​  
角色转向 Tracking  
████████████  
​  
攻击判定 Hit  
          ███████  
​  
闪避取消 Dodge Cancel  
                █████████████████████  
​  
技能取消 Skill Cancel  
                       ███████████████  
​  
连招输入 Combo  
                    ███████████

这意味着在不同时间：

t = 0.10  
​  
可以：  
转向敌人  
​  
不能：  
造成伤害  
闪避取消  
接Attack02

到了：

t = 0.30  
​  
可以：  
攻击命中  
转向  
闪避取消

到了：

t = 0.55  
​  
可以：  
接Attack02  
闪避  
技能取消  
​  
但是：  
攻击HitBox已经关闭

所以所谓：

**Combat Window（战斗窗口）**

就是：

> 在一个动作的某段时间内，临时开放某种 Gameplay 权限或 Gameplay 行为。

---

# 2. Window 和 State 一定要区分

这是整个架构非常重要的一点。

## State —— 状态

例如：

State.Action.Attacking  
正在攻击  
​  
State.Action.Dodging  
正在闪避  
​  
State.Defense.SuperArmor  
处于霸体  
​  
State.Control.Stunned  
处于眩晕

它回答：

> 角色现在处于什么状态？

---

## Window —— 窗口

例如：

Window.Cancel.Dodge  
当前允许闪避取消  
​  
Window.Cancel.Skill  
当前允许技能取消  
​  
Window.Combo.Attack  
当前允许接下一段普攻  
​  
Window.Defense.Invincible  
当前处于无敌时间窗口  
​  
Window.Defense.PerfectDodge  
当前处于极限闪避判定窗口  
​  
Window.Motion.Tracking  
当前允许攻击追踪

它回答：

当前这一小段时间允许发生什么？

所以：

State  
角色现在是什么状态  
​  
Window  
当前动作开放什么权限

这是两个层级。

---

# 3. 为什么 Combat Window 非常适合 Gameplay Tag？

我们不希望写：

bool bCanDodgeCancel;  
bool bCanSkillCancel;  
bool bCanAttackCancel;  
bool bCanParry;  
bool bCanTrackTarget;  
bool bIsInvincible;  
bool bCanSwitch;  
...

因为随着系统复杂：

几十个 bool

会逐渐失控。

更好的做法是全部抽象成：

FGameplayTag

例如：

Window.Cancel.Attack  
Window.Cancel.Dodge  
Window.Cancel.Skill  
Window.Cancel.Switch  

Window.Combo.Attack  

Window.Defense.Invincible  
Window.Defense.PerfectDodge  
Window.Defense.Parry  

Window.Motion.Tracking  

Window.Hit.Melee

Gameplay Tag 本身是层级结构，所以查询：

Window.Cancel

就可以表达：

> 所有动作取消窗口。

UE 的 Tag 层级匹配就是为这类逻辑分类设计的。[Epic Games Developers](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-gameplay-tags-in-unreal-engine?utm_source=chatgpt.com)

---

# 4. 推荐的第一版目录结构

项目可以逐渐整理成：

Source/Game/  

Combat/  
│  
├── Ability/  
│ ├── GA_AttackBase  
│ ├── GA_NormalAttack  
│ └── GA_Dodge  
│  
├── Input/  
│ └── CombatInputComponent  
│  
├── Window/  
│ ├── CombatWindowComponent  
│ └── ANS_CombatWindow  
│  
├── Hit/  
│ ├── HitDetectionComponent  
│ ├── ANS_HitWindow  
│ └── CombatResolver  
│  
├── Tags/  
│ └── CombatGameplayTags  
│  
└── Data/  
    └── CombatActionData

这里：

ANS

我建议作为：

Anim Notify State  
动画通知状态

的命名前缀。

例如：

ANS_CombatWindow  
通用战斗窗口通知  

ANS_HitWindow  
攻击命中窗口通知  

ANS_RootMotionWarp  
MotionWarp窗口

---

# 5. 先创建 Gameplay Tags

如果使用 C++ Native Gameplay Tags，可以建立：

// CombatGameplayTags.h  

#pragma once  

#include "NativeGameplayTags.h"  

// 动作取消窗口  
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_Window_Cancel_Attack);  
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_Window_Cancel_Dodge);  
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_Window_Cancel_Skill);  
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_Window_Cancel_Switch);  

// 连招窗口  
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_Window_Combo_Attack);  

// 防御相关窗口  
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_Window_Defense_Invincible);  
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_Window_Defense_PerfectDodge);  
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_Window_Defense_Parry);  

// 运动相关窗口  
UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_Window_Motion_Tracking);

然后：

// CombatGameplayTags.cpp  

#include "CombatGameplayTags.h"  

UE_DEFINE_GAMEPLAY_TAG(  
    TAG_Window_Cancel_Attack,  
    "Window.Cancel.Attack"  
);  

UE_DEFINE_GAMEPLAY_TAG(  
    TAG_Window_Cancel_Dodge,  
    "Window.Cancel.Dodge"  
);  

UE_DEFINE_GAMEPLAY_TAG(  
    TAG_Window_Cancel_Skill,  
    "Window.Cancel.Skill"  
);  

UE_DEFINE_GAMEPLAY_TAG(  
    TAG_Window_Cancel_Switch,  
    "Window.Cancel.Switch"  
);  

UE_DEFINE_GAMEPLAY_TAG(  
    TAG_Window_Combo_Attack,  
    "Window.Combo.Attack"  
);  

UE_DEFINE_GAMEPLAY_TAG(  
    TAG_Window_Defense_Invincible,  
    "Window.Defense.Invincible"  
);  

UE_DEFINE_GAMEPLAY_TAG(  
    TAG_Window_Defense_PerfectDodge,  
    "Window.Defense.PerfectDodge"  
);  

UE_DEFINE_GAMEPLAY_TAG(  
    TAG_Window_Defense_Parry,  
    "Window.Defense.Parry"  
);  

UE_DEFINE_GAMEPLAY_TAG(  
    TAG_Window_Motion_Tracking,  
    "Window.Motion.Tracking"  
);

现在程序就拥有了一套统一“战斗语言”。

---

# 6. 为什么不用 FString？

不要这样：

if (WindowName == "DodgeCancel")

因为字符串：

"DodgeCancel"  
"Dodge_Cancel"  
"DodgeCancle"

写错了编译器也不知道。

而：

FGameplayTag

提供统一 Tag Dictionary（标签字典）和层级查询。

所以大型项目更适合：

FGameplayTag

而不是到处：

FString  
FName  
bool  
enum

描述复杂 Gameplay 分类。

---

# 7. 核心类：UCombatWindowComponent

接下来创建：

UCombatWindowComponent

中文：

> **战斗窗口组件**

它的职责非常简单：

打开窗口  
关闭窗口  
查询窗口  
通知其他系统窗口发生变化

注意：

它不负责：

播放动画  
扣血  
执行技能  
Trace

它只是一个：

> 战斗权限窗口管理器。

---

# 8. 最简单版本的数据结构

可以这样：

UCLASS(ClassGroup=(Combat), meta=(BlueprintSpawnableComponent))  
class YOURGAME_API UCombatWindowComponent  
    : public UActorComponent  
{  
    GENERATED_BODY()  

public:  

    void OpenWindow(FGameplayTag WindowTag);  
  
    void CloseWindow(FGameplayTag WindowTag);  
  
    bool IsWindowOpen(FGameplayTag WindowTag) const;  

private:  

    TMap<FGameplayTag, int32> ActiveWindowCounts;  

};

这里出现：

## `TMap`

中文：

> **键值映射 / 字典**

类似 C#：

Dictionary<Key, Value>

这里：

TMap<FGameplayTag, int32>

表示：

WindowTag → 当前打开次数

例如：

Window.Cancel.Dodge → 1  

Window.Motion.Tracking → 1

---

# 9. 为什么 Value 是 int32，而不是 bool？

这一点非常重要。

初学时很容易：

TMap<FGameplayTag, bool>

但是假设两个系统都打开：

Window.Defense.Invincible

例如：

Dodge Ability  
       ↓  
开启 Invincible  

某Buff  
       ↓  
也开启 Invincible

那么：

Invincible Count = 2

Dodge 结束：

Close Invincible

如果你是：

bool = false;

那么 Buff 明明还存在，角色却失去无敌。

---

正确逻辑应该类似：

第一次Open  
Count = 1  

第二次Open  
Count = 2  

一次Close  
Count = 1  

再Close  
Count = 0

只有：

Count <= 0

才算真正关闭。

这叫：

# Reference Counting

中文：

> **引用计数**

战斗状态系统里非常实用。

---

# 10. OpenWindow

可以实现：

void UCombatWindowComponent::OpenWindow(  
    FGameplayTag WindowTag)  
{  
    if (!WindowTag.IsValid())  
    {  
        return;  
    }  

    int32& Count = ActiveWindowCounts.FindOrAdd(WindowTag);  
  
    ++Count;  
  
    if (Count == 1)  
    {  
        OnWindowOpened.Broadcast(WindowTag);  
    }  

}

这里：

FindOrAdd

意思：

> 如果已经存在就找到它，否则创建。

---

假设第一次：

Window.Cancel.Dodge

不存在：

Count = 0

然后：

Count = 1

触发：

OnWindowOpened

---

# 11. Delegate——委托 / 事件

这里：

OnWindowOpened.Broadcast()

涉及 UE 非常重要的概念：

# Delegate

中文：

> **委托 / 事件回调**

它和你之前 Unity C# 里的：

event  
Action  
delegate

概念很接近。

例如：

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(  
    FCombatWindowChangedSignature,  
    FGameplayTag,  
    WindowTag  
);

然后：

UPROPERTY(BlueprintAssignable)  
FCombatWindowChangedSignature OnWindowOpened;

意思：

> 任何系统都可以监听“某个窗口打开”的事件。

例如：

CombatInputComponent

监听：

OnWindowOpened

当：

Window.Combo.Attack

打开：

马上检查InputBuffer

这就变成：

# Event Driven

中文：

> **事件驱动**

---

# 12. 不建议每 Tick 检查 Input Buffer

比较差的方案：

Tick()  
{  
    CheckInputBuffer();  
}

60 FPS：

每秒检查60次

即使什么都没有发生。

更好的方式：

玩家输入  
       ↓  
TryExecute  

如果失败  
       ↓  
进入InputBuffer  

窗口打开事件  
       ↓  
TryConsumeInputBuffer

也就是说只有：

输入发生  

或者  

窗口状态变化

时检查。

这叫：

> Event-Driven Architecture（事件驱动架构）。

---

# 13. CloseWindow

void UCombatWindowComponent::CloseWindow(  
    FGameplayTag WindowTag)  
{  
    int32* Count = ActiveWindowCounts.Find(WindowTag);  

    if (!Count)  
    {  
        return;  
    }  
  
    --(*Count);  
  
    if (*Count <= 0)  
    {  
        ActiveWindowCounts.Remove(WindowTag);  
  
        OnWindowClosed.Broadcast(WindowTag);  
    }  

}

所以：

Count > 0

窗口仍然存在。

Count == 0

真正关闭。

---

# 14. IsWindowOpen

bool UCombatWindowComponent::IsWindowOpen(  
    FGameplayTag WindowTag) const  
{  
    const int32* Count =  
        ActiveWindowCounts.Find(WindowTag);  

    return Count && *Count > 0;  

}

以后任何系统都可以：

if (  
    CombatWindow->IsWindowOpen(  
        TAG_Window_Cancel_Dodge  
    )  
)  
{  
    StartDodge();  
}

中文：

> 如果当前“闪避取消窗口”处于开启状态，则允许从当前动作切换到 Dodge。

---

# 15. 接下来真正把它放进动画

现在有了：

UCombatWindowComponent

但是什么时候调用：

OpenWindow()

？

什么时候：

CloseWindow()

？

答案就是：

# AnimNotifyState

也就是：

> 动画通知状态。

---

# 16. 创建 ANS_CombatWindow

UCLASS()  
class YOURGAME_API UANS_CombatWindow  
    : public UAnimNotifyState  
{  
    GENERATED_BODY()  

public:  

    UPROPERTY(EditAnywhere, BlueprintReadOnly)  
    FGameplayTag WindowTag;  
  
    virtual void NotifyBegin(  
        USkeletalMeshComponent* MeshComp,  
        UAnimSequenceBase* Animation,  
        float TotalDuration,  
        const FAnimNotifyEventReference& EventReference  
    ) override;  
  
    virtual void NotifyEnd(  
        USkeletalMeshComponent* MeshComp,  
        UAnimSequenceBase* Animation,  
        const FAnimNotifyEventReference& EventReference  
    ) override;  

};

官方当前 `UAnimNotifyState` 就提供 `NotifyBegin`、`NotifyTick`、`NotifyEnd`，所以非常适合表达“某段动画时间内有效”的行为。[Epic Games Developers](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/Engine/UAnimNotifyState?utm_source=chatgpt.com)

---

# 17. NotifyBegin

void UANS_CombatWindow::NotifyBegin(  
    USkeletalMeshComponent* MeshComp,  
    UAnimSequenceBase* Animation,  
    float TotalDuration,  
    const FAnimNotifyEventReference& EventReference)  
{  
    Super::NotifyBegin(  
        MeshComp,  
        Animation,  
        TotalDuration,  
        EventReference  
    );  

    if (!MeshComp)  
    {  
        return;  
    }  
  
    AActor* Owner = MeshComp->GetOwner();  
  
    if (!Owner)  
    {  
        return;  
    }  
  
    UCombatWindowComponent* WindowComponent =  
        Owner->FindComponentByClass<UCombatWindowComponent>();  
  
    if (WindowComponent)  
    {  
        WindowComponent->OpenWindow(WindowTag);  
    }  

}

也就是：

动画进入NotifyState  
        ↓  
找到角色  
        ↓  
找到CombatWindowComponent  
        ↓  
OpenWindow

---

# 18. NotifyEnd

void UANS_CombatWindow::NotifyEnd(  
    USkeletalMeshComponent* MeshComp,  
    UAnimSequenceBase* Animation,  
    const FAnimNotifyEventReference& EventReference)  
{  
    Super::NotifyEnd(  
        MeshComp,  
        Animation,  
        EventReference  
    );  

    if (!MeshComp)  
    {  
        return;  
    }  
  
    AActor* Owner = MeshComp->GetOwner();  
  
    if (!Owner)  
    {  
        return;  
    }  
  
    UCombatWindowComponent* WindowComponent =  
        Owner->FindComponentByClass<UCombatWindowComponent>();  
  
    if (WindowComponent)  
    {  
        WindowComponent->CloseWindow(WindowTag);  
    }  

}

这样：

NotifyState左边界  
       ↓  
Open  

██████████████████  

NotifyState右边界  
       ↓  
Close

---

# 19. 动画师 / 策划实际看到什么？

在：

AM_Attack01

Montage 里面右键：

Add Notify State  
    ↓  
ANS_CombatWindow

然后拖出来：

Attack01  

0.00 ───────────────────────────── 0.80  

ANS_CombatWindow  
Window.Cancel.Dodge  
             █████████████████████  

ANS_CombatWindow  
Window.Combo.Attack  
                   █████████

选中第一个：

WindowTag =  
Window.Cancel.Dodge

第二个：

WindowTag =  
Window.Combo.Attack

这样窗口时间完全变成：

> **动画时间轴数据。**

---

# 20. 这比代码里写时间强很多

不要：

Delay(0.25f);  

OpenDodge();  

Delay(0.3f);  

CloseDodge();

原因非常明显。

假如动画：

PlayRate = 1.2

也就是：

> 动画播放速度提高 20%。

代码里的：

0.25秒

不会自动对应动画关键帧。

而 Notify 在动画 Timeline（时间轴）上：

动画变快  
↓  
Notify跟着变快  

动画变慢  
↓  
Notify跟着变慢

因此：

> **动画相关的战斗时机应该尽可能跟动画时间走，而不是用大量裸 Delay。**

---

# 21. 现在接 Input Buffer

假设玩家正在：

Attack01

当前：

State.Action.Attacking

玩家在 `0.35s` 时按：

Attack

但是：

Window.Combo.Attack

还没打开。

---

CombatInputComponent：

void UCombatInputComponent::HandleAttackInput()  
{  
    if (TryExecuteAttack())  
    {  
        return;  
    }  

    BufferInput(TAG_Input_Attack);  

}

逻辑：

尝试执行  
   │  
   ├── 成功  
   │  
   └── 失败  
        ↓  
     InputBuffer

---

# 22. Buffered Input——缓存输入

可以定义：

USTRUCT()  
struct FBufferedCombatInput  
{  
    GENERATED_BODY()  

    UPROPERTY()  
    FGameplayTag InputTag;  
  
    UPROPERTY()  
    float ExpireTime = 0.f;  

};

中文：

InputTag  
缓存的是哪条输入  

ExpireTime  
这个输入什么时候过期

例如：

Attack  
缓存时间 0.20秒

玩家：

10.00秒按Attack

那么：

ExpireTime = 10.20

超过：

10.20

还没执行，就扔掉。

---

# 23. 为什么输入必须过期？

否则玩家：

一秒前按了一次Attack

角色被打飞。

一秒后站起来。

系统突然：

自动攻击

显然不对。

所以：

Input Buffer

必须有：

# Buffer Lifetime

中文：

> **输入缓存生命周期 / 有效时间**

一般会根据游戏手感配置，而不是写死。

---

# 24. Window 打开后通知 InputBuffer

UCombatInputComponent

监听：

CombatWindowComponent->OnWindowOpened

例如：

void UCombatInputComponent::OnCombatWindowOpened(  
    FGameplayTag WindowTag)  
{  
    TryConsumeBufferedInput();  
}

于是：

0.35  
玩家Attack  
↓  
不能执行  
↓  
Buffer Attack  

0.48  
Window.Combo.Attack Open  
↓  
OnWindowOpened  
↓  
检查InputBuffer  
↓  
发现Attack  
↓  
执行Attack02

这就是完整闭环。

---

# 25. 这里产生一个极重要概念：Transition

Transition

中文：

> **状态转换 / 动作转换**

当前：

Attack01

目标：

Attack02

玩家输入：

Attack

规则：

Attack01  
+  
Input.Attack  
+  
Window.Combo.Attack  
=  

Attack02

可以写成：

Current Action  
当前动作  

+  

Input  
玩家意图  

+  

Window  
当前开放权限  

+  

Gameplay State  
角色状态  

↓  

Next Action  
下一个动作

这基本就是后面：

# Combo System

连招系统的数学核心。

---

# 26. 下一步不要硬编码 Attack01 → Attack02

第一版可能：

if (CurrentAttackIndex == 1)  
{  
    StartAttack02();  
}

但是做到复杂角色以后：

Attack01  
 ├── Attack02  
 ├── HeavyAttack  
 ├── Skill01  
 ├── Dodge  
 ├── JumpAttack  
 └── Switch

所以最终应该演变成：

# Transition Rule

中文：

> **动作转换规则**

例如：

USTRUCT(BlueprintType)  
struct FCombatTransitionRule  
{  
    GENERATED_BODY()  

    // 玩家输入  
    UPROPERTY(EditAnywhere)  
    FGameplayTag InputTag;  
  
    // 当前必须开放的Window  
    UPROPERTY(EditAnywhere)  
    FGameplayTagContainer RequiredWindows;  
  
    // 当前不能存在的状态  
    UPROPERTY(EditAnywhere)  
    FGameplayTagContainer BlockedStates;  
  
    // 最终激活哪个Ability  
    UPROPERTY(EditAnywhere)  
    FGameplayTag TargetAbilityTag;  
  
    // 优先级  
    UPROPERTY(EditAnywhere)  
    int32 Priority = 0;  

};

这几个名词分别解释一下。

---

## Required Windows

中文：

> **必须存在的窗口**

比如：

Input.Attack

要求：

Window.Combo.Attack

---

## Blocked States

中文：

> **禁止状态**

例如：

State.Control.Stunned  
State.Dead

存在这些状态：

不能执行。

---

## Target Ability

中文：

> **目标技能**

规则通过以后：

激活哪个 GA？

比如：

Ability.Attack.Normal.02

---

## Priority

中文：

> **优先级**

这个以后非常重要。

假设同时允许：

Attack  
Dodge  
Skill

而 InputBuffer 里：

Attack  
Dodge

谁先执行？

可以定义：

Dodge = 100  

Skill = 80  

Attack = 50

那么：

Dodge优先

---

# 27. 为什么优先级很重要？

假设玩家几乎同时按：

Attack  
Dodge

如果你的系统永远：

谁先进入Array谁执行

手感可能很怪。

高速动作游戏经常存在：

# Input Priority

中文：

> **输入优先级**

例如设计：

Ultimate  
100  

Dodge  
90  

Skill  
80  

Switch  
70  

Attack  
50

当然具体数值是游戏设计问题。

程序系统只需要支持：

> 可以配置。

---

# 28. GAS 在这里到底放在哪？

现在加入：

Gameplay Ability System

也就是 GAS。

整个结构应该变成：

玩家输入  
    ↓  
CombatInputComponent  
    ↓  
InputBuffer  
    ↓  
Transition Rule  
    │  
    ├──检查 CombatWindow  
    ├──检查 Gameplay Tags  
    └──检查 Ability Condition  
    ↓  
ASC  
AbilitySystemComponent  
    ↓  
TryActivateAbility  
    ↓  
GA_Attack02

GAS 官方本身就支持 Ability 激活条件、阻止和取消，并使用 Gameplay Tags 参与这些条件；Ability 也可以在执行期间播放 Montage、响应输入及管理跨多帧流程。[Epic Games Developers](https://dev.epicgames.com/documentation/en-us/unreal-engine/understanding-the-unreal-engine-gameplay-ability-system?utm_source=chatgpt.com)

---

# 29. GA_Attack01 的职责是什么？

可以先把：

GA_Attack01

理解成：

> “Attack01 这次攻击行为的总导演。”

它负责：

检查Ability是否合法  
        ↓  
Commit Ability  
确认消耗/冷却  
        ↓  
添加攻击状态  
        ↓  
播放Attack01 Montage  
        ↓  
等待Montage完成  
        ↓  
结束Ability

大致：

ActivateAbility(...)  
{  
    if (!CommitAbility(...))  
    {  
        EndAbility(...);  
        return;  
    }  

    // 播放动画  
    PlayAttackMontage();  

}

---

# 30. AbilityTask

真正 GAS 里经常会使用：

UAbilityTask

中文：

> **技能任务 / 异步技能任务**

例如：

PlayMontageAndWait

中文：

> 播放 Montage 并等待它结束。

因为技能不是：

PlayAnimation();  
Damage();  
End();

一帧执行完。

而是：

第1帧  
Ability开始  

↓  

未来某帧  
攻击判定  

↓  

未来某帧  
允许连招  

↓  

未来某帧  
动画结束  

↓  

Ability结束

所以 Ability Task 是：

> 让一个 Gameplay Ability 可以跨多帧等待某件事情发生。

官方也明确指出 `UAbilityTask` 用于 Gameplay Ability 中的异步工作，可以通过 Delegate 或蓝图执行引脚继续 Ability 流程。[Epic Games Developers](https://dev.epicgames.com/documentation/unreal-engine/gameplay-ability-tasks-in-unreal-engine?utm_source=chatgpt.com)

---

# 31. 那 CombatWindow 应不应该直接放进 ASC？

这里开始进入真正工程架构问题。

有两种方案。

## 方案 A：所有 Window 都作为 ASC Gameplay Tag

例如：

Window.Cancel.Dodge

打开时：

ASC->AddLooseGameplayTag(  
    TAG_Window_Cancel_Dodge  
);

关闭：

ASC->RemoveLooseGameplayTag(  
    TAG_Window_Cancel_Dodge  
);

优点是：

Ability  
Effect  
GameplayTagQuery

全部能直接查询。

官方确实提供 `AddLooseGameplayTag`，用于添加“不由 GameplayEffect 支撑”的 Gameplay Tag。[Epic Games Developers](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/GameplayAbilities/UAbilitySystemComponent/AddLooseGameplayTag?utm_source=chatgpt.com)

---

# 32. 但是 Loose Gameplay Tag 有个重要注意点

Loose Gameplay Tag

中文可以理解：

> **松散 Gameplay Tag / 由代码直接管理的 Tag**

官方特别指出：

> Loose Tag 并不自动替你解决所有客户端/服务器同步问题，调用方需要保证在需要的客户端与服务器正确添加它。[Epic Games Developers](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/GameplayAbilities/UAbilitySystemComponent/AddLooseGameplayTag?utm_source=chatgpt.com)

所以不能形成错误理解：

AddLooseGameplayTag()  
=  

任何网络情况全部自动解决

不是。

---

# 33. 我更推荐的架构

如果目标是大型高速动作游戏，我会分成：

Gameplay State  
长期/重要Gameplay状态  
        ↓  
ASC Gameplay Tags  

Combat Window  
超短时间权限窗口  
        ↓  
UCombatWindowComponent

例如：

## 放 ASC

State.Action.Attacking  

State.Action.Dodging  

State.Defense.SuperArmor  

State.Control.Stunned  

State.Forte.Enhanced

因为这些是真正：

> Gameplay State。

---

## 放 CombatWindowComponent

Window.Combo.Attack  

Window.Cancel.Dodge  

Window.Cancel.Skill  

Window.Motion.Tracking

因为这些主要属于：

> 当前动画局部时间权限。

---

必要的时候再把重要窗口映射：

Window.Defense.Invincible  

Window.Defense.Parry

给 Combat Resolver 查询。

这样不会把 ASC 变成：

> 每个 Montage 一秒不停 Add/Remove 一堆临时 Tag 的唯一窗口系统。

---

# 34. Hit Window 不建议完全做成普通 CombatWindow

这个区别尤其重要。

例如：

Window.Cancel.Dodge

只需要回答：

Open / Closed

所以通用：

ANS_CombatWindow

很好。

但是：

# Hit Window

命中窗口需要更多数据。

例如：

哪种Hit Profile？  

使用Sword还是拳头？  

Sphere半径多少？  

是否多段命中？  

造成什么攻击等级？  

是否能弹刀？

所以建议：

ANS_HitWindow

单独设计。

---

# 35. Hit Profile 是什么？

Hit Profile

中文：

> **攻击判定配置 / 命中配置**

例如：

USTRUCT(BlueprintType)  
struct FHitWindowData  
{  
    GENERATED_BODY()  

    UPROPERTY(EditAnywhere)  
    FName TraceProfile;  
  
    UPROPERTY(EditAnywhere)  
    float Radius = 15.f;  
  
    UPROPERTY(EditAnywhere)  
    FGameplayTag AttackType;  
  
    UPROPERTY(EditAnywhere)  
    bool bAllowMultiHit = false;  

};

比如：

Attack01  

TraceProfile:  
Sword_Long  

Radius:  
15  

AttackType:  
Attack.Melee.Light  

MultiHit:  
false

---

# 36. ANS_HitWindow

就可以：

NotifyBegin  
    ↓  
BeginHitDetection()  

NotifyTick  
    ↓  
PerformWeaponSweep()  

NotifyEnd  
    ↓  
EndHitDetection()

注意：

普通：

Cancel Window

不需要 Tick。

而：

Hit Window

可能需要每帧 Sweep。

所以不能为了“统一”硬把所有东西塞进一个万能：

ANS_CombatWindow

---

# 37. 一个重要的软件设计原则

这叫：

# Generalization vs Specialization

中文：

> **通用化与专用化**

能通用的：

Cancel  
Combo  
Tracking Permission  
Switch Permission

使用：

ANS_CombatWindow

需要特殊 Runtime 行为的：

Hit Window  
Motion Warping  
Weapon Trail

使用：

ANS_HitWindow  
ANS_MotionWarpWindow  
ANS_WeaponTrail

不要最后做出：

UANS_Everything  
{  
    bool bHit;  
    bool bDodge;  
    bool bParry;  
    bool bTrack;  
    bool bWarp;  
    bool bVFX;  
    ...  
}

那会重新变成垃圾桶。

---

# 38. 现在看 Attack01 的完整 Montage

我们已经可以设计：

AM_Attack01  

0.00 ───────────────────────────────────── 0.80  

Tracking  
攻击追踪  
████████████████  

Hit  
攻击判定  
        ███████  

Dodge Cancel  
闪避取消  
             ███████████████████████  

Combo  
连招窗口  
                     ██████████  

Skill Cancel  
技能取消  
                         █████████████  

Switch Cancel  
切人取消  
                  ████████████████████

Animation Montage 里面实际上放：

ANS_CombatWindow  
Window.Motion.Tracking  

ANS_HitWindow  
Attack01HitData  

ANS_CombatWindow  
Window.Cancel.Dodge  

ANS_CombatWindow  
Window.Combo.Attack  

ANS_CombatWindow  
Window.Cancel.Skill  

ANS_CombatWindow  
Window.Cancel.Switch

这就是一条完整：

# Combat Timeline

中文：

> **战斗时间轴**

---

# 39. 玩家执行 Attack01 的完整时序

现在可以真正完整走一遍。

玩家按 Attack  
      │  
      ▼  
Enhanced Input  
      │  
      ▼  
IA_Attack  
      │  
      ▼  
CombatInputComponent  
      │  
      ▼  
当前是否可以直接Attack？  
      │  
     Yes  
      │  
      ▼  
TryActivateAbility  
      │  
      ▼  
GA_Attack01  
      │  
      ▼  
添加  
State.Action.Attacking  
      │  
      ▼  
播放  
AM_Attack01

动画到：

0.00

打开：

Window.Motion.Tracking

角色开始：

向目标修正Rotation

---

到：

0.18

进入：

ANS_HitWindow

执行：

BeginHitDetection

每帧：

Sword Sweep

命中敌人：

FHitResult  
     ↓  
CombatResolver  
     ↓  
NormalHit  
     ↓  
GameplayEffect  
     ↓  
Damage

---

到：

0.28

打开：

Window.Cancel.Dodge

现在玩家可以：

Attack01 → Dodge

---

到：

0.45

打开：

Window.Combo.Attack

系统检查：

InputBuffer

如果之前玩家已经按：

Attack

立即：

Consume Input  
      ↓  
Attack Transition  
      ↓  
GA_Attack02

---

# 40. Attack02 如何中断 Attack01？

这里以后会进入：

# Ability Cancellation

中文：

> **技能取消 / Ability 中断**

例如：

GA_Attack01

当前运行。

系统确认：

Window.Combo.Attack

打开。

然后：

GA_Attack02

允许激活。

可以选择：

Cancel GA_Attack01  
      ↓  
结束Attack01 Montage  
      ↓  
开始Attack02 Montage

或者：

同一个GA_NormalAttack  
     ↓  
Montage Jump Section  
     ↓  
Attack01 → Attack02

这其实就是两种架构。

---

# 41. “每段攻击一个 GA”还是“整套普攻一个 GA”？

这个问题很关键。

## 方案 A

GA_Attack01  
GA_Attack02  
GA_Attack03  
GA_Attack04

优点：

每段完全独立  
逻辑非常清晰  
特殊攻击容易修改

缺点：

Ability数量很多  
连招管理复杂  
Ability切换频繁

---

## 方案 B

GA_NormalAttack

内部：

Attack01  
↓  
Attack02  
↓  
Attack03  
↓  
Attack04

使用：

Montage Section

控制。

优点：

普攻连段统一  
动画衔接方便

缺点：

GA内部逻辑越来越复杂

---

# 42. 我会怎么选？

对于《鸣潮》这种角色：

普攻1  
普攻2  
普攻3  
普攻4

关系非常紧密的连续动作，我倾向：

GA_NormalAttack

一个 GA。

内部维护：

int32 ComboIndex;

然后：

Section_Attack01  
Section_Attack02  
Section_Attack03  
Section_Attack04

但是：

重击  
闪避反击  
共鸣技能  
空中攻击  
下落攻击

则是不同 Ability。

例如：

GA_NormalAttack  

GA_HeavyAttack  

GA_AirAttack  

GA_PlungeAttack  

GA_Dodge  

GA_DodgeCounter

这个粒度通常更合理。

---

# 43. 一个非常容易踩的 AnimNotifyState 坑

不要在：

UANS_CombatWindow

内部保存角色 Runtime State（运行时角色状态）。

例如不要：

UPROPERTY()  
ACharacter* CurrentCharacter;

然后：

NotifyBegin  
CurrentCharacter = A  

NotifyBegin  
CurrentCharacter = B

因为动画 Notify 对象属于动画资产体系，并不是应该承载“某一个角色这次攻击实例状态”的地方。

正确原则：

AnimNotifyState  
只描述：  

“这条动画这里是什么窗口”

Runtime 数据：

窗口开了几层  
当前Ability是谁  
命中过哪些敌人  
当前输入是什么

全部放：

Character Component  
Gameplay Ability  
ASC

里。

---

# 44. 第二个非常容易踩的坑：窗口清理

假设：

Attack01  

       Dodge Window  
       ███████████████

但是角色在窗口中间：

被Boss打飞

Attack Montage 被：

Interrupt

中文：

> **中断**

这个时候必须确保：

Window.Cancel.Dodge

被清理。

否则角色以后可能永久：

CanDodgeCancel = true

这种 Bug 特别难查。

---

# 45. 所以需要 Cleanup

Cleanup

中文：

> **清理 / 收尾**

不能只依赖：

理想情况下NotifyEnd肯定执行

更成熟的系统会有第二层保险。

例如 Ability End：

void UGA_AttackBase::EndAbility(...)  
{  
    CombatWindowComponent->ClearWindowsBySource(this);  

    Super::EndAbility(...);  

}

所以以后窗口最好进一步拥有：

# Source

中文：

> **来源**

例如：

Window.Cancel.Dodge  

Source:  
GA_NormalAttack Activation #123

Ability 被取消：

Clear all windows from #123

---

# 46. 更成熟的数据结构

最后可能从：

TMap<FGameplayTag, int32>

升级为：

struct FActiveCombatWindow  
{  
    FGameplayTag WindowTag;  

    TWeakObjectPtr<UObject> Source;  
  
    int32 InstanceId;  

};

这里：

## `TWeakObjectPtr`

中文：

> **弱对象指针**

简单理解：

> 指向 UObject，但不会强制让这个 UObject 因为你保存了指针而一直存活。

---

## Instance ID

中文：

> **窗口实例编号**

例如：

DodgeWindow #1001  

DodgeWindow #1002

即使 Tag 相同：

Window.Cancel.Dodge

也知道是哪一次开的。

这是以后处理：

并发  
动画打断  
Ability取消

非常有价值的能力。

---

# 47. Debug 系统必须尽早做

动作游戏最大的痛苦之一是：

> “为什么这一刀不能闪避取消？”

如果没有 Debug，你只能猜。

所以我非常建议一开始就做：

# Combat Debug HUD

例如屏幕左侧实时显示：

----- Combat State -----  

State.Action.Attacking  

----- Active Windows -----  

Window.Cancel.Dodge  
Window.Combo.Attack  
Window.Motion.Tracking  

----- Input Buffer -----  

Input.Attack 0.11s  
Input.Skill 0.07s  

----- Active Ability -----  

GA_NormalAttack  
ComboIndex = 2

这样一眼就知道：

输入有没有进来？  
窗口开没开？  
Ability有没有激活？

调手感效率会高非常多。

---

# 48. Combat Window System 到这里真正解决了什么？

现在我们已经有：

Animation  
负责时间

AnimNotifyState  
负责告诉Gameplay窗口变化

CombatWindowComponent  
负责窗口生命周期

GameplayTag  
负责描述窗口类型

InputBuffer  
负责保存玩家提前输入

GAS  
负责真正执行Ability

形成：

             Animation Timeline  
                    │  
                    ▼  
             AnimNotifyState  
                    │  
               Open / Close  
                    │  
                    ▼  
          CombatWindowComponent  
                    │  
         ┌──────────┴──────────┐  
         ▼                     ▼  
     InputBuffer          Combat Gameplay  
         │  
         ▼  

Transition Rule  
         │  
         ▼  
      GAS / ASC  
         │  
         ▼  
Gameplay Ability

这已经不是一个 Demo 结构了，而是一套可以继续扩展的动作战斗骨架。

---

# 49. 下一章就是 Combo System

现在最自然的下一步已经不是再讲 Combat Window，而是：

# **Combo System —— 连招系统**

因为目前我们已经解决：

什么时候允许Attack02？

但是还没解决：

Attack01之后究竟应该进入谁？  

Attack01 + Attack  
→ Attack02  

Attack01 + HoldAttack  
→ HeavyAttack  

Attack01 + Skill  
→ Skill01  

Attack01 + Jump  
→ AirAttack？  

Attack02 + Dodge + Attack  
→ DodgeCounter？  

空中 + Attack  
→ AirAttack  

空中 + HeavyAttack  
→ PlungeAttack

这就需要真正的：

Combat Action  
战斗动作  

Combat Transition  
动作转换  

Combo Graph  
连招图  

Transition Condition  
转换条件  

Input Priority  
输入优先级  

Action Data Asset  
动作数据资产

最后会得到这种结构：

                    Attack01  
                       │  
          ┌────────────┼───────────────┐  
          │            │               │  
     Input.Attack  Input.Heavy    Input.Dodge  
          │            │               │  
          ▼            ▼               ▼  
      Attack02    HeavyAttack        Dodge  
          │  
     ┌────┼─────────┐  
     │              │  

 Attack03 Skill01  
     │  
 Attack04

而且我们不会把它写成几十个：

if  
else if  
else if

而会进一步做成 **Data Asset（数据资产）+ Gameplay Tag + Transition Rule（转换规则）驱动的连招图**。

这一步做完以后，你就已经能够搭出一个很像《鸣潮》的“普攻/重击/技能/闪避可以互相派生和取消”的基础战斗框架了。