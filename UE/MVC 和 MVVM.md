MVC 和 MVVM 都是常见的软件架构模式，主要用于分离业务逻辑、数据和 UI 显示，从而降低代码耦合度，提高项目的可维护性。

在 Unity、UE5 等游戏开发中，这两种架构经常用于背包系统、角色属性面板、商城、任务系统等 UI 模块。

两者最大的区别：

- MVC： 以 Controller（控制器）为核心，负责处理用户操作，并协调 Model 和 View。
    
- MVVM： 以 ViewModel（视图模型）为核心，通过数据绑定和变更通知，让 UI 随数据变化自动更新。
    

## 一、MVC 架构详解

MVC 全称 Model-View-Controller（模型-视图-控制器）。

它把系统分成三个部分：

|组件|中文|主要职责|
|---|---|---|
|Model|模型|保存数据、处理业务规则|
|View|视图|显示 UI，接收用户输入|
|Controller|控制器|处理输入，协调 Model 与 View|

### 1. MVC 的工作流程

以游戏中的角色血条为例：

<img width="736" height="432" alt="image" src="https://github.com/user-attachments/assets/50d62717-6917-4d1a-bf5c-eeeb576c6b46" />


这里要注意：MVC 并不强制 Controller 直接调用 View 刷新。 在经典 MVC 中，View 可以观察 Model 的变化；在某些实际项目中，也会由 Controller 主动调用 View 的刷新方法。

### 2. MVC 的优缺点

优点：

- 职责划分比较清晰，容易理解。
    
- 处理流程明确，方便追踪用户操作。
    
- 简单 UI 的实现成本较低。
    
- 不需要专门的数据绑定框架。
    

缺点：

- 复杂 UI 中 Controller 容易承担过多职责，形成庞大的控制器。
    
- UI 刷新逻辑可能需要手动编写。
    
- 如果 Controller 直接依赖大量 UI 控件，耦合度会提高。
    
- 多个界面展示相同数据时，可能出现重复的刷新代码。
    

## 二、MVVM 架构详解

MVVM 全称 Model-View-ViewModel（模型-视图-视图模型）。

它也把系统分为三个部分：

|组件|中文|主要职责|
|---|---|---|
|Model|模型|保存业务数据，负责业务逻辑|
|View|视图|负责 UI 的显示与交互|
|ViewModel|视图模型|将业务数据转换为 UI 需要的数据，暴露属性和命令，通知 UI 更新|

MVVM 最重要的设计思想是：ViewModel 不应该直接依赖具体的 UI 控件。

例如，ViewModel 不应该关心血条控件是不是 `ProgressBar`，也不应该直接调用 `ProgressBar.SetPercent()`。

它只需要提供一个属性：

```
float HealthPercent = 0.8f;
```

View 绑定这个属性即可。

### 1. MVVM 的工作流程

<img width="736" height="326" alt="image" src="https://github.com/user-attachments/assets/cb023fbc-67eb-4d3e-8d24-3f369dfc3b27" />


假设角色受到攻击：

1. Model 中的血量从 100 减少到 60。
    
2. ViewModel 接收到 Model 的数据变化。
    
3. ViewModel 更新 `HealthPercent = 0.6f`。
    
4. ViewModel 发出属性变化通知。
    
5. View 通过绑定机制，将血条显示更新为 60%。
    

整个过程中，ViewModel 并不需要知道血条控件的存在。

这就是 MVVM 的核心优势：UI 显示与数据处理解耦。

### 2. MVVM 的数据绑定

MVVM 常见的绑定方式包括：

|绑定方式|数据流向|适用场景|
|---|---|---|
|One Time|ViewModel → View（一次）|初始化后不变的 UI|
|One Way|ViewModel → View|血条、金币、等级|
|Two Way|ViewModel ↔ View|音量设置、用户输入|

例如：

- 玩家血量变化，血条自动更新：单向绑定。
    
- 玩家拖动音量滑块，ViewModel 更新设置；外部改变音量设置，滑块也同步变化：双向绑定。
    

需要强调，MVVM 并不要求所有属性都使用双向绑定。在游戏 HUD 中，单向绑定通常更常见。

## 三、MVC 和 MVVM 的核心区别

|对比项|MVC|MVVM|
|---|---|---|
|核心组件|Controller|ViewModel|
|UI 更新|手动调用或事件通知|属性绑定与变更通知|
|数据和 UI 的关系|View 可观察 Model|View 通常绑定 ViewModel|
|用户输入|Controller 处理|ViewModel 的命令或方法处理|
|UI 耦合度|取决于实现方式|通常更低|
|自动数据同步|需要自行实现|通常由绑定框架支持|
|代码量|简单场景较少|需要额外的 ViewModel 和绑定机制|
|单元测试|可以测试 Controller|ViewModel 更容易独立测试|
|典型场景|简单 UI、交互流程|复杂 UI、多界面数据同步|
|性能|取决于具体实现|取决于绑定与通知机制|

注意，MVVM 并不一定比 MVC 性能更好。MVVM 的主要优势在于解耦和维护性，而不是绝对的运行效率。

## 四、通过代码理解两者的区别

下面以 Unity C# 为例，使用同一个需求：

角色受到伤害后更新血条。

### 1. MVC 实现

```
// Model：角色血量数据
public class HealthModel
{
    public int CurrentHP = 100;
    public int MaxHP = 100;

    public void TakeDamage(int damage)
    {
        CurrentHP = Math.Max(0, CurrentHP - damage);
    }
}

// View：负责 UI 显示
public class HealthView : MonoBehaviour
{
    public Slider healthSlider;

    public void UpdateHealth(float percent)
    {
        // 直接操作 UI 控件
        healthSlider.value = percent;
    }
}

// Controller：负责协调数据和 UI
public class HealthController
{
    private HealthModel model;
    private HealthView view;

    public HealthController(HealthModel model, HealthView view)
    {
        this.model = model;
        this.view = view;
    }

    public void TakeDamage(int damage)
    {
        // 1. 修改数据
        model.TakeDamage(damage);

        // 2. 计算显示百分比
        float percent = (float)model.CurrentHP / model.MaxHP;

        // 3. 主动刷新 UI
        view.UpdateHealth(percent);
    }
}
```

这是一种由 Controller 主动刷新的 MVC 实现。

关键代码：

```
view.UpdateHealth(percent);
```

Controller 必须明确调用 View 的更新方法。

如果后来需要显示血量数字、血条颜色、低血量警告，Controller 可能还要协调更多 UI 刷新操作。

当然，也可以通过 Model 事件让 View 自行刷新，避免 Controller 承担全部更新职责。

### 2. MVVM 实现

MVVM 的 Model 同样保存角色的真实血量，但增加数据变化事件。

```
// Model：负责角色血量数据
public class HealthModel
{
    public int CurrentHP { get; private set; } = 100;
    public int MaxHP { get; private set; } = 100;

    // 数据变化通知
    public event Action OnHealthChanged;

    public void TakeDamage(int damage)
    {
        CurrentHP = Math.Max(0, CurrentHP - damage);

        // 通知数据已经变化
        OnHealthChanged?.Invoke();
    }
}
```

ViewModel 负责将真实血量转换成 UI 需要的数据。

```
// ViewModel：负责 UI 数据和通知
public class HealthViewModel
{
    private readonly HealthModel model;

    // 通知 View：血量百分比发生变化
    public event Action<float> OnHealthPercentChanged;

    public float HealthPercent =>
        model.MaxHP > 0
            ? (float)model.CurrentHP / model.MaxHP
            : 0f;

    public HealthViewModel(HealthModel model)
    {
        this.model = model;

        // 监听 Model 变化
        model.OnHealthChanged += OnModelHealthChanged;
    }

    private void OnModelHealthChanged()
    {
        // 将变化通知给绑定的 View
        OnHealthPercentChanged?.Invoke(HealthPercent);
    }

    public void Dispose()
    {
        // 解除事件订阅，避免生命周期问题
        model.OnHealthChanged -= OnModelHealthChanged;
    }
}
```

View 只负责绑定 ViewModel：

```
public class HealthView : MonoBehaviour
{
    public Slider healthSlider;

    private HealthViewModel viewModel;

    public void Bind(HealthViewModel vm)
    {
        Unbind();

        viewModel = vm;

        // 初始化 UI
        UpdateHealth(viewModel.HealthPercent);

        // 绑定属性变化事件
        viewModel.OnHealthPercentChanged += UpdateHealth;
    }

    private void UpdateHealth(float percent)
    {
        healthSlider.value = percent;
    }

    public void Unbind()
    {
        if (viewModel == null)
            return;

        viewModel.OnHealthPercentChanged -= UpdateHealth;
        viewModel = null;
    }

    private void OnDestroy()
    {
        Unbind();
    }
}
```

这段代码使用手动事件订阅来模拟 MVVM 的属性绑定。成熟的 MVVM 框架通常会提供自动化的绑定机制。

最大的变化是：

MVC 中，Controller 通常明确调用 View 的刷新方法。

MVVM 中，ViewModel 只负责提供数据和发出变化通知，具体哪些 UI 控件需要更新，由 View 的绑定关系决定。

例如，同一个 `HealthPercent` 可以同时绑定血条、百分比文本和其他状态显示，无须在 ViewModel 中分别操作这些控件。

## 五、在 UE5 中如何使用 MVVM？

UE5 的 UMG 支持 MVVM 工作流。通过启用 UMG Viewmodel 插件，可以使用 ViewModel、FieldNotify 和 View Binding 实现事件驱动的 UI 更新。

### 1. UE5 中的架构对应

以角色血量为例：

|MVVM 组件|UE5 对应实现|
|---|---|
|Model|Character、AttributeSet、其他业务数据对象|
|View|UUserWidget、Widget Blueprint|
|ViewModel|UMVVMViewModelBase|
|属性变化通知|FieldNotify|
|数据绑定|View Binding|

假设使用 GAS（Gameplay Ability System）管理角色血量。

<img width="734" height="351" alt="image" src="https://github.com/user-attachments/assets/d82f88ba-cdad-4ad1-ac0a-a3ae888bd350" />


这样，战斗系统不需要直接操作 UI，UI 也不需要反复从角色身上获取血量。

### 2. UE5 C++ ViewModel 示例

下面是一个简化示例，展示关键的属性通知逻辑。

```
// HealthViewModel.h

#pragma once

#include "CoreMinimal.h"
#include "MVVMViewModelBase.h"
#include "HealthViewModel.generated.h"

UCLASS(BlueprintType)
class UHealthViewModel : public UMVVMViewModelBase
{
    GENERATED_BODY()

public:
    // 设置血量百分比
    void SetHealthPercent(float NewValue)
    {
        NewValue = FMath::Clamp(NewValue, 0.f, 1.f);

        // 值变化时才赋值并发出 FieldNotify
        UE_MVVM_SET_PROPERTY_VALUE(HealthPercent, NewValue);
    }

private:
    // 声明可以被 UI 绑定的属性
    UPROPERTY(BlueprintReadOnly, FieldNotify,
        meta=(AllowPrivateAccess="true"))
    float HealthPercent = 1.0f;
};
```

当 GAS 的血量属性发生变化时，业务侧就可以调用：

```
float Percent = CurrentHealth / MaxHealth;

// 更新 ViewModel
HealthViewModel->SetHealthPercent(Percent);

// 不需要手动调用：
// HealthWidget->ProgressBar->SetPercent(Percent);
```

上面假设 `MaxHealth` 大于 0，实际工程中应处理除零和无效引用。

在 UMG 的 View Binding 中将：

`UHealthViewModel.HealthPercent` → `ProgressBar.Percent`

设置为 `One Way to Widget`。

这样，`UE_MVVM_SET_PROPERTY_VALUE` 在数值变化时发出通知，绑定的 UI 就会更新。

### 3. 为什么在 UE5 中采用 MVVM？

例如你正在开发一个动作游戏，拥有多个 UI：

- 玩家 HUD：显示血量、能量、技能 CD。
    
- 角色属性面板：显示等级、攻击力、防御力。
    
- 背包界面：显示装备、数量、品质。
    
- Boss 血条：显示阶段、护盾、生命值。
    

如果 Widget 直接访问 Character、ASC、AttributeSet 和背包系统，依赖关系会越来越复杂。

MVVM 可以让不同 Widget 只依赖自己需要的 ViewModel。

例如 `UPlayerHUDViewModel` 负责 HUD 展示，`UInventoryViewModel` 负责背包展示，它们可以读取同一个角色的业务数据，但分别提供适合各自 UI 的属性。

另外，事件通知式绑定通常比每帧轮询数据更合适，特别是大量不需要每帧变化的 UI 属性。UE 官方文档也将 ViewModel 的变化通知作为相对于传统原始属性绑定的一项效率优势。

不过，也需要避免短时间内产生大量无意义的属性通知。

## 六、MVC 和 MVVM 应该怎么选？

|游戏开发场景|推荐方案|原因|
|---|---|---|
|简单确认弹窗|MVC 或直接事件处理|结构简单，无须额外 ViewModel|
|单个按钮交互|简单控制逻辑|避免过度设计|
|角色 HUD|MVVM|血量、能量等属性需要同步|
|背包系统|MVVM|数据复杂，且有多个 UI 状态|
|商城系统|MVVM|商品数据、状态、货币同步|
|角色属性面板|MVVM|属性多、计算和格式转换多|
|复杂界面导航流程|Controller / Coordinator|适合集中协调页面跳转|
|战斗技能系统|领域架构 + GAS 等|MVC/MVVM 本身不是战斗逻辑架构|

MVC 和 MVVM 也不是非此即彼。

大型游戏中完全可以组合使用：由 Controller 或 UI Manager 管理界面的打开、关闭与导航，使用 MVVM 管理具体界面的数据展示。

## 七、面试中怎么回答？

如果面试官问：“MVC 和 MVVM 的区别是什么？”

可以这样回答：

> MVC 和 MVVM 都是用于实现界面与业务逻辑分离的架构模式。
> 
> MVC 分为 Model、View 和 Controller。Controller 负责处理用户输入以及协调数据和界面，View 可以通过事件或 Controller 的调用完成更新。
> 
> MVVM 分为 Model、View 和 ViewModel。相比 MVC，MVVM 引入 ViewModel 来封装 UI 展示状态与交互逻辑，利用数据绑定和属性变化通知更新 View。
> 
> 两者最核心的区别是 UI 的数据交互方式。MVC 通常需要手动协调界面更新，MVVM 则更强调声明式数据绑定，使 ViewModel 不必直接操作 UI 控件。
> 
> MVC 更适合结构较简单、交互流程明确的界面，而 MVVM 更适合属性多、数据同步复杂的大型 UI。
> 
> 在 UE5 中，可以通过 UMVVMViewModelBase、FieldNotify 和 View Binding 实现 MVVM。对于 GAS 的属性变化，可以通过事件更新 ViewModel，再由绑定机制自动刷新 UMG。

最后需要理解一个非常重要的区别：MVC/MVVM 主要解决的是 UI 层如何组织的问题，而 GAS、ECS、行为树等解决的是其他领域的问题，它们完全可以同时存在。

对于 UE5 游戏客户端开发，建议重点掌握三个问题：ViewModel 如何监听 GAS 属性变化、FieldNotify 如何避免 UI 每帧轮询、多个 Widget 如何共享或隔离 ViewModel 的生命周期。 这三个问题比单纯背诵 MVC/MVVM 的定义更接近实际项目开发。
