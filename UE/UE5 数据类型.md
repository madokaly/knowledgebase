# UE5 数据类型完整详解（C++ + 蓝图 + 底层原理 + 面试重点）

UE 的数据类型比标准 C++ 丰富得多。除了 `int32`、`float`、`bool` 等基本类型，还包含引擎专用的字符串、数学结构体、容器、智能指针、反射对象、资源引用、委托以及各种游戏系统数据类型。

本文以 UE5 游戏客户端开发为主，重点讲解实际项目常用、底层原理重要、面试高频的类型。 UE 引擎包含数以千计的具体类和结构体，不可能逐一枚举，但可以完整梳理其主要类型体系。

## 一、UE 数据类型的整体分类

|类别|代表类型|主要作用|
|---|---|---|
|1. 基础类型|`int32`、`float`、`bool`|数值、逻辑判断|
|2. 字符串类型|`FString`、`FName`、`FText`|字符处理、标识符、本地化|
|3. 数学类型|`FVector`、`FRotator`、`FQuat`、`FTransform`|坐标、旋转、缩放|
|4. 几何类型|`FBox`、`FSphere`、`FPlane`|包围体、空间计算|
|5. 枚举类型|`enum class`、`UENUM`|状态、类型定义|
|6. 结构体类型|`struct`、`USTRUCT`|组合多个字段|
|7. 容器类型|`TArray`、`TMap`、`TSet`|数据集合|
|8. UObject 对象|`UObject`、`AActor`、`UActorComponent`|引擎对象及游戏实体|
|9. 对象引用类型|`TObjectPtr`、`TWeakObjectPtr`、`TSoftObjectPtr`|UObject 生命周期与资源引用|
|10. 智能指针类型|`TSharedPtr`、`TUniquePtr`、`TWeakPtr`|普通 C++ 对象内存管理|
|11. 类与接口类型|`UClass`、`TSubclassOf`、`TScriptInterface`|类引用、动态创建、多态|
|12. 委托类型|`DECLARE_DELEGATE` 等|事件回调|
|13. 资源及配置类型|`UDataAsset`、`UDataTable`、`FPrimaryAssetId`|数据驱动、资源加载|
|14. 游戏系统类型|`FHitResult`、`FGameplayTag`、`FGameplayAbilitySpec` 等|碰撞、战斗、GAS|
|15. 其他工具类型|`FGuid`、`FDateTime`、`FTimerHandle` 等|唯一标识、时间、定时器|

## 二、基础数据类型（Primitive Types）

UE 在标准 C++ 基础上，提供了更明确的整数类型别名。

### 2.1 整数类型

|类型|字节数|范围|常见用途|
|---|---|---|---|
|`int8`|1|-128 ~ 127|小范围有符号整数|
|`uint8`|1|0 ~ 255|字节数据、枚举底层类型|
|`int16`|2|-32768 ~ 32767|小范围整数|
|`uint16`|2|0 ~ 65535|协议字段、压缩数据|
|`int32`|4|约 ±21.47 亿|最常用的整数类型|
|`uint32`|4|0 ~ 42.95 亿|位标志、无符号计数|
|`int64`|8|约 ±9.22×10¹⁸|大整数、时间戳|
|`uint64`|8|0 ~ 约 1.84×10¹⁹|大范围 ID、位掩码|

示例：

```
// 角色等级
int32 Level = 10;

// 角色金币
int64 Gold = 10000000000LL;

// 小范围枚举值、字节数据
uint8 State = 1;

// 位掩码
uint32 Flags = 0;

// 血量值
int32 Health = 100;
```

为什么 UE 推荐 `int32` 而不是 `int`？

主要是为了让数据宽度明确。

标准 C++ 的 `int` 在语言层面并不保证一定是 32 位，而 UE 的 `int32` 明确表示 32 位有符号整数。

在序列化、网络协议和二进制文件中，数据宽度尤其重要。

例如：

```
struct FNetworkPacket
{
    int32 ActorId;  // 明确使用 4 字节
    uint16 SkillId; // 明确使用 2 字节
    uint8 State;    // 明确使用 1 字节
};
```

注意：结构体可能由于内存对齐产生填充字节，不能直接认为结构体总大小就是各字段字节数之和，也不应随意将原始结构体内存当作稳定网络协议。

### 2.2 浮点数类型

|类型|大小|有效十进制精度|用途|
|---|---|---|---|
|`float`|4 字节|约 6～7 位|动画参数、速度、各种比例|
|`double`|8 字节|约 15～16 位|高精度位置、大世界坐标|

```
float MoveSpeed = 600.0f;
float AttackRate = 1.25f;

double WorldDistance = 1000000.123456;
```

### float 与 double 的主要区别

- `float`：单精度浮点数，32 位。
    
- `double`：双精度浮点数，64 位。
    

它们都采用近似数值表示，并不能精确表示所有十进制小数。

比如：

```
float A = 0.1f;
float B = 0.2f;

// 不应依赖 A + B 与 0.3f 严格相等
bool bEqual = FMath::IsNearlyEqual(A + B, 0.3f);
```

对于游戏项目，通常使用误差比较，而不是直接通过 `==` 判断浮点计算结果。

UE5 的重要变化：大世界坐标（LWC）

UE4 中常见的 `FVector` 使用单精度浮点，而 UE5 中 `FVector` 默认使用双精度分量。

因此：

```
FVector Position;    // UE5：通常为双精度向量
FVector3f PositionF;  // 明确使用 float 分量
FVector3d PositionD;  // 明确使用 double 分量
```

Epic 也推荐一般情况下使用 `FVector` 别名，仅在必须明确精度时使用 `FVector3f` 或 `FVector3d`。

Epic Developer Community+1

### 2.3 布尔类型

```
bool bIsDead = false;
bool bIsAttacking = true;
bool bCanJump = false;
```

UE 习惯使用 `b` 前缀命名布尔成员：

```
UPROPERTY(EditAnywhere, BlueprintReadWrite)
bool bCanMove = true;
```

在普通 C++ 对象中，`bool` 通常占 1 字节，但反射属性、位域和网络序列化的实际存储形式可能不同。

### 2.4 字符类型

```
char AsciiChar = 'A';

TCHAR Letter = TEXT('A');

const TCHAR* Message = TEXT("Hello Unreal");
```

其中：

`char` 是标准 C++ 字符类型。

`TCHAR` 是 UE 使用的字符类型抽象，服务于引擎的字符串接口和平台兼容。

`TEXT()` 是用于生成符合 UE 字符表示方式的字符串字面量的宏。

另外还有 `ANSICHAR`、`UTF8CHAR`、`WIDECHAR` 等类型，常见于字符编码转换、文件或第三方库交互。

## 三、UE 三大字符串类型（面试高频）

这是 UE 必须熟练掌握的内容。

![[局部截取_20261120_004156.png]]

### 3.1 FString：可修改的字符串

`FString` 是 UE 中常用的动态字符串类型，主要存储和处理文本内容。

从实现上看，它是基于动态字符数组的字符串容器，拥有自己的字符存储空间。

示例：

```
FString PlayerName = TEXT("Player01");

// 字符串拼接
PlayerName += TEXT("_Test");

// 获取长度
int32 Length = PlayerName.Len();

// 判断是否包含指定内容
bool bContains = PlayerName.Contains(TEXT("Player"));

// 查找子字符串
int32 Index = PlayerName.Find(TEXT("Test"));

// 修改字符串
PlayerName.ReplaceInline(TEXT("Test"), TEXT("New"));

// 转换为数字
int32 Number = FCString::Atoi(TEXT("123"));

// 数字转字符串
FString NumberText = FString::FromInt(100);
```

#### FString 的底层特点

`FString` 的核心是动态数组式存储：

```
FString
   |
   +-- 字符数据指针
   +-- 当前字符数量
   +-- 分配的容量
          |
          v
      H e l l o \0
```

这是一种简化的概念示意。

字符串长度增加并超过当前容量时，可能触发新的内存分配、复制或移动数据。

因此，频繁拼接大量字符串可能带来额外的内存分配开销。

常见场景： 日志信息、玩家输入、文件路径处理、解析配置、动态生成字符串。

### 3.2 FName：不可变的名字标识

`FName` 是 UE 专门为高效的名称存储与比较设计的类型。

例如：

```
FName SocketName = TEXT("WeaponSocket");

FName BoneName = TEXT("hand_r");

FName StateName = TEXT("Attack");

FName ParameterName = TEXT("BaseColor");
```

它经常出现在骨骼名称、动画通知名称、Socket 名称、材质参数名以及对象标识符等场景。

#### FName 的底层原理

UE 维护一个全局的名字池（Name Pool），相同的名字内容可以复用名字池中的记录。

![[局部截取_20261120_004216.png]]

核心特点：

1. `FName` 是不可变标识符，不能像 `FString` 一样直接拼接修改。
    
2. 默认比较不区分大小写。
    
3. 名字相等性比较通常不需要逐字符比较，效率很高。
    
4. 名字本身并不等同于显示文本，也没有 `FText` 的本地化语义。
    

例如：

```
FName A(TEXT("Attack"));
FName B(TEXT("attack"));

bool bEqual = (A == B); // true
```

为什么骨骼名称、Socket 名称要使用 FName？

因为游戏运行过程中会大量进行名字查找和比较。

例如：

```
Mesh->GetSocketLocation(TEXT("WeaponSocket"));
```

通过 `FName` 可以避免许多普通字符串逐字符比较操作。

需要注意：从字符串首次构建或查找 `FName` 仍有成本，不能理解为所有涉及 `FName` 的操作都是 O(1)。

### 3.3 FText：支持本地化的文本

`FText` 主要用于玩家可以看到的文字。

例如：

```
FText SkillName = FText::FromString(TEXT("烈焰斩"));

FText UIMessage = NSLOCTEXT(
    "Combat",
    "NotEnoughEnergy",
    "能量不足"
);
```

其特点是支持语言本地化、文本格式化、数字和时间的文化格式等。

例如同一 UI 文字可能需要根据游戏语言显示不同内容。

注意：

```
FText::FromString(TEXT("攻击"));
```

虽然能够生成 `FText`，但并不意味着文本自动获得翻译资源。通常要通过 `LOCTEXT`、`NSLOCTEXT` 等本地化机制创建可收集、可翻译的文本。

### 3.4 三种字符串类型对比

|对比项|FString|FName|FText|
|---|---|---|---|
|能否直接修改内容|可以|不可以|不直接修改原内容|
|核心目的|字符串处理|标识与比较|文本显示与本地化|
|比较性能|通常需比较字符|相等比较高效|取决于比较语义|
|支持本地化|不直接支持|不支持|支持|
|是否区分大小写|可指定比较方式|名称相等比较默认不区分|依比较 API 与文化规则|
|常见用途|日志、文件名、输入|骨骼名、参数名|UI、对话、任务|

Epic 官方文档明确区分了三者的设计目标。

Epic Developer Community+2

### 3.5 三种类型之间的转换

```
FString Str = TEXT("Attack");

// FString -> FName
FName Name(*Str);

// FName -> FString
FString NameStr = Name.ToString();

// FString -> FText
FText Text = FText::FromString(Str);

// FText -> FString
FString TextStr = Text.ToString();

// FName -> FText
FText NameText = FText::FromName(Name);
```

特别注意：`FText` 转为 `FString` 后再转换回 `FText`，可能丢失原来的本地化历史信息。

面试重点：FString、FName、FText 如何选？

可以总结为：需要字符串运算使用 `FString`；需要频繁比较的名字标识使用 `FName`；需要向用户显示且考虑本地化的文本使用 `FText`。

## 四、UE 数学数据类型（角色移动、动画、战斗高频）

### 4.1 常用数学类型汇总

|类型|中文名称|主要用途|
|---|---|---|
|`FVector2D`|二维向量|UI 坐标、二维移动输入|
|`FVector`|三维向量|世界坐标、方向、速度|
|`FVector4`|四维向量|四维数学运算|
|`FIntPoint`|二维整数点|屏幕尺寸、网格索引|
|`FIntVector`|三维整数向量|三维整数坐标|
|`FRotator`|欧拉角旋转|角色朝向、摄像机角度|
|`FQuat`|四元数旋转|旋转插值、骨骼动画|
|`FTransform`|变换结构体|位置、旋转、缩放|
|`FMatrix`|4×4 矩阵|坐标空间变换|
|`FLinearColor`|线性颜色|材质、光照和颜色计算|
|`FColor`|8 位通道颜色|颜色数据存储|

### 4.2 FVector：三维向量

这是 UE 使用频率最高的数学类型之一。

```
FVector Position(100.0, 200.0, 300.0);

FVector Direction(1.0, 0.0, 0.0);

FVector Velocity(300.0, 0.0, 0.0);
```

一个向量由三个分量组成：

$$
\mathbf{V}=(X,Y,Z)
$$

![[局部截取_20261120_004240.png]]

常用函数：

```
FVector A(1.0, 2.0, 3.0);
FVector B(4.0, 5.0, 6.0);

// 向量加法
FVector Sum = A + B;

// 向量长度
double Length = A.Length();

// 单位化
FVector Unit = A.GetSafeNormal();

// 点积：衡量方向相似程度
double Dot = FVector::DotProduct(A, B);

// 叉积：得到垂直于两向量的向量
FVector Cross = FVector::CrossProduct(A, B);

// 两个位置之间的距离
double Distance = FVector::Distance(A, B);
```

#### 点积在战斗系统中的应用

例如判断敌人是否在角色前方。

$$
A\cdot B=|A||B|\cos\theta
$$

对于单位向量：

- 点积接近 1：方向几乎一致。
    
- 点积为 0：方向垂直。
    
- 点积接近 -1：方向相反。
    

```
FVector Forward = Character->GetActorForwardVector();

FVector ToEnemy = (
    Enemy->GetActorLocation() -
    Character->GetActorLocation()
).GetSafeNormal();

// 计算敌人与角色正前方夹角的余弦值
double Dot = FVector::DotProduct(Forward, ToEnemy);

// 敌人位于角色前方约 90 度的扇形区域内
bool bInFront = Dot > 0.707;
```

这里还应注意：使用三维方向判断会受到高度差影响。如果是地面战斗中的水平扇形攻击，通常需要先把向量投影到水平面。

### 4.3 FRotator：欧拉角旋转

```
FRotator Rotation(0.0, 90.0, 0.0);
```

包含三个分量：

|分量|含义|典型旋转轴|
|---|---|---|
|Pitch|俯仰角|Y 轴|
|Yaw|偏航角|Z 轴|
|Roll|横滚角|X 轴|

游戏中最常见的是 Yaw，用于表示角色水平朝向。

```
FRotator CurrentRotation = Character->GetActorRotation();

// 修改角色朝向
CurrentRotation.Yaw += 30.0;

Character->SetActorRotation(CurrentRotation);
```

UE 的 Rotator 角度通常使用度数，而不是弧度。

欧拉角的主要问题：万向节锁（Gimbal Lock）。

当旋转组合导致两个旋转自由度重合时，可能失去一个独立旋转自由度，因此复杂三维旋转和动画混合经常使用四元数。

### 4.4 FQuat：四元数

```
FQuat RotationQuat = FRotator(0.0, 90.0, 0.0).Quaternion();
```

四元数有四个分量：

\[ q=(x,y,z,w) \]

其中表示纯旋转的单位四元数满足：

\[ x^2+y^2+z^2+w^2=1 \]

相比直接操作欧拉角，四元数特别适合旋转组合和球面插值。

```
FQuat Start = FQuat::Identity;

FQuat End = FRotator(0.0, 180.0, 0.0).Quaternion();

// 球面线性插值
FQuat Result = FQuat::Slerp(Start, End, 0.5);
```

应用场景包括骨骼动画、摄像机旋转、复杂角色朝向插值。

面试常问：为什么骨骼动画通常使用四元数，而不直接使用欧拉角？

因为四元数适合复合旋转与平滑插值，并能避免欧拉角表示中的万向节锁问题。

### 4.5 FTransform：变换结构体

`FTransform` 由三个部分组成：

- Translation：平移。
    
- Rotation：旋转，内部使用四元数。
    
- Scale3D：缩放。
    

```
FTransform Transform;

Transform.SetLocation(FVector(100, 0, 0));

Transform.SetRotation(
    FRotator(0, 90, 0).Quaternion()
);

Transform.SetScale3D(FVector(1, 1, 1));
```

常用方法：

```
FVector WorldPoint = Transform.TransformPosition(
    FVector(10, 0, 0)
);

// 将世界空间位置转换为局部空间位置
FVector LocalPoint = Transform.InverseTransformPosition(
    WorldPoint
);
```

它经常用于：

- Actor 的世界变换。
    
- 组件的相对变换。
    
- 骨骼的局部变换与组件空间变换。
    
- 技能特效的挂点变换。
    
- 网络同步的位置、旋转和缩放信息。
    

### 4.6 FMatrix：矩阵

`FMatrix` 是 4×4 变换矩阵，可表示坐标变换及更一般的仿射变换。

`FTransform` 与 `FMatrix` 的重要区别：

|类型|优点|局限|
|---|---|---|
|FTransform|平移、旋转、缩放分离，适合游戏对象操作|不能完整表达所有仿射变换，例如剪切变换|
|FMatrix|方便矩阵乘法及统一变换计算|一般操作更复杂，存储成本较高|

需要特别注意：存在非均匀缩放与旋转组合时，多个 `FTransform` 的组合不一定等价于任意完整矩阵变换，因为结果可能包含剪切，而 `FTransform` 不显式保存剪切分量。

### 4.7 FLinearColor 与 FColor

```
// 浮点颜色，常用于线性空间颜色计算
FLinearColor LinearColor(1.0f, 0.5f, 0.0f, 1.0f);

// 每个分量使用 8 位整数
FColor ByteColor(255, 128, 0, 255);
```

|类型|数据表示|常见用途|
|---|---|---|
|FLinearColor|四个 float 分量|材质、灯光、线性颜色混合|
|FColor|RGBA 四个 uint8 分量|像素与颜色数据存储|

这两个类型不能单纯理解为“高精度版颜色”和“低精度版颜色”，因为转换时还可能涉及线性颜色空间与 sRGB 编码之间的关系。

## 五、枚举类型（Enum）

枚举用于表达有限的状态或选项。

例如角色可能有待机、移动、攻击、死亡等状态。

### 5.1 普通 C++ 枚举

```
enum class ECharacterState : uint8
{
    Idle,
    Move,
    Attack,
    Dead
};

ECharacterState State = ECharacterState::Idle;
```

`enum class` 属于强类型枚举，不允许像传统枚举那样随意转换成整数。

### 5.2 UENUM：支持 UE 反射的枚举

```
UENUM(BlueprintType)
enum class ECharacterState : uint8
{
    Idle    UMETA(DisplayName = "待机"),
    Move    UMETA(DisplayName = "移动"),
    Attack  UMETA(DisplayName = "攻击"),
    Dead    UMETA(DisplayName = "死亡")
};
```

这样就能够作为蓝图变量的类型。

```
UPROPERTY(EditAnywhere, BlueprintReadWrite)
ECharacterState CharacterState =
    ECharacterState::Idle;
```

`UENUM(BlueprintType)` 并不是一种新的整数存储类型，而是为 C++ 枚举增加 UE 反射能力。

这里的 `: uint8` 表示使用 8 位无符号整数作为底层表示，特别适合这种状态种类很少的枚举。

面试注意： 枚举适合有限且相对稳定的状态集合。如果状态非常多，需要动态扩展或进行层级匹配，例如战斗中的 Buff、Debuff、技能阻断标签，通常更适合使用 `FGameplayTag`。

## 六、结构体类型（Struct / USTRUCT）

结构体将多个数据字段组合为一个逻辑整体。

### 6.1 普通 C++ struct

```
struct FPlayerData
{
    int32 Level = 1;
    float Health = 100.0f;
    FVector Position = FVector::ZeroVector;
};
```

使用：

```
FPlayerData PlayerData;

PlayerData.Level = 10;
PlayerData.Health = 80.0f;
```

普通 `struct` 的生命周期遵循 C++ 规则，可以作为局部变量、成员变量、数组元素，也可以动态分配。

它本身不属于 `UObject` GC 管理体系。

### 6.2 USTRUCT：可以被引擎反射的结构体

```
USTRUCT(BlueprintType)
struct FSkillData
{
    GENERATED_BODY()

    // 技能 ID
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 SkillId = 0;

    // 技能名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FText SkillName;

    // 基础伤害
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float Damage = 100.0f;

    // 技能动画：软引用，不要求配置加载时立刻加载动画
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSoftObjectPtr<UAnimMontage> AttackMontage;
};
```

`USTRUCT(BlueprintType)` 表示这个结构体可作为蓝图支持的结构体类型。

`UPROPERTY` 则使字段参与 UE 的属性反射系统，并根据其声明参数和上下文支持编辑器、序列化、GC 引用追踪等功能。

### 6.3 USTRUCT 与 UObject 的区别

|比较项|USTRUCT|UObject|
|---|---|---|
|类型本质|C++ 值类型结构体|UE 对象|
|创建方式|普通构造、值初始化|通常使用 NewObject 等|
|GC 管理|结构体自身不经过 UObject GC|由 UObject GC 系统管理|
|值拷贝|通常可以直接拷贝字段数据|通常使用对象引用，不按值拷贝 UObject|
|蓝图支持|使用 USTRUCT(BlueprintType)|使用 UCLASS 及对应反射声明|
|常见场景|配置、状态、战斗数据|Actor、组件、资产对象|

例如：

```
FSkillData A;
FSkillData B = A;

// B 是独立的结构体值，之后修改 B.Damage 不影响 A.Damage
B.Damage = 200.0f;
```

但如果结构体包含对象引用，复制结构体通常只是复制引用，不会自动深拷贝引用的 UObject。

这也是游戏配置系统经常使用结构体的原因之一。

### 6.4 FInstancedStruct：运行时可变结构体类型

这是 UE5 项目中值得了解的较高级类型。

普通 `USTRUCT` 的具体类型在编译期确定，而 `FInstancedStruct` 可以容纳不同类型的反射结构体实例。

例如技能配置可能存在：

- `FMeleeSkillData`：近战技能参数。
    
- `FProjectileSkillData`：投射物技能参数。
    
- `FHealSkillData`：治疗技能参数。
    

可以通过 `FInstancedStruct` 实现更灵活的异构数据存储，但需要在使用时检查实际结构体类型。

适用于复杂配置系统、数据驱动系统，普通固定类型配置不需要优先使用它。

Epic Developer Community+1

## 七、容器类型（Containers，重点）

UE 最核心的三个容器：

![[局部截取_20261120_004329.png]]

### 7.1 TArray：动态数组

类似 `std::vector`。

```
TArray<int32> Numbers;

// 添加
Numbers.Add(10);
Numbers.Add(20);

// 原地构造
Numbers.Emplace(30);

// 访问
int32 First = Numbers[0];

// 查询
bool bFound = Numbers.Contains(20);

// 删除
Numbers.Remove(10);

// 获取数量
int32 Count = Numbers.Num();

// 清空
Numbers.Empty();
```

#### 底层原理

默认 `TArray` 采用连续内存存储：

```
TArray
  |
  +-- Data：元素内存地址
  +-- Num：当前元素数量
  +-- Max：当前分配容量

内存：
[10][20][30][40][空闲][空闲]
           Num = 4
           Max = 6
```

它需要扩容时，可能申请一块更大的连续内存，并重新定位已有元素。

因此，扩容后原有元素指针、引用和迭代器可能失效。

#### 时间复杂度

|操作|平均/典型复杂度|
|---|---|
|根据下标访问|O(1)|
|尾部 Add|摊销 O(1)|
|Contains|O(n)|
|中间 Insert|O(n)|
|保持顺序的 RemoveAt|O(n)|
|不保持顺序的 RemoveAtSwap|O(1)，不计可选缩容|

`TArray` 提供两个常见的删除方法：

```
Numbers.RemoveAt(1);

// 用最后一个元素覆盖被删除位置，不保证顺序
Numbers.RemoveAtSwap(1);
```

如果不需要维持数组顺序，`RemoveAtSwap` 通常更高效。

还可以提前预留容量：

```
TArray<FVector> Positions;

Positions.Reserve(1000);
```

这样可以减少添加大量元素时的重复扩容。

### 7.2 TMap：键值对映射

类似 `std::unordered_map`，常用来根据 Key 快速查找 Value。

```
TMap<int32, FString> PlayerNames;

PlayerNames.Add(1001, TEXT("Alice"));
PlayerNames.Add(1002, TEXT("Bob"));

// 根据 Key 查找
if (FString* Name = PlayerNames.Find(1001))
{
    UE_LOG(LogTemp, Log, TEXT("%s"), **Name);
}

// 删除
PlayerNames.Remove(1002);
```

底层核心是哈希查找。

使用时必须注意：Key 需要具有合适的哈希函数及相等性判断。

如果是自定义结构体作为 Key，通常需要实现 `operator==` 和 `GetTypeHash`。

```
struct FSkillKey
{
    int32 SkillId = 0;

    bool operator==(const FSkillKey& Other) const
    {
        return SkillId == Other.SkillId;
    }

    friend uint32 GetTypeHash(const FSkillKey& Key)
    {
        return GetTypeHash(Key.SkillId);
    }
};
```

平均查找复杂度通常接近 O(1)，但极端哈希冲突情况下可能退化。

另外，TMap 不保证固定遍历顺序。

### 7.3 TSet：哈希集合

类似 `std::unordered_set`。

```
TSet<FName> ActiveStates;

ActiveStates.Add(TEXT("Attack"));
ActiveStates.Add(TEXT("Dodge"));

// 判断元素是否存在
bool bAttacking = ActiveStates.Contains(TEXT("Attack"));

// 删除元素
ActiveStates.Remove(TEXT("Attack"));
```

主要特点是元素唯一、没有稳定的顺序，适合快速判断某个值是否存在。

### 7.4 三种容器选择

|场景|推荐类型|原因|
|---|---|---|
|所有存活敌人列表|TArray|需要遍历|
|根据玩家 ID 查询玩家数据|TMap|根据 Key 查找|
|保存已经击败的 Boss ID|TSet|保证唯一、快速判断|
|技能按照释放顺序排列|TArray|需要顺序|
|技能 ID → 技能配置|TMap|根据 ID 检索|
|保存不重复的对象标记|TSet|自动去重|

这些容器及其基础行为在 Epic 文档中有明确描述。

Epic Developer Community+2

### 7.5 其他值得掌握的容器和模板类型

|类型|作用|典型场景|
|---|---|---|
|`TQueue<T>`|队列，先进先出|消息队列|
|`TDeque<T>`|双端队列（版本支持需确认）|双端插入删除|
|`TMultiMap<K,V>`|同一 Key 对应多个值|多值关联|
|`TPair<A,B>`|两个值组成一对|返回键值组合|
|`TTuple<...>`|多个类型组成的元组|临时组合数据|
|`TOptional<T>`|可选值，可能有值也可能无值|查询结果|
|`TArrayView<T>`|数组的非拥有视图|避免不必要的数据复制|
|`FStringView`|字符串的非拥有视图|只读字符串处理|
|`TBitArray<>`|紧凑位数组|大量布尔状态|
|`TFunction<...>`|类型擦除的可调用对象|回调、Lambda|
|`TFunctionRef<...>`|非拥有的可调用对象视图|临时调用回调|

例如 `TOptional`：

```
TOptional<int32> Result;

// 可能存在查询结果
Result = 100;

if (Result.IsSet())
{
    int32 Value = Result.GetValue();
}
```

`TOptional<int32>` 与使用 `-1` 表示无效值相比，能更明确表达“值不存在”的语义。

注意 `TArrayView` 和 `FStringView` 不拥有底层数据，使用它们必须确保原始数据在视图使用期间仍然有效。

## 八、UObject 对象类型

这是 UE 与普通 C++ 最大的区别之一。

### 8.1 UObject 是什么？

`UObject` 是 UE 对象系统的核心基类，支持反射、对象生命周期管理、GC、序列化等引擎功能。

UE 中常见的对象继承关系可以简化为：

```
UObject
 |
 +-- AActor
 |    |
 |    +-- APawn
 |    |    |
 |    |    +-- ACharacter
 |    |
 |    +-- APlayerController
 |    |
 |    +-- AGameModeBase
 |
 +-- UActorComponent
 |    |
 |    +-- USceneComponent
 |         |
 |         +-- UPrimitiveComponent
 |
 +-- UUserWidget
 |
 +-- UDataAsset
 |
 +-- UAnimInstance
 |
 +-- UGameplayAbility
```

这只是部分继承关系，并不代表所有对象都属于这几类。

### 8.2 UObject 与普通 C++ class 的区别

|对比|普通 C++ class|UObject|
|---|---|---|
|对象创建|`new`、栈上构造等|一般使用 UE 对象创建 API|
|生命周期|C++ 规则及自定义管理|UObject GC 系统|
|反射|无 UE 原生反射|支持|
|蓝图访问|不能直接作为反射对象|可以通过声明暴露|
|引擎序列化|需自行实现或集成|有反射属性支持|
|运行时类型|RTTI 或其他 C++ 机制|UClass 类型系统|

创建普通 UObject：

```
UMyObject* Object = NewObject<UMyObject>(this);
```

创建 Actor：

```
AMyEnemy* Enemy = GetWorld()->SpawnActor<AMyEnemy>(
    EnemyClass,
    SpawnTransform
);
```

Actor 应使用 Actor 对应的生成与生命周期接口，而不是直接通过 `NewObject` 当作普通对象创建。

面试重点：UObject 并不意味着只能通过 GC 销毁，也不意味着所有 UObject 都拥有游戏世界中的位置。

例如 `UTexture2D`、`UDataAsset` 都是 UObject，但不是 Actor。

## 九、对象引用与指针类型（非常重要）

这一部分直接关系到 UE 项目中的崩溃、悬空指针、垃圾回收和资源加载。

### 9.1 UE 常见指针汇总

|类型|是否可用于 UObject|是否保持 UObject 存活|核心用途|
|---|---|---|---|
|`UObject*`|是|被 GC 正确追踪时可以|临时访问、函数参数|
|`TObjectPtr<T>`|是|在 UPROPERTY 等正确追踪条件下可以|UObject 成员强引用|
|`TWeakObjectPtr<T>`|是|否|非拥有引用、锁定目标|
|`TSoftObjectPtr<T>`|是|否|资源软引用、按需加载|
|`TSoftClassPtr<T>`|是，引用类资产|不保持目标类资产常驻|蓝图类软引用|
|`TStrongObjectPtr<T>`|是|是|非 UObject 拥有者中的强引用|
|`TSubclassOf<T>`|代表类|类引用语义|限定子类类型|

UE 官方的对象指针文档也明确区分了这些类型在 GC、序列化和按需加载方面的行为。

Epic Developer Community

### 9.2 TObjectPtr：UObject 强引用

```
UPROPERTY()
TObjectPtr<AActor> TargetActor;
```

典型应用：

```
UPROPERTY(VisibleAnywhere)
TObjectPtr<USkeletalMeshComponent> Mesh;
```

`TObjectPtr` 是 UE5 推荐用于反射类成员的 UObject 引用表示方式之一。

它有助于引擎追踪 UObject 之间的引用关系。

但非常容易忽略的一点是：

```
// UObject 类中的成员

UPROPERTY()
TObjectPtr<UObject> A; // 可被 UE GC 反射系统追踪

TObjectPtr<UObject> B; // 不能仅因为用了 TObjectPtr 就保证被 GC 追踪
```

对于普通的反射成员字段，`UPROPERTY` 依然非常重要。

另外，强引用可以阻止正常的 GC 回收，但不能阻止 Actor 被显式调用 Destroy()。

### 9.3 TWeakObjectPtr：弱引用

```
TWeakObjectPtr<AActor> LockedTarget;
```

适合：

- 当前锁定的敌人。
    
- AI 的临时目标。
    
- 缓存某个 Actor。
    
- 异步逻辑中的对象观察引用。
    

使用：

```
if (AActor* Target = LockedTarget.Get())
{
    // UObject 尚有效，但具体游戏业务可能还需判断 Actor 状态
    if (!Target->IsActorBeingDestroyed())
    {
        // 可以继续进行相关逻辑
    }
}
```

`TWeakObjectPtr` 不负责保持对象存活。

当对应 UObject 失效时，弱引用不会像普通裸指针那样继续持有一个可直接解引用的悬空地址。

### 9.4 TSoftObjectPtr：资源软引用

这是大型 UE 项目资源管理必须掌握的类型。

```
UPROPERTY(EditAnywhere)
TSoftObjectPtr<UAnimMontage> SkillMontage;

UPROPERTY(EditAnywhere)
TSoftObjectPtr<UNiagaraSystem> SkillEffect;
```

与硬引用相比，它通常只记录资源路径及已加载对象的弱引用，不要求引用目标在配置加载时立即加载。

### 硬引用与软引用的区别

![[局部截取_20261120_004401.png]]

例如角色拥有 100 个技能，而当前只装备了 5 个。

如果技能配置对所有动画和特效都采用硬引用，就可能产生大量无用资源加载。

软引用使配置与资源的加载时机分离。

```
// 同步加载：操作简单，但可能阻塞游戏线程
UAnimMontage* Montage = SkillMontage.LoadSynchronous();
```

实际战斗项目一般更推荐在技能预加载或角色加载阶段，通过 `FStreamableManager`、`UAssetManager` 等异步加载资源，并在使用期间维持必要的有效引用。

注意：软引用不会自动保证已加载对象始终存活。 仅仅调用过一次加载函数，不代表资源以后绝对不会被卸载。

### 9.5 TSubclassOf：引用类，而非对象实例

```
UPROPERTY(EditAnywhere)
TSubclassOf<AActor> EnemyClass;
```

这是一个非常经典的区别。

```
AActor* Enemy;            // 某个已经存在的 Actor 实例

TSubclassOf<AActor> Type; // AActor 的某个子类类型
```

例如：

```
GetWorld()->SpawnActor<AActor>(
    EnemyClass,
    SpawnTransform
);
```

这里需要的是一个类，而不是场景中已经存在的对象。

`TSubclassOf<AActor>` 可以保存满足继承约束的蓝图类或 C++ 类。

### 9.6 TSoftClassPtr：软类引用

```
UPROPERTY(EditAnywhere)
TSoftClassPtr<AActor> EnemyClass;
```

与 `TSubclassOf` 的区别是，它不要求目标类资产在当前时刻已经加载。

经常用于：

- 大量角色蓝图。
    
- 敌人类型配置。
    
- 可下载或按需加载的玩法资产。
    
- 动态生成的 UI 蓝图类。
    

## 十、普通 C++ 智能指针类型

UE 有自己的共享指针体系，但它与 UObject GC 是两套独立的内存生命周期机制。

### 10.1 四种常见智能指针

|类型|中文名称|主要作用|
|---|---|---|
|`TSharedPtr<T>`|共享指针|多个持有者共享对象所有权|
|`TSharedRef<T>`|共享引用|不允许为空的共享所有权引用|
|`TWeakPtr<T>`|弱指针|观察共享对象，不增加强引用计数|
|`TUniquePtr<T>`|独占指针|独占对象所有权|

### TSharedPtr

```
struct FPlayerInfo
{
    int32 Level = 1;
};

TSharedPtr<FPlayerInfo> Info =
    MakeShared<FPlayerInfo>();

Info->Level = 10;
```

内部依靠引用计数管理对象：

```
TSharedPtr A ─┐
              ├──> FPlayerInfo 对象
TSharedPtr B ─┘

强引用计数 = 2
```

最后一个共享所有者释放引用后，对象通常被销毁。

### TWeakPtr

```
TSharedPtr<FPlayerInfo> Strong =
    MakeShared<FPlayerInfo>();

TWeakPtr<FPlayerInfo> Weak = Strong;

if (TSharedPtr<FPlayerInfo> Pinned = Weak.Pin())
{
    // 成功取得临时强引用
    int32 Level = Pinned->Level;
}
```

`TWeakPtr` 不会增加强引用计数，主要用来避免共享所有权循环引用。

### TUniquePtr

```
TUniquePtr<FPlayerInfo> Info =
    MakeUnique<FPlayerInfo>();
```

它不能被普通复制，但可以通过移动语义转移所有权。

```
TUniquePtr<FPlayerInfo> Other = MoveTemp(Info);
```

转移后，`Info` 不再拥有原来的对象。

### 10.2 智能指针与 UObject 指针的核心区别

```
// 普通 C++ 类型
TSharedPtr<FPlayerInfo> Data =
    MakeShared<FPlayerInfo>();

// UObject 类型
UPROPERTY()
TObjectPtr<UMyObject> MyObject;
```

不要这样写：

```
// 错误的管理思路：不要使用 TSharedPtr 管理 UObject
TSharedPtr<UMyObject> Object;
```

`UObject` 由 UE 的对象生命周期和 GC 系统管理，不能与普通共享指针的销毁机制直接混用。

Epic Developer Community

## 十一、类、接口与委托类型

### 11.1 UClass

`UClass` 是 UE 反射系统中描述 UObject 类类型的对象。

例如：

```
UClass* Class = ACharacter::StaticClass();
```

也可以获取已有对象的实际运行时类型：

```
UClass* ActualClass = Character->GetClass();
```

这与 `TSubclassOf` 的关系可以理解为：

`UClass*` 是类类型对象的指针，`TSubclassOf<T>` 在此基础上增加了特定子类的类型约束。

### 11.2 TScriptInterface

`TScriptInterface<T>` 用于持有实现了指定 UE 接口的对象引用。

```
UPROPERTY(EditAnywhere, BlueprintReadWrite)
TScriptInterface<IMyInteractInterface> Interactable;
```

UE 接口分为 `UInterface` 反射声明类和 `IInterface` 风格的 C++ 接口部分。

与普通 C++ 接口指针不同，UE 的接口引用需要兼顾 UObject 和反射系统，特别是接口由蓝图实现时，应按接口函数的声明形式使用 `Execute_` 等调用方式。

适用于不同类型的 Actor 共享交互、受击、拾取等行为。

### 11.3 委托（Delegate）

委托是一种事件回调机制。

|类型|作用|常见用途|
|---|---|---|
|单播委托|绑定一个回调|异步任务完成|
|多播委托|绑定多个回调|多系统事件通知|
|动态委托|支持 UObject 反射绑定|反射事件|
|动态多播委托|支持蓝图绑定多个回调|UI、血量变化通知|

例如：

```
// 声明一个动态多播委托
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(
    FOnHealthChanged,
    float,
    NewHealth
);
```

在 UObject 类中：

```
UPROPERTY(BlueprintAssignable)
FOnHealthChanged OnHealthChanged;
```

广播：

```
OnHealthChanged.Broadcast(80.0f);
```

实际应用：角色受到伤害后广播血量变化事件，血条 UI、音效系统和受击反馈系统分别响应，而不需要让角色直接依赖所有接收方。

注意： 非动态委托通常开销较低；动态委托支持反射绑定和序列化相关能力，但并不是所有委托类型都能直接暴露给蓝图。

Epic Developer Community

## 十二、碰撞、几何与物理相关类型

### 12.1 几何结构体

|类型|中文名称|主要用途|
|---|---|---|
|`FBox`|轴对齐包围盒 AABB|包围体、空间查询|
|`FSphere`|球形包围体|球形范围与粗略碰撞判断|
|`FBoxSphereBounds`|包围盒与包围球组合|渲染、组件 Bounds|
|`FPlane`|平面|几何裁剪、空间计算|
|`FIntRect`|整数矩形|屏幕区域、像素范围|
|`FBox2D`|二维包围盒|2D 空间范围|

例如：

```
FBox Bounds(
    FVector(-100, -100, -100),
    FVector(100, 100, 100)
);

// 判断点是否在包围盒内
bool bInside = Bounds.IsInside(
    FVector(10, 20, 30)
);
```

### 12.2 FHitResult：碰撞检测结果

这是动作游戏中非常重要的结构体。

当射线、球形 Sweep 或其他碰撞查询检测到目标时，可以通过 `FHitResult` 保存命中结果。

```
FHitResult Hit;

FVector Start = GetActorLocation();
FVector End = Start + GetActorForwardVector() * 1000.0;

bool bHit = GetWorld()->LineTraceSingleByChannel(
    Hit,
    Start,
    End,
    ECC_Visibility
);

if (bHit)
{
    FVector ImpactPoint = Hit.ImpactPoint;
    AActor* HitActor = Hit.GetActor();
}
```

常用字段：

|字段|含义|
|---|---|
|`bBlockingHit`|是否产生阻挡命中|
|`ImpactPoint`|实际接触点|
|`Location`|查询形状在命中时的位置；Sweep 与射线语义有区别|
|`ImpactNormal`|接触表面法线|
|`Normal`|查询结果法线，Sweep 中可能与 ImpactNormal 不同|
|`Distance`|从查询起点到命中的距离|
|`Time`|沿查询路径的归一化命中时间|
|`BoneName`|命中的骨骼名称|
|`GetActor()`|获取命中的 Actor|
|`GetComponent()`|获取命中的组件|

这些字段在武器 Trace、受击特效、命中部位判断中非常常见。

Epic Developer Community

其他相关类型：

`FCollisionQueryParams` 用于指定查询时忽略哪些对象、是否进行复杂碰撞等。

`FCollisionShape` 用于创建球体、胶囊体和盒体等查询形状。

`ECollisionChannel` 用于选择碰撞通道。

## 十三、数据配置与资源管理类型

大型 UE 游戏项目非常依赖数据驱动设计。

### 13.1 常用配置与资源类型

|类型|用途|
|---|---|
|`UDataAsset`|存储一份独立配置资产|
|`UPrimaryDataAsset`|支持 Asset Manager 主资产管理|
|`UDataTable`|保存多行结构化数据|
|`UCurveTable`|表格化曲线数据|
|`UCurveFloat`|单一浮点曲线|
|`FDataTableRowHandle`|引用某个 DataTable 中的一行|
|`FSoftObjectPath`|通过路径标识软引用资产|
|`FPrimaryAssetId`|主资产标识|
|`FPrimaryAssetType`|主资产类型|

例如技能表可以使用：

```
USTRUCT(BlueprintType)
struct FSkillTableRow : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    int32 SkillId = 0;

    UPROPERTY(EditAnywhere)
    float Cooldown = 1.0f;

    UPROPERTY(EditAnywhere)
    float Damage = 100.0f;

    UPROPERTY(EditAnywhere)
    TSoftObjectPtr<UAnimMontage> SkillAnimation;
};
```

这样技能数值和动画资源引用可以保存在 DataTable 中。

`UPrimaryDataAsset` 则特别适合需要通过 Asset Manager 按类型、资产 ID、Asset Bundle 进行统一管理的配置资产。

Epic Developer Community+2

## 十四、GAS 战斗系统中的重要数据类型

如果开发类似《鸣潮》的动作战斗系统，这些类型的优先级非常高。

|类型|中文解释|主要作用|
|---|---|---|
|`FGameplayTag`|游戏玩法标签|表示状态、技能类别、限制条件|
|`FGameplayTagContainer`|标签容器|保存多个 Gameplay Tag|
|`FGameplayTagQuery`|标签查询表达式|复杂逻辑匹配|
|`FGameplayAttributeData`|属性数据|血量、攻击力等基础值和当前值|
|`FGameplayAttribute`|属性引用描述|标识某个 AttributeSet 属性|
|`FGameplayAbilitySpec`|技能授予规格|描述 ASC 持有的技能|
|`FGameplayAbilitySpecHandle`|技能规格句柄|标识授予的某个技能规格|
|`FGameplayEffectSpec`|效果实例规格|描述一次具体 GE 应用|
|`FGameplayEffectSpecHandle`|效果规格句柄|持有或访问效果规格|
|`FActiveGameplayEffectHandle`|已激活效果句柄|标识已经应用的持续性 GE|
|`FGameplayEffectContextHandle`|效果上下文句柄|记录来源、Instigator、HitResult 等|
|`FGameplayEventData`|Gameplay 事件数据|事件标签、来源、目标等|
|`FScalableFloat`|可缩放数值|按等级或曲线计算数值|

### 14.1 FGameplayTag

```
FGameplayTag AttackTag =
    FGameplayTag::RequestGameplayTag(
        FName(TEXT("State.Attacking"))
    );
```

例如一个角色可能拥有：

```
State.Attacking
State.Stunned
State.Dodging
Weapon.Sword
Buff.AttackSpeed
```

与枚举相比，Gameplay Tag 的优势在于可扩展性、层级结构、复杂匹配及与 GAS 系统的集成。

### 14.2 FGameplayAttributeData

例如：

```
UPROPERTY(BlueprintReadOnly)
FGameplayAttributeData Health;
```

它通常包含基础值（Base Value）和当前值（Current Value）。

在 GAS 中，一些 Gameplay Effect 修改的是基础值，另一些则会通过持续性修饰器影响当前值。

因此不能简单地把 GAS 属性理解成一个普通 `float`。

### 14.3 FGameplayAbilitySpec 与 FGameplayEffectSpec

这是非常容易混淆的地方。

FGameplayAbilitySpec 表示 ASC 中已经授予的技能规格。

例如角色获得一个攻击技能：

```
FGameplayAbilitySpec Spec(
    AttackAbilityClass,
    1
);

FGameplayAbilitySpecHandle Handle =
    ASC->GiveAbility(Spec);
```

其中 `FGameplayAbilitySpec` 描述这个技能在 ASC 中的授予信息，而 `FGameplayAbilitySpecHandle` 是引用这一授予规格的标识。

FGameplayEffectSpec 表示一次具体的 Gameplay Effect 应用规格。

例如一个伤害 GE 可以根据施法者等级、效果上下文、SetByCaller 数据等生成不同的 Spec。

`FGameplayEffectSpecHandle` 用于持有该 Spec，`FActiveGameplayEffectHandle` 则用于引用已应用并处于活动状态的 GE。

这一点在区分 `ApplyGameplayEffectToSelf` 与 `ApplyGameplayEffectSpecToSelf` 时非常关键。

Epic Developer Community+3

## 十五、网络同步、输入、动画、UI 与工具类型

### 15.1 网络同步类型

|类型|用途|
|---|---|
|`FVector_NetQuantize`|网络量化向量|
|`FVector_NetQuantize10`|保留约 0.1 精度的量化向量|
|`FVector_NetQuantize100`|保留约 0.01 精度的量化向量|
|`FVector_NetQuantizeNormal`|适用于法线的网络量化向量|
|`FFastArraySerializer`|数组差量复制辅助结构|
|`FFastArraySerializerItem`|Fast Array 的元素基础结构|
|`FNetworkGUID`|网络对象标识|
|`FRepMovement`|Actor 移动复制数据|

为什么需要量化向量？

因为如果每次复制角色位置都发送完整高精度分量，可能产生不必要的带宽开销。

通过量化可以在满足游戏精度要求的情况下，降低网络数据体积。但量化不意味着不再需要预测、插值和纠错。

`FFastArraySerializer` 则适用于诸如背包列表、Buff 列表等需要进行元素级变化同步的容器，具体实现需要自定义序列化和正确标记脏数据。

Epic Developer Community

### 15.2 Enhanced Input 输入类型

|类型|作用|
|---|---|
|`FInputActionValue`|输入动作的值|
|`FInputActionInstance`|输入动作实例及触发信息|
|`ETriggerEvent`|Started、Triggered、Completed 等触发状态|
|`FKey`|具体按键|
|`FInputChord`|组合按键描述|

`FInputActionValue` 可以表示布尔、1D、2D 和 3D 输入。

```
void AMyCharacter::Move(
    const FInputActionValue& Value
)
{
    // 读取二维移动输入
    FVector2D MoveInput = Value.Get<FVector2D>();

    AddMovementInput(
        GetActorForwardVector(),
        MoveInput.Y
    );

    AddMovementInput(
        GetActorRightVector(),
        MoveInput.X
    );
}
```

这里假设 Input Action 配置为 Axis2D。

Epic Developer Community

### 15.3 动画与 UI 常见类型

|类型|场景|
|---|---|
|`FPoseContext`|动画姿态计算上下文|
|`FCompactPose`|紧凑骨骼姿态|
|`FBoneReference`|骨骼引用|
|`FAnimMontageInstance`|Montage 播放实例|
|`FSlateBrush`|UI 图片样式资源|
|`FMargin`|UI 边距|
|`FGeometry`|UI 布局几何信息|
|`FWidgetTransform`|UI 元素渲染变换|
|`FReply`|Slate 输入事件响应|
|`FSlateColor`|Slate 颜色描述|

### 15.4 通用工具类型

|类型|作用|
|---|---|
|`FGuid`|全局唯一标识|
|`FDateTime`|日期时间|
|`FTimespan`|时间间隔|
|`FTimerHandle`|定时器句柄|
|`FDelegateHandle`|委托绑定句柄|
|`FStringBuilderBase`|高效字符串构建辅助|
|`FPlatformTime`|平台时间相关接口所在的工具类型|

其中 `FDateTime`、`FTimespan`、`FGuid` 属于数据类型，而 `FPlatformTime` 更准确地说是提供时间操作接口的工具类，这些概念不能完全混为一谈。

## 十六、蓝图数据类型与 C++ 的关系

在蓝图变量列表中，你会看到 Boolean、Integer、Float、Vector、Object Reference、Class Reference 等选项。

它们与 C++ 类型的常见对应如下：

|蓝图类型|C++ 对应类型|
|---|---|
|Boolean|`bool`|
|Byte|`uint8`|
|Integer|`int32`|
|Integer64|`int64`|
|Float|`float`|
|Double|`double`|
|String|`FString`|
|Name|`FName`|
|Text|`FText`|
|Vector|`FVector`|
|Rotator|`FRotator`|
|Transform|`FTransform`|
|Object Reference|`UObject` 派生类引用|
|Class Reference|`TSubclassOf<T>` 等类引用|
|Soft Object Reference|`TSoftObjectPtr<T>`|
|Soft Class Reference|`TSoftClassPtr<T>`|
|Interface|UE 接口引用|
|Array|`TArray<T>`|
|Set|`TSet<T>`|
|Map|`TMap<K,V>`|

并不是所有 C++ 类型都能直接成为蓝图变量类型。它需要引擎反射系统支持，而且部分类型还受到蓝图系统的限制。

Epic Developer Community

一个必须理解的区别：蓝图中的对象引用不等于创建对象。

例如创建一个 `BP_Enemy` 类型的 Object Reference 变量，只是声明一个可以引用敌人实例的变量，不代表场景中自动生成了敌人。

类似地，Class Reference 保存的是类型，常用于 Spawn Actor，而不是指向已经生成的 Actor。

## 十七、UE 数据类型与反射系统的关系

UE 数据类型和普通 C++ 数据类型真正的分界，不只是有没有 `F`、`U`、`A` 前缀，而在于能不能参与 UE 反射系统。

|宏|用途|
|---|---|
|`UCLASS()`|声明反射类|
|`USTRUCT()`|声明反射结构体|
|`UENUM()`|声明反射枚举|
|`UINTERFACE()`|声明 UE 接口|
|`UPROPERTY()`|声明反射属性|
|`UFUNCTION()`|声明反射函数|
|`GENERATED_BODY()`|插入 UHT 生成代码所需的声明|

例如：

```
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 Level = 1;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float Health = 100.0f;

    UPROPERTY(VisibleAnywhere)
    TObjectPtr<USkeletalMeshComponent> CharacterMesh;

    UPROPERTY(EditAnywhere)
    TSoftObjectPtr<UAnimMontage> AttackAnimation;

    UPROPERTY()
    TArray<TObjectPtr<AActor>> NearbyEnemies;
};
```

要注意以下几个区别：

`UPROPERTY` 本身并不自动意味着蓝图可读写。需要 `BlueprintReadOnly`、`BlueprintReadWrite` 等属性说明符。

`UPROPERTY` 也不意味着自动网络同步，网络复制仍需要对应的复制声明和实现。

此外，UHT 只支持一部分 C++ 类型及模板组合，不能随意把任意复杂的模板结构暴露为反射属性。

Epic Developer Community

## 十八、UE 数据类型面试高频知识总结

|高频问题|核心答案|
|---|---|
|1. FString、FName、FText 区别？|可变字符串、名称标识、本地化文本|
|2. UE5 FVector 是 float 还是 double？|UE5 默认 FVector 使用双精度分量|
|3. FVector 与 FRotator 有什么区别？|向量表示坐标/方向；Rotator 表示欧拉角旋转|
|4. FQuat 为什么适合动画？|适合旋转组合、球面插值，避免欧拉角万向节锁|
|5. FTransform 和 FMatrix 区别？|TRS 分离表示与通用矩阵表示|
|6. TArray 的底层是什么？|默认是连续动态数组，类似 vector|
|7. TMap 底层是什么？|哈希映射，平均 O(1) 查找|
|8. TSet 与 TArray 区别？|唯一性及哈希查找 vs 有序数组|
|9. USTRUCT 与 UObject 区别？|值类型结构体 vs GC 管理的对象|
|10. TObjectPtr 与 TWeakObjectPtr 区别？|被正确追踪的强引用 vs 不保持对象存活的弱引用|
|11. TSoftObjectPtr 有什么作用？|软引用，降低硬依赖，支持按需加载|
|12. TSubclassOf 和 UObject* 区别？|类类型引用 vs 对象实例引用|
|13. TSharedPtr 能否管理 UObject？|不应该，UObject 使用专门的 GC 系统|
|14. FGameplayTag 与枚举区别？|标签更适合扩展、层级与动态状态组合|
|15. FGameplayAbilitySpec 是什么？|ASC 所持有技能的授予规格|
|16. FHitResult 是什么？|碰撞检测结果，包含命中点、法线、对象等|
|17. UPROPERTY 是否自动保证对象存活？|只有符合 GC 追踪条件的 UObject 引用才有相应作用|
|18. UObject 强引用能阻止 Actor Destroy 吗？|不能|
|19. TArray 中保存 UObject 引用是否都能防止 GC？|必须处于 GC 可追踪的引用链中|
|20. UE 为什么需要自己的类型与容器？|为引擎反射、序列化、GC、平台兼容、内存管理等体系提供支持|

## 十九、按 UE 游戏客户端开发的重要性划分

第一梯队：必须熟练掌握

`int32`、`float`、`double`、`FString`、`FName`、`FText`、`FVector`、`FRotator`、`FQuat`、`FTransform`、`TArray`、`TMap`、`TSet`、`USTRUCT`、`UObject`、`TObjectPtr`、`TWeakObjectPtr`、`TSoftObjectPtr`、`TSubclassOf`

第二梯队：战斗客户端项目重点

`FHitResult`、`FCollisionShape`、`FGameplayTag`、`FGameplayTagContainer`、`FGameplayAttributeData`、`FGameplayAbilitySpec`、`FGameplayEffectSpecHandle`、`FDataTableRowHandle`、`FPrimaryAssetId`、`FInputActionValue`、`FFastArraySerializer`

第三梯队：深入引擎源码和专项系统时掌握

`FMatrix`、`FInstancedStruct`、`FCompactPose`、`FPoseContext`、`TArrayView`、`TOptional`、`TFunctionRef`、`TStrongObjectPtr` 等

最重要的是区分四种不同概念：

- 值类型： 如 `FVector`、`FTransform`、`FSkillData`，关注数据值与拷贝语义。
    
- 对象类型： 如 `UObject`、`AActor`，关注对象身份、生命周期和 GC。
    
- 容器类型： 如 `TArray`、`TMap`，关注存储方式、复杂度和元素失效。
    
- 引用与句柄类型： 如 `TObjectPtr`、`TSoftObjectPtr`、`FGameplayAbilitySpecHandle`，关注它引用什么、是否拥有目标、是否参与 GC 或资源加载。
    

对于 UE5 C++ 游戏客户端岗位，最有价值的学习顺序是：先掌握前三大字符串、数学结构体和三大容器，再深入 UObject / USTRUCT、对象引用与内存生命周期，之后学习 GAS、资源管理和网络同步中的专项数据类型。 这样不仅能知道各个类型怎么用，也能理解它们为什么这样设计。