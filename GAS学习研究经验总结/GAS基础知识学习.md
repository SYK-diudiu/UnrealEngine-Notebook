# GAS基础知识学习

[TOC]

## 前言

本文大部分内容涉及GAS基础知识，包括但不限于以下部分：`Ability System Component`、`Attribute Set`、`Gameplay Ability`、`Ability Task`、`Gameplay Effect`、`Gameplay Cue`以及`Gameplay Tag`。

## Ability System Component

## Attribute Set

在GAS框架中`UAttributeSet`就是我们说的**Attribute Set**（即：属性集），我会写一个`UAttributeSet`的派生类来定义我自己的属性集，也就是往里面添加我需要的属性，比如角色的生命值、魔法值、攻击力等。

### **FGameplayAttributeData**

`FGameplayAttributeData` 是一个专门用于定义游戏属性（如血量、法力值、攻击力等）的结构体。它被设计用来与 GAS 的核心功能深度集成，支持**网络同步**、**属性修改**（如加减乘除）和**事件触发**。

下面是该结构体的源码定义

```cpp
USTRUCT(BlueprintType)
struct GAMEPLAYABILITIES_API FGameplayAttributeData
{
	GENERATED_BODY()
	FGameplayAttributeData()
		: BaseValue(0.f)
		, CurrentValue(0.f)
	{}

	FGameplayAttributeData(float DefaultValue)
		: BaseValue(DefaultValue)
		, CurrentValue(DefaultValue)
	{}

	virtual ~FGameplayAttributeData()
	{}

	/** Returns the current value, which includes temporary buffs */
	float GetCurrentValue() const;

	/** Modifies current value, normally only called by ability system or during initialization */
	virtual void SetCurrentValue(float NewValue);

	/** Returns the base value which only includes permanent changes */
	float GetBaseValue() const;

	/** Modifies the permanent base value, normally only called by ability system or during initialization */
	virtual void SetBaseValue(float NewValue);

protected:
	UPROPERTY(BlueprintReadOnly, Category = "Attribute")
	float BaseValue;

	UPROPERTY(BlueprintReadOnly, Category = "Attribute")
	float CurrentValue;
};
```

- **`BaseValue`**：属性的基准值，通常由角色初始值或装备提供。
- **`CurrentValue`**：实际生效的值，会受到 `GameplayEffect` 临时修改。

#### 属性修改

通过 `GameplayEffect` 动态修改 `CurrentValue`，而 `BaseValue` 保持不变：

```cpp
// 例如：一个治疗效果将 CurrentValue 增加 20
Health.CurrentValue += 20;
```

#### 网络同步

配合 `AttributeSet` 使用，自动同步到客户端。

需在 `GetLifetimeReplicatedProps` 中注册：

```cpp
DOREPLIFETIME_CONDITION_NOTIFY(UAuraAttributeSet, Health, COND_None, REPNOTIFY_Always);
```

#### 与GameplayEffect交互

`GameplayEffect` 可以通过 `Modifiers` 修改 `FGameplayAttributeData`：

```cpp
//一个效果永久增加最大生命值（BaseValue）
EffectSpec.Modifiers.Add(FScalableFloat(10.f), EGameplayModOp::Additive, Health);
```

#### 常用操作

##### **获取属性值**

```cpp
float CurrentHealth = Health.GetCurrentValue();
float MaxHealth = Health.GetBaseValue(); // 假设BaseValue为最大值
```

##### **修改属性**

```cpp
// 直接修改（需在服务器端执行）
Health.SetCurrentValue(50.f);
Health.SetBaseValue(100.f);

// 通过GameplayEffect（我比较推荐使用）
UAbilitySystemComponent* ASC = GetAbilitySystemComponent();
ASC->ApplyModToAttribute(Health, EGameplayModOp::Additive, 20.f);
```

##### **响应属性变化**

绑定委托监听变化：

```cpp
AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(HealthAttribute)
    .AddUObject(this, &UAuraAttributeSet::OnHealthChanged);
```

### **如何添加属性**

在**Attribute Set**中添加属性并不是写一个`int a`或者`float b`那样简单。我们需要用到统一的数据类型，即`FGameplayAttributeData`。比如添加角色生命值`Health`，那我们就需要在属性集的头文件中写`FGameplayAttributeData Health`，这就算是添加了一个属性。但是只有这段代码还远远不够，在我们开发游戏时需要考虑到网络层面代码编写。即使我们的游戏当前是一个单机游戏，但在开发过程中我们也要时刻为今后可能的多人游戏编写可扩展的代码，也就是网络复制相关内容，那对于**Attribute Set**中的属性，自然需要被考虑在内，在多人游戏中我们需要服务器与客户端的属性信息保持同步。

有了以上网络层面的考虑之后，我们添加一个属性就不单单是写一段`FGameplayAttributeData Health`可以解决的了，还需要写该属性的网络复制功能。

这部分功能在我的文章《虚幻引擎网络复制》中有讲到，简单概括以下就是我们需要做三件事：**声明属性变量**、**声明并定义属性的回调函数**，**注册属性到复制系统**。

#### 声明属性变量

在我们自己的属性集头文件中，编写如下代码。

```cpp
UPROPERTY(ReplicatedUsing = OnRep_Health)
FGameplayAttributeData Health;
```

其中`ReplicatedUsing = OnRep_Health`表示这个属性在网络复制过程中使用的回调函数的函数名。

如果需要添加其它功能，比如蓝图可读可写、分类什么的，可以这样写：

```cpp
UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "xxx")
FGameplayAttributeData Health;
```

#### 声明并定义属性的回调函数

在声明属性变量的时候我们就看到了`ReplicatedUsing = OnRep_Health`，现在就是声明这个函数的时候。

我们在头文件和源文件创建它的声明和定义。

```cpp
//头文件
UFUNCTION()
void OnRep_Health(const FGameplayAttributeData& OldHealth) const;

//源文件
void UCustomAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth) const
{
	//GAS属性在网络复制中用到的宏
	GAMEPLAYATTRIBUTE_REPNOTIFY(UCustomAttributeSet, Health, OldHealth);
}
```

这个回调函数可以用于处理一些属性变化后需要执行的逻辑，比如血量减少，进而播放流血特效、音效等，虽然我不会这么写，但是它的确可以这样用，我们清楚这个函数在代码中是一个怎样的地位即可。

##### 关于函数参数为什么要这样写

首先我们需要使用`GAMEPLAYATTRIBUTE_REPNOTIFY`宏，而宏需要旧值。

其次在客户端接收到服务器发来的新的数据之后，引擎会保存原来的`Health`旧值，旧值就会以函数参数的形式传递进来。

#### 注册属性到复制系统

在我的网络复制文章中有提到，属性网络复制必须重写一个函数`GetLifetimeReplicatedProps`具体代码如下所示。

```cpp
//头文件
virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

//源文件
void UCustomAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	//注册属性的代码
	DOREPLIFETIME_CONDITION_NOTIFY(UCustomAttributeSet, Health, COND_None, REPNOTIFY_Always);
}
```

我们需要像代码中所示那样，将`Health`属性注册到网络复制的复制系统中。

其中使用到了一个宏：`DOREPLIFETIME_CONDITION_NOTIFY`

这个宏用于精细控制属性的网络复制行为，它扩展了标准 `DOREPLIFETIME` 的功能，增加了 **复制条件（Condition）** 和 **通知策略（Notify Policy）** 的支持。

具体作用就是：

1. **声明 `Health` 属性需要网络复制**。
2. **指定复制条件**，在代码中就是`COND_None`所在位置。
3. **定义通知策略**，在代码中就是`REPNOTIFY_Always`所在位置。

#### 总结

以上过程就是我们在GAS的`Attribute Set`中添加一个属性的代码，这会形成一个样板，新添加的属性只需要在原来代码的基础上修改属性名并复制粘贴即可，因此我们应该熟悉这其中的代码样板，也就是上面讲到的三个步骤。

**完整代码**

```cpp
//头文件
#pragma once

#include "CoreMinimal.h"
#include "AttributeSet.h"
#include "CustomAttributeSet.generated.h"

UCLASS()
class XXX_API UCustomAttributeSet : public UAttributeSet
{
	GENERATED_BODY()

public:

	UCustomAttributeSet();

	virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

	UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Vital Attribute")
	FGameplayAttributeData Health;

	UFUNCTION()
	void OnRep_Health(const FGameplayAttributeData& OldHealth) const;
};

```

```cpp
//源文件

#include "AbilitySystem/CustomAttributeSet.h"
#include "AbilitySystemComponent.h"
#include "Net/UnrealNetwork.h"

UCustomAttributeSet::UCustomAttributeSet()
{
}

void UCustomAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	//注册属性
	DOREPLIFETIME_CONDITION_NOTIFY(UCustomAttributeSet, Health, COND_None, REPNOTIFY_Always);
}

void UCustomAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth) const
{
	//GAS属性在网络复制中用到的宏
	GAMEPLAYATTRIBUTE_REPNOTIFY(UCustomAttributeSet, Health, OldHealth);
}
```



## Gameplay Ability

## Ability Task

## Gameplay Effect

## Gameplay Cue

## Gameplay Tag