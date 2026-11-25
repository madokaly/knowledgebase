# **游戏 AI 状态机（State Machine）详解及经典面试题**

状态机（State Machine）是游戏开发中最基础、最常用的逻辑控制架构之一，尤其适用于角色控制、怪物 AI、Boss AI、战斗系统以及动画系统。

下面以 **UE5 游戏客户端开发、类似《鸣潮》的动作战斗游戏**为背景，重点讲解状态机的底层原理、实现方式、架构设计、优缺点以及高频面试问题。

## **一、状态机的核心知识点**

|核心知识|中文解释|重要程度|
|---|---|---|
|FSM|有限状态机|★★★★★|
|State|状态|★★★★★|
|Transition|状态转换|★★★★★|
|Enter / Update / Exit|状态生命周期|★★★★★|
|Guard Condition|状态转换条件|★★★★★|
|Transition Priority|状态转换优先级|★★★★★|
|HFSM|分层状态机|★★★★★|
|Parallel FSM|并行状态机|★★★★|
|Pushdown FSM|下推状态机、栈式状态机|★★★|
|State Pattern|状态设计模式|★★★★★|
|State Explosion|状态爆炸|★★★★★|
|StateTree|UE 分层状态逻辑框架|★★★★|

# **二、有限状态机（FSM）**

## **2.1 什么是有限状态机？**

FSM（Finite State Machine，有限状态机）是一种将复杂行为拆分为有限数量的状态，并根据事件或者条件在不同状态之间进行切换的控制模型。

普通 FSM 在同一时刻只有一个当前活动状态。

例如，一个普通怪物具有以下状态：

- Idle（待机）
    
- Patrol（巡逻）
    
- Chase（追击）
    
- Attack（攻击）
    
- Hit（受击）
    
- Dead（死亡）
    

怪物根据玩家的位置、自身血量、技能冷却、受击情况等信息决定是否进行状态切换。

![[局部截取_20261126_005746.png]]

状态机主要负责解决两个问题：

1. 当前应该执行什么行为？
    
2. 满足什么条件时，应该切换到其他行为？
    

例如，怪物正在追击玩家，当距离小于攻击距离时，状态机退出 Chase 状态并进入 Attack 状态。

## **2.2 状态机的组成**

|组成部分|作用|游戏中的例子|
|---|---|---|
|State（状态）|描述当前行为|追击、攻击|
|Current State（当前状态）|当前正在运行的状态|Chase|
|Initial State（初始状态）|初始化时进入的状态|Idle|
|Event（事件）|外部或内部发生的事件|受到攻击、动画结束|
|Transition（转换）|描述状态之间的切换关系|Chase → Attack|
|Guard（守卫条件）|判断是否允许转换|距离小于攻击距离|
|Action（动作）|状态或转换过程中执行的逻辑|播放动画、释放技能|
|Context（上下文）|状态机需要访问的数据|血量、目标、距离、冷却|

可以将 FSM 的核心逻辑理解为：

\[ NextState=\delta(CurrentState,Input) \]

即根据当前状态和输入事件或条件，决定下一个状态。

游戏中的状态机通常还会结合角色运行时数据进行判断，这种包含额外变量、条件和动作的模型，也称为 EFSM（Extended Finite State Machine，扩展有限状态机）。

# **三、状态机的生命周期**

一个游戏状态通常具有三个核心生命周期函数。

|函数|中文含义|执行时机|
|---|---|---|
|OnEnter|进入状态|刚切换到该状态时|
|OnUpdate / Tick|更新状态|状态持续运行期间|
|OnExit|退出状态|切换到其他状态时|

![[局部截取_20261126_005815.png]]

以 Chase（追击状态）为例：

```
void ChaseState::OnEnter()
{
    // 进入追击状态时执行一次
    SetMoveSpeed(RunSpeed);
    StartMoveToTarget();
}

void ChaseState::OnUpdate(float DeltaTime)
{
    // 追击状态持续运行期间执行

    // 玩家目标丢失，返回巡逻
    if (!HasValidTarget())
    {
        ChangeState(EState::Patrol);
        return;
    }

    // 进入攻击范围，切换攻击状态
    if (GetTargetDistance() <= AttackRange)
    {
        ChangeState(EState::Attack);
        return;
    }
}

void ChaseState::OnExit()
{
    // 退出状态时执行一次
    StopMovement();
}
```

**为什么需要 OnEnter 和 OnExit？**

因为有些逻辑只应该执行一次，而不应该每帧执行。

例如，进入攻击状态时播放攻击动画，如果将播放动画放在 OnUpdate 中，每帧重新播放，就可能导致动画不断从第一帧开始。

OnExit 则主要负责清理当前状态产生的副作用，例如停止移动、清理定时器、解除事件绑定、取消不应继续运行的异步任务等。

# **四、状态机的 C++ 实现方式**

游戏开发中，状态机常见的实现方式有三种。

|实现方式|适用场景|特点|
|---|---|---|
|Enum + Switch|小型状态机|简单直接|
|State Pattern（状态模式）|中大型角色、AI 系统|封装性、扩展性好|
|Data-Driven FSM（数据驱动状态机）|大型商业游戏|配置化、可视化、便于策划使用|

## **4.1 Enum + Switch 实现**

最简单、最直接的状态机实现方式。

```
// 定义所有可能的怪物状态
enum class EAIState : uint8
{
    Idle,       // 待机
    Patrol,     // 巡逻
    Chase,      // 追击
    Attack,     // 攻击
    Hit,        // 受击
    Dead        // 死亡
};
```

定义当前状态：

```
EAIState CurrentState = EAIState::Idle;
```

每帧执行对应状态的逻辑：

```
void UpdateAI(float DeltaTime)
{
    // 全局高优先级转换：死亡
    if (Health <= 0 && CurrentState != EAIState::Dead)
    {
        ChangeState(EAIState::Dead);
        return;
    }

    switch (CurrentState)
    {
        case EAIState::Idle:
        {
            if (HasValidTarget())
            {
                ChangeState(EAIState::Chase);
            }
            break;
        }

        case EAIState::Chase:
        {
            if (!HasValidTarget())
            {
                ChangeState(EAIState::Idle);
            }
            else if (TargetDistance <= AttackRange)
            {
                ChangeState(EAIState::Attack);
            }
            else
            {
                UpdateMovement();
            }
            break;
        }

        case EAIState::Attack:
        {
            // 攻击结束后重新判断目标
            if (IsAttackFinished())
            {
                ChangeState(
                    HasValidTarget()
                        ? EAIState::Chase
                        : EAIState::Idle
                );
            }
            break;
        }

        case EAIState::Dead:
        {
            // 死亡状态不再执行普通 AI 逻辑
            break;
        }

        default:
            break;
    }
}
```

状态转换函数：

```
void ChangeState(EAIState NewState)
{
    // 避免重复进入同一个状态
    if (CurrentState == NewState)
    {
        return;
    }

    // 1. 退出旧状态
    OnExitState(CurrentState);

    // 2. 修改当前状态
    CurrentState = NewState;

    // 3. 进入新状态
    OnEnterState(CurrentState);
}
```

优点是实现简单、执行开销较小、容易理解。

缺点是当状态数量增加时，Switch 和条件判断会越来越复杂，所有状态逻辑容易集中在一个类中，导致维护困难。

因此，这种实现适合状态较少、转换关系简单的系统。

## **4.2 State Pattern（状态设计模式）**

状态模式是一种行为型设计模式。

核心思想是：

**将每个状态封装成独立的类，让不同状态拥有自己的数据和行为。**

例如，将攻击、巡逻、追击分别封装为 AttackState、PatrolState、ChaseState。

![[局部截取_20261126_005830.png]]

### **第一步：定义状态基类**

```
class IState
{
public:
    virtual ~IState() = default;

    // 进入状态
    virtual void OnEnter() {}

    // 更新状态
    virtual void OnUpdate(float DeltaTime) {}

    // 退出状态
    virtual void OnExit() {}
};
```

### **第二步：定义具体状态**

```
class FAttackState : public IState
{
public:
    virtual void OnEnter() override
    {
        // 播放攻击动画
        // 激活攻击技能
        // 初始化攻击计时器
    }

    virtual void OnUpdate(float DeltaTime) override
    {
        // 更新攻击状态
        // 检测攻击阶段是否结束
    }

    virtual void OnExit() override
    {
        // 清理攻击相关事件
        // 关闭攻击检测窗口
        // 取消尚未完成的相关任务
    }
};
```

### **第三步：实现状态机管理器**

下面使用 UE 的 `TMap` 和 `TUniquePtr` 管理普通 C++ 状态对象。

```
class FStateMachine
{
private:
    // 状态机持有所有状态对象的所有权
    TMap<EAIState, TUniquePtr<IState>> States;

    // 当前状态，不拥有对象
    IState* CurrentState = nullptr;

    // 防止状态切换过程中发生重入
    bool bIsChangingState = false;

public:
    void RegisterState(
        EAIState StateID,
        TUniquePtr<IState> State)
    {
        // 状态只允许注册一次
        if (State && !States.Contains(StateID))
        {
            States.Add(StateID, MoveTemp(State));
        }
    }

    void ChangeState(EAIState NewStateID)
    {
        if (bIsChangingState)
        {
            return;
        }

        auto* FoundState = States.Find(NewStateID);

        if (!FoundState || !FoundState->IsValid())
        {
            return;
        }

        IState* NewState = FoundState->Get();

        // 避免重复进入
        if (CurrentState == NewState)
        {
            return;
        }

        bIsChangingState = true;

        // 退出旧状态
        if (CurrentState)
        {
            CurrentState->OnExit();
        }

        // 更新当前状态
        CurrentState = NewState;

        // 进入新状态
        CurrentState->OnEnter();

        bIsChangingState = false;
    }

    void Tick(float DeltaTime)
    {
        if (CurrentState)
        {
            CurrentState->OnUpdate(DeltaTime);
        }
    }
};
```

使用方式：

```
FStateMachine StateMachine;

// 提前创建并注册状态
StateMachine.RegisterState(
    EAIState::Attack,
    MakeUnique<FAttackState>()
);

// 切换状态
StateMachine.ChangeState(EAIState::Attack);

// 每帧更新
StateMachine.Tick(DeltaTime);
```

这是一个教学用的简化实现。实际项目还需要完善转换规则、上下文注入、事件队列、状态机停止与销毁时的清理等逻辑。上述重入保护会拒绝切换过程中的新请求，商业项目一般会将合法请求延迟到安全时机处理。

### **状态模式的优缺点**

|优点|缺点|
|---|---|
|状态逻辑相互独立|类的数量增加|
|增加状态更加方便|状态管理结构更复杂|
|符合单一职责原则|状态之间可能产生耦合|
|便于单元测试|存在一定的间接调用开销|
|可以复用状态对象|仍然无法直接解决状态爆炸|

## **4.3 数据驱动状态机**

Data-Driven FSM（数据驱动有限状态机）是将状态以及状态转换规则从硬编码中抽离出来，通过配置和可视化编辑器进行管理。

例如，在 UE 中可以使用 DataAsset 配置状态机：

|当前状态|目标状态|转换条件|优先级|
|---|---|---|---|
|Idle|Chase|发现目标|10|
|Chase|Attack|进入攻击范围且技能可用|20|
|Chase|Patrol|丢失目标|10|
|Attack|Chase|攻击结束|10|
|Attack|Hit|受到有效打断|80|
|任意存活状态|Dead|HP ≤ 0|100|

此表采用自定义规则：数值越大，优先级越高。

运行时由通用状态机框架读取配置、执行条件判断和转换，而具体行为由 C++、蓝图、GAS 等系统完成。

大型游戏使用这种方式的主要目的是降低代码与玩法配置的耦合，让策划可以调整状态转换关系，而不需要每次都修改底层代码。

需要注意，**状态机配置数据和运行时状态数据必须分离**。多个怪物可以共享同一份 DataAsset，但每个怪物应独立保存当前状态、目标、计时器等运行时数据。

# **五、状态转换机制详解**

状态机真正复杂的地方通常不是状态本身，而是状态之间如何转换。

## **5.1 主动转换与集中式转换**

### **主动转换（State-Driven Transition）**

由当前状态自行决定下一个状态。

```
void FChaseState::OnUpdate(float DeltaTime)
{
    if (TargetDistance <= AttackRange)
    {
        StateMachine->ChangeState(EAIState::Attack);
    }
}
```

优点是实现简单，每个状态可以管理自己的转换逻辑。

缺点是状态需要了解其他状态，随着转换关系增加，容易产生耦合。

### **集中式转换（Centralized Transition）**

将状态转换规则集中交给状态机管理器处理。

```
void FStateMachine::Tick(float DeltaTime)
{
    // 1. 检查全局高优先级转换
    if (CheckGlobalTransition())
    {
        return;
    }

    // 2. 检查当前状态的转换规则
    if (CheckLocalTransition())
    {
        return;
    }

    // 3. 没有转换时执行当前状态逻辑
    if (CurrentState)
    {
        CurrentState->OnUpdate(DeltaTime);
    }
}
```

这种方式更加方便统一处理死亡、受击、优先级、调试日志以及状态合法性检查。

大型项目也可以采用混合方式：具体状态负责产生转换请求，状态机管理器负责最终裁决。

## **5.2 状态转换优先级**

假设 Boss 当前正在攻击，同一帧出现以下情况：

- 攻击动画结束。
    
- 玩家离开攻击范围。
    
- Boss 韧性值被打空。
    
- Boss 血量降低至零。
    

此时应该切换到哪个状态？

需要通过 Transition Priority（转换优先级）解决冲突。

例如：

|状态转换|优先级|
|---|---|
|死亡 Dead|100|
|韧性击破 Break|90|
|强制受击 Hit|80|
|闪避 Dodge|60|
|攻击 Attack|40|
|追击 Chase|20|
|待机 Idle|10|

实际运行效果可以通过下面的例子理解。

![[局部截取_20261126_005904.png]]

这里即使有多个转换条件同时成立，也只选择优先级最高的合法转换。

实际项目中还需要区分：

- Transition Priority（转换优先级）：多个转换同时成立时，优先执行哪一个。
    
- Interruptibility（可打断性）：当前状态是否允许被某种行为打断。
    
- Guard Condition（守卫条件）：某条转换是否满足执行要求。
    

例如，Boss 在霸体攻击期间，即使受击状态优先级高，也可能因为不可打断条件而不能进入受击状态。

**优先级高不代表一定可以打断当前状态。**

## **5.3 Tick 驱动与事件驱动**

|对比|Tick 驱动|Event 驱动|
|---|---|---|
|中文含义|每帧轮询|事件触发|
|触发时机|每帧或固定间隔|特定事件发生时|
|实现方式|不断检测条件|接收事件后检查|
|CPU 开销|与更新频率相关|与事件频率相关|
|适合场景|距离、移动、持续转向|死亡、受击、技能结束|

Tick 驱动示例：

```
void Tick(float DeltaTime)
{
    // 每帧检查是否进入攻击范围
    if (Distance <= AttackRange)
    {
        RequestState(EAIState::Attack);
    }
}
```

事件驱动示例：

```
void OnHealthChanged(float NewHealth)
{
    // 血量发生变化时才检查死亡
    if (NewHealth <= 0)
    {
        RequestState(EAIState::Dead);
    }
}
```

商业游戏中通常采用混合驱动方式：

持续性行为使用 Tick，离散事件使用 Delegate（委托）、Gameplay Event（玩法事件）等进行通知。

这样既能保证响应及时，也能减少不必要的轮询。

# **六、状态爆炸（State Explosion）**

这是状态机面试中非常高频的问题。

## **6.1 什么是状态爆炸？**

当角色具有多种相互独立的行为维度时，如果使用一个普通 FSM 表示所有组合，就可能导致状态数量急剧增长。

例如一个动作游戏角色具有：

|行为维度|状态|
|---|---|
|移动|待机、行走、奔跑、跳跃、下落|
|战斗|空闲、普攻、重击、技能|
|异常|正常、眩晕、冻结|

如果将这些维度全部组合成一个扁平状态机：

$$
5 \times 4 \times 3 = 60
$$

最多可能产生 60 种组合状态（实际合法组合通常少于这个数字）。

例如：

```
Idle_Normal
Idle_Attack
Run_Normal
Run_Attack
Jump_Attack
Fall_Attack
Run_HeavyAttack
Jump_HeavyAttack
……
```

当进一步增加游泳、攀爬、闪避、武器切换等功能时，状态数量和转换关系都会迅速增加。

对于 N 个状态，如果每个状态都可以直接转换到其他所有状态，最多需要：

$$
N(N-1)
$$

条有向转换关系。

![[局部截取_20261126_010009.png]]

## **6.2 如何解决状态爆炸？**

常用解决方式包括：

- HFSM（分层状态机）：将共同行为和转换抽取到父状态。
    
- Parallel FSM（并行状态机）：将独立行为拆分成多个状态机。
    
- Gameplay Tag（玩法标签）：表达可叠加的状态属性和限制条件。
    
- GAS（玩法能力系统）：管理技能的激活、取消、互斥和效果。
    
- Behavior Tree（行为树）：组织复杂的 AI 决策逻辑。
    
- Utility AI（效用 AI）：通过评分选择行为，减少大量显式转换关系。
    

其中，分层状态机与并行状态机是解决传统 FSM 复杂度问题的核心方案。

# **七、HFSM（Hierarchical Finite State Machine，分层有限状态机）**

## **7.1 核心原理**

分层状态机将状态按照层级进行组织，一个父状态内部可以包含多个子状态。

例如 Boss AI：

![[局部截取_20261126_010036.png]]

当 Boss 正在攻击时，其活动状态路径可以表示为：

```
Alive（存活）
    └── Combat（战斗）
          └── Attack（攻击）
```

与普通 FSM 不同，HFSM 可以同时激活一条父子状态路径。

Attack 负责攻击逻辑，Combat 负责战斗相关的公共逻辑，Alive 负责所有存活状态共享的逻辑。

## **7.2 为什么分层状态机可以减少转换关系？**

普通 FSM 中，每个存活状态可能都需要一条死亡转换：

```
Idle   → Dead
Patrol → Dead
Chase  → Dead
Attack → Dead
Hit    → Dead
```

使用 HFSM 后，可以将这些共用规则统一放在 Alive 父状态：

```
Alive → Dead

转换条件：HP <= 0
```

无论当前子状态是巡逻、追击还是攻击，只要满足父状态的死亡转换条件，就可以统一退出存活状态。

这样可以减少重复逻辑和转换关系。

## **7.3 分层状态机的进入和退出顺序**

假设当前状态为：

```
Alive → Combat → Attack
```

现在需要切换为：

```
Alive → NonCombat → Patrol
```

状态机首先找到 LCA（Lowest Common Ancestor，最近公共祖先），也就是 Alive。

然后从当前叶子状态向上退出，再从公共祖先向下进入目标状态。

![[局部截取_20261126_010051.png]]

父状态不需要重新进入或退出，避免重复初始化共享逻辑。

如果仅从 Attack 切换到 Chase，那么 Combat 和 Alive 都可以保持激活。

## **7.4 HFSM 的优缺点**

优点：

- 可以复用父状态的公共逻辑。
    
- 减少重复状态转换。
    
- 结构更加清晰，适合复杂角色和 Boss AI。
    
- 方便处理全局事件和统一打断。
    

缺点：

- 父子状态的生命周期管理更加复杂。
    
- 父子转换规则可能发生冲突。
    
- 层级过深会增加调试难度。
    
- 无法完全解决多个独立行为维度之间的组合爆炸。
    

HFSM 是对普通 FSM 的扩展，而不是万能的解决方案。

# **八、Parallel FSM（并行状态机）**

## **8.1 核心原理**

并行状态机是将不同的行为维度拆分为多个相对独立的状态机，让它们同时运行。

例如，动作游戏中的角色可以同时处于：

- 移动状态：Run（奔跑）
    
- 战斗状态：Attack（攻击）
    
- 装备状态：Sword（持剑）
    
- 异常状态：Poison（中毒）
    

不需要专门创建一个 Run_Attack_Sword_Poison 状态。

![[局部截取_20261126_010108.png]]
## **8.2 并行状态机如何处理冲突？**

这是并行状态机最重要的问题。

例如：

|当前状态|产生的冲突|
|---|---|
|奔跑 + 站立重击|是否允许移动？|
|攻击 + 眩晕|是否取消攻击？|
|跳跃 + 地面技能|是否允许释放技能？|
|死亡 + 任意技能|是否强制终止？|

如果各个状态机完全独立，没有统一协调机制，就可能产生互相矛盾的行为。

在 UE 中，可以使用 Gameplay Tag（玩法标签）进行跨状态机协调。

```
// 示例 Gameplay Tags

State.Movement.Running        // 奔跑
State.Movement.InAir          // 空中

State.Combat.Attacking        // 攻击中
State.Combat.SuperArmor       // 霸体

State.Control.Stunned         // 眩晕
State.Control.Frozen          // 冻结

State.Life.Dead               // 死亡
```

攻击技能的激活条件可以配置为：

```
Activation Blocked Tags（激活阻止标签）

State.Control.Stunned
State.Control.Frozen
State.Life.Dead
```

这样，技能系统不需要直接访问异常状态机的内部实现，只需要通过标签判断角色是否允许释放技能。

不过，中毒、燃烧等可以同时存在的 Buff，通常更适合使用 Gameplay Effect 或独立效果容器管理，而不是全部放进一个互斥状态机。

## **8.3 并行是否意味着多线程？**

**不是。**

并行状态机的并行指逻辑上多个状态机同时处于活动状态，不代表它们必须运行在不同的 CPU 线程中。

完全可以在游戏线程中依次更新：

```
void AMyCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    LifeFSM.Tick(DeltaTime);
    MovementFSM.Tick(DeltaTime);
    CombatFSM.Tick(DeltaTime);
    WeaponFSM.Tick(DeltaTime);
}
```

需要注意更新顺序，否则可能出现死亡状态已经发生，但战斗状态机仍然执行攻击的问题。

实际项目可以采用统一的状态协调器，先处理高优先级事件，再执行各个子状态机。

# **九、Pushdown FSM（下推状态机、栈式状态机）**

普通状态机切换时，通常直接退出旧状态。

栈式状态机则可以将当前状态暂停，把新状态压入栈中，当新状态结束后恢复之前的状态。

例如 NPC 正在巡逻，临时进入调查状态。

![[局部截取_20261126_010132.png]]

栈式状态机一般还会增加两个生命周期函数：

|函数|作用|
|---|---|
|OnEnter|首次进入状态|
|OnUpdate|更新状态|
|OnExit|完全退出状态|
|OnPause|被其他状态暂时覆盖|
|OnResume|恢复之前的状态|

适合临时中断后需要恢复的行为，例如 NPC 巡逻、菜单系统、临时交互。

但对于动作游戏的受击打断，需要谨慎使用。角色受击结束后，通常不能直接恢复已经被取消的攻击，否则可能出现动画、攻击检测、技能消耗不一致的问题。

# **十、动作游戏中的战斗状态机设计**

以类似《鸣潮》的动作游戏为例，下面是一种可行的通用架构，并非该游戏实际项目的内部实现。

## **10.1 整体架构**

![[局部截取_20261126_010153.png]]

## **10.2 攻击状态的细分**

一个完整的攻击动作通常可以分为三个阶段。

|阶段|中文含义|主要逻辑|
|---|---|---|
|Startup|攻击前摇|播放起手动作、准备攻击|
|Active|攻击有效期|开启攻击检测、计算命中|
|Recovery|攻击后摇|关闭检测、等待动作恢复|

![[局部截取_20261126_010224.png]]

这些阶段可以实现为 Attack 父状态的子状态，也可以使用技能内部阶段变量、Ability Task（能力任务）、AnimNotify（动画通知）等实现。

不一定每个攻击阶段都需要创建独立的状态类。

## **10.3 攻击打断机制**

攻击状态是否可以被其他状态打断，通常取决于当前攻击阶段和角色属性。

下面是一份示例打断表：

|当前阶段|普通受击|韧性击破|闪避取消|死亡|
|---|---|---|---|---|
|攻击前摇|允许|允许|不允许|允许|
|攻击有效期|霸体时不允许|允许|不允许|允许|
|攻击后摇|允许|允许|窗口内允许|允许|

其核心实现思路：

```
bool CanInterrupt(EAIState TargetState)
{
    // 死亡具有最高打断权限
    if (TargetState == EAIState::Dead)
    {
        return true;
    }

    // 韧性击破强制打断
    if (TargetState == EAIState::Break)
    {
        return true;
    }

    // 普通受击需要检查霸体
    if (TargetState == EAIState::Hit)
    {
        return !bHasSuperArmor;
    }

    // 闪避需要检查取消窗口
    if (TargetState == EAIState::Dodge)
    {
        return bCanDodgeCancel;
    }

    return false;
}
```

这里的 `Break`、`Dodge` 是前面基础枚举需要扩展的状态，代码仅用于说明打断规则。

**打断机制不仅是状态切换，还必须处理旧状态的资源清理。**

例如攻击被眩晕打断时，需要关闭伤害检测、取消技能任务、清理计时器、解除委托绑定，以及按照玩法规则处理动画、Root Motion（根运动）和移动限制。

## **10.4 Input Buffer（输入缓冲）**

动作游戏中，玩家可能在第一段普攻尚未结束时，就按下了第二次攻击键。

如果直接丢弃这个输入，玩家会感觉操作不流畅；如果立即切换状态，又可能导致攻击动作被错误打断。

通常会通过输入缓冲解决。

![[局部截取_20261126_010245.png]]

输入缓冲负责记录输入类型和时间，状态机负责在合法的连招窗口或者取消窗口中消费输入。

还需要设置输入过期时间，防止很久之前的输入导致角色突然执行动作。

# **十一、UE5 中常见的状态机系统**

## **11.1 Animation State Machine（动画状态机）**

UE 动画蓝图中的状态机主要用于控制动画姿势及其混合关系。

![[局部截取_20261126_010311.png]]

常见动画状态：

```
Locomotion（基础移动）
    ├── Idle（待机）
    ├── Walk（行走）
    └── Run（奔跑）

Jump（起跳）
    ↓
Fall Loop（下落循环）
    ↓
Land（落地）
```

核心组成：

|UE 名词|中文解释|
|---|---|
|State|动画状态|
|Transition Rule|动画转换规则|
|Blend Duration|动画混合时长|
|Priority Order|转换优先级|
|Conduit|转换条件汇聚与分流节点|
|State Alias|状态别名，简化共享转换关系|
|Max Transitions Per Frame|每帧最大转换次数|

动画状态机允许在不同状态之间进行动画混合。

Epic Developer Community+1

### **动画状态机与逻辑状态机的区别**

|对比|逻辑状态机|动画状态机|
|---|---|---|
|负责内容|游戏玩法逻辑|动画姿势输出|
|攻击判定|决定能否攻击|表现攻击动作|
|状态转换|玩法事件、条件|动画参数、转换规则|
|网络同步|通常需要权威校验|通常根据同步数据驱动|
|状态数量|根据玩法设计|根据动画需求设计|

例如，角色逻辑已经进入落地状态，但动画仍然在进行下落到落地的混合，这是正常的表现层过渡。

逻辑状态和动画状态不一定需要一一对应。

**动画状态机不应该成为整个战斗系统唯一的权威状态来源。**

尤其是多人游戏中，服务器需要独立掌握技能激活、命中、伤害等关键玩法信息。

### **UE 动画状态机多个转换同时成立怎么办？**

例如：

```
Jump → Fall Loop
Jump → Land
```

如果两个转换条件同时成立，UE 动画状态机会优先选择 `Priority Order` 数值较小的转换。

Epic Developer Community

例如：

|转换|Priority Order|
|---|---|
|Jump → Land|1|
|Jump → Fall Loop|2|

此时 Land 优先。

如果两个转换的优先级相同，不建议依赖内部遍历顺序，最好设置明确的优先级，或者使转换条件互斥。

需要注意，UE 动画状态机是数值越小优先级越高，与前面自定义 FSM 示例的规则不同。

## **11.2 StateTree（状态树）**

StateTree 是 UE 提供的一种通用分层状态逻辑框架，将行为树的状态选择能力与状态机的状态转换机制相结合。

![[局部截取_20261126_010335.png]]

主要组件：

|组件|中文解释|作用|
|---|---|---|
|State|状态|组织行为|
|Task|任务|执行移动、等待、攻击等行为|
|Evaluator|数据评估器|提供运行时决策数据|
|Enter Condition|进入条件|判断状态能否被选择|
|Transition|转换|控制状态切换|
|Context Data|上下文数据|提供角色、控制器等外部对象|

StateTree 选中叶子状态时，其父状态路径也可以保持激活，相应的 Task 可以共同运行。任务结果、事件和条件都可以参与状态切换。

它适合组织复杂 NPC 行为、分层 AI、交互流程等逻辑。


# **十二、FSM、行为树、效用 AI 的区别**

这是游戏 AI 开发中非常经典的对比面试题。

![[局部截取_20261126_010357.png]]

|对比项|FSM|Behavior Tree|Utility AI|
|---|---|---|---|
|中文名称|状态机|行为树|效用 AI|
|核心思想|状态转换|层级任务选择|行为评分|
|行为选择|显式转换规则|节点条件及执行规则|效用分数|
|当前行为|通常持续到转换|由树的执行机制决定|可定期重新评估|
|可控性|高|高|取决于评分设计|
|扩展性|普通 FSM 较差|较好|较好|
|主要问题|状态爆炸|树结构复杂|评分调参、行为抖动|
|适合场景|战斗动作、角色状态|NPC、怪物决策|Boss 技能选择、动态决策|

大型动作游戏完全可以将三者结合使用。

例如：

```
Behavior Tree / Utility AI
    负责决定 Boss 下一步做什么
                 ↓
Combat FSM / HFSM
    负责管理攻击、前摇、后摇、打断
                 ↓
GAS
    负责技能激活、效果、冷却、消耗
                 ↓
Animation System
    负责动画播放与混合
```

效用 AI 决定 Boss 应该使用哪个技能，状态机保证技能按照合法的阶段执行。

# **十三、状态机常见性能优化**

|优化方式|原理|
|---|---|
|事件驱动|减少每帧重复检测|
|降频更新|非关键 AI 每隔一定时间更新|
|AI LOD|根据重要程度调整 AI 更新频率|
|状态对象复用|避免每次状态切换都分配内存|
|缓存上下文|避免重复计算距离、目标、感知数据|
|条件短路|优先检测低成本、高优先级条件|
|分帧更新|避免大量 AI 在同一帧执行复杂判断|
|限制转换次数|避免同一帧发生无限状态切换|
|批量数据处理|大规模 AI 使用数据导向设计提高缓存命中率|

例如，场景中存在 1000 个怪物，不需要让所有怪物都以相同频率执行技能选择和寻路。

近距离战斗怪物可以高频更新，远距离怪物可以降低决策频率，不重要的怪物甚至可以暂停部分 AI 逻辑。

但死亡、受击等关键事件仍应根据玩法要求及时处理。

状态机的主要性能开销往往不是状态切换本身，而是状态内部执行的寻路、射线检测、环境感知、目标搜索等复杂操作。

# **十四、状态机经典高频面试题及答案**

## **1. 什么是有限状态机？ ★★★★★**

有限状态机是一种由有限数量的状态、输入事件、转换条件和转换规则组成的控制模型。

普通 FSM 同一时刻只有一个当前活动状态，通过输入或条件决定下一步状态。

游戏中通常通过 OnEnter、OnUpdate、OnExit 管理状态生命周期。

## **2. 状态机和状态设计模式有什么区别？ ★★★★★**

状态机是一种描述状态及其转换关系的控制模型。

状态模式是一种面向对象设计模式，将不同状态的行为封装到独立对象中。

状态机可以使用 Switch 实现，也可以使用状态模式实现。

状态模式是状态机的一种实现手段，而不是状态机本身。

## **3. 状态机为什么需要 Enter、Update、Exit？ ★★★★★**

主要是为了管理不同生命周期阶段的逻辑。

Enter 负责初始化，Update 负责持续行为，Exit 负责清理副作用。

例如攻击状态的 Enter 激活技能，Update 处理持续逻辑，Exit 清理攻击任务。

这样可以避免重复初始化、内存泄漏以及旧状态逻辑残留。

## **4. 什么是状态爆炸？如何解决？ ★★★★★**

当状态数量增加，尤其是多个独立行为维度相互组合时，状态数量和转换关系会急剧增加。

解决方案：

|问题|解决方案|
|---|---|
|重复公共状态转换|HFSM（分层状态机）|
|多维度状态组合|Parallel FSM（并行状态机）|
|可叠加状态属性|Gameplay Tag、Buff 系统|
|AI 决策转换过多|行为树、效用 AI|
|状态行为代码过于集中|状态模式|

核心思想是减少重复逻辑和不必要的状态组合。

## **5. 普通 FSM 与 HFSM 有什么区别？ ★★★★★**

普通 FSM 中的状态通常处于同一层级。

HFSM 支持父子状态结构，子状态可以共享父状态的行为和转换规则。

例如将所有存活状态的死亡转换统一放在 Alive 父状态中，从而减少重复判断。

HFSM 切换时还需要管理父子状态的进入、退出顺序。

## **6. 多个状态转换条件同时成立怎么办？ ★★★★★**

通常通过统一的优先级机制进行裁决。

先检查转换条件是否成立，再判断当前状态是否允许被打断，然后选择优先级最高的合法转换。

相同优先级需要具有明确、稳定的决胜规则。

多人确定性模拟中，尤其不能依赖不稳定的容器遍历顺序。

## **7. 如何防止状态机频繁来回切换？ ★★★★★**

使用 Hysteresis（滞回机制）、最短状态持续时间、冷却时间、状态锁等方式。

例如怪物的攻击范围为 3 米，如果进入和退出攻击状态都使用 3 米作为判断条件，那么目标在边界附近移动时，怪物就可能不断切换攻击和追击状态。

可以设置两个不同的阈值。

![[局部截取_20261126_010458.png]]

这样即使玩家在 3 米附近频繁移动，也不会导致状态不断抖动。

该机制也经常用于 AI 决策、LOD 切换和移动速度判断。

## **8. 状态机的状态对象应该每次创建还是复用？ ★★★★**

通常优先复用。

可以在角色初始化时创建状态对象，切换状态时只修改当前状态引用。

这样可以减少动态内存分配、释放以及潜在的 GC 开销。

不过，状态对象复用时需要注意重置计时器、目标引用和其他临时数据，避免上一次执行的数据影响下一次进入。

对于普通 C++ 对象可以使用 `TUniquePtr` 管理；对于 UObject 状态对象则应使用 UE 的 UObject 生命周期与 GC 引用管理机制。

## **9. 如何处理状态机切换时的重入问题？ ★★★★★**

重入（Reentrancy）是指状态机正在执行一次转换时，又触发了新的状态转换。

例如：

```
ChangeState(Attack)
    → Attack.OnEnter()
        → 激活技能
            → 技能立即失败
                → ChangeState(Idle)
```

如果处理不当，可能出现重复 Exit、状态覆盖、生命周期顺序错误等问题。

解决方式通常是使用转换锁和 Pending Transition（待处理转换请求）队列。

在当前转换完整结束后，再统一检查并执行新的合法请求，同时限制单帧转换次数，防止循环转换。

## **10. 如何处理状态机的全局事件？ ★★★★★**

例如死亡、强制眩晕、脱离战斗等事件，不应该在每个状态中重复编写处理逻辑。

可以使用：

- 全局转换规则（Global Transition）。
    
- HFSM 父状态转换。
    
- 统一事件分发器。
    
- 状态协调器。
    

例如 HP 降低到零后，统一请求进入 Dead 状态，再由状态机取消当前活动行为。

## **11. 状态机和行为树哪个更好？ ★★★★★**

不存在绝对的优劣。

状态机适合状态明确、转换关系固定、动作控制严格的系统。

行为树适合复杂条件组合、层级决策、任务序列等场景。

对于 Boss AI，可以使用行为树或者效用 AI 负责决策，使用 FSM 管理具体攻击动作。

两者可以互相配合，而不是互相替代。

## **12. 状态机如何处理异步任务？ ★★★★★**

例如 AI 发起异步寻路之后，还没有收到结果就进入了死亡状态。

如果寻路完成时直接执行回调，可能导致已经死亡的 AI 重新开始移动。

通常通过以下方式解决：

- 在 OnExit 中取消异步任务。
    
- 解除相关 Delegate 绑定。
    
- 使用弱引用检查目标对象是否有效。
    
- 使用 Generation ID（状态版本号）过滤过期回调。
    

例如：

```
// 每次状态切换时增加版本号
++StateVersion;

// 发起异步请求时记录当前版本
const uint32 RequestVersion = StateVersion;

// 异步回调执行时
if (RequestVersion != StateVersion)
{
    // 状态已经发生变化，丢弃旧任务结果
    return;
}
```

如果回调跨线程，还需要正确处理线程同步和对象生命周期。

## **13. 多人游戏中状态机如何同步？ ★★★★★**

对于服务器权威的多人动作游戏，服务器通常负责最终战斗状态和合法性校验。

客户端可以对移动、技能激活等操作进行预测。

需要同步的数据可能包括：

|同步数据|作用|
|---|---|
|StateID|当前逻辑状态|
|StateVersion|状态版本|
|EnterServerTime|状态进入的服务器时间|
|SkillID|当前技能|
|SkillPhase|当前技能阶段|
|PredictionKey|将客户端预测与服务器结果关联|

客户端预测进入攻击状态后，如果服务器发现角色已经眩晕，则拒绝该攻击。

客户端需要取消错误预测，并将逻辑状态校正为服务器认可的状态，视觉层根据情况平滑处理。

UE 的 GAS 也提供了一定的技能预测与校正支持，但并不代表任意自定义状态机都能自动获得完整预测能力。

## **14. 如何保证状态机在回滚时的正确性？ ★★★★**

如果项目采用支持回滚的同步架构，状态机必须能够保存和恢复影响后续模拟的运行时数据，而不仅仅是当前状态 ID。

例如：

```
struct FStateSnapshot
{
    EAIState CurrentState;     // 当前状态
    uint32 StateVersion;       // 状态版本
    int32 StateEnterFrame;     // 进入状态的帧号
    int32 AttackPhase;         // 攻击阶段
    int32 ComboIndex;          // 连招段数
};
```

回滚后恢复快照，再使用历史输入重新模拟。

同时要处理伤害、技能消耗、音效、特效等副作用，避免重新模拟造成重复执行。

确定性模拟还需要保证转换顺序、时间推进、随机数和相关逻辑具有一致性。

## **15. 如何调试复杂的状态机？ ★★★★**

实际项目建议记录每次状态转换的信息：

```
Time: 12.35
Actor: Boss_01
OldState: Attack
NewState: Break
Reason: ToughnessZero
Priority: 90
StateVersion: 28
```

除了当前状态，还需要记录转换原因、转换条件、状态持续时间、失败原因和待处理请求。

可以结合 UE 的 Gameplay Debugger、Visual Logger、Unreal Insights，以及自定义状态机调试面板。

对于复杂状态机，能够看到历史转换路径通常比只显示当前状态更有价值。

# **十五、状态机面试重点总结**

如果是 UE 游戏客户端、动作战斗系统方向的面试，建议重点掌握以下知识：

|优先级|知识点|
|---|---|
|S|FSM 核心原理与生命周期|
|S|状态模式的 C++ 实现|
|S|状态爆炸及解决方案|
|S|HFSM 分层状态机|
|S|状态优先级、打断机制|
|S|状态切换重入与异步任务清理|
|A|并行状态机及冲突处理|
|A|输入缓冲、连招、取消窗口|
|A|Tick 驱动与事件驱动|
|A|FSM、行为树、效用 AI 的区别|
|A|逻辑状态机与动画状态机的区别|
|A|GAS 与 Gameplay Tag 的结合|
|A|多人网络同步、预测与校正|
|B|栈式状态机|
|B|数据驱动状态机与 StateTree|

**状态机的核心难点并不是如何实现 ChangeState，而是如何在复杂游戏系统中保证状态转换的合法性、互斥性、优先级、生命周期完整性以及逻辑一致性。**

这也是初级状态机实现与大型商业动作游戏状态机架构之间最重要的区别。