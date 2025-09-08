[TOC]

# GAS经验积累

# GAS自制轮子

## 输出AttributeSet所有属性值

### 使用须知

如果你不知道其它快速查看角色属性的方法并准备使用该轮子，我推荐你在蓝图中或运行游戏之后运行下图所示节点或节点中的命令。

![命令行命令](GAS经验积累.assets/命令行命令.png)

上图所示命令只能查看玩家当前控制的角色的属性等其它`ASC`信息，如果你希望方便的查看玩家没有控制的角色属性信息，可以尝试使用本轮子。

本轮子可以在`GAS`框架中输出`ASC`下指定`AttributeSet`的所有属性值，包括属性名、BaseValue、CurrentValue。

建议了解`FGameplayAttribute`、`FGameplayAttributeData`的结构与基本关系后使用。

### 代码

代码所示函数编写在了`Character`中，这是因为`ASC`选择添加到`Character`。如果你的`ASC`在`PlayerState`中那么代码所示函数应当编写在`PlayerState`中，总之就是在`ASC`所在`Actor`处添加该函数即可。

#### 函数声明：

```C++
// 输出ASC下AttributeSet所有属性值（包括属性名、BaseValue、CurrentValue）
// 参数解释：
// 1、Src：AttributeSet指针，决定输出哪一个属性集的属性值
// 2、TimeToDisplay：调式信息的输出时间
// 3、DisplayColor：调试信息输出颜色
// 使用方式：
// 法一：子类可以封装或重写该函数暴露到蓝图使用
// 法二：直接在蓝图中使用该函数，但是需要使用ASC获取AttributeSet并传参到本函数
UFUNCTION(BlueprintCallable, Category = "Print")
virtual void PrintAS(UAttributeSet* Src , float TimeToDisplay = 5.0f , FColor DisplayColor = FColor::Yellow);
```

#### 函数定义

```C++
void ACSCharacterBase::PrintAS(UAttributeSet* Src , float TimeToDisplay, FColor DisplayColor)
{
	// 检测ASC
	if (ASC == nullptr)
	{
		UE_LOG(LogTemp, Warning, TEXT("PrintAS函数：ASC为空"));
		return;
	}

	// 检测Src
	if (Src == nullptr)
	{
		UE_LOG(LogTemp, Warning, TEXT("PrintAS函数：Src为空"));
		return;
	}

	// 获取ASC下所有属性
	TArray<FGameplayAttribute> PrintAttributeArray;
	ASC->GetAllAttributes(PrintAttributeArray);

	// 输出属性值的循环语句
	for (int32 index = 0 ; index < PrintAttributeArray.Num() ; ++index)
	{
		// 过滤非Src的属性
		FGameplayAttributeData* TempData = PrintAttributeArray[index].GetGameplayAttributeData(Src);

		// 如果属性为空则跳出循环
		if (TempData == nullptr)
		{
			break;
		}

		// 编写输出属性名、BaseValue、CurrentValue格式字符串
		FString PrintStr = FString::Printf(
			TEXT("%s-----BaseValue: %f | CurrentValue: %f"), 
			*PrintAttributeArray[index].AttributeName , 
			TempData->GetBaseValue(), 
			TempData->GetCurrentValue()
		);

		// 打印调试信息到屏幕
		GEngine->AddOnScreenDebugMessage(index, 5.0, FColor::Yellow, PrintStr);
	}
}
```

