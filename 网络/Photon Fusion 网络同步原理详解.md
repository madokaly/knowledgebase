Photon Fusion 是 Photon 为 Unity 提供的一套高性能多人游戏网络框架，尤其适合动作游戏、射击游戏、多人合作游戏等需要实时同步的场景。

Fusion 最核心的技术是：基于 Tick 的状态同步 + 客户端预测（Client-Side Prediction）+ 回滚重模拟（Rollback & Resimulation）+ 快照插值（Snapshot Interpolation）。

它不是简单地把玩家的位置、旋转发送给其他客户端，而是构建了一套完整的网络模拟系统，让玩家在有网络延迟的情况下，仍然能够获得接近单机游戏的操作体验。

下面以 Photon Fusion 2 的 Host / Dedicated Server 模式为主要讲解对象，因为这种模式更适合研究战斗游戏的网络同步原理。

## 一、Fusion 的核心架构

### 1. Fusion 本质上是什么同步方式？

如果用传统网络游戏的分类来看：

|同步类型|主要思路|
|---|---|
|帧同步（Lockstep）|各客户端交换操作输入，并按一致的逻辑计算游戏状态|
|传统状态同步|服务器计算角色位置、血量等状态，再发送给客户端|
|Fusion 的客户端-服务器模式|服务器权威状态同步 + 客户端输入预测 + 回滚重模拟|

Fusion 的客户端-服务器模式本质上属于服务器权威的状态同步框架。

但它融合了帧同步中的一些思想，例如：

- 固定 Tick 驱动游戏逻辑。
    
- 客户端和服务器可以使用相同的输入执行相同的模拟。
    
- 保存输入历史和网络状态快照。
    
- 发生预测偏差时，回到服务器确认的状态重新模拟。
    

所以，Fusion 虽然具有类似帧同步的输入驱动机制，但它并不是传统意义上的全客户端确定性帧同步。

### 2. Fusion 支持三种主要网络架构

Dedicated Server（专用服务器模式）

专用游戏服务器

所有对象的最终状态权威

客户端 A

客户端 B

适合竞技射击、多人动作战斗，对安全性要求高的游戏。

Host（玩家主机模式）

玩家 A + 服务器

Host 运行权威模拟

客户端 B

客户端 C

适合小规模多人合作，节省专用服务器成本，但 Host 拥有天然优势。

Shared Mode（共享权威模式）

玩家 A

拥有部分对象权威

玩家 B

拥有部分对象权威

玩家 C

拥有部分对象权威

Photon Cloud 房间

适合休闲联机、社交游戏、轻量多人协作。不同对象可以由不同客户端负责计算权威状态。

这里有一个很重要的区别：

Host / Dedicated Server 模式和 Shared Mode 的内部同步机制并不完全相同。

在 Host / Dedicated Server 模式下，Fusion 支持典型的客户端预测、服务器校正和回滚重模拟。

在 Shared Mode 下，则主要由拥有对象状态权威的客户端执行正向模拟，再把结果复制给其他客户端；它不运行上述客户端-服务器模式中的同一套回滚重模拟循环。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

Fusion Update Loop

+1

因此，下面重点分析 Host / Dedicated Server 的完整同步链路。

## 二、Fusion 最重要的两个权限

Fusion 中，每个 `NetworkObject` 都有权限概念，其中最重要的是：

- `StateAuthority`：状态权威。
    
- `InputAuthority`：输入权威。
    

### 1. StateAuthority：谁决定正确的游戏状态？

它负责计算并确定网络对象的最终状态。

比如：

```
using Fusion;

public class PlayerHealth : NetworkBehaviour
{
    // Networked：Fusion 自动管理的网络同步状态
    [Networked]
    public int Health { get; set; }

    public void TakeDamage(int damage)
    {
        // 只有状态权威方能正式修改玩家血量
        if (!HasStateAuthority)
            return;

        Health -= damage;

        // Fusion 会将权威血量同步到其他客户端
        // 不需要开发者手动向每个客户端发送血量
    }
}
```

在 Host / Dedicated Server 模式下，服务器拥有所有网络对象的状态权威。

客户端即使直接修改自己的 `Health`，也不能将该修改变成服务器认可的最终结果，后续会受到权威状态的覆盖。

### 2. InputAuthority：谁负责提供玩家操作？

例如：

玩家 A 控制角色 A，玩家 B 控制角色 B。

|网络对象|StateAuthority|InputAuthority|
|---|---|---|
|角色 A|服务器|玩家 A|
|角色 B|服务器|玩家 B|
|怪物 Boss|服务器|通常没有|
|场景机关|服务器|通常没有|

角色 A 的操作输入来自玩家 A，但角色 A 的最终位置、血量以及攻击是否有效，仍然由服务器确定。

InputAuthority 不等于可以直接决定最终状态。

对于代码而言：

```
if (HasStateAuthority)
{
    // 当前运行端拥有状态权威
    // 可以决定权威状态
}

if (HasInputAuthority)
{
    // 当前运行端拥有该对象的输入权威
    // 通常表示该对象由本地玩家控制
}

if (IsProxy)
{
    // 当前端只是代理对象
    // 主要负责接收网络状态并呈现
}
```

需要注意，`InputAuthority` 主要适用于 Host / Dedicated Server 模式；Shared Mode 中采用不同的权限管理方式。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

NetworkObject - Syncing Unity GameObjects

+1

## 三、Fusion 的 Tick 网络模拟机制

这是理解 Fusion 的基础。

### 1. 什么是 Tick？

Tick（网络逻辑帧）是 Fusion 用于推进游戏模拟的固定时间单位。

例如，设置：

```
Tick Rate = 60

每秒执行 60 次网络模拟

每个 Tick 的时间：

DeltaTime = 1 / 60 ≈ 0.01667 秒
```

在 Unity 中，普通游戏逻辑可能写在：

```
private void Update()
{
    // 根据画面帧率执行
    // 60 FPS：约每秒调用 60 次
    // 144 FPS：约每秒调用 144 次
}
```

Fusion 则使用：

```
public override void FixedUpdateNetwork()
{
    // Fusion 的固定网络逻辑帧回调

    // 当前网络 Tick
    int tick = Runner.Tick;

    // 当前网络 Tick 的模拟时间间隔
    float deltaTime = Runner.DeltaTime;

    // 在这里执行：
    // 角色移动
    // 战斗判定
    // 技能冷却
    // 状态改变
}
```

假设 Tick Rate 为 60：

|Tick|模拟时间|角色 X 位置|
|---|---|---|
|100|1.667s|0|
|101|1.683s|0.1|
|102|1.700s|0.2|
|103|1.717s|0.3|
|104|1.733s|0.4|

Fusion 会以 Tick 为单位管理输入、状态和模拟历史。

但有一个非常重要的区别：

固定 Tick 不代表客户端和服务器必须等待彼此执行相同的 Tick。

客户端可以运行在服务器时间线之前，以提前预测玩家操作。

另外，网络 Tick Rate 和画面 FPS 也不是一回事。即使画面以 144 FPS 渲染，网络模拟仍然可以按照配置的 60 Tick/s 执行。

### 2. Fusion 中的三种关键时间

|时间|主要作用|
|---|---|
|Server Simulation Time（服务器模拟时间）|确定最终游戏结果|
|Client Prediction Time（客户端预测时间）|提前模拟本地玩家，隐藏网络延迟|
|Client Interpolation Time（客户端插值时间）|根据历史快照平滑显示其他玩家|

可以理解为：

```
时间方向 ───────────────────────────────→

客户端显示其他玩家：
Tick 96 → 97 → 98
                   ↑
             插值显示时间

服务器权威模拟：
               Tick 100
                   ↑
              服务器当前时间

客户端本地角色预测：
                         Tick 102 → 103
                                     ↑
                                客户端预测时间
```

这些 Tick 数字只是示意，不是 Fusion 固定的时间差。

为什么这么设计？

因为本地角色需要立即响应，但其他角色可以稍微延迟显示，以获得更平滑的动画和移动效果。

这也意味着：

同一个客户端画面里，本地玩家与远程玩家可能处在不同的模拟时间线上。

这正是后面需要延迟补偿（Lag Compensation）的原因。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

Time Synchronization

+1

## 四、Fusion 完整网络同步流程

假设玩家 A 控制角色向前移动。

网络延迟 RTT（往返时间）为 100ms，服务器运行 60 Tick/s。

Fusion Host / Server 模式的核心数据流（示意图）

这个过程可以分解为五步：

1. 客户端在 `OnInput()` 中采集输入，并保存输入历史。
    
2. 客户端不等待网络返回，立即使用输入进行本地预测。
    
3. 服务器接收输入，在对应的网络模拟中计算权威状态。
    
4. 服务器把状态快照发送给客户端，客户端回滚到已收到的权威状态，再重新模拟后续 Tick。
    
5. 渲染阶段显示本地预测结果，并对远程对象使用历史快照插值。
    

Photon 官方文档也明确将这套流程称为 Client-Side Prediction、Rollback 和 Re-Simulation。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

Player Input - OnInput & INetworkInput

## 五、客户端预测（Client-Side Prediction）

客户端预测主要解决的是：为什么一个网络延迟 100ms 的玩家，按下移动键时不会感觉人物过了 100ms 才开始移动？

### 1. 没有预测会怎样？

假设用户按下 W 键，RTT 为 100ms：

```
0ms
玩家按下 W 键
    ↓
50ms
服务器收到移动输入
    ↓
服务器计算新位置
    ↓
100ms
客户端收到服务器位置
    ↓
角色才开始移动
```

这种设计会让角色操作产生明显延迟。

### 2. 有客户端预测会怎样？

```
0ms
玩家按下 W
    │
    ├── 客户端立即模拟移动
    │       ↓
    │   角色马上前进
    │
    └── 同时把输入发送给服务器
            ↓
50ms
服务器收到输入并产生权威状态
            ↓
100ms
客户端收到权威状态
            ↓
回滚重模拟，修正预测偏差
```

客户端预测的核心思想是：

我先假设自己这次移动会成功，立即表现出来；服务器之后告诉我真正的结果。

### 3. 预测使用什么数据？

假设一位玩家每秒移动 6 米，Tick Rate 为 60。

每个 Tick 的理想移动距离：

\[ \Delta x=v\cdot\Delta t \]

\[ \Delta x=6\times\frac{1}{60}=0.1m \]

那么：

```
Tick 100：
玩家位置 X = 0

Tick 101：
输入 W
客户端预测 X = 0.1

Tick 102：
继续输入 W
客户端预测 X = 0.2

Tick 103：
继续输入 W
客户端预测 X = 0.3
```

实际移动通常还会受到碰撞、角色状态、加速度、技能效果等影响。

但关键点在于，客户端和服务器都执行相同的模拟逻辑：

\[ S_{t+1}=F(S_t,I_t,\Delta t) \]

其中：

- \(S_t\)：当前网络状态（State）。
    
- \(I_t\)：玩家在 Tick \(t\) 的输入（Input）。
    
- \(F\)：游戏模拟函数。
    
- \(\Delta t\)：固定 Tick 时间。
    
- \(S_{t+1}\)：模拟结束后的新状态。
    

例如：

```
// 概念性示例：网络模拟函数
Vector3 SimulateMovement(
    Vector3 currentPosition,
    Vector3 inputDirection,
    float moveSpeed,
    float deltaTime)
{
    // 输入 + 上一个状态 → 新状态

    return currentPosition
           + inputDirection * moveSpeed * deltaTime;
}
```

这也是为什么 Fusion 强调把核心游戏逻辑放在 `FixedUpdateNetwork()` 中。

## 六、回滚与重模拟（Rollback & Resimulation）

这是 Fusion 网络同步机制中技术含量最高的部分之一。

### 1. 为什么已经预测了，还需要回滚？

假设客户端认为：

```
Tick 100 → 位置 0
Tick 101 → 位置 1
Tick 102 → 位置 2
Tick 103 → 位置 3
Tick 104 → 位置 4
Tick 105 → 位置 5
```

但服务器检测到角色受阻，因此：

```
服务器 Tick 102：

客户端预测位置 = 2
服务器权威位置 = 1.5
```

客户端此时已经预测到了 Tick 105。

如果直接把当前位置从 5 改成 1.5，人物就会突然瞬移回去，随后继续向前移动。

因此，正确的方法是：

### 2. 回滚到权威状态，再重新计算

第一阶段：本地预测

T100

0

T101

1

T102

2

T103

3

T104

4

T105

5

收到服务器 Tick 102 的权威位置：1.5

第二阶段：回滚到服务器确认的 Tick

T100

—

T101

—

T102

1.5

T103

—

T104

—

T105

—

重新执行 Tick 103、104、105 的历史输入

第三阶段：重新模拟后的状态

T100

—

T101

—

T102

1.5

T103

2.5

T104

3.5

T105

4.5

这里假设每个后续 Tick 都能够正常前进 1 个位置单位，实际结果取决于碰撞和模拟逻辑。

重模拟后的结果：

```
服务器确认 Tick 102：X = 1.5

重新模拟 Tick 103：X = 2.5
重新模拟 Tick 104：X = 3.5
重新模拟 Tick 105：X = 4.5

原预测结果：X = 5.0
新预测结果：X = 4.5
```

最终客户端不是直接退回到 Tick 102，而是回到那个状态重新演算，最终获得 Tick 105 的新预测结果。

### 3. Fusion 内部的重模拟流程

可以抽象为：

```
// 注意：这是用于解释 Fusion 原理的伪代码
// 并不是 Fusion 对外提供的实际 API

void OnReceiveServerSnapshot(
    int serverTick,
    NetworkState authoritativeState)
{
    // 第一步：获取客户端当前预测到的 Tick
    int targetTick = currentPredictedTick;

    // 第二步：恢复服务器确认的权威状态
    RestoreState(authoritativeState);

    // 第三步：从服务器确认 Tick 的下一个 Tick 开始
    for (int tick = serverTick + 1;
         tick <= targetTick;
         tick++)
    {
        // 找回该 Tick 保存的历史输入
        PlayerInput input = inputHistory[tick];

        // 使用历史输入重新执行游戏逻辑
        SimulateTick(input);

        // 重新生成该 Tick 的预测状态
    }

    // 第四步：渲染最新预测状态
}
```

开发者通常不需要自己编写这个回滚循环，Fusion 的网络模拟系统会负责管理。

但是，开发者需要保证自己的模拟逻辑能够被安全地重复执行。

### 4. 重模拟最容易出现什么 Bug？

最典型的问题就是：在 `FixedUpdateNetwork()` 里直接触发不可逆的副作用。

例如：

```
public override void FixedUpdateNetwork()
{
    if (ShouldAttack())
    {
        // 错误示例：
        // 如果这个 Tick 被重复模拟，
        // 这里可能重复播放声音

        audioSource.Play();

        // 这里也可能重复创建特效
        Instantiate(hitEffectPrefab);
    }
}
```

因为 Fusion 可能在一个渲染帧内多次调用 `FixedUpdateNetwork()`，其中部分调用是在重新模拟历史 Tick。

因此通常要将：

- 游戏状态计算放在 `FixedUpdateNetwork()`。
    
- 表现层逻辑放在 `Render()` 或专门的表现系统。
    
- 需要保证只执行一次的外部操作放在受控的、非重模拟执行路径。
    

还需要利用 Tick、事件序号或预测事件去重，防止击中特效和攻击音效在校正过程中重复出现。

这对技能释放、受击反馈、弹道生成尤其重要。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

Networking Controller Code in Unity

+1

## 七、快照插值（Snapshot Interpolation）

解决了本地玩家操作延迟，接下来还有另一个问题：

其他玩家在网络上每秒只更新几十次，为什么我们看到的人物移动仍然很平滑？

答案就是快照插值。

### 1. 直接显示网络位置会发生什么？

假设服务器每隔一段时间发送一次角色位置：

```
Tick 100：X = 0
Tick 101：X = 1
Tick 102：X = 2
Tick 103：X = 3
```

客户端每次收到数据才更新位置，就可能出现：

```
X = 0 → 停顿 → X = 1 → 停顿 → X = 2
```

形成明显的跳跃感。

### 2. Fusion 使用历史快照进行插值

假设客户端持有：

```
Snapshot A：
Tick 100
Position = (0, 0, 0)

Snapshot B：
Tick 101
Position = (1, 0, 0)
```

当当前渲染时间处于两个快照之间时：

\[ P_{render}=\operatorname{Lerp}(P_A,P_B,\alpha) \]

其中：

\[ 0\leq\alpha\leq1 \]

快照插值演示

线性插值

α = 0.50

位置 = 0.50

拖动滑块，可以观察 Fusion 使用两个网络快照计算中间渲染位置的基本原理。

Unity 中类似的计算：

```
Vector3 InterpolatePosition(
    Vector3 from,
    Vector3 to,
    float alpha)
{
    // from：上一份网络快照
    // to：下一份网络快照
    // alpha：插值系数，取值为 0~1

    return Vector3.Lerp(from, to, alpha);
}
```

旋转通常使用四元数插值：

```
Quaternion result = Quaternion.Slerp(
    rotationA,
    rotationB,
    alpha
);
```

Fusion 的 `NetworkTransform` 已经集成了相关状态同步与插值能力，正常使用时无需手动实现这些基础运算。

### 3. 为什么插值需要延迟？

因为网络包的到达时间并不稳定。

例如：

```
服务器发送：

Tick 100
Tick 101
Tick 102
Tick 103
Tick 104

客户端接收：

Tick 100
Tick 101
         网络抖动
Tick 102、103 一起到达
Tick 104
```

如果客户端始终显示最新收到的 Tick，就可能因为缺少下一个快照而卡顿。

所以 Fusion 会维护一定的插值缓冲，让远程角色的显示时间相对于服务器权威时间稍微滞后。

这样它更有机会持续获得两份可以插值的快照。

这是一种典型的取舍：

增加少量显示延迟，换取更稳定的远程角色运动。

Fusion 2.1 还提供了专门的时间同步配置，允许调整输入接收缓冲、预测负载和状态插值缓冲。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

Time Synchronization

+1

## 八、Fusion 如何在 Unity 中实现同步？

下面通过一个角色移动、攻击和血量同步的示例，把前面的原理与实际代码对应起来。

### 1. 定义网络输入数据

首先，定义玩家需要上传的输入：

```
using Fusion;
using UnityEngine;

// 玩家按钮类型
public enum PlayerButton
{
    Attack = 0,   // 普通攻击
    Jump = 1,     // 跳跃
    Dodge = 2     // 闪避
}

// 网络输入数据
// 客户端和服务器会使用相同结构
public struct PlayerInputData : INetworkInput
{
    // 移动方向
    // x：左右
    // y：前后
    public Vector2 MoveDirection;

    // 按钮状态
    // Fusion 会使用位标记管理多个按钮
    public NetworkButtons Buttons;
}
```

这里需要注意：

`PlayerInputData` 不保存玩家的最终位置，而是保存玩家的操作意图。

例如：

```
玩家按下 W：

MoveDirection = (0, 1)

服务器收到的不是：
“我的位置已经变成 (10, 0, 5)”

而是：
“我希望往前移动”
```

服务器会根据速度、碰撞、角色状态等信息，计算角色真正应该到达的位置。

### 2. 采集客户端输入

下面使用 Fusion 2 的 `NetworkEvents` 接收 `OnInput` 回调。

```
using Fusion;
using UnityEngine;

public class FusionInputCollector : MonoBehaviour
{
    // 攻击按钮的临时缓存
    private bool _attackPressed;

    private void Update()
    {
        // Update 用于采集 Unity 输入
        //
        // 因为玩家按键的时间可能非常短，
        // 直接在 OnInput 里使用 GetKeyDown，
        // 可能会错过某次按键。

        if (Input.GetKeyDown(KeyCode.J))
        {
            // 将按键缓存，等待下一个网络输入 Tick
            _attackPressed = true;
        }
    }

    // 该方法绑定到 NetworkEvents.OnInput
    public void OnInput(NetworkRunner runner, NetworkInput input)
    {
        PlayerInputData data = new PlayerInputData();

        // 获取 WASD 移动输入
        float horizontal = Input.GetAxisRaw("Horizontal");
        float vertical = Input.GetAxisRaw("Vertical");

        data.MoveDirection =
            new Vector2(horizontal, vertical).normalized;

        // 将缓存的攻击操作写入本次网络输入
        data.Buttons.Set(
            PlayerButton.Attack,
            _attackPressed
        );

        // 把输入交给 Fusion
        //
        // Fusion 负责：
        // 1. 记录输入历史
        // 2. 在客户端预测时提供输入
        // 3. 将输入传输到服务器
        // 4. 在回滚重模拟时重新提供历史输入
        input.Set(data);

        // 本次操作已提交，清除缓存
        _attackPressed = false;
    }
}
```

使用时，将脚本挂到本地输入对象上，并把 `OnInput` 方法绑定到 `NetworkRunner` 所使用的 `NetworkEvents.OnInput` 事件。

这里使用的是 Unity Legacy Input Manager。如果项目使用新版 Input System，只需要更换输入采集部分。

### 3. 使用同一套代码进行预测和权威模拟

下面是最关键的角色控制逻辑。

```
using Fusion;
using UnityEngine;

public class FusionPlayerController : NetworkBehaviour
{
    [SerializeField]
    private float moveSpeed = 6.0f;

    [SerializeField]
    private NetworkCharacterController controller;

    // 玩家血量
    [Networked]
    public int Health { get; set; }

    // 记录上一个 Tick 的按钮状态
    //
    // 必须参与网络模拟状态，
    // 这样回滚后才能恢复正确的历史按钮状态。
    [Networked]
    private NetworkButtons PreviousButtons { get; set; }

    // 网络攻击冷却计时器
    [Networked]
    private TickTimer AttackCooldown { get; set; }

    public override void Spawned()
    {
        // 服务器初始化玩家血量
        if (HasStateAuthority)
        {
            Health = 100;
        }
    }

    public override void FixedUpdateNetwork()
    {
        // 获取当前 Tick 对应的玩家输入
        //
        // 本地输入权威：
        // 使用它进行客户端预测
        //
        // 服务器状态权威：
        // 使用它进行权威模拟
        //
        // 客户端回滚：
        // 使用保存的历史输入重新模拟
        if (!GetInput(out PlayerInputData input))
        {
            return;
        }

        // ---------- 角色移动 ----------

        Vector3 direction = new Vector3(
            input.MoveDirection.x,
            0,
            input.MoveDirection.y
        );

        // 服务器仍需要验证输入是否合法
        // 防止客户端构造超出正常范围的移动输入
        direction = Vector3.ClampMagnitude(direction, 1f);

        // 使用 Fusion 的时间步长
        Vector3 movement =
            direction * moveSpeed * Runner.DeltaTime;

        // 在网络模拟中移动角色
        //
        // 需要角色预制体配置
        // NetworkObject 和 NetworkCharacterController
        controller.Move(movement);

        // ---------- 攻击输入 ----------

        // 比较当前 Tick 与上一 Tick 的按钮状态
        bool attackPressed =
            input.Buttons.WasPressed(
                PreviousButtons,
                PlayerButton.Attack
            );

        // 保存本次按钮状态
        PreviousButtons = input.Buttons;

        if (attackPressed &&
            AttackCooldown.ExpiredOrNotRunning(Runner))
        {
            // 用网络 Tick 记录攻击冷却
            AttackCooldown =
                TickTimer.CreateFromSeconds(Runner, 0.5f);

            // 只有服务器真正执行攻击判定
            if (HasStateAuthority)
            {
                ProcessAttack();
            }
        }
    }

    private void ProcessAttack()
    {
        // 这里属于服务器权威逻辑：
        //
        // 1. 检测攻击是否命中
        // 2. 检查目标是否合法
        // 3. 计算真实伤害
        // 4. 修改目标的 Networked 血量

        Debug.Log("服务器处理攻击判定");
    }
}
```

这个示例省略了视角旋转、动画和具体碰撞检测，主要展示网络同步结构。

其中最重要的一点：

`FixedUpdateNetwork()` 并不是只在服务器执行。

同一个角色控制代码可以执行于：

|运行位置|主要作用|
|---|---|
|本地客户端|预测角色移动|
|服务器|计算权威位置和攻击结果|
|本地客户端重模拟|使用权威状态重新预测后续移动|
|其他客户端的代理对象|通常不通过 `GetInput()` 获取该玩家输入，而是依据接收的状态进行表现|

也就是说，Fusion 通过统一模拟代码、输入历史和状态回滚，减少开发者分别编写客户端与服务器移动逻辑的成本。

官方 Host Mode 教程也使用 `NetworkCharacterController.Move()` 配合 `GetInput()` 实现这种移动模型。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

Host Mode Tutorial — Prediction

+1

## 九、Networked 属性到底是怎么同步的？

前面使用了：

```
[Networked]
public int Health { get; set; }
```

那么 Fusion 如何把普通 C# 属性变成网络状态？

### 1. 编译阶段：IL Weaving

Fusion 使用 IL Weaving（中间语言织入）处理网络属性。

简单理解，普通 C# 自动属性原本相当于：

```
public int Health
{
    get;
    set;
}
```

但加入 `[Networked]` 后，Fusion 会在编译处理阶段生成访问网络状态内存的相关代码。

概念上可以理解为：

```
// 伪代码，用于解释内部访问思想

public int Health
{
    get
    {
        // 从 NetworkObject 对应的状态内存读取
        return ReadNetworkState<int>(healthOffset);
    }

    set
    {
        // 将值写入网络状态内存
        WriteNetworkState(healthOffset, value);
    }
}
```

这不是 Fusion 真实生成代码的逐行还原，只是为了说明原理。

关键点在于：

`[Networked]` 属性不再只是一般的 C# 变量，而是与 Fusion 管理的对象网络状态内存关联。

### 2. Tick 结束后生成 Snapshot

每个 Tick 模拟完成后，Fusion 会捕获相应的网络对象状态。

一个角色的快照可以抽象为：

```
Snapshot：Tick 100

NetworkObjectId = 15

Position = (1.0, 0.0, 2.0)
Rotation = (0, 0, 0, 1)

Health = 80
IsDead = false
AttackCooldown = ...
```

这里位置、旋转通常由 `NetworkTransform` 对应的网络状态管理，血量等自定义属性由 `NetworkBehaviour` 中的 `[Networked]` 管理。

服务器把需要复制的对象状态发送给相关客户端。

### 3. 是否每次都发送所有数据？

不是。

Fusion 会利用状态变化检测、数据编码和复制调度等机制减少网络流量。

例如：

```
Tick 100：
Health = 100
Position = (0, 0, 0)

Tick 101：
Health = 100
Position = (1, 0, 0)
```

血量没有变化，就没有必要把它当作每次都必须完整传输的新值。

输入数据也有优化机制，例如 Fusion 的 `Redundancy` 输入传输模式会采用差异压缩和冗余输入策略，以提高带宽利用率和抗丢包能力。

但需要特别区分：Fusion 2 已移除了旧版的 Delta Snapshots 状态传输模式，不要把 Fusion 1 的所有快照压缩细节原样套用到 Fusion 2。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

Networked Properties & State Sync

+2

### 4. 一个容易误解的问题

假设服务器执行：

```
Health = 100;
Health = 90;
Health = 80;
```

客户端不一定会看到：

```
100 → 90 → 80
```

可能只看到：

```
100 → 80
```

这是因为 `[Networked]` 表达的是需要复制的状态，不是每一次赋值所产生的事件。

Fusion 的网络属性同步遵循最终一致性（Eventual Consistency）机制，中间变化可能被合并。

如果需要记录“受到三次攻击”这种过程，就不能只依赖血量变化次数来推断。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

Reliability Exceptions for RPCs and State

## 十、RPC 与状态同步有什么区别？

Fusion 提供两种经常使用的数据同步方式：

|对比|`[Networked]`|`[Rpc]`|
|---|---|---|
|主要用途|持续存在的游戏状态|一次性网络消息|
|典型例子|血量、状态、位置|聊天、请求操作、临时通知|
|新玩家加入|可收到当前状态|不会自动重放历史 RPC|
|同步特点|关注当前权威状态|关注方法调用|
|是否适合持续的位置同步|是，通常配合 NetworkTransform|不适合|

例如，玩家请求修改角色名称：

```
using Fusion;

public class NetworkPlayerInfo : NetworkBehaviour
{
    [Networked]
    public NetworkString<_32> PlayerName { get; set; }

    // 输入权威客户端向服务器发送请求
    [Rpc(RpcSources.InputAuthority, RpcTargets.StateAuthority)]
    public void RPC_SetPlayerName(NetworkString<_32> newName)
    {
        // 真实项目中，服务器还应验证名称的合法性
        PlayerName = newName;
    }
}
```

运行流程：

```
客户端 A
    │
    │ RPC_SetPlayerName("PlayerA")
    ↓
服务器
    │
    │ 验证名称
    │ 修改 Networked PlayerName
    ↓
Fusion 状态复制
    │
    ├── 客户端 A 收到名称
    ├── 客户端 B 收到名称
    └── 后来加入的客户端 C 也能获得当前名称
```

因此，一个重要的设计原则是：

RPC 适合发起请求，Networked 属性适合保存结果。

RPC 默认走可靠通道，但可靠性并不意味着无条件保证，例如断线或对象已被销毁时存在例外。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

Remote Procedure Calls in Unity

## 十一、射击与动作游戏如何解决命中判定问题？

如果只是实现网络移动，客户端预测与快照插值已经非常重要。

但对于射击游戏，还有一个更复杂的问题。

假设玩家 A 正在瞄准玩家 B。

玩家 A 看到的玩家 B，是经过插值显示的较早状态。

而服务器在当前 Tick 中看到的玩家 B，可能已经移动到了别的位置。

玩家 A 瞄准的是自己屏幕上的目标位置，而服务器当前状态中的目标已经移动。

如果服务器仅使用当前物理场景：

```
Physics.Raycast(
    origin,
    direction,
    out RaycastHit hit
);
```

就可能出现玩家明明瞄准了敌人，却没有命中的情况。

### 1. Fusion 的 Lag Compensation（延迟补偿）

Fusion 的 Host / Server 模式提供了内置的延迟补偿系统。

其工作原理是：

1. 服务器保存历史 Hitbox（命中盒）位置。
    
2. 玩家提交射击输入。
    
3. 服务器根据射击玩家的网络时间信息，查询对应的历史目标状态。
    
4. 使用历史 Hitbox 执行射线或范围检测。
    
5. 由服务器决定命中结果。
    

Fusion 甚至支持 Sub-tick Accuracy（子 Tick 精度），在历史快照之间进行进一步的插值判定。

### 2. 简化代码

```
using Fusion;
using UnityEngine;

public class FusionWeapon : NetworkBehaviour
{
    [SerializeField]
    private LayerMask hitMask;

    [SerializeField]
    private float attackRange = 100f;

    // 示例：由服务器模拟流程调用
    public void Fire(Vector3 origin, Vector3 direction)
    {
        // 最终命中检测由服务器负责
        if (!HasStateAuthority)
            return;

        // direction 应来自经过验证的瞄准数据
        direction = direction.normalized;

        // 使用 Fusion 的历史命中盒进行检测
        bool hasHit = Runner.LagCompensation.Raycast(
            origin,
            direction,
            attackRange,
            Object.InputAuthority,
            out LagCompensatedHit hit,
            hitMask,
            HitOptions.IgnoreInputAuthority |
            HitOptions.SubtickAccuracy
        );

        if (!hasHit)
            return;

        // 获取命中的 Hitbox
        Hitbox targetHitbox = hit.Hitbox;

        if (targetHitbox == null)
            return;

        // 查找目标网络角色
        var target = targetHitbox
            .GetComponentInParent<FusionPlayerController>();

        if (target == null)
            return;

        // 这里继续执行服务器权威伤害逻辑
        // 例如验证目标、攻击范围、队伍关系后扣血

        Debug.Log("攻击命中目标：" + target.name);
    }
}
```

这个例子还需要在角色上配置 `HitboxRoot` 和 `Hitbox`，并在网络配置中启用 Lag Compensation。正式项目中还需要验证射击起点、方向、射速和弹药等数据，避免客户端构造非法命中请求。

这套机制不是简单地相信客户端说“我打中了”，而是让服务器依据历史信息自己验证。

对于动作游戏，也可以使用历史碰撞信息处理部分近战攻击；但高速连续挥动的武器、持续存在的攻击盒和复杂的多目标攻击，需要额外设计采样与判定规则。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

Lag Compensation & Hitbox Queries

## 十二、Fusion 的网络同步性能如何优化？

当场景中有几十个甚至上百个网络对象时，不能把所有对象的全部状态无差别地发送给所有玩家。

Fusion 主要提供以下几个方向的优化。

### 1. AOI：Area of Interest（兴趣区域）

例如一张大型游戏地图：

```
玩家 A 位于地图左侧
玩家 B 位于地图右侧
```

如果双方距离很远，玩家 A 不需要持续接收玩家 B 附近所有小怪和可破坏物件的状态。

因此可以设置兴趣区域，只向某个玩家复制与其相关的对象状态。

需要注意：AOI 是网络复制过滤，不代表服务器不再模拟 AOI 之外的对象。

### 2. 优先级与复制调度

例如：

|网络对象|相对优先级|
|---|---|
|当前交战的敌方玩家|高|
|附近 Boss|高|
|附近移动机关|中|
|远处装饰性网络对象|低|

当网络带宽不足时，可以根据对象优先级和复制调度策略安排更新。

这不是说低优先级对象永远不更新，而是让有限带宽优先服务重要数据。

### 3. 减少不必要的 Networked 属性

例如没有必要把下列内容全部按网络状态传输：

```
// 一般不推荐把纯本地表现数据持续同步
// CameraShakeAmount
// LocalUIAlpha
// ParticleLifetime
// TemporaryScreenEffect
```

通常应重点同步会影响游戏结果或需要远端还原的数据，例如：

```
[Networked]
public int Health { get; set; }

[Networked]
public int CurrentSkillId { get; set; }

[Networked]
public NetworkBool IsStunned { get; set; }

[Networked]
public TickTimer SkillCooldown { get; set; }
```

特效、摄像机震动和音效则尽量由本地表现系统生成。

### 4. 不要滥用网络实体

例如连续射击的子弹：

如果每秒产生数百发子弹，全部创建独立 `NetworkObject`，会增加对象管理、生成销毁和状态同步成本。

对于不需要长期存在的视觉弹道，可以考虑：

- 服务器权威验证命中。
    
- 客户端本地预测弹道表现。
    
- 使用必要的网络数据还原弹道。
    
- 对持续存在且具有复杂交互的弹丸，才根据需求选择网络对象同步。
    

Photon 也提供了专门的 Projectiles 技术示例，可以进一步研究弹丸同步设计。

## 十三、Fusion 开发最容易踩的坑

|常见问题|原因|正确处理|
|---|---|---|
|角色移动不一致|在普通 `Update()` 中运行核心移动模拟|使用 `FixedUpdateNetwork()`|
|攻击特效重复播放|历史 Tick 重模拟触发了副作用|将模拟与表现分离，进行事件去重|
|预测结果经常被校正|客户端和服务器运动逻辑不同|尽量统一模拟输入和规则|
|攻击按键偶尔失效|短按键没有被 `OnInput` 采集|在 Update 中缓存短暂按键|
|客户端修改血量无效|没有状态权威|在服务器修改权威血量|
|远程角色抖动|插值设置、输入和物理同步处理不合理|检查 NetworkTransform 与网络时间配置|
|物理模拟回滚不正确|普通 Rigidbody 状态没有正确纳入预测恢复|使用合适的 Fusion 物理同步方案|
|玩家后加入看不到状态|只使用了历史 RPC|把持续状态保存在 Networked 属性|

其中一个高级问题值得特别强调：

Fusion 支持回滚重模拟，不代表所有 Unity 物理逻辑天然都能正确回滚。

例如普通 Rigidbody 的速度、接触状态、外力以及自定义缓存，如果没有正确纳入可恢复的模拟过程，就可能导致客户端重新执行同样输入时得到不同结果。

Fusion 2 的 Physics Forecast、Forecast Disabled 和 Physics Addon 提供不同的物理同步方案。涉及复杂可预测刚体交互时，应根据项目实际需求选择。

![](https://www.google.com/s2/favicons?domain=https://doc.photonengine.com&sz=32)

doc.photonengine.com

## 十四、Fusion 与其他网络同步方案的区别

|对比维度|传统状态同步|传统帧同步|Fusion Host / Server|
|---|---|---|---|
|权威状态|服务器|通常依赖确定性模拟及仲裁设计|服务器|
|玩家输入上传|可选|核心机制|核心机制|
|客户端预测|需要自行实现|依实现而定|框架内置|
|回滚重模拟|需要自行实现|依实现而定|框架内置|
|快照插值|需要自行实现|不一定需要|框架内置|
|严格跨平台确定性|不要求|通常非常重要|不要求全局严格确定性|
|服务器状态复制|核心机制|不一定需要|核心机制|
|典型适用类型|RPG、多人在线游戏|高确定性竞技战斗|实时多人动作、射击与合作|

需要注意，传统状态同步和传统帧同步都可以根据需求额外实现预测、插值等技术，所以表格比较的是典型架构，而不是绝对技术限制。

## 十五、如果开发类似《鸣潮》的多人合作战斗，应该怎么使用 Fusion？

假设游戏具有玩家角色、Boss、技能、闪避、受击和伤害等系统。

我更倾向于采用以下分层设计：

|系统|推荐同步设计|
|---|---|
|玩家移动|客户端预测 + 服务器权威位置|
|玩家闪避|本地即时反馈 + 服务器验证无敌帧|
|玩家攻击|本地预测动画 + 服务器权威判定|
|Boss AI|服务器权威执行行为树或状态机|
|Boss 位置|服务器模拟 + 客户端插值|
|Boss 技能状态|Networked 状态 + Tick 时间管理|
|玩家血量|服务器权威状态同步|
|技能命中|服务器判定，必要时使用延迟补偿|
|战斗特效|本地表现 + 必要的网络事件|
|战斗伤害数字|依据服务器认可的伤害事件生成|

注意，Boss AI 并不应该让每个客户端独立运行一份完整的权威决策逻辑。

更合理的方式是：

```
服务器：
    Boss 感知
        ↓
    行为树 / 状态机决策
        ↓
    选择攻击目标
        ↓
    计算技能释放
        ↓
    执行移动、碰撞、伤害判定
        ↓
    生成 Boss 权威状态
        ↓
    Fusion 同步相关状态

客户端：
    接收 Boss 状态
        ↓
    快照插值
        ↓
    驱动动画表现
        ↓
    播放技能特效和声音
```

而玩家自身通常采用：

```
客户端输入
    ↓
本地移动 / 闪避 / 攻击预测
    ↓
立即播放操作反馈
    ↓
服务器权威计算
    ↓
状态校正
    ↓
客户端调整表现
```

这里真正困难的不是把角色位置同步过去，而是如何让攻击、闪避、受击、硬直、技能冷却和 Boss 行为在预测及校正之后仍然保持一致。

## 十六、总结：Fusion 最值得掌握的 8 个核心技术

|优先级|核心技术|必须理解的问题|
|---|---|---|
|1|Tick-Based Simulation（固定 Tick 模拟）|为什么游戏逻辑必须基于网络 Tick|
|2|State / Input Authority（状态与输入权威）|谁负责输入，谁决定最终结果|
|3|Client-Side Prediction（客户端预测）|为什么本地操作不必等待服务器|
|4|Rollback & Resimulation（回滚重模拟）|如何利用权威状态修正历史预测|
|5|Snapshot Interpolation（快照插值）|为什么其他玩家移动可以保持平滑|
|6|Networked State（网络状态同步）|网络属性如何捕获、复制和更新|
|7|Lag Compensation（延迟补偿）|为什么能打中自己屏幕上看到的敌人|
|8|Interest Management（兴趣管理）|如何控制多人场景中的带宽消耗|

其中，最值得深入研究的是客户端预测、回滚重模拟和快照插值三者之间的关系。

它们分别解决了三个不同的问题：

- 客户端预测：解决本地操作延迟。
    
- 回滚重模拟：解决预测状态与权威状态不一致。
    
- 快照插值：解决远程对象显示不平滑。
    

这三种机制共同构成了 Fusion 实时网络同步体验的核心。

进一步阅读可参考 Photon 官方的 [Fusion Update Loop](https://doc.photonengine.com/fusion/v2/concepts-and-patterns/fusion-update-loop)、[Player Input](https://doc.photonengine.com/fusion/v2/manual/input/player-input) 和 [Lag Compensation](https://doc.photonengine.com/fusion/v2/manual/advanced/lag-compensation)。

如果继续往底层深入，下一阶段最有价值的是研究 Fusion 如何管理 Snapshot Buffer（快照缓冲区）、Input History（输入历史）、Resimulation Loop（重模拟循环）以及网络 Tick 的时间同步。这几个模块可以进一步串联成一套类似 Fusion 的简化版 Unity 网络同步框架。