# **一、什么是效用 AI（Utility AI）**

**Utility AI（效用 AI）是一种通过对候选行为进行量化评分，从而选择当前最合适行为的 AI 决策方式。**

它的核心思想不是简单地判断一个行为能不能执行，而是判断一个行为在当前环境下有多值得执行。

在游戏 AI 中，效用 AI 通常用于：

- Boss 战斗行为决策、技能选择。
    
- NPC 的进食、休息、工作、社交等行为决策。
    
- 敌人的攻击、闪避、撤退、防御、追击。
    
- 多人战斗中的攻击目标选择。
    
- 射击游戏中的寻找掩体、换弹、压制、绕后。
    
- 根据角色性格、战斗风格动态调整行为。
    

效用 AI 通常采用人工设计的数学函数进行评分，不需要机器学习或神经网络训练。其核心是将复杂的环境信息转换为可比较的行为效用值。

gdcvault.com+1

下面主要结合 **UE5、类鸣潮动作战斗游戏、Boss AI 设计以及游戏客户端开发面试**进行讲解。这里的战斗系统属于通用架构设计示例，并非鸣潮实际内部源码实现。

## **1.1 传统 AI 与效用 AI 的区别**

假设一个 Boss 具有以下行为：

- 普通攻击
    
- 重攻击
    
- 闪避
    
- 追击
    
- 后撤
    
- 恢复生命
    

传统的条件判断可能是：

```
if (HealthPercent < 0.2f)
{
    Heal();       // 恢复生命
}
else if (IsPlayerAttacking)
{
    Dodge();      // 闪避
}
else if (Distance > AttackRange)
{
    Chase();      // 追击
}
else
{
    Attack();     // 攻击
}
```

这种方式具有明显的固定优先级。

例如，只要生命值低于 20%，Boss 就优先尝试恢复生命，即使玩家已经释放了危险技能，也可能无法合理选择闪避。

而效用 AI 会综合考虑当前所有可用行为，并分别计算分数。

<img width="740" height="366" alt="image" src="https://github.com/user-attachments/assets/d842e15b-2cef-4a8f-8322-8a6ac96636e5" />

当玩家停止攻击后，闪避评分降低，恢复生命可能成为评分最高的行为。

这种方式不需要为每一种环境组合单独编写大量的 `if-else`。

**效用 AI 的本质就是将布尔决策（是或否）转换为连续数值决策（适合程度是多少）。** 但它仍然可以保留必要的布尔条件作为行为准入限制。

# **二、效用 AI 的核心组成**

|核心名词|中文解释|作用|
|---|---|---|
|Context|上下文|保存当前 AI 的环境信息|
|Action|候选行为|攻击、闪避、追击等|
|Consideration|考量因素|血量、距离、威胁程度等|
|Normalization|归一化|将不同量纲的数据统一到相同范围|
|Response Curve|响应曲线|将输入转换为效用评分|
|Utility Score|效用评分|表示行为当前的适合程度|
|Weight|权重|调整因素或行为的重要程度|
|Aggregation|分数聚合|将多个考量因素合并为最终评分|
|Selector|行为选择器|根据评分选择行为|
|Inertia|行为惯性|避免行为频繁切换|
|Execution|行为执行|执行最终选中的行为|

## **2.1 整体运行流程**

<img width="740" height="1024" alt="image" src="https://github.com/user-attachments/assets/fbba7979-6368-472c-b789-4eb3af4b2d7c" />

# **三、Consideration（考量因素）**

Consideration 是影响某个行为效用评分的独立因素，也可以称为评分维度。

例如 Boss 的普通攻击行为，可以包含以下考量因素：

|考量因素|评分倾向|
|---|---|
|与玩家的距离|距离合适时评分高|
|玩家是否处于硬直状态|硬直时评分高|
|自身耐力|耐力越充足，评分越高|
|玩家威胁程度|威胁越低，评分越高|
|攻击使用频率|最近使用次数越多，评分越低|

不同的行为可以共享考量因素，但是使用不同的评分曲线。

例如，同样是生命值：

- 普通攻击：生命值越高，攻击意愿可能越强。
    
- 恢复生命：生命值越低，恢复意愿越强。
    
- 撤退：生命值越低，撤退意愿越强。
    
- 狂暴攻击：生命值越低，攻击意愿反而越强。
    

因此，**考量因素负责提供决策依据，响应曲线负责表达具体的行为倾向。**

## **3.1 数据归一化（Normalization）**

不同考量因素的数值范围可能完全不同。

|属性|原始数值范围|
|---|---|
|生命值|0～10000|
|玩家距离|0～3000 cm|
|技能冷却时间|0～30 秒|
|耐力值|0～100|
|威胁程度|0～1|

为了方便计算，通常将数据统一映射到 0～1。

公式：

$$
x=\operatorname{Clamp} \left(\frac{Value-Min}{Max-Min},0,1\right)
$$

UE C++ 实现：

```
float NormalizeValue(float Value, float Min, float Max)
{
    // 防止除零或无效区间
    if (Max <= Min)
    {
        return 0.0f;
    }

    // 将原始数据映射到 [0, 1]
    return FMath::Clamp(
        (Value - Min) / (Max - Min),
        0.0f,
        1.0f
    );
}
```

例如 Boss 最大生命值为 10000，当前生命值为 3000，那么归一化后的生命值就是 0.3。

但要注意：**归一化后的数值不一定就是效用评分。**

生命值为 0.3，不代表恢复生命行为的评分是 0.3。对于恢复行为，血量越低反而应该具有越高的评分。

这就需要响应曲线。

# **四、Response Curve（响应曲线）详解**

响应曲线是效用 AI 最核心的部分之一，它决定了**环境数据发生变化时，AI 的行为意愿应该如何变化**。

其本质就是一个映射函数：

$$
Score=f(x)
$$

- x：归一化后的输入数据。
    
- f：响应曲线函数。
    
- Score：该考量因素的效用评分。
    

## **4.1 常用响应曲线**

<img width="739" height="1090" alt="image" src="https://github.com/user-attachments/assets/bbf47ffe-c4bc-4fc6-bb3b-b187065fda35" />

这些曲线可以用来表达不同的 AI 行为性格和决策倾向。响应曲线的选择直接影响最终行为表现。

### **① Linear（线性曲线）**

$$
y=x
$$

```
float Score = StaminaPercent;
```

耐力越高，攻击意愿越强，且两者按照相同的比例增长。

适用于简单的评分关系。

### **② Inverse Linear（反向线性曲线）**

$$
y=1-x
$$

```
// 血量越低，恢复意愿越强
float HealScore = 1.0f - HealthPercent;
```

|当前血量|恢复评分|
|---|---|
|100%|0.0|
|80%|0.2|
|50%|0.5|
|20%|0.8|
|10%|0.9|

### **③ Power Curve（幂函数曲线）**

$$
y=x^k
$$

例如恢复生命的意愿：

```
// 血量较高时，几乎没有恢复意愿
// 血量较低时，恢复意愿快速提升
float HealScore = FMath::Pow(
    1.0f - HealthPercent, 3.0f
);
```

|当前血量|线性恢复评分|三次方恢复评分|
|---|---|---|
|80%|0.200|0.008|
|60%|0.400|0.064|
|40%|0.600|0.216|
|20%|0.800|0.512|
|10%|0.900|0.729|

这样可以避免 Boss 在血量还比较健康时就频繁恢复生命。

### **④ Sigmoid（S 型曲线）**

$$
y=\frac{1}{1+e^{-k(x-t)}}
$$

- k：控制曲线变化的陡峭程度。
    
- t：控制意愿发生明显变化的阈值。
    
- x：输入数值。
    

适合实现：

- 血量低于 30% 时，撤退意愿快速提升。
    
- 威胁程度超过一定程度时，防御意愿快速提升。
    
- 仇恨值达到一定程度时，攻击意愿快速提升。
    

相比直接使用 `if` 判断，S 型曲线能够提供更加平滑的过渡。

### **⑤ Bell Curve（钟形曲线）**

$$
y=e^{-\frac{(x-\mu)^2}{2\sigma^2}}
$$

- μ：最理想的输入值。
    
- σ：可接受范围的宽松程度。
    

例如 Boss 有一个中距离冲刺斩技能：

- 玩家距离太近：不适合释放。
    
- 玩家距离适中：非常适合释放。
    
- 玩家距离太远：无法有效命中。
    
<img width="743" height="834" alt="image" src="https://github.com/user-attachments/assets/81e9e743-ab88-4e93-9d9d-afc0d376535a" />

在 UE 中，可以直接使用 `UCurveFloat` 曲线资产来配置响应曲线，让策划调整技能的评分倾向，而不需要修改 C++ 代码。

```
// 读取策划配置的响应曲线
float Score = ResponseCurve->GetFloatValue(Input);

// 限制评分范围，防止曲线超出预期
Score = FMath::Clamp(Score, 0.0f, 1.0f);
```

# **五、多个考量因素如何计算最终评分**

一个行为通常包含多个 Consideration。

例如重攻击：

|考量因素|评分|
|---|---|
|距离适合程度|0.9|
|耐力充足程度|0.8|
|玩家硬直程度|0.7|
|自身安全程度|0.6|

如何将这些分数合并成最终评分？

常见的方法有以下几种。

## **5.1 Weighted Sum（加权求和）**

$$
U=\sum_{i=1}^{n}w_i s_i
$$

通常将权重之和设为 1，或者使用加权平均。

```
float Score =
      DistanceScore * 0.4f   // 距离权重 40%
    + StaminaScore  * 0.2f   // 耐力权重 20%
    + StunScore     * 0.3f   // 硬直权重 30%
    + SafetyScore   * 0.1f;  // 安全权重 10%
```

最终评分：

$$
U=0.36+0.16+0.21+0.06=0.79
$$

**优点：** 简单直观、容易调参、不同因素之间可以互相补偿。

**缺点：** 某个因素评分为 0 时，最终评分仍然可能很高。

例如技能距离完全不满足，但是其他因素评分很高，最终仍可能选中该技能。

因此，加权求和通常需要配合硬性条件过滤。

## **5.2 Multiplication（乘法聚合）**

$$
U=\prod_{i=1}^{n}s_i
$$

```
float Score =
      DistanceScore
    * StaminaScore
    * StunScore
    * SafetyScore;
```

最终评分：

$$
U=0.9\times0.8\times0.7\times0.6=0.3024
$$

乘法聚合有一个重要特点：只要某个考量因素为 0，最终评分就为 0。

它适合多个条件都需要满足的行为。

但是乘法存在一个非常经典的问题：**考量因素越多，最终评分越容易偏低。**

假设每个考量因素评分都是 0.8。

<img width="780" height="445" alt="image" src="https://github.com/user-attachments/assets/9d0cd81d-d2f3-43c2-81fb-4895b285d50f" />

如果普通攻击只有两个考量因素，而重攻击拥有八个考量因素，那么重攻击可能长期处于评分劣势。

即使两个行为的每个考量因素都同样优秀。

## **5.3 Compensation Factor（补偿因子）**

为了缓解乘法聚合产生的评分衰减，可以使用 Dave Mark 提出的补偿方法之一。

首先计算修正系数：

$$
M=1-\frac{1}{n}
$$

然后对每个考量因素进行补偿：
$$

s'_i=s_i+(1-s_i)\times M\times s_i
$$

最后再进行乘法：

$$
U=\prod_{i=1}^{n}s'_i
$$

C++ 实现：

```
float CalculateUtility(const TArray<float>& Scores)
{
    const int32 Count = Scores.Num();

    if (Count == 0)
    {
        return 0.0f;
    }

    // 考量因素数量越多，补偿力度越大
    const float ModificationFactor =
        1.0f - 1.0f / Count;

    float FinalScore = 1.0f;

    for (float Score : Scores)
    {
        Score = FMath::Clamp(Score, 0.0f, 1.0f);

        // 保证零分仍然是零分
        if (Score <= 0.0f)
        {
            return 0.0f;
        }

        // 计算补偿值
        const float Compensation =
            (1.0f - Score) * ModificationFactor * Score;

        const float AdjustedScore = Score + Compensation;

        FinalScore *= AdjustedScore;
    }

    return FinalScore;
}
```

补偿因子可以缓解评分衰减，但不能保证不同数量、不同分布的考量因素完全公平，最终仍需要进行行为测试和分数校准。

## **5.4 Geometric Mean（几何平均数）**

另一种常见解决方案：

$$
U=\sqrt[n]{\prod_{i=1}^{n}s_i}
$$

例如：

$$
\sqrt{0.8\times0.8}=0.8
$$

$$
\sqrt[3]{0.8\times0.8\times0.8}=0.8
$$

这样，当每项评分相同时，不会因为考量因素数量增加而导致分数下降。

```
float FinalScore = FMath::Pow(
    ProductScore,
    1.0f / ConsiderationCount
);
```

它也具有零分否决的特性，但可能弱化多个一般因素共同带来的负面影响。

## **5.5 Min / Max（最小值与最大值）**

取最小值：

$$
U=\min(s_1,s_2,\ldots,s_n)
$$

类似连续数值版本的 AND（逻辑与），由最不满足的因素决定评分。

取最大值：

$$
U=\max(s_1,s_2,\ldots,s_n)
$$

类似连续数值版本的 OR（逻辑或），只要有一个因素非常适合，就可以获得高分。

UE5 的 StateTree 原生效用选择器就支持这种 AND、OR 聚合方式。

## **5.6 各种评分聚合方式总结**

|聚合方式|零分否决|考量因素数量影响|特点|
|---|---|---|---|
|加权平均|否|较小|容易理解和配置|
|直接乘法|是|大|强调多因素共同满足|
|补偿乘法|是|有一定影响|缓解乘法评分衰减|
|几何平均|是|相同评分下无影响|便于不同维度数量的比较|
|Min|是|不直接产生乘法衰减|强调最弱因素|
|Max|否|不直接产生乘法衰减|强调最强因素|

实际项目中没有必须使用的统一公式，应根据行为的设计需求选择。

# **六、Hard Constraint（硬性约束）与 Soft Constraint（软性约束）**

这是效用 AI 在实际游戏开发中非常重要的设计。

## **6.1 硬性约束**

硬性约束决定一个行为**能不能执行**。

例如：

- 技能是否处于 CD（冷却时间）。
    
- 自身是否处于眩晕状态。
    
- 耐力是否满足技能消耗。
    
- 当前技能是否被 Gameplay Tag 禁止。
    
- 玩家是否已经死亡。
    
- 技能是否属于当前 Boss 阶段。
    

```
bool CanExecuteHeavyAttack()
{
    if (IsStunned())
    {
        return false;
    }

    if (IsSkillOnCooldown())
    {
        return false;
    }

    if (Stamina < RequiredStamina)
    {
        return false;
    }

    return true;
}
```

如果硬性约束不满足，就直接将该行为从候选列表中排除，通常不需要继续计算效用评分。

## **6.2 软性约束**

软性约束决定一个行为**有多适合执行**。

例如：

```
float Score =
      DistanceScore
    * PlayerStunScore
    * SafetyScore;
```

距离适中时评分高，距离过近时评分低，但是只要满足技能实际的释放条件，就仍然有可能被选择。

## **6.3 两者的区别**

|对比|硬性约束|软性约束|
|---|---|---|
|决策结果|True / False|连续评分|
|作用|判断是否允许执行|判断是否适合执行|
|CD 冷却限制|适合|不适合作为唯一限制|
|攻击距离偏好|视需求而定|适合|
|战斗风格|不适合|适合|
|技能消耗限制|适合|不适合作为唯一限制|

**先判断能不能做，再计算值不值得做。**

例如距离限制可以同时存在于两者中：

技能最大释放距离为 1000 cm，这是硬性限制；最佳释放距离为 600 cm，这是软性评分。

# **七、行为选择策略（Action Selection）**

获得所有行为评分后，不一定要直接选择最高分行为。

## **7.1 Max Utility（最大效用选择）**

```
// 选择评分最高的可用行为
if (Score > BestScore)
{
    BestScore = Score;
    BestAction = Action;
}
```

优点是确定性强、行为容易预测、调试简单。

缺点是当环境长期不变时，Boss 可能一直释放同一个技能，导致战斗表现机械化。

## **7.2 Weighted Random（加权随机选择）**

根据行为评分计算对应的选择概率：

$$
P_i=\frac{U_i}{\sum_j U_j}
$$

例如：

<img width="737" height="251" alt="image" src="https://github.com/user-attachments/assets/5ff08b3b-9596-47d3-879e-bde91d668145" />

这种方式可以增加 Boss 的行为多样性，但有可能选中评分较低的行为。

如果所有分数均为 0，则不能直接进行概率计算，需要执行兜底行为。

## **7.3 Top-K / Threshold（前 K 名或阈值筛选）**

首先筛选出评分最高的几个行为，或者只保留评分接近最高分的行为，再进行加权随机。

例如：

|行为|评分|是否进入候选池|
|---|---|---|
|重攻击|0.95|是|
|普通攻击|0.89|是|
|冲刺攻击|0.84|是|
|后撤|0.31|否|
|追击|0.12|否|

如果使用 Top-3，那么只在前三个行为中进行随机选择。

这种方式兼顾了合理性和随机性，比较适合动作游戏的 Boss 技能选择。

## **7.4 Softmax（软最大值选择）**

$$
P_i=\frac{e^{U_i/T}}{\sum_j e^{U_j/T}}
$$

其中 T 为 Temperature（温度参数），用于控制随机程度。

<img width="742" height="359" alt="image" src="https://github.com/user-attachments/assets/6bb24d60-738f-4c3b-a5cc-1649b87047d0" />

温度越低，越倾向于最高分行为；温度越高，行为选择越接近均匀随机。

注意：Softmax 会给有限的低分行为分配非零概率，因此必须先排除不合法的行为。

# **八、如何解决效用 AI 的行为频繁切换**

这是**效用 AI 高频面试问题，也是实际项目中非常常见的问题**。

假设每隔 0.1 秒进行一次决策。

|时间|攻击评分|闪避评分|选择行为|
|---|---|---|---|
|0.0 秒|0.81|0.80|攻击|
|0.1 秒|0.80|0.81|闪避|
|0.2 秒|0.82|0.80|攻击|
|0.3 秒|0.79|0.81|闪避|
|0.4 秒|0.83|0.80|攻击|

AI 会在攻击和闪避之间反复切换，导致动画、移动、技能执行出现异常。

这种现象称为 Thrashing（决策抖动）。

## **8.1 Inertia（行为惯性）**

给当前正在执行的行为一个额外评分。

```
if (Action == CurrentAction)
{
    Score *= 1.2f; // 当前行为获得 20% 评分加成
}
```

使当前行为不会因为其他行为略微超过自身评分就被立即替换。

## **8.2 Hysteresis（滞回机制）**

只有新行为的评分超过当前行为一定幅度时，才允许切换。

```
const float SwitchThreshold = 0.15f;

if (NewScore > CurrentScore + SwitchThreshold)
{
    SwitchAction(NewAction);
}
```

<img width="740" height="482" alt="image" src="https://github.com/user-attachments/assets/07a38ecf-f861-41eb-8bd6-7252fa028ab7" />

例如当前行为评分为 0.8，切换阈值为 0.15，那么新行为需要超过 0.95 才会发生切换。

需要注意，如果评分上限为 1，固定阈值有可能让高分行为永远无法被普通行为替换，因此需要配合行为结束、最大持续时间和强制打断机制。

## **8.3 Commitment（行为承诺机制）**

一旦行为开始执行，在特定阶段内不允许普通决策打断。

例如重攻击技能：

<img width="742" height="302" alt="image" src="https://github.com/user-attachments/assets/67003f83-77d3-47b0-99c3-afcf1b5080f9" />

这是一个示例取消规则，实际项目中需要针对每个技能单独配置。

在 UE 中，可以结合：

- Anim Notify State（动画通知状态）。
    
- Gameplay Tag（玩法标签）。
    
- GAS Ability Cancel（技能取消机制）。
    
- 技能阶段状态机。
    
- 最短执行时间与最大执行时间。
    

实现行为锁定和取消窗口。

## **8.4 Cooldown（行为冷却）**

执行完一个行为之后，在一定时间内不允许再次选择该行为。

```
if (CurrentTime < Action.NextAvailableTime)
{
    return false;
}
```

除了实际技能 CD，还可以设置专门的 AI 决策冷却时间。

例如技能本身没有 CD，但是为了增加战斗多样性，AI 在 3 秒内不能再次使用同一个技能。

## **8.5 Repetition Penalty（重复行为惩罚）**

如果某个行为被连续使用多次，就降低它的评分。

```
float Penalty = FMath::Pow(
    0.7f,
    ConsecutiveUseCount
);

FinalScore *= Penalty;
```

|连续使用次数|评分系数|
|---|---|
|0|1.0|
|1|0.7|
|2|0.49|
|3|0.343|
|4|0.2401|

这种方式不是强制禁止某个技能，而是逐渐降低使用意愿。

适合防止 Boss 无限重复使用同一种攻击。

# **九、效用 AI 在 Boss 战斗中的实际应用**

假设设计一个类鸣潮的近战 Boss，拥有以下技能：

|技能|适用场景|考量因素|
|---|---|---|
|普通连击|近距离持续攻击|距离、耐力、重复次数|
|重攻击|玩家处于硬直|距离、硬直、自身安全|
|冲刺斩|中距离接近玩家|距离、路径、玩家移动状态|
|范围攻击|多人近距离包围|敌人数量、距离|
|后撤|自身处于危险环境|威胁程度、血量|
|闪避|玩家即将命中|威胁程度、闪避资源|
|远程攻击|玩家距离较远|距离、视线|
|狂暴技能|特定战斗阶段|血量、阶段、技能 CD|

## **9.1 Boss 技能决策模拟**

下面是一个简化的效用 AI 模型，可以调整环境数据，观察 Boss 行为如何变化。

<img width="741" height="1071" alt="image" src="https://github.com/user-attachments/assets/4506bb7a-6a4a-4383-87a2-7f9e3c5bdfd1" />

## **9.2 效用 AI 不应该控制所有战斗逻辑**

大型动作游戏的 Boss AI 通常适合采用混合架构，而不是完全由效用评分控制。

例如 Boss 血量下降至 50% 时，必须进入二阶段。

这种确定性的流程不应该让效用 AI 来决定是否执行。

<img width="746" height="738" alt="image" src="https://github.com/user-attachments/assets/6e65e8a0-72d0-4128-8129-0d1a7a3173d3" />

这样每一层都有明确的职责：

|系统|负责的问题|
|---|---|
|FSM（有限状态机）|Boss 当前处于什么阶段？|
|Utility AI（效用 AI）|当前应该使用哪个技能？|
|Behavior Tree（行为树）|如何完成这个行为？|
|EQS（环境查询系统）|应该移动到哪个位置？|
|GAS（游戏技能系统）|如何激活技能、处理消耗和伤害？|
|Animation（动画系统）|如何表现这个行为？|

# **十、UE5 中如何实现效用 AI**

## **10.1 常用 UE 组件及职责**

|UE 类或组件|中文说明|职责|
|---|---|---|
|`AAIController`|AI 控制器|管理 AI 决策和控制|
|`UAIPerceptionComponent`|AI 感知组件|获取视野、听觉等信息|
|`UBlackboardComponent`|黑板组件|缓存决策所需的数据|
|`UBehaviorTreeComponent`|行为树组件|执行 AI 行为|
|`UStateTreeComponent`|状态树组件|管理状态和行为选择|
|`UCurveFloat`|浮点曲线资产|配置响应曲线|
|`UDataAsset`|数据资产|保存行为配置|
|`UAbilitySystemComponent`|技能系统组件 ASC|管理技能、标签、属性等|
|`EQS`|环境查询系统|查询并评分环境中的候选位置|

## **10.2 定义 AI 决策上下文**

```
// AI 决策上下文
// 将本次决策需要的数据集中起来
struct FUtilityAIContext
{
    // 当前生命值百分比
    float HealthPercent = 1.0f;

    // 当前耐力百分比
    float StaminaPercent = 1.0f;

    // 玩家距离，UE 默认单位为 cm
    float TargetDistance = 0.0f;

    // 玩家威胁程度，范围 0~1
    float ThreatLevel = 0.0f;

    // 目标是否处于硬直
    bool bTargetStunned = false;

    // 自身是否处于硬直
    bool bSelfStunned = false;

    // 目标是否有效
    bool bHasValidTarget = false;

    // 当前 Boss 战斗阶段
    int32 CombatPhase = 1;

    // 目标对象，弱引用，不负责保持目标存活
    TWeakObjectPtr<AActor> Target;
};
```

为什么需要 Context？

因为很多行为都会读取相同的数据。如果每个行为都单独获取玩家距离、血量、耐力、感知数据，就会产生不必要的重复计算。

更合理的方式是每轮决策先构建一次 Context，然后让所有行为共享这份数据快照。

## **10.3 定义候选行为基类**

```
UCLASS(Abstract, Blueprintable)
class UUtilityAction : public UObject
{
    GENERATED_BODY()

public:

    // 行为基础权重
    UPROPERTY(EditAnywhere, Category = "Utility")
    float BaseWeight = 1.0f;

    // 判断行为是否允许执行
    virtual bool CanExecute(
        const FUtilityAIContext& Context
    ) const
    {
        return true;
    }

    // 计算行为效用
    virtual float CalculateScore(
        const FUtilityAIContext& Context
    ) const
    {
        return 0.0f;
    }
};
```

每种具体行为继承该基类，实现自己的硬性约束和评分逻辑。

## **10.4 实现重攻击行为**

```
UCLASS()
class UHeavyAttackAction : public UUtilityAction
{
    GENERATED_BODY()

public:

    // 距离响应曲线，由策划配置
    UPROPERTY(EditAnywhere)
    TObjectPtr<UCurveFloat> DistanceCurve;

    // 技能实际最大释放距离
    UPROPERTY(EditAnywhere)
    float MaxAttackRange = 400.0f;

    virtual bool CanExecute(
        const FUtilityAIContext& Context
    ) const override
    {
        // 自身硬直，禁止攻击
        if (Context.bSelfStunned)
        {
            return false;
        }

        // 目标无效
        if (!Context.bHasValidTarget ||
            !Context.Target.IsValid())
        {
            return false;
        }

        // 超过最大攻击距离
        if (Context.TargetDistance > MaxAttackRange)
        {
            return false;
        }

        // 耐力不足
        if (Context.StaminaPercent < 0.3f)
        {
            return false;
        }

        // 实际项目还需要检查技能 CD、GAS Tag 等
        return true;
    }

    virtual float CalculateScore(
        const FUtilityAIContext& Context
    ) const override
    {
        // 距离归一化
        const float DistanceInput =
            FMath::Clamp(
                Context.TargetDistance / MaxAttackRange,
                0.0f,
                1.0f
            );

        // 距离评分
        const float DistanceScore = DistanceCurve
            ? FMath::Clamp(
                DistanceCurve->GetFloatValue(DistanceInput),
                0.0f,
                1.0f)
            : 1.0f - DistanceInput;

        // 耐力越高，攻击意愿越高
        const float StaminaScore =
            Context.StaminaPercent;

        // 玩家处于硬直时，更适合重攻击
        const float StunScore =
            Context.bTargetStunned ? 1.0f : 0.4f;

        // 玩家威胁越高，自身攻击意愿越低
        const float SafetyScore =
            1.0f - Context.ThreatLevel;

        // 多因素评分聚合
        return FMath::Max(0.0f, BaseWeight)
            * DistanceScore
            * StaminaScore
            * StunScore
            * SafetyScore;
    }
};
```

## **10.5 实现效用 AI 决策组件**

```
UCLASS(ClassGroup = (AI), meta = (BlueprintSpawnableComponent))
class UUtilityAIComponent : public UActorComponent
{
    GENERATED_BODY()

public:

    // 当前 AI 实例拥有的候选行为
    // 每个行为对象需要在初始化时正确创建
    UPROPERTY()
    TArray<TObjectPtr<UUtilityAction>> Actions;

    // 当前正在执行的行为
    UPROPERTY()
    TObjectPtr<UUtilityAction> CurrentAction;

    // 当前行为惯性系数
    UPROPERTY(EditAnywhere)
    float InertiaMultiplier = 1.2f;

    UUtilityAction* SelectBestAction(
        const FUtilityAIContext& Context
    )
    {
        UUtilityAction* BestAction = nullptr;

        float BestScore = 0.0f;

        for (UUtilityAction* Action : Actions)
        {
            if (!IsValid(Action))
            {
                continue;
            }

            // 第一步：硬性约束过滤
            if (!Action->CanExecute(Context))
            {
                continue;
            }

            // 第二步：计算效用评分
            float Score =
                Action->CalculateScore(Context);

            // 排除非法数值
            if (!FMath::IsFinite(Score) || Score <= 0.0f)
            {
                continue;
            }

            // 第三步：增加当前行为惯性
            if (Action == CurrentAction)
            {
                Score *= InertiaMultiplier;
            }

            // 第四步：寻找最高分行为
            if (Score > BestScore)
            {
                BestScore = Score;
                BestAction = Action;
            }
        }

        return BestAction;
    }
};
```

这里的代码是核心架构示意，并不是已经编译测试过的完整 UE 插件。实际项目还需要处理对象初始化、行为生命周期、取消规则、配置校验、技能激活失败及兜底行为等。

特别需要注意：当前技能开始执行后可能立刻进入 CD，而 CD 只应该限制再次激活，不一定代表正在执行的技能已经不合法。因此，实际项目最好区分：

- `CanStart`：是否允许开始执行。
    
- `CanContinue`：是否允许继续执行。
    
- `CanInterrupt`：是否允许被其他行为打断。
    

# **十一、效用 AI 与行为树结合**

Behavior Tree（行为树）本身的普通 Selector（选择节点），通常按照固定顺序选择满足条件的子节点。

效用 AI 则根据当前环境动态确定行为优先级。

## **11.1 普通行为树**

<img width="745" height="374" alt="image" src="https://github.com/user-attachments/assets/655696ef-cab2-4f2d-a86c-cbe43d3ebba3" />

## **11.2 Utility Selector（效用选择节点）**

<img width="741" height="372" alt="image" src="https://github.com/user-attachments/assets/061d9cf9-53b1-4a93-a4df-2db0141462dc" />

图中的 Utility Selector 是概念上的自定义节点，并非 UE 普通行为树的内置 Selector 节点。

## **11.3 UE 行为树常用结合方式**

一种比较容易落地的方式是通过 `BTService` 计算效用评分，然后将结果写入 Blackboard（黑板）。

```
Root
 └─ Selector
     ├─ 死亡分支
     ├─ 强制硬直分支
     └─ 战斗分支
         ├─ Service：更新效用评分
         │
         └─ Selector
             ├─ Decorator：BestAction == Attack
             │   └─ Task：释放攻击技能
             │
             ├─ Decorator：BestAction == Dodge
             │   └─ Task：执行闪避
             │
             ├─ Decorator：BestAction == Chase
             │   └─ Task：追击目标
             │
             └─ Task：待机或调整站位
```

工作流程：

1. `BTService` 周期性更新 AI Context。
    
2. 调用 Utility AI 组件计算所有行为评分。
    
3. 将最合适的行为写入 Blackboard 的 `BestAction`。
    
4. 通过 Decorator（装饰器）选择对应的行为分支。
    
5. 通过 Task（任务节点）执行移动、攻击等具体逻辑。
    
6. 通过 Observer Aborts（观察者中止）和行为锁定机制控制是否允许打断。

UE 的行为树具有事件驱动特性，也可以通过 Service 周期性更新决策信息。

# **十二、UE5 StateTree 原生效用 AI**

**从 UE5.5 开始，StateTree 已经加入了原生的效用选择机制。** 不一定需要完全手写 Utility AI 框架。

StateTree 提供两种效用选择方式：

|Selection Behavior|中文说明|
|---|---|
|Try Select Children With Highest Utility|选择效用评分最高的子状态|
|Try Select Children at Random Weighted by Utility|根据子状态效用评分进行加权随机选择|

例如：

```
Root
 ├─ Idle（待机）
 ├─ Dead（死亡）
 └─ Combat（战斗）
     │
     │ Selection Behavior:
     │ Try Select Children With Highest Utility
     │
     ├─ Attack（攻击）
     │   ├─ Enter Conditions：攻击是否合法
     │   ├─ Consideration：距离评分
     │   ├─ Consideration：耐力评分
     │   └─ Task：执行攻击
     │
     ├─ Dodge（闪避）
     │   ├─ Enter Conditions：闪避是否合法
     │   ├─ Consideration：威胁评分
     │   └─ Task：执行闪避
     │
     └─ Chase（追击）
         ├─ Consideration：距离评分
         └─ Task：追击玩家
```

StateTree 中常用的评分组件包括：

|组件|作用|
|---|---|
|Float Input Consideration|浮点数输入，通过响应曲线评分|
|Enum Input Consideration|根据枚举类型获取对应评分|
|Constant Input Consideration|固定评分|
|AND|取两个评分中的最小值|
|OR|取两个评分中的最大值|
|Weight|对整个状态的评分进行加权|

例如：

$$
U=Weight\times\min(DistanceScore,StaminaScore)
$$

这里需要注意，**StateTree 的原生 AND 并不是乘法聚合，而是取最小值，OR 则是取最大值。** 加权后的最终状态评分也不一定限制在 0～1。

另外，效用评分是在相关状态选择过程中进行的，并不是只要某个 Consideration 分数变化，就必然自动切换当前状态。还需要配置相应的 Transition（状态转换）或重新选择机制。

# **十三、效用 AI 与 FSM、行为树、GOAP 的区别**

|对比项|FSM 状态机|BT 行为树|Utility AI 效用 AI|GOAP 目标导向行动规划|
|---|---|---|---|---|
|核心思想|状态与转换|树形条件与任务|行为评分|搜索行动序列|
|决策依据|状态转换条件|节点条件与优先级|多因素效用评分|目标、前置条件、效果、代价|
|优先级|通常显式配置|普通 Selector 固定顺序|动态计算|根据规划结果确定|
|行为灵活性|较低|较高|高|高|
|长期规划|弱|需要额外设计|通常较弱|强|
|调试难点|状态转换|树结构|评分与权重|搜索和规划结果|
|性能开销|通常较低|取决于节点数量|取决于评分复杂度|取决于搜索空间|
|适合场景|状态管理|复杂任务流程|动态战斗决策|多步骤目标实现|

例如 AI 当前的目标是击杀玩家，但没有武器。

- FSM：通过配置好的状态进入寻找武器状态。
    
- 行为树：判断没有武器，执行寻找武器分支。
    
- Utility AI：比较寻找武器、徒手攻击、撤退等行为评分。
    
- GOAP：规划出寻找武器 → 拾取武器 → 接近玩家 → 攻击的行动序列。
    

**Utility AI 主要解决当前做什么，GOAP 主要解决为了达成目标，需要依次做什么。**

它们也可以结合，例如先通过 Utility AI 选择当前最重要的目标，再由 GOAP 为这个目标生成行动序列。

# **十四、效用 AI 的性能优化**

假设场景中存在：

- 1000 个 AI。
    
- 每个 AI 有 20 个候选行为。
    
- 每个行为有 5 个 Consideration。
    

如果每帧都计算：

$$
1000\times20\times5=100000
$$

在 60 FPS 下，每秒就需要执行 600 万次考量因素评估。

如果这些考量因素还涉及射线检测、寻路、环境查询等操作，性能开销就会更加明显。

## **14.1 降低决策频率**

效用 AI 不需要每帧执行。

|AI 类型|示例决策间隔|
|---|---|
|高精度战斗 Boss|0.05～0.2 秒|
|近距离战斗敌人|0.1～0.3 秒|
|普通 NPC|0.3～1 秒|
|远距离 NPC|1～3 秒|
|非活跃 NPC|暂停或事件驱动|

这些是可供参考的初始配置，并非统一行业标准，需要通过实际性能分析确定。

动画、移动和碰撞检测仍然可以按照各自所需频率更新。

## **14.2 Event Driven（事件驱动）**

重要事件发生时立即触发决策，不需要等待下一轮周期更新。

例如：

- 自身受到攻击。
    
- 玩家进入或离开感知范围。
    
- 当前技能执行完成。
    
- 技能冷却完成。
    
- Boss 血量达到阶段阈值。
    
- 自身进入强制硬直。
    

普通评分可以低频更新，紧急行为则通过事件立即响应。

## **14.3 Context Cache（上下文缓存）**

错误设计：

```
// 每个行为都重新计算相同的数据
AttackAction->GetTargetDistance();
DodgeAction->GetTargetDistance();
ChaseAction->GetTargetDistance();
```

优化设计：

```
// 每轮决策只计算一次
Context.TargetDistance = FVector::Dist(
    SelfLocation,
    TargetLocation
);

// 所有行为共享 Context
```

还可以缓存视线检测、可达性、敌人数量等昂贵结果，并在需要时重新计算。

## **14.4 Early Out（提前退出）**

先执行计算成本低、过滤能力强的判断。

```
// 第一层：技能冷却判断
if (IsOnCooldown)
{
    return 0.0f;
}

// 第二层：距离判断
if (DistanceSquared > MaxRangeSquared)
{
    return 0.0f;
}

// 第三层：较昂贵的检测
if (!HasLineOfSight())
{
    return 0.0f;
}

// 最后计算评分
return CalculateUtilityScore();
```

在只需要比较距离大小时，可以使用距离平方，减少不必要的开平方计算。

乘法聚合中，如果某一项已经为 0，也可以直接结束后续评分。

## **14.5 Time Slicing（时间切片）**

避免所有 AI 在同一帧进行决策。

<img width="742" height="624" alt="image" src="https://github.com/user-attachments/assets/4b2baa42-c3ef-4300-8f32-6b9feac7f44e" />

## **14.6 AI LOD（AI 分级更新）**

根据 AI 与玩家的距离、重要程度、是否参与战斗等信息，使用不同的决策精度。

近距离 Boss 可以评估所有复杂行为；远距离普通 NPC 可以只保留巡逻、待机等简单行为。

## **14.7 多线程计算**

效用 AI 中纯数学评分部分比较适合并行处理。

但在 UE 中，不能直接将整个 AI 决策逻辑随意放到工作线程中。

推荐流程：

```
Game Thread（游戏线程）
    ↓
获取 UObject、Actor、GAS 数据
    ↓
构建纯数据快照
    ↓
Worker Thread（工作线程）
    ↓
并行计算效用评分
    ↓
Game Thread（游戏线程）
    ↓
校验结果是否仍然有效
    ↓
执行技能、移动或行为切换
```

避免在工作线程中随意操作 Actor、UObject、GAS 或调用不保证线程安全的引擎接口。

# **十五、效用 AI 的优缺点**

|优点|说明|
|---|---|
|决策灵活|可以综合多个环境因素|
|扩展性较好|新增行为通常不需要修改大量状态转换|
|自然的行为表现|可以表达渐进变化的行为意愿|
|数据驱动|策划可以配置曲线、权重和评分规则|
|便于实现战斗性格|不同敌人可共享逻辑但使用不同权重|
|容易与其他 AI 架构结合|可结合行为树、状态机、GOAP|

|缺点|说明|
|---|---|
|调参复杂|多个曲线和权重之间存在相互影响|
|行为抖动|评分接近时可能频繁切换|
|计算成本|需要评估多个候选行为|
|缺乏长期规划|通常只关注当前最合适的行为|
|分数难以统一|不同行为的评分可能存在尺度偏差|
|调试难度较高|需要知道每一个因素对最终分数的影响|

因此实际项目最好开发专门的 Utility AI 调试工具，显示每次决策时的原始数据、归一化结果、曲线输出、最终评分、行为过滤原因和行为切换原因。

# **十六、效用 AI 高频面试问题及参考答案**

以下重点保留 UE 游戏客户端、动作战斗 AI 开发中比较有价值的问题。

## **面试题 1：什么是效用 AI？核心原理是什么？**

**参考答案：**

效用 AI 是一种基于数值评分的 AI 决策系统。

它将当前环境信息作为输入，通过归一化和响应曲线将不同考量因素转换为效用评分，再通过乘法、加权求和等方式计算候选行为的最终评分，最后选择评分最高或者按评分随机选择的行为。

相比传统固定优先级决策，它能够根据环境动态调整行为优先级。

## **面试题 2：效用 AI 和行为树有什么区别？**

**参考答案：**

普通行为树通过树形结构、条件判断和节点优先级组织行为，主要优势是行为流程清晰。

效用 AI 通过动态评分比较多个行为的适合程度，能够综合距离、血量、威胁等因素。

实际项目中两者并不冲突，可以使用效用 AI 决定选择哪个行为，再使用行为树执行该行为的具体流程。

## **面试题 3：为什么效用 AI 需要归一化？**

**参考答案：**

不同考量因素的单位和数值范围不同，例如生命值可能是 0～10000，距离可能是 0～3000 cm，耐力可能是 0～100。

如果直接使用这些数值计算评分，大数值因素可能会不合理地主导最终结果。

归一化可以统一数值范围，再通过响应曲线和权重表达不同因素的重要程度。

但归一化只能统一数值尺度，不能自动保证不同评分具有相同的行为含义。

## **面试题 4：响应曲线有什么作用？为什么不直接使用归一化后的数值？**

**参考答案：**

归一化只负责统一数值范围，而响应曲线负责表达 AI 的行为倾向。

例如生命值越低，恢复意愿应该越高，因此需要使用反向曲线。

某些技能只适合在中距离释放，则可以使用钟形曲线，让距离过近或过远时评分降低。

响应曲线能够让策划通过数学函数或者曲线资产灵活控制 AI 行为，而不是编写大量条件分支。

## **面试题 5：评分使用乘法还是加法？有什么区别？**

**参考答案：**

加权求和允许多个因素互相补偿，适合综合评价，但是一个因素为 0 时，最终评分仍可能很高。

乘法更加重视多个因素共同满足，只要一个因素为 0，最终结果就是 0。

乘法的主要问题是考量因素数量增加时，评分容易发生衰减。

实际项目中可以使用硬性约束配合加权平均，也可以使用补偿乘法、几何平均等方式。

## **面试题 6：为什么乘法评分会导致考量因素越多，行为评分越低？如何解决？**

**参考答案：**

因为所有考量因素通常处于 0～1 之间，多个小于 1 的数值不断相乘会产生指数式衰减。

例如：

$$
0.8^2=0.64,\qquad 0.8^5=0.32768
$$

考量因素较多的行为会因此受到额外惩罚。

可以使用 Compensation Factor（补偿因子）、Geometric Mean（几何平均数），或者采用其他经过校准的评分聚合方式来解决。

## **面试题 7：如何解决效用 AI 在多个行为之间反复切换？**

**参考答案：**

可以通过行为惯性、滞回阈值、最短执行时间、行为承诺机制、决策冷却以及技能取消窗口解决。

例如当前攻击评分为 0.8，闪避评分为 0.81，不应该因为 0.01 的差值立即切换。

只有闪避评分超过一定阈值，或者出现紧急攻击事件，并且满足当前行为的取消条件时，才允许打断攻击。

对于动作游戏，还必须考虑攻击前摇、生效帧、后摇等技能阶段。

## **面试题 8：如何防止 Boss 一直释放评分最高的技能？**

**参考答案：**

可以采用加权随机、Top-K 随机、Softmax、技能冷却和重复行为惩罚等机制。

例如先筛选评分最高的三个合法技能，再根据评分随机选择，使 Boss 的行为具有一定多样性，同时避免执行明显不合理的技能。

但是随机性不应该覆盖死亡、强制硬直和阶段切换等确定性逻辑。

## **面试题 9：效用 AI 是否需要每帧计算？**

**参考答案：**

不需要。

可以根据 AI 类型和战斗重要程度设置不同的决策频率，例如近战敌人每隔 0.2 秒进行一次评分。

紧急事件通过事件驱动立即触发决策，避免低频更新导致闪避、硬直等重要行为响应不及时。

决策频率与移动、动画、碰撞等执行系统的更新频率应该相互独立。

## **面试题 10：大量 NPC 使用效用 AI 时，如何进行性能优化？**

**参考答案：**

主要通过降低决策频率、事件驱动、上下文缓存、候选行为预过滤、提前退出、时间切片、AI LOD 和多线程纯数据计算等方式优化。

尤其应该避免在每个 Consideration 中重复执行寻路、射线检测、EQS 查询等昂贵操作。

可以先统一获取并缓存环境信息，再让不同评分函数共享数据。

## **面试题 11：如果所有候选行为的效用评分都是 0，应该怎么办？**

**参考答案：**

应该设计 Fallback Action（兜底行为），例如待机、观察、重新寻找目标、调整站位等。

不能直接对评分进行加权随机，因为评分总和为 0 会导致除零问题。

还需要区分所有行为都不合法和所有合法行为评分过低这两种情况，必要时继续当前合法行为或者重新构建上下文。

## **面试题 12：效用 AI 如何与 GAS 结合？**

**参考答案：**

效用 AI 负责决定使用哪个技能，GAS 负责技能的实际激活和执行。

在效用 AI 评分前，可以根据 GAS 的 Gameplay Tag、Attribute、Cooldown、Cost 等信息过滤非法技能。

选择技能后，可以通过 `TryActivateAbility` 等接口尝试激活对应 Ability。

技能执行完成后，通过结束事件触发新一轮决策。

但是 `CanActivateAbility` 检查通过，不代表后续激活一定成功，因此仍然需要处理激活失败、目标失效和重新决策。

## **面试题 13：效用 AI 与 EQS 有什么区别？**

**参考答案：**

EQS 和效用 AI 都可以使用评分机制，但职责不同。

Utility AI 主要负责行为决策，例如选择攻击、撤退还是寻找掩体。

EQS 主要负责环境查询和候选结果评分，例如寻找最适合的掩体、攻击站位或者移动位置。

两者可以组合，先由效用 AI 选择撤退行为，再通过 EQS 查找最合适的撤退位置。

## **面试题 14：多人游戏中如何保证效用 AI 的决策同步？**

**参考答案：**

在服务器权威架构下，通常由服务器负责 AI 感知、效用评分、行为选择和技能激活。

客户端不需要独立计算并决定 Boss 的行为，而是通过 Actor Replication（Actor 复制）、技能状态同步、动画同步等方式表现服务器决定的结果。

这样可以避免客户端环境信息、随机数和决策时间不同造成 AI 行为不一致。

客户端可以进行必要的视觉平滑，但伤害、技能合法性和关键战斗状态应以服务器权威结果为准。

## **面试题 15：如何设计一个类鸣潮的 Boss AI 系统？**

**参考答案：**

我会采用分层混合 AI 架构。

高层使用 FSM 或 StateTree 管理 Boss 阶段、死亡、硬直和特殊机制。

中层使用 Utility AI，根据玩家距离、威胁程度、自身属性、技能冷却、攻击历史等数据动态选择技能。

执行层使用行为树或状态树 Task 管理移动、转向、技能释放和等待完成。

底层结合 GAS 管理技能生命周期、Gameplay Tag、属性消耗、伤害和冷却，结合动画系统控制攻击前摇、攻击生效期和后摇。

同时加入行为惯性、技能取消窗口、重复行为惩罚、加权随机、事件驱动和上下文缓存，保证战斗行为的合理性、多样性、稳定性以及性能。

# **十七、效用 AI 核心知识总结**

|知识点|面试重要程度|
|---|---|
|Utility AI 核心原理|★★★★★|
|Consideration 考量因素|★★★★★|
|Response Curve 响应曲线|★★★★★|
|数据归一化|★★★★★|
|加法与乘法评分区别|★★★★★|
|硬性约束与软性约束|★★★★★|
|行为惯性与决策抖动|★★★★★|
|与行为树、状态机结合|★★★★★|
|Boss 技能决策设计|★★★★★|
|性能优化|★★★★★|
|与 GAS 结合|★★★★★|
|补偿因子|★★★★|
|加权随机与 Softmax|★★★★|
|StateTree 原生效用选择器|★★★★|
|多人游戏 AI 同步|★★★★|

最后，对于 UE 动作游戏客户端开发，需要重点理解三个问题：

**行为是否允许执行（Hard Constraint） → 行为是否值得执行（Utility Score） → 行为是否允许切换（Execution & Interrupt）。**

这三个问题分别对应行为合法性、行为决策和行为生命周期。只有将这三部分分离设计，才能使效用 AI 真正适用于复杂的商业游戏战斗系统，而不只是一个简单的技能评分器。
