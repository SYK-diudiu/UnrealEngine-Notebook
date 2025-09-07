[TOC]

# GAS工程创建操作指北

## 前言

本文所介绍的内容大多在`C++`项目工程下进行，如果为纯蓝图工程可能有所出入。

文中部分专有名词为缩略写法，详见**常用术语缩略**。

本文不会过多解释`GAS`相关内容的理论知识，重在描述如何快速搭建相关功能、创建相关组件，迅速实现功能的编写。

本文不考虑`GAS`网络相关内容，默认为单机工程。

## 开启GAS插件

通过虚幻编辑器的**插件浏览器**找到`GAS`插件并启动。

![image-20250902174440830](GAS工程创建操作指北.assets/image-20250902174440830.png)

![image-20250902174630383](GAS工程创建操作指北.assets/image-20250902174630383.png)

勾选上图所示插件即可启动我们所说的`GAS`，虚幻编辑器应该会弹出一个更新`.uproject`文件的提示，选择更新即可，更新的内容其实就是将`GAS`插件设置为开启状态。

## 为模块添加插件

开启了`GAS`插件之后，我们并不能直接在C++类中添加相关头文件开始编写`GAS`代码。

在需要使用`GAS`相关功能的模块的`.Build.cs`文件中更改代码如下所示。

```C++
using UnrealBuildTool;

public class GASTestDemo : ModuleRules
{
	public GASTestDemo(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

		PublicDependencyModuleNames.AddRange(new string[] {
			"Core",
			"CoreUObject",
			"Engine",
			"InputCore",
			"EnhancedInput"
		});

		PrivateDependencyModuleNames.AddRange(new string[] {
            "GameplayAbilities",
            "GameplayTags"          // GAS 标签系统模块（可选，若用 GameplayTag）
        });

	}
}
```

在该模块中添加`GameplayAbilities`模块、`GameplayTags`模块，这里两个模块是`GAS`插件中较为常用的模块，其中`GameplayAbilities`模块是我们接下来为角色添加`AbilitySystemComponent`组件等其它组件的基础。此处我将模块添加到了私有依赖中，这不是必须项，只是我的个人选择，你也可以写在共有依赖中。

## 为Character添加ASC组件

### 添加ASC组件

在需要添加`ASC`的`Character`头文件中添加头文件。

```C++
#include "AbilitySystemComponent.h"   // GAS 核心组件头文件
```

添加`ASC`指针。

```C++
// --------------------------
// 标记为 VisibleAnywhere：编辑器中可见但不可修改
// 标记为 BlueprintReadOnly：蓝图可读取但不可修改
// 标记为 Category = "GAS"：归类到 GAS 分组
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "GAS", meta = (AllowPrivateAccess = "true"))
UAbilitySystemComponent* AbilitySystemComponent;
```

在构造函数中添加实例化`ASC`的代码。

```C++
AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
```

### 重写GetAbilitySystemComponent()函数。

添加头文件。

```C++
#include "AbilitySystemInterface.h"
```

让当前类继承`IAbilitySystemInterface`接口，类似下列所示代码。

```C++
class AGASTestDemoCharacter : public ACharacter , public IAbilitySystemInterface
{
	GENERATED_BODY()    
	//...此处省略
};
```

实现接口中的函数，也就是重写`GetAbilitySystemComponent()`函数。

```C++
// --------------------------
// 关键：获取 ASC 的接口（供外部调用，如技能系统）
// --------------------------
UFUNCTION(BlueprintCallable, Category = "GAS")
UAbilitySystemComponent* GetAbilitySystemComponent() const override;
```

在`CPP`文件中的具体实现如下：

```C++
UAbilitySystemComponent* AGASTestDemoCharacter::GetAbilitySystemComponent() const
{
    //这里返回的就是我们前面在构造函数中实例化的ASC组件
	return AbilitySystemComponent;
}
```

### 初始化ASC组件

我们知道`ASC`需要初始化它的`OwnerActor`和`AvatarActor`，初始化操作需要使用到：

```C++
AbilitySystemComponent->InitAbilityActorInfo(OwnerActor*, AvatarActor*);
```

#### 核心前提：明确`InitAbilityActorInfo`的依赖条件

调用`InitAbilityActorInfo`前，必须确保两个核心条件已就绪，否则会导致绑定失败：

1. **ASC 实例已创建并初始化**（如通过`CreateDefaultSubobject`或动态创建）；
2. **角色（Actor）的`ActorInfo`容器已有效**（包含角色的`Mesh`、`PlayerController`等关键信息，通常在角色`BeginPlay`或被控制器控制后初始化）。

简单来说：必须先有`ASC`和 “可被绑定的角色”，才能执行初始化绑定。

我这里重写了父类`Pawn`中的`PossessedBy`函数，并在其基础上添加初始化`ASC`的代码。

**为什么不推荐放在`BeginPlay`？**
`BeginPlay`在 “角色生成时” 执行，但玩家角色的`PlayerController`可能未完成接管（如加载场景时角色先生成，控制器后连接），此时`InitAbilityActorInfo`的第二个参数（控制器）为空，会导致 “技能无法响应玩家输入”。而`PossessedBy`在控制器接管时触发，能确保控制器信息有效。

```C++
virtual void PossessedBy(AController* NewController) override;
```

在`CPP`文件中的具体实现如下：

```C++
void AGASTestDemoCharacter::PossessedBy(AController* NewController)
{
	Super::PossessedBy(NewController);

	if (AbilitySystemComponent)
	{
		AbilitySystemComponent->InitAbilityActorInfo(this, this);
	}

	SetOwner(NewController);
}
```

**补充**：

我们知道上述`PossessedBy`函数的函数体是`ASC`存在于`Character`上的情况。还有一种情况是我们的`ASC`存在于`PlayerState`上，那么此时在`Character`中重写的这个`PossessedBy`函数的实现方式如下：

```C++
void AGASTestDemoCharacter::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// 人工智能不会有玩家控制器，所以我们可以在这里再次初始化以确保安全。对于有玩家控制器的英雄来说，初始化两次也没有坏处。
		PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
	}
    
	SetOwner(NewController);
	//...
}
```

## 为ASC添加AttributeSet

`ASC`的工作离不开`Attribute`，角色的属性都放在了`AttributeSet`里，因此`AttributeSet`应该添加到`ASC`所在的类里面。

如果`ASC`存在与`Character`，那么`AttributeSet`就应该存在于`Character`；如果`ASC`存在于`PlayerState`，那么`AttributeSet`就应该存在于`PlayerState`。

### 继承UAttributeSet

此外还需要添加以下代码到头文件中

```C++
// 这里的宏主要用于帮我们自动生成各个属性的相关Get、Set函数
// 需要在声明的每个属性下面使用该宏
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)
```

`ATTRIBUTE_ACCESSORS`这个宏会自动生成以下函数(以Health属性为例)

```C++
GetHealthAttribute();
GetHealth();
SetHealth(float NewValue);
InitHealth(float BaseValue);
```

下面是头文件代码

```C++
#pragma once

#include "CoreMinimal.h"
#include "AttributeSet.h"
#include "AbilitySystemComponent.h"
#include "MyAttributeSet.generated.h"

// 这里的宏主要用于帮我们自动生成各个属性的相关Get、Set函数
// 需要在声明的每个属性下面使用该宏
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

UCLASS()
class GASTESTDEMO_API UMyAttributeSet : public UAttributeSet
{
	GENERATED_BODY()
public:
    // --------------------------
    // 示例属性：生命值
    // --------------------------
    UPROPERTY(BlueprintReadOnly, Category = "Health")
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health)
        
    //写到这里需要添加头文件
    //#include "AbilitySystemComponent.h"

    // --------------------------
    // 示例属性：最大生命值
    // --------------------------
    UPROPERTY(BlueprintReadOnly, Category = "Health")
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth)
};
```

源文件暂时不需要写什么内容，只看头文件即可。

现在有了自己的`AttributeSet`类，这里就是`UMyAttributeSet`类，类中有两个`Attribute`，分别代表**生命值**和**最大生命值**。

### 添加原始AS

在`ASC`所在类的头文件中添加`UMyAttributeSet`类指针。

```C++
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "GAS", meta = (AllowPrivateAccess = "true"))
UMyAttributeSet* OriginAS;
```

**重点**：在`ASC`的`OwnerActor`的构造函数中创建的`AttributeSet`将会自动注册到`ASC`。

如果`ASC`的`OwnerActor`是`Character`，就写在`Character`类的构造函数中，如果`ASC`的`OwnerActor`是`PlayerState`就写在`PlayerState`的构造函数中，这种情况下实例化出来的`AS`可以自动注册到`ASC`，无需调用接口手动注册。

在`ASC`的`OwnerActor`的构造函数中实例化`OriginAS`。

```C++
OriginAS = CreateDefaultSubobject<UMyAttributeSet>(TEXT("MyAttributeSet"));
```

到此即成功将我们的自定义`AS`添加到`ASC`上了。

# 注意事项

ASC、GA、AttributeSet等相关变量名、指针名，尽量不要存在与引擎内部相同的可能，否则可能会出现难以预料的问题。

# 常用术语缩略

|                术语                |        缩略         |
| :--------------------------------: | :-----------------: |
|       AbilitySystemComponent       |         ASC         |
|            AbilityTask             |         AT          |
| Action RPG Sample Project by Epic  |  ARPG, ARPG Sample  |
|     CharacterMovementComponent     |         CMC         |
|          GameplayAbility           |         GA          |
|       GameplayAbilitySystem        |         GAS         |
|            GameplayCue             |         GC          |
|           GameplayEffect           |         GE          |
| GameplayEffectExecutionCalculation | ExecCalc, Execution |
|            GameplayTag             |       Tag, GT       |
|    ModiferMagnitudeCalculation     |   ModMagCalc, MMC   |