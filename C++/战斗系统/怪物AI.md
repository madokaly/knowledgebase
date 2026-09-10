类《鸣潮》Boss AI，可以把核心总结成一句话：

> **不是“发现玩家 → 追过去 → CD好了放技能”，而是让 Boss 持续判断战斗局势、推测玩家意图、争夺空间与节奏，再选择一段合适的战斗行为。**

可以把整个设计拆成下面这套结构。

### 1. AI主体：BT + FSM + Utility + Attack Scheduler

推荐不是单独依赖行为树，而是分层：

- **Behavior Tree（行为树）**：负责大的决策流程，例如战斗、脱战、转阶段、处决特殊机制。
- **Combat FSM（战斗状态机）**：管理 `Idle / Approach / Attack / Recover / Stagger / PhaseChange` 等明确状态。
- **Utility AI（效用评分）**：回答“现在做什么最合适？”
- **Attack Scheduler（攻击调度器）**：负责具体选择哪套攻击、Combo、变招。

也就是说：

```
行为树
  ↓
现在处于战斗状态
  ↓
Utility判断
  ↓
该压迫？拉开？反制？攻击？等待？
  ↓
Attack Scheduler
  ↓
选择具体 Attack / Combo
```

这样 Boss 才不会变成简单的：

```
Distance > 5 → Chase
Distance < 5 → Attack
```

---

### 2. Boss真正应该判断的是“玩家正在干什么”

高级 Boss 的关键不只是读取玩家位置，而是建立一个 **Player Intent Model（玩家意图模型）**。

例如推测玩家现在是在：

```
Approaching       主动接近
Retreating        撤退
Circling          绕侧面
Aggressive        连续进攻
Defensive         防守/等闪避
Healing           治疗
Charging          蓄力
Airborne          空中行动
Recovering        技能后摇
Dodging           高频闪避
WaitingForParry   等待弹反
```

因此 Boss 的输入不只是：

```
Distance
PlayerPosition
```

而应该包括：

```
距离变化
相对方向
玩家速度
玩家朝向
玩家最近技能
攻击频率
闪避频率
玩家是否刚攻击完
玩家是否连续后撤
玩家是否长期绕背
玩家资源状态
Boss自己的技能历史
```

核心思想是：

> **位置告诉 Boss 玩家“在哪”，历史行为告诉 Boss 玩家“想干什么”。**

---

### 3. 玩家意图不要只看当前帧，要看多个时间窗口

这是之前讨论里一个很重要的点。

可以同时维护：

```
短期：0.2 ~ 0.8 秒
中期：1 ~ 3 秒
长期：5 ~ 15 秒
```

它们回答的是不同问题。

例如玩家：

```
过去0.3秒：
    向Boss冲刺

过去2秒：
    连续攻击Boss

过去10秒：
    每次Boss起手都会立刻闪避
```

于是 AI 可以得到：

```
ShortTerm:
    PlayerApproaching = 0.9

MidTerm:
    PlayerAggression = 0.8

LongTerm:
    EarlyDodgeHabit = 0.85
```

不能问：

> “到底哪个时间窗口为主？”

因为它们负责的决策层不同。

比如：

```
下一个Combo动作
→ 短期最重要

现在是压制还是拉扯
→ 中期最重要

是否应该使用延迟刀骗闪
→ 长期行为模式更重要
```

---

### 4. 技能不是随机选，而是做评分

例如：

```
Score =
    DistanceScore
  + AngleScore
  + PlayerStateScore
  + IntentScore
  + ComboScore
  + PhaseScore
  + HistoryScore
  - CooldownPenalty
  - RepeatPenalty;
```

假设 Boss 有：

```
突进斩
横扫
抓取
远程投射物
延迟重击
后撤反击
```

玩家一直后退：

```
突进斩        +40
远程攻击      +30
横扫          -30
抓取          -40
```

玩家疯狂贴脸：

```
横扫          +35
后撤反击      +40
抓取          +20
远程攻击      -40
```

玩家习惯看到起手就闪：

```
普通快刀      -10
延迟攻击      +40
追踪攻击      +20
```

于是 Boss 的行为会自然产生针对性，而不是写成一堆：

```
if (PlayerDodging)
    UseDelayedAttack();
```

---

### 5. Combo不是“AI完全失去决策”

这是之前那个问题的关键。

确实，如果已经进入：

```
A → B → C
```

那么 B、C 有时是固定的。

但真正的 Boss Combo 通常应该设计成 **Combo Graph（连招图）**：

```
       → B → C
A
       → D → E
       → Stop
```

比如：

```
斩击A
 │
 ├─ 玩家继续贴脸
 │      ↓
 │    横扫B
 │      ↓
 │    下砸C
 │
 ├─ 玩家后撤
 │      ↓
 │    突进D
 │
 └─ 玩家闪避到背后
        ↓
      转身反击E
```

因此：

> **AI主要决定Combo入口、分支、终结和取消，而不是每一帧重新随机选技能。**

这会同时保证两件事：

```
动作设计有节奏、有编排
+
Boss又能根据玩家产生变化
```

---

### 6. Boss必须有“空间意识”

优秀 Boss 不应该只有：

```
离玩家远 → 靠近
```

而应该有 **Preferred Combat Distance（理想战斗距离）**。

例如：

```
0 ~ 2m      太近
2 ~ 5m      最佳攻击区间
5 ~ 8m      追击区间
8m+         远程/突进区间
```

同时考虑：

```
玩家在Boss正面？
侧面？
背后？

Boss靠墙了吗？
玩家靠墙了吗？
战斗场地中心在哪？
```

于是移动行为可以变成：

```
Approach        接近
Retreat         后撤
Strafe          横移
Circle          绕行
Reposition      重定位
KeepDistance    控距
CutOff          封走位
```

而不是永远 MoveTo(Player)。

---

### 7. Boss还应该“争夺战斗节奏”

Boss AI 可以理解成双方轮流争夺 **Initiative（主动权）**。

例如玩家刚放完大后摇技能：

```
Player Vulnerable
↓
Boss Aggression ↑
↓
立即突进压制
```

Boss自己刚打完一套：

```
Boss Recovery
↓
主动攻击欲望下降
↓
后撤 / Walk / Observe
```

因此 Boss 应该有：

```
Aggression
Pressure
Threat
Initiative
Opportunity
```

这类高层战斗变量。

这也是为什么优秀 Boss 有时会：

```
慢慢走
停顿
横移
盯着玩家
突然突进
后撤重新组织攻击
```

而不是一直攻击。

---

### 8. 要记录“攻击历史”，避免机器人感

Attack Scheduler 最好维护：

```
LastAttack
LastNAttacks
LastCombo
AttackRepeatCount
LastAttackTime
```

然后加入：

```
RepeatPenalty
```

例如：

```
刚使用过横扫：

横扫
Score -= 60

突进
Score += 10
```

否则即使使用 Utility AI，也可能连续出现：

```
横扫
横扫
横扫
横扫
```

---

### 9. Boss应该“适应玩家”，但不能读心

这是设计边界。

好的 Boss：

```
玩家连续3次早闪
→ 延迟刀概率提高
```

不好的 Boss：

```
玩家按下闪避键这一帧
→ Boss立刻修改动画追踪玩家
```

前者表现为：

> Boss 学会了我的习惯。

后者表现为：

> AI 偷看输入。

因此 AI 应尽量基于：

```
Observed Behavior
历史行为
当前可观察状态
```

而不是直接读取玩家未来输入。

---

### 10. 最终可以形成这样一套架构

```
                ┌───────────────┐
                │   Boss Brain  │
                └───────┬───────┘
                        │
             ┌──────────▼──────────┐
             │    Perception       │
             │    战斗感知          │
             └──────────┬──────────┘
                        │
             ┌──────────▼──────────┐
             │ Player Intent Model │
             │ 玩家意图分析         │
             │                     │
             │ Short / Mid / Long  │
             └──────────┬──────────┘
                        │
              ┌─────────▼─────────┐
              │ Combat Context    │
              │ 战斗上下文         │
              └─────────┬─────────┘
                        │
               ┌────────▼────────┐
               │   Utility AI    │
               │ 高层行为选择     │
               └────────┬────────┘
                        │
       ┌────────────────┼────────────────┐
       │                │                │
   Approach          Reposition       Attack
   接近               调整位置          │
                                         ▼
                              ┌──────────────────┐
                              │ Attack Scheduler │
                              │ 攻击调度器        │
                              └────────┬─────────┘
                                       │
                              ┌────────▼─────────┐
                              │   Combo Graph    │
                              │   连招图          │
                              └────────┬─────────┘
                                       │
                                    Ability
```

如果压缩成最重要的 **5 个设计原则**，就是：

1. **决策不要建立在“距离”上，而要建立在完整 Combat Context 上。**
2. **不要只判断玩家状态，要推测玩家意图。**
3. **短、中、长期时间窗口分别负责即时反应、战术判断和玩家习惯。**
4. **Utility AI 决定“现在应该干什么”，Attack Scheduler + Combo Graph 决定“具体怎么打”。**
5. **Boss的目标不是无限攻击，而是控制距离、制造压力、骗玩家反应、抓机会、主动让出节奏再重新夺回来。**

如果我们接下来继续往工程实现走，我认为最值得先展开的是 **`BossCombatContext + PlayerIntentAnalyzer + AttackScheduler + ComboGraph`** 这四个模块，因为这四个基本就是整个类《鸣潮》Boss“大脑”的核心。