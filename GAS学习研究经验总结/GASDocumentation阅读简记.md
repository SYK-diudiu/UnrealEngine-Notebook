#  GASDocumentation阅读简记

## 前言：

​	本文内容多半为阅读`Github`上的`GASDocumentation_Chinese`文章时对概念、技术的解读和联想，随读随记，笔记结构与文章结构相同。

# 4. GAS概念

## 4.1 ASC

#### `OwnerActor`和`AvatarActor`

这里有一个难点就是对`ASC`中`OwnerActor`和`AvatarActor`的理解。

- `OwnerActor`是`ASC`的 "逻辑所有者"（通常是玩家状态`PlayerState` 、 玩家控制器`PlayerController`或 AI 控制器`AIController`），负责持久化保存能力数据（如技能配置、属性基础值）；
- `AvatarActor`是 "表现载体"，随角色实体的创建 / 销毁而变化（如角色死亡后销毁，重生时重新创建）。

这种分离设计让能力系统更灵活：即使`AvatarActor`销毁（如角色死亡），`OwnerActor`仍能保留`ASC`的核心数据，待新的`AvatarActor`生成后重新关联。

##### 示例场景：第三人称动作游戏中的玩家角色

假设我们有一个典型的第三人称游戏：

- 玩家通过`PlayerController`（玩家控制器）控制一个名为`HeroCharacter`的角色；
- 角色可以释放技能、受伤害、死亡后重生；
- 能力系统（`ASC`）需要处理技能逻辑、属性计算（如血量、攻击力）等。

##### 1. `OwnerActor`：逻辑所有者（通常是`PlayerController`）

`OwnerActor`是`ASC`的 "逻辑归属者"，负责**持久化保存能力系统的核心数据**，不随角色的生死而消失。

**具体作用：**

- **保存永久数据**：玩家解锁的技能、属性基础值（如初始血量）、天赋配置等会存在`OwnerActor`中。即使角色死亡，这些数据也不会丢失。
- **作为输入源头**：玩家按下技能键时，`PlayerController`（`OwnerActor`）会接收输入，然后通知`ASC`激活对应的技能。
- **维持能力系统存续**：`OwnerActor`通常与玩家的游戏进程绑定（如从进入游戏到退出游戏），`ASC`会随`OwnerActor`的生命周期存在，确保能力系统不会中途失效。

**举例：**
当玩家的`HeroCharacter`被击败（`AvatarActor`销毁）时，`PlayerController`（`OwnerActor`）依然保留着玩家的所有技能数据。当角色重生时，`ASC`只需重新关联新的`AvatarActor`，就能立即恢复所有技能功能。

##### 2. `AvatarActor`：表现载体（通常是`HeroCharacter`）

`AvatarActor`是`ASC`的 "物理表现者"，负责**将能力系统的逻辑结果具象化呈现**，是玩家在游戏世界中看到的实体。

**具体作用：**

- **执行表现逻辑**：技能释放时的动画（如挥剑动作）、特效（如火焰粒子）、碰撞检测（如技能攻击范围）都通过`AvatarActor`实现。
- **承载临时状态**：角色当前的位置、速度、受击后的硬直状态等临时数据由`AvatarActor`管理。例如，`ASC`施加 "减速" 效果时，实际是修改`AvatarActor`移动组件的速度。
- **作为交互对象**：其他角色（如敌人）的攻击、场景中的陷阱等，都是与`AvatarActor`发生物理交互，然后通过`AvatarActor`上的`ASC`处理伤害逻辑。

**举例：**
当玩家释放 "火球术" 时：

1. `OwnerActor`（`PlayerController`）接收输入，通知`ASC`激活火球术能力；
2. `ASC`计算技能消耗（如魔力值）和伤害后，通过`GetAvatarActor()`获取`HeroCharacter`；
3. `AvatarActor`（`HeroCharacter`）播放施法动画，并在手部位置生成火球特效；
4. 火球的碰撞检测基于`AvatarActor`的位置计算，命中敌人后通过`ASC`施加伤害效果。

##### 3. 两者的协作关系

- **数据流向**：`OwnerActor`保存的永久数据（如技能配置）会同步到`ASC`，`ASC`再根据这些数据控制`AvatarActor`的表现。
- **生命周期配合**：`AvatarActor`可能频繁创建 / 销毁（如角色死亡重生），但`OwnerActor`和`ASC`保持稳定，确保能力系统的逻辑连续性。
- **职责分离**：`OwnerActor`专注于 "谁在控制" 和 "有哪些能力"，`AvatarActor`专注于 "如何表现" 和 "与世界交互"。

通过这个例子可以看出：`OwnerActor`是能力系统的 **"大脑"**，负责决策和记忆；`AvatarActor`是 **"身体"**，负责执行和呈现。这种分离设计让 GAS 在处理角色切换、重生、多人游戏等场景时更加灵活。

#### ASC中GA和GE的保存位置

`ASC`在`FActiveGameplayEffectContainer ActiveGameplayEffect`中保存其当前活跃的`GameplayEffect`.  

`ASC`在`FGameplayAbilitySpecContainer ActivatableAbility`中保存其授予的`GameplayAbility`.

### 4.1.1 网络同步

注意`MixedMode`的启用方法。

### 4.1.2 设置和初始化

#### 什么位置开始创建ASC？

`ASC`一般在`OwnerActor`的构建函数中创建并且需要明确标记为`Replicated`. **这必须在C++中完成.** 

比如说我们需要`PlayerState`作为`OwnerActor`，那么就建议在`PlayerState`的构造函数添加如下代码：

```cpp
AGDPlayerState::AGDPlayerState()
{
	// 创建能力系统组件，并将其设置为显式复制。
	AbilitySystemComponent = CreateDefaultSubobject<UGDAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
	AbilitySystemComponent->SetIsReplicated(true); //设置为显式复制的函数
	//...
}
```

#### ASC初始化位置和初始化时机

- **初始化位置**：`OwnerActor`和`AvatarActor`的`ASC`在**服务端和客户端上均需初始化**。
- **初始化时机**：你应该在`Pawn`的`Controller`设置之后初始化(Possess之后), 单人游戏只需参考服务端的做法。
- 关于`ASC`的初始化，其实就是设置`ASC`的`OwnerActor`和`AvatarActor`，使用`InitAbilityActorInfo(OwnerActor* , AvatarActor*)`函数并设置对应函数参数即可完成初始化，通常将`PlayerState`作为`OwnerActor`，将`ACharacter`作为`AvatarActor`，更重要的是初始化的**位置**和**时机**需要仔细考虑。

上述内容**强调ASC 初始化必须满足两个前提**：一是服务端和客户端都要分别初始化，二是初始化操作必须在`Controller`成功 “控制”`Pawn`（即`Possess`操作完成）之后。

##### 先理解几个关键概念：

- **`Possess`操作**：在虚幻引擎中，`Controller`（如玩家控制器`PlayerController`、AI 控制器`AAIController`）通过`Possess(APawn* InPawn)`函数获得对`Pawn`（如角色`ACharacter`）的控制权。这个过程会建立`Controller`与`Pawn`的绑定关系（`Pawn->GetController()`从此能正确返回对应的`Controller`）。
- **服务端与客户端的独立性**：在联网游戏中，服务端（权威逻辑）和客户端（本地表现）是两个独立的运行环境，很多对象（如`ASC`、`Controller`、`Pawn`）在两端都有各自的实例，需要分别初始化才能保证逻辑一致。

##### 逐句解释：

###### 1. “`OwnerActor`和`AvatarActor`的`ASC`在服务端和客户端上均需初始化”

- **`ASC`的网络特性**：`ASC`是 GAS 的核心组件，它既需要在服务端处理权威逻辑（如技能是否允许激活、伤害计算、属性修改等），也需要在客户端处理本地预测（如技能输入响应、特效动画播放等）。因此，服务端和客户端的`ASC`实例需要分别初始化，否则会出现 “服务端有逻辑但客户端没表现” 或 “客户端操作不被服务端认可” 的问题。
- **`OwnerActor`和`AvatarActor`的两端一致性**：`OwnerActor`（通常是`Controller`）和`AvatarActor`（通常是`Pawn`）在服务端和客户端都有对应的实例（如客户端的`PlayerController`和服务端的`PlayerController`是不同对象，但通过网络同步关联）。`ASC`初始化时需要在两端分别绑定对应的`OwnerActor`和`AvatarActor`，才能保证两端的能力系统都能正确找到 “逻辑所有者” 和 “表现载体”。

###### 2. “你应该在`Pawn`的`Controller`设置之后初始化（`Possess`之后）”

- **`Possess`是`Controller`与`Pawn`绑定的标志**：在`Possess`之前，`Pawn`的`Controller`可能尚未设置（`Pawn->GetController()`返回`null`），或者绑定的是临时 / 错误的`Controller`。而`ASC`初始化的核心函数`InitAbilityActorInfo(Owner, Avatar)`需要明确的`OwnerActor`（通常是`Controller`）和`AvatarActor`（通常是`Pawn`），如果`Controller`还没绑定`Pawn`，会导致：
  - `OwnerActor`参数（`Controller`）可能为`null`，`ASC`无法关联到正确的逻辑所有者，后续输入处理（如技能按键响应）会失效；
  - `AvatarActor`（`Pawn`）与`OwnerActor`（`Controller`）的绑定关系未稳定，可能导致`ASC`后续获取`Owner`或`Avatar`时出现错误（如角色死亡重生后`Controller`重新绑定，但`ASC`未更新）。
- **举例说明**：
  玩家角色（`Pawn`）生成后，`PlayerController`需要先通过`Possess`获得控制权（此时`Pawn->Controller`被正确设置为该`PlayerController`）。之后再调用`ASC->InitAbilityActorInfo(PC, Pawn)`，才能确保`ASC`的`OwnerActor`是正确的`PlayerController`，`AvatarActor`是被控制的`Pawn`。如果在`Possess`之前初始化，`PC`可能还未绑定`Pawn`，`ASC`会因`OwnerActor`无效而无法正常工作（如技能无法被输入触发）。

#### 针对不同情况的ASC初始化

##### 对于玩家控制的`Character`且`ASC`位于`Pawn`

这种情况就是说`ASC`中`OwnerActor`和`AvatarActor`都将会是`Pawn`自身（`Character`也是`Pawn`），没有分离开来，那么这时候就在`Pawn`的构造函数中创建`ASC`（因为`OwnerActor`是`Pawn`）之后一般在服务端`Pawn`的`PossessedBy()`函数中初始化, 在客户端`PlayerController`的`AcknowledgePossession()`函数中初始化。

```c++
void APACharacterBase::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	if (AbilitySystemComponent)
	{
		AbilitySystemComponent->InitAbilityActorInfo(this, this);
	}

	// ASC混合模式复制要求ASC所有者的所有者是控制器。
	SetOwner(NewController);
}
```

```C++
void APAPlayerControllerBase::AcknowledgePossession(APawn* P)
{
	Super::AcknowledgePossession(P);

	APACharacterBase* CharacterBase = Cast<APACharacterBase>(P);
	if (CharacterBase)
	{
		CharacterBase->GetAbilitySystemComponent()->InitAbilityActorInfo(CharacterBase, CharacterBase);
	}

	//...
}
```

##### 对于玩家控制的`Character`且`ASC`位于`PlayerState`

这种情况就是说`ASC`的`OwnerActor`和`AvatarActor`是不同的对象，其中`PlayerState`是`OwnerActor`，而`Pawn`是`AcatarActor`。此时一般在服务端`Pawn`的`PossessedBy()`函数中初始化，在客户端`PlayerController`的`OnRep_PlayerState()`函数中初始化, 这确保了`PlayerState`存在于客户端上。

```C++
void AGDHeroCharacter::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		// Set the ASC on the Server. Clients do this in OnRep_PlayerState()
        // 在服务器上设置ASC。客户端在OnRep_PlayerState()中执行此操作。
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// AI won't have PlayerControllers so we can init again here just to be sure. No harm in initing twice for heroes that have PlayerControllers.
        // 人工智能不会有玩家控制器，所以我们可以在这里再次初始化以确保安全。对于拥有玩家控制器的英雄来说，初始化两次也没有坏处。
		PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
	}
	
	//...
}
```

```c++
void AGDHeroCharacter::OnRep_PlayerState()
{
	Super::OnRep_PlayerState();

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		// Set the ASC for clients. Server does this in PossessedBy.
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// Init ASC Actor Info for clients. Server will init its `ASC` when it possesses a new Actor.
		AbilitySystemComponent->InitAbilityActorInfo(PS, this);
	}

	// ...
}
```



## 4.2 GameplayTags

### FGameplayTag::RequestGameplayTag()函数

`FGameplayTag::RequestGameplayTag(FName("Your.GameplayTag.Name"))` 是虚幻引擎中获取 `GameplayTag` 实例的核心方法，其主要作用是**根据指定的标签字符串（如 `"Your.GameplayTag.Name"`），从引擎的全局标签库中查询并返回对应的 `FGameplayTag` 对象**。

#### 具体解析：

1. **`GameplayTag` 的本质**
   `GameplayTag` 是一种结构化的字符串标识（格式通常为 `Group.SubGroup.Tag`，如 `"Ability.Skill.Fireball"` 或 `"Status.Burning"`），用于标记游戏中的状态、能力、交互条件等。但在代码中，它并非直接以字符串形式使用，而是通过 `FGameplayTag` 结构体的实例来表示，这个实例会与引擎内部的标签系统关联。
2. **函数的核心功能**
   - **查询标签**：当你调用 `RequestGameplayTag` 并传入一个标签字符串（包装为 `FName` 类型）时，它会去引擎维护的**全局标签注册表**中查找该字符串对应的 `FGameplayTag` 实例。
   - **返回结果**：如果找到匹配的标签，返回有效的 `FGameplayTag` 对象；如果未找到（例如标签未在项目中定义），则返回一个 “无效标签”（可通过 `IsValid()` 方法判断）。
3. **为什么需要这个方法？**
   - `GameplayTag` 必须先在项目中定义（通常在 `.uasset` 标签表中配置），才能被代码使用。`RequestGameplayTag` 确保你使用的标签是 “已注册” 的，避免拼写错误或使用未定义的标签导致逻辑错误。
   - 它提供了类型安全的标签访问方式，相比直接使用字符串，`FGameplayTag` 实例可以更高效地参与比较、查询等操作（例如在 `FGameplayTagContainer` 中检查是否包含某个标签）。

#### 使用场景举例：

##### （1）检查角色是否有某个状态标签

```cpp
// 获取"燃烧"状态标签的实例
FGameplayTag BurningTag = FGameplayTag::RequestGameplayTag(FName("Status.Burning"));

// 从角色的标签容器中检查是否包含该标签
if (Character->GetGameplayTagContainer().HasTag(BurningTag))
{
    // 执行燃烧相关逻辑（如每秒掉血）
}
```

##### （2）激活技能时指定标签条件

```cpp
// 获取"技能可用"标签
FGameplayTag AbilityReadyTag = FGameplayTag::RequestGameplayTag(FName("Ability.Ready.Fireball"));

// 当满足标签条件时激活技能
if (ASC->HasMatchingGameplayTag(AbilityReadyTag))
{
    ASC->TryActivateAbilityByTag(AbilityReadyTag);
}
```

#### 注意事项：

- **标签必须预先定义**：如果传入的标签字符串未在项目的标签表中定义，`RequestGameplayTag` 会返回无效标签，可能导致逻辑失效。可以在编辑器的 `Edit > Project Settings > Gameplay Tags` 中管理标签。
- **性能考虑**：`RequestGameplayTag` 内部会进行哈希表查询，建议在初始化时（如 `BeginPlay`）提前获取标签实例并缓存，避免在高频逻辑（如 `Tick`）中重复调用。

#### 总结：

`FGameplayTag::RequestGameplayTag` 是连接 **“标签字符串” 和 “代码中可用的标签实例” 的桥梁**，因为不可能说是我们拿着标签字符串在代码中使用，因为代码是不能直接识别你的标签字符串的，所以我们就需要让这个标签字符串在编写代码时具备一个可以指代、可以像变量一样拿来拿去使用的这么一个载体，这就是所谓的的标签实例，也就是`FGameplayTag::RequestGameplayTag`函数返回的对象。该标签实例确保你能安全、高效地使用预定义的 `GameplayTag` 进行逻辑判断和交互。

### FGameplayTagContainer类的核心功能

`FGameplayTagContainer` 是虚幻引擎 GAS（Gameplay Ability System）模块中用于管理 `GameplayTag` 的核心容器类，它的主要作用是**集中存储和管理多个不重复的 `GameplayTag`，提供高效的标签查询、添加、移除等操作**。

#### 核心功能与用途：

1. **存储不重复的 `GameplayTag`**
   `FGameplayTagContainer` 本质是一个 “集合”（类似数学中的集合概念），它确保同一个 `GameplayTag` 在容器中只会存在一次（不会重复存储）。
   例如，容器中添加两次 `"Status.Burning"` 标签，最终只会保留一个实例。
2. **高效的标签查询与判断**
   它提供了便捷的接口来检查容器中是否包含某个标签，或是否包含某类标签（通过标签的层级关系）：
   - 检查是否包含特定标签：`HasTag(FGameplayTag::RequestGameplayTag(FName("Status.Burning")))`
   - 检查是否包含某类标签（如所有 `Status` 开头的标签）：`HasAnyTagMatching(FGameplayTagContainer(FGameplayTag::RequestGameplayTag(FName("Status"))))`
3. **作为标签的 “载体” 传递信息**
   在 GAS 中，`FGameplayTagContainer` 常被用作数据载体，在不同系统间传递标签信息：
   - 技能激活条件：某个技能可能要求目标容器中必须包含 `"State.Alive"` 标签才能生效；
   - 效果过滤：施加 `GameplayEffect` 时，通过容器指定 “哪些标签的目标会被影响”；
   - 状态同步：角色的状态（如 “中毒”“隐身”）可以通过容器集中管理，方便其他系统查询。
4. **与其他 GAS 组件协同工作**
   它是 GAS 中多个核心类的基础依赖：
   - `GameplayAbility`：通过容器定义技能的激活标签、阻塞标签（如 “死亡状态” 下无法激活技能）；
   - `GameplayEffect`：通过容器指定效果的应用条件或目标标签；
   - `AttributeSet`：结合标签容器记录属性相关的状态（如 “是否处于防御状态”）。

#### 举例说明：

假设一个角色有以下状态标签：`"Status.Burning"`（燃烧）、`"Status.Poisoned"`（中毒）、`"State.Invincible"`（无敌）。
这些标签会被存储在角色的 `FGameplayTagContainer` 中：

- 当角色受到攻击时，攻击逻辑会检查容器是否包含 `"State.Invincible"`，如果包含则免疫伤害；
- 当角色使用 “解控技能” 时，技能逻辑会从容器中移除 `"Status.Burning"` 和 `"Status.Poisoned"`；
- UI 系统会查询容器，显示当前角色身上的所有状态图标（燃烧、中毒等）。

### FGameplayTagCountContainer类的核心功能

它是一个用于存储`GameplayTag`（游戏标签）并记录每个标签实例数量的容器，其中的`TagMap`负责具体保存这些标签及其对应的计数。

#### 逐部分解析：

1. **`FGameplayTagCountContainer`**
   这是虚幻引擎中 GAS（Gameplay Ability System）模块提供的一个容器类，专门用于管理`GameplayTag`的 “数量”。它不仅能存储哪些标签存在，还能记录每个标签出现了多少次。
2. **保存于其中的`GameplayTag`**
   `GameplayTag`是 GAS 中用于标记游戏状态、行为或属性的字符串标识（格式通常为`"Group.SubGroup.Tag"`，例如`"Status.Burning"`表示 “燃烧状态”）。`FGameplayTagCountContainer`的核心作用就是存储这些标签。
3. **`TagMap`的作用**
   `TagMap`是`FGameplayTagCountContainer`内部的一个数据结构（通常是哈希表），它的关键功能是：
   - **关联`GameplayTag`与其实例数**：每个`GameplayTag`作为键（Key），对应的数值（Value）是这个标签的 “实例数量”（即该标签被添加了多少次）。
   - **高效管理计数**：支持对标签进行 “添加”（计数 + 1）、“移除”（计数 - 1）、“清零” 等操作，当一个标签的计数减到 0 时，通常会从`TagMap`中移除。

#### 举例说明：

假设在游戏中，一个角色可能同时被多个 “减速” 效果影响：

- 当角色被第一个减速技能命中时，向`FGameplayTagCountContainer`添加`"Status.Slowed"`标签，`TagMap`中该标签的计数变为`1`。
- 紧接着被第二个减速技能命中，再次添加该标签，计数变为`2`。
- 当第一个减速效果结束时，移除一次该标签，计数减为`1`。
- 当第二个减速效果结束时，再次移除，计数变为`0`，此时`"Status.Slowed"`标签从`TagMap`中移除。

通过`TagMap`，系统可以快速知道：当前角色有多少个同类标签（如多少个减速效果），从而决定是否执行对应的逻辑（如只要计数≥1，就应用减速）。

#### 总结：

`FGameplayTagCountContainer`通过内部的`TagMap`，实现了对`GameplayTag`的 “数量化管理”—— 不仅记录哪些标签存在，还精确跟踪每个标签的实例数量，这在处理叠加效果（如多层减速、多段伤害）等场景中非常实用。

### `FGameplayTagContainer`与 `FGameplayTagCountContainer` 的区别：

- `FGameplayTagContainer` 只关心标签 “是否存在”（不记录数量），适合管理 “非叠加” 状态（如 “无敌” 要么存在，要么不存在）；
- `FGameplayTagCountContainer` 会记录标签的 “实例数量”（如叠加 3 层减速），适合管理 “可叠加” 状态。

### 4.2.1 响应Gameplay Tags的变化

关于文章中该部分的内容我们重点关注`AGDPlayerState::StunTagChanged`函数的触发时机以及函数参数的来源以及作用即可。

#### 代码作用详解：

1. **注册标签事件监听**
   `AbilitySystemComponent->RegisterGameplayTagEvent(...)` 是`ASC`提供的注册方法，用于监听指定`GameplayTag`的变化。
   - 第一个参数：指定要监听的标签（这里是`"State.Debuff.Stun"`，通过`RequestGameplayTag`获取其实例）。
   - 第二个参数：`EGameplayTagEventType::NewOrRemoved` 表示监听的事件类型 —— 当标签 “被添加”“被移除” 或 “计数变化”（如叠加层数改变）时，都会触发回调。
   - `.AddUObject(this, &AGDPlayerState::StunTagChanged)`：将回调函数`StunTagChanged`绑定到这个事件上，当事件触发时，`AGDPlayerState`类的`StunTagChanged`方法会被自动调用。
2. **回调函数的作用**
   `virtual void StunTagChanged(const FGameplayTag CallbackTag, int32 NewCount)` 是事件触发时的处理函数，用于响应 “眩晕标签” 的变化：
   - 当角色被施加眩晕效果（标签被添加，`NewCount`从 0 变为 1），可以在此处处理眩晕逻辑（如禁用移动、播放眩晕动画）。
   - 当眩晕效果结束（标签被移除，`NewCount`从 1 变为 0），可以在此处恢复角色状态（如启用移动、播放恢复动画）。
   - 若支持眩晕效果叠加（如多层眩晕延长持续时间），`NewCount`的变化（如从 1 变为 2）也能在此处被捕获并处理。

#### 为什么回调函数需要这两个参数？

这两个参数是委托（Delegate）传递的**负载参数**，用于向回调函数提供 “事件相关的关键信息”，确保回调逻辑能准确处理具体情况：

1. **`const FGameplayTag CallbackTag`**
   - 作用：明确告知回调函数 “哪个标签发生了变化”。
   - 必要性：在实际开发中，一个回调函数可能监听多个标签（或同一类标签的不同子标签），通过该参数可以区分具体是哪个标签的事件。例如，若同时监听`"State.Debuff.Stun"`和`"State.Debuff.Silence"`，`CallbackTag`能告诉函数是 “眩晕” 还是 “沉默” 发生了变化。
2. **`int32 NewCount`**
   - 作用：表示该标签当前的计数（即`TagMapCount`的最新值）。
   - 必要性：标签可能存在 “叠加” 情况（如多层减速、多段眩晕），`NewCount`能反映当前标签的活跃数量：
     - `NewCount > 0`：标签处于 “活跃” 状态（如`NewCount=2`表示有 2 个眩晕效果同时生效）。
     - `NewCount = 0`：标签已被完全移除（所有眩晕效果都已结束）。
       回调函数可以根据`NewCount`的数值判断是否需要启用 / 禁用对应逻辑（如只要`NewCount≥1`就保持眩晕状态）。

#### 注意：

`virtual void StunTagChanged(const FGameplayTag CallbackTag, int32 NewCount)`函数中的两个参数必须写出来，不可以省略，因为这是由`AbilitySystemComponent->RegisterGameplayTagEvent(...)` 注册函数返回的委托的定义决定，类似这样的委托定义`DECLARE_DELEGATE_TwoParams(FGameplayTagDelegate, const FGameplayTag&, int32);`明确了与该委托绑定的函数必须有两个参数，其类型分别为`const FGameplayTag&`和`int32`。

#### 总结：

- 这段代码通过`ASC`的委托机制，实现了对 “眩晕标签” 变化的监听，确保标签状态改变时能自动触发处理逻辑。
- 回调函数的两个参数`CallbackTag`和`NewCount`是委托传递的负载参数，分别用于标识 “哪个标签变化” 和 “变化后的计数”，为回调逻辑提供了必要的上下文信息，使其能准确处理不同的标签状态场景。

## 4.3 Attribute

### 4.3.1 Attribute定义

### 4.3.2 BaseValue 和 CurrentValue

### 4.3.3 Meta（元）Attribute

**元属性（Meta Attribute）** 是一种特殊类型的`Attribute`（属性），其核心特点是**不直接存储数值，而是通过计算其他 “基础属性” 的数值动态生成结果**。它本质是一个 “计算型属性”，用于派生、组合或转换基础属性的数值，而非独立维护自身状态。

#### 1. 核心特征：“计算依赖” 而非 “独立存储”

要理解元属性，首先需要区分它与 “基础属性（Base Attribute）” 的差异：

- **基础属性**：如`Health`（生命值）、`MaxHealth`（最大生命值）、`AttackDamage`（攻击力），这类属性有明确的 “存储值”，数值变化（如受伤减少`Health`、升级增加`MaxHealth`）会直接修改其内部存储的变量。
- **元属性**：如`HealthPercent`（生命值百分比）、`EffectiveDamage`（最终有效伤害，受攻击力 + 暴击倍率影响），这类属性**没有独立存储的数值**，其值永远是通过预设的 “计算逻辑” 从基础属性中实时推导而来（例如`HealthPercent = Health / MaxHealth`）。

#### 2. 为什么需要元属性？

元属性的设计核心是为了解决 “属性数值联动” 和 “逻辑复用” 的问题，典型场景包括：

- **简化百分比 / 比例计算**：无需手动每次计算 “当前生命值占比”，直接通过`HealthPercent`元属性获取，避免重复代码。
- **整合多属性影响**：例如 “最终伤害” 可能受`BaseAttack`（基础攻击力）、`AttackMultiplier`（攻击倍率）、`DamageReduction`（敌方减伤）等多个基础属性影响，用元属性`EffectiveDamage`封装计算逻辑，其他系统（如伤害结算）只需直接读取元属性结果，无需关心内部计算细节。
- **保证数值一致性**：元属性的数值完全依赖基础属性，只要基础属性正确，元属性就一定正确，不会出现 “基础属性变了但派生值没同步” 的 bug（例如`Health`减少后，`HealthPercent`会自动实时更新）。

#### 3. 元属性的实现方式（关键：`GetGameplayAttributeValue`重写）

在 UE 中，所有`Attribute`都需要继承`FGameplayAttributeData`或实现`IGameplayAttributeValue`接口，而元属性的核心是**重写 “数值获取逻辑”**，而非使用默认的 “读取存储值” 逻辑。

##### 典型实现步骤（以`HealthPercent`为例）：

1. **定义元属性标识**
   在属性定义时，通过`Meta = (IsMetaAttribute = true)`标记该属性为元属性（可选，但便于框架识别和管理）：

   ```cpp
   // 在Attribute类中声明（如UMyCharacterAttributes）
   UPROPERTY(BlueprintReadOnly, Category = "Health", Meta = (IsMetaAttribute = true))
   FGameplayAttributeData HealthPercent;
   ```

2. **重写数值计算逻辑**
   重写`GetGameplayAttributeValue`函数，指定元属性的计算规则（从基础属性推导数值）：

   ```cpp
   // 重写属性值的获取逻辑
   virtual float GetGameplayAttributeValue(const FGameplayAttribute& Attribute) const override
   {
       // 若请求的是HealthPercent元属性，则计算并返回结果
       if (Attribute == GetHealthPercentAttribute())
       {
           // 获取基础属性Health和MaxHealth的当前值
           float CurrentHealth = GetHealth();
           float MaxHealth = GetMaxHealth();
           
           // 避免除零错误，返回0或1（根据实际需求）
           return (MaxHealth > 0.0f) ? (CurrentHealth / MaxHealth) : 0.0f;
       }
       
       // 非元属性（如Health、MaxHealth），使用默认存储值逻辑
       return Super::GetGameplayAttributeValue(Attribute);
   }
   ```

3. **使用元属性**
   其他系统（如 UI 显示、技能逻辑）获取元属性时，与获取基础属性的方式完全一致 —— 框架会自动调用重写后的`GetGameplayAttributeValue`，返回实时计算的结果：

   ```cpp
   // 获取ASC（能力系统组件）
   UAbilitySystemComponent* ASC = GetOwner()->FindComponentByClass<UAbilitySystemComponent>();
   if (ASC)
   {
       // 获取UMyCharacterAttributes实例
       const UMyCharacterAttributes* Attrs = ASC->GetAttributeSet<UMyCharacterAttributes>();
       if (Attrs)
       {
           // 直接获取HealthPercent元属性（自动触发计算）
           float CurrentHealthPercent = Attrs->GetHealthPercent();
           // 用途：UI显示“生命值百分比”、技能判断“是否处于低血量状态”等
       }
   }
   ```

#### 4. 元属性的关键注意点

- **无独立修改接口**：元属性无法通过`SetBaseValue`、`AddModifiers`等方式直接修改（因为没有存储值），只能通过修改其依赖的 “基础属性” 间接改变结果（例如要让`HealthPercent`增加，只能增加`Health`或减少`MaxHealth`）。
- **计算性能**：由于元属性是 “实时计算” 的，若计算逻辑复杂（如依赖大量基础属性），频繁读取可能带来微小性能开销，但 UE 框架已对属性获取做了优化，常规场景下无需担心。
- **与属性修改器（Modifier）的关系**：元属性的计算结果**会受基础属性的修改器影响**（例如`MaxHealth`被一个 “+20%” 的 Modifier 增强后，`HealthPercent = Health / (MaxHealth * 1.2)`会自动反映这个变化），但元属性自身不能直接添加 Modifier。

#### 总结

元属性是 UE 能力系统中用于**动态派生基础属性数值**的 “计算型属性”，核心价值是：

1. 封装复杂的数值计算逻辑，降低代码耦合；
2. 保证派生数值与基础属性的实时一致性；
3. 简化外部系统对 “组合 / 比例型属性” 的使用。

它的本质是 “用计算逻辑替代存储”，是构建灵活、可扩展属性系统的重要工具（如角色面板、伤害结算、状态判断等场景均会大量用到）。

### 4.3.4 响应Attribute变化

重点就是使用`AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate`函数获取`Attribute`的委托，当`Attribute`发生变化时，与委托绑定的函数便会触发。

### 4.3.5 自动推导Attribute

**自动推导 Attribute（Auto-Derived Attribute）** 是一种特殊的属性类型，它能够**根据预设的规则自动从其他属性（通常是基础属性或其他推导属性）计算并更新自身的值**，无需手动编写计算逻辑。它本质上是引擎框架提供的 **“自动化元属性（Meta Attribute）”**，简化了属性间依赖关系的维护。

#### 核心特点：依赖规则驱动的自动计算

自动推导 Attribute 的核心是**通过配置 “依赖关系” 和 “计算规则”**，让引擎自动处理数值更新，无需像普通元属性那样手动重写`GetGameplayAttributeValue`方法。

例如：

- 若定义`HealthPercent`为自动推导属性，依赖`Health`（当前生命值）和`MaxHealth`（最大生命值），并指定规则为`Health / MaxHealth`，则每当`Health`或`MaxHealth`变化时，`HealthPercent`会自动重新计算并更新。

#### 与普通元属性（Meta Attribute）的区别

| 特性             | 普通元属性（Meta Attribute）                | 自动推导 Attribute（Auto-Derived Attribute） |
| ---------------- | ------------------------------------------- | -------------------------------------------- |
| 计算逻辑实现方式 | 需要手动重写`GetGameplayAttributeValue`方法 | 无需写代码，通过配置依赖规则自动计算         |
| 数值更新时机     | 每次读取时实时计算                          | 依赖属性变化时自动更新，缓存计算结果         |
| 适用场景         | 复杂计算逻辑（多步骤、条件判断）            | 简单数学关系（比例、差值、求和等）           |

#### 实现方式：通过`AttributeSet`中的规则配置

自动推导 Attribute 的核心是在`AttributeSet`（属性集）中定义其 “依赖属性” 和 “计算表达式”，通常通过以下步骤实现：

1. **在`AttributeSet`中声明自动推导属性**
   与普通属性类似，但需要通过元数据（Meta）或代码标记为自动推导类型，并指定依赖项和计算规则：

   ```cpp
   UCLASS()
   class MYGAME_API UMyAttributeSet : public UAttributeSet
   {
       GENERATED_BODY()
   
   public:
       // 基础属性：当前生命值
       UPROPERTY(BlueprintReadOnly, Category = "Health")
       FGameplayAttributeData Health;
       ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health);
   
       // 基础属性：最大生命值
       UPROPERTY(BlueprintReadOnly, Category = "Health")
       FGameplayAttributeData MaxHealth;
       ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth);
   
       // 自动推导属性：生命值百分比（依赖Health和MaxHealth）
       UPROPERTY(BlueprintReadOnly, Category = "Health", Meta = (AutoDerived = true))
       FGameplayAttributeData HealthPercent;
       ATTRIBUTE_ACCESSORS(UMyAttributeSet, HealthPercent);
   
   protected:
       // 重写属性初始化，定义自动推导规则
       virtual void PostInitProperties() override;
   };
   ```

2. **配置推导规则**
   在`PostInitProperties`或专门的初始化函数中，为自动推导属性指定依赖的基础属性和计算表达式（如除法、减法等）：

   ```cpp
   void UMyAttributeSet::PostInitProperties()
   {
       Super::PostInitProperties();
   
       // 为HealthPercent配置自动推导规则：Health / MaxHealth
       auto& HealthAttr = GetHealthAttribute();
       auto& MaxHealthAttr = GetMaxHealthAttribute();
       auto& HealthPercentAttr = GetHealthPercentAttribute();
   
       // 注册推导规则：当Health或MaxHealth变化时，自动计算HealthPercent
       AbilitySystemComponent->RegisterAutoDerivedAttribute(
           HealthPercentAttr,
           [this](const FGameplayAttribute& DerivedAttr, const FGameplayAttributeData* SourceData) {
               float Health = GetHealth();
               float MaxHealth = GetMaxHealth();
               return (MaxHealth > 0.0f) ? (Health / MaxHealth) : 0.0f;
           },
           { HealthAttr, MaxHealthAttr } // 依赖的属性列表
       );
   }
   ```

3. **自动更新机制**
   当依赖的基础属性（如`Health`或`MaxHealth`）发生变化时，引擎会自动触发推导规则的重新计算，并更新`HealthPercent`的值。其他系统读取`HealthPercent`时，直接获取最新的缓存结果即可。

#### 核心优势

1. **减少重复代码**：无需手动编写和维护属性间的计算逻辑，尤其适合简单的数学关系（如百分比、差值、总和等）。
2. **自动同步更新**：依赖属性变化时，推导属性会自动更新，避免手动同步导致的遗漏或错误。
3. **提升性能**：计算结果会被缓存，而非每次读取时重新计算（与普通元属性的实时计算不同），适合高频访问的场景。

#### 适用场景

- 百分比属性：如`HealthPercent`（生命值百分比）、`ManaPercent`（法力值百分比）。
- 差值属性：如`ShieldRemaining`（剩余护盾 = 总护盾 - 已受损护盾）。
- 总和属性：如`TotalDamage`（总伤害 = 物理伤害 + 魔法伤害）。

#### 总结

自动推导 Attribute 是 GAS 中一种**通过配置依赖规则实现自动计算和更新的属性类型**，它简化了属性间依赖关系的维护，尤其适合简单的数学推导场景。与手动实现的元属性相比，它更高效、更不易出错，是构建清晰、可扩展属性系统的重要工具

## 4.4 AttributeSet

### 4.4.1 定义AttributeSet

### 4.4.2 设计AttributeSet

1. **"Attribute 在内部被引用为 AttributeSetClassName.AttributeName"**
   每个属性都有个 "全名"，格式是 "清单名。属性名"。
   比如：

   - "角色基础属性清单" 里的血量，全名是`BaseAttributeSet.Health`；
   - "角色战斗属性清单" 里的攻击力，全名是`CombatAttributeSet.Attack`。
     这样命名是为了区分不同清单里的同名属性（比如可能两个清单都有`Speed`，但含义不同）。

2. **"继承 AttributeSet 时，父类的 Attribute 仍保留父类名作为前缀"**
   当你 "抄作业"（继承）时，父清单里的属性不会改名字。
   比如：

   - 父类`BaseAttributeSet`有个`Health`属性（全名`BaseAttributeSet.Health`）；
   - 你新建了个子类`PlayerAttributeSet`继承它，这个子类会自动有`Health`属性，但它的全名**仍然是`BaseAttributeSet.Health`**，不会变成`PlayerAttributeSet.Health`。

   这就像你抄了同桌练习册上的题，题目的编号（前缀）还是同桌练习册上的，不会变成你的。

### 4.4.3 定义Attribute

注意可以在头文件中使用宏自动为所有的`Attribute`生成`Getter`和`Setter`函数，如果我们创建的`Attribute`需要属性同步，也可以添加`OnRep`函数，同时不要忘记在`GetLifetimeReplicatedProps`函数中注册属性，否则同步不会生效。

### 4.4.4 初始化Attribute

初始化`Attribute`就是将`Attribute`中的`BaseValue`和`CurrentValue`设置为某一基础数值，文章中介绍了两种方法。

第一个就是推荐使用`即刻(Instant)GameplayEffect`来初始化`Attribute`，因为我们知道`即刻(Instant)GameplayEffect`是可以修改`BaseValue`数值的。

第二个就是当我们为`Attribute`使用了`ATTRIBUTE_ACCESSORS`宏之后，在`AttributeSet`中会自动为每个`Attribute`生成一个初始化函数。

```C++
// InitHealth(float InitialValue) is an automatically generated function for an Attribute 'Health' defined with the `ATTRIBUTE_ACCESSORS` macro
// InitHealth（float InitialValue）是为通过`ATTRIBUTE_ACCESSORS`宏定义的“Health”属性自动生成的函数。
AttributeSet->InitHealth(100.0f);
// 通过调用上述函数，即可初始化BaseValue和CurrentValue的数值为100.0f
```

### 4.4.5 PreAttributeChange()

`PreAttributeChange` 是 `AttributeSet` 中一个非常关键的函数，它的核心作用是 **“在属性的当前值（CurrentValue）被修改之前，对即将生效的新值（NewValue）进行检查、限制或调整”**，相当于给属性值的修改加了一道 “前置过滤器”。

#### 通俗理解：

想象你有一个 “生命值（Health）” 属性，正常情况下它的范围应该是 `0` 到 `最大生命值（MaxHealth）` 之间。如果没有任何限制，可能会出现一些不合理的情况：比如生命值被错误地改成负数，或者超过最大生命值。

`PreAttributeChange` 就像是在 “生命值真正被修改” 前的一道关卡，它会先拿到 “即将要设置的新值（NewValue）”，然后根据规则（比如 “不能小于 0，不能大于 MaxHealth”）对这个新值进行调整，确保最终生效的数值是合理的。

#### 具体作用解析：

1. **在属性修改前介入**
   当任何操作（比如技能回血、受伤扣血、属性加成等）要改变某个属性的 `CurrentValue` 时，`PreAttributeChange` 会在 “修改实际发生” 前被自动调用。
   这时候，你可以通过这个函数提前干预，避免不合理的数值进入系统。

2. **通过 `NewValue` 引用限制数值范围**
   函数的第二个参数 `float& NewValue` 是一个 “引用参数”—— 这意味着你可以直接修改它的值，而修改后的结果会成为最终生效的属性值。
   最常见的用法是 “数值 clamping（夹取）”，即限制数值在合理范围内：

   ```cpp
   void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
   {
       Super::PreAttributeChange(Attribute, NewValue);
   
       // 如果是生命值（Health）的修改
       if (Attribute == GetHealthAttribute())
       {
           // 获取最大生命值作为上限
           float MaxHealth = GetMaxHealth();
           // 限制 NewValue 不能小于0，也不能大于 MaxHealth
           NewValue = FMath::Clamp(NewValue, 0.0f, MaxHealth);
       }
   }
   ```

   这样一来，无论外部操作想把生命值改成多少（比如 `-50` 或 `1000`，而最大生命值是 `500`），最终生效的都会被限制在 `0` 到 `500` 之间。

3. **处理特殊属性的逻辑**
   除了简单的范围限制，还可以在这里处理更复杂的规则：

   - 比如 “最大生命值（MaxHealth）” 提升时，自动按比例调整当前生命值（例如 MaxHealth 从 100 升到 200，当前 Health 从 50 自动升到 100，保持 50% 比例）；
   - 比如 “法力值（Mana）” 不能为负数，且修改时需要触发某些日志记录或调试信息。

#### 为什么需要这个函数？

- **保证属性数值的合理性**：防止因外部逻辑错误（如技能效果计算错误）导致属性值出现异常（如负数生命值、超过上限的能量值）。
- **集中管理规则**：将属性的限制逻辑统一放在 `PreAttributeChange` 中，避免在每个修改属性的地方重复写限制代码，便于维护。

#### 总结：

`PreAttributeChange` 是属性值修改的 “前置检查点”，通过它可以在属性真正变化前，对新值进行限制、调整或特殊处理，确保属性数值始终符合游戏设计的规则（比如生命值不会为负、不会超过最大值等）。它是维护属性系统稳定性的重要工具。

### 4.4.6 PostGameplayEffectExecute()

`PostGameplayEffectExecute` 是 `AttributeSet` 中的一个关键函数，专门用于在**即刻生效的 GameplayEffect（瞬时效果）修改属性的基础值（BaseValue）之后**，执行额外的逻辑处理。它的核心作用是**在属性被瞬时效果修改后，对结果进行进一步处理、同步或触发关联逻辑**。

#### 通俗理解：

想象你玩游戏时，使用了一个 “瞬间回血” 的技能（这就是一种 “即刻 GameplayEffect”），这个技能直接修改了你的 “生命值（Health）基础值”。
`PostGameplayEffectExecute` 就像是这个回血操作完成后的 “收尾程序”—— 它会在血量已经增加后被自动调用，可以用来做一些后续工作，比如：播放回血特效、更新 UI 显示的血量、如果血量回满了就触发 “满血状态” 的 buff 等。

#### 具体作用解析：

1. **仅响应 “即刻生效的 GameplayEffect”**
   GameplayEffect 有多种类型，其中 “即刻型（Instant）” 是指**立即对属性产生一次性修改**（比如瞬间回血、瞬时伤害、立即增加攻击力等），而不是持续一段时间的效果（如持续 5 秒的中毒）。
   `PostGameplayEffectExecute` 只会在这种 “即刻效果” 修改了属性的 BaseValue 后被触发，其他类型的效果（如持续效果）不会触发它。
2. **在属性修改 “之后” 介入**
   与 `PreAttributeChange`（修改前干预）不同，这个函数是在属性的 BaseValue 已经被成功修改后才被调用的。此时你可以：
   - 获取修改后的最新属性值；
   - 基于新值执行后续逻辑（如判断是否达到某个阈值）；
   - 同步其他系统（如 UI、动画、音效）。
3. **通过 `Data` 参数获取修改详情**
   函数的参数 `const FGameplayEffectModCallbackData& Data` 包含了这次属性修改的完整信息，例如：
   - 哪个 GameplayEffect 导致了修改（可以判断是回血技能还是伤害技能）；
   - 被修改的属性是什么（是生命值、法力值还是攻击力）；
   - 修改前后的数值变化（比如生命值从 50 涨到了 80，增加了 30）。

#### 典型使用场景举例：

##### （1）处理伤害 / 治疗后的反馈

```cpp
void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    // 检查是否是生命值（Health）被修改
    if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        // 获取修改后的最新生命值
        float NewHealth = GetHealth();
        
        // 如果生命值变为0，触发死亡逻辑
        if (NewHealth <= 0.0f)
        {
            AMyCharacter* Character = GetOwningCharacter();
            if (Character)
            {
                Character->OnDeath(); // 触发死亡动画、游戏结束等逻辑
            }
        }
        // 否则，播放回血/受伤特效
        else
        {
            // 计算变化量（正数为回血，负数为受伤）
            float Delta = NewHealth - Data.EvaluatedData.OldValue;
            if (Delta > 0)
                PlayHealVFX(); // 播放回血特效
            else
                PlayHurtVFX(); // 播放受伤特效
        }
        
        // 更新UI显示的生命值
        UpdateHealthUI(NewHealth, GetMaxHealth());
    }
}
```

##### （2）处理属性修改后的连锁反应

比如 “攻击力提升后，自动同步到武器伤害计算系统”：

```cpp
if (Data.EvaluatedData.Attribute == GetAttackPowerAttribute())
{
    float NewAttackPower = GetAttackPower();
    // 通知武器系统更新伤害计算
    GetWeaponComponent()->UpdateDamageBasedOnAttack(NewAttackPower);
}
```

#### 为什么需要这个函数？

- **集中处理瞬时效果的后续逻辑**：将 “属性被瞬时修改后” 的所有关联操作（如特效、UI、状态判断）集中在这里，避免在每个 GameplayEffect 中重复编写这些逻辑。
- **确保逻辑在属性修改后执行**：有些操作必须依赖 “修改完成后的最终值”（如死亡判断需要知道血量是否真的降到 0），这个函数提供了可靠的执行时机。

#### 总结：

`PostGameplayEffectExecute` 是 “即刻型 GameplayEffect 修改属性后” 的专属回调函数，用于在属性值确定更新后，执行各种后续逻辑（如反馈特效、状态判断、系统同步等）。它是连接瞬时属性修改与游戏表现 / 逻辑的重要桥梁。

### 4.4.7 OnAttributeAggregatorCreated()

在虚幻引擎 GAS（Gameplay Ability System）中，`OnAttributeAggregatorCreated()` 是 `UAttributeSet` 中的一个特殊回调函数，主要作用是**在属性的 “聚合器（Attribute Aggregator）” 被创建时执行初始化逻辑**。

要理解这个函数，首先需要了解什么是 “属性聚合器（Attribute Aggregator）”：
在 GAS 中，当属性（Attribute）需要处理复杂的修改逻辑（如叠加多个效果、处理网络同步、计算最终值等）时，引擎会为其创建一个 “聚合器”。这个聚合器负责管理该属性的所有修改器（Modifiers）、追踪数值变化，并计算最终生效的属性值。简单说，它是属性的 “数值计算管理器”。

#### `OnAttributeAggregatorCreated()` 的具体作用：

当引擎为某个属性创建聚合器后，会自动调用 `OnAttributeAggregatorCreated()` 函数，开发者可以在这个函数中：

1. **对聚合器进行自定义配置**
   例如，设置属性数值的精度（如保留小数点后几位）、配置网络同步的策略（如是否需要高频同步）、或指定聚合器的计算优先级。
2. **绑定聚合器的事件**
   聚合器本身会触发一些事件（如数值计算完成、修改器被添加 / 移除等），可以在这个函数中绑定这些事件的回调，以便在特定时机执行逻辑。
   例如：当聚合器计算出属性的最终值后，自动同步到 UI 显示。
3. **初始化与聚合器相关的辅助数据**
   如果属性需要依赖聚合器的某些中间结果（如修改器的总叠加层数），可以在这里初始化存储这些数据的变量。

#### 典型使用场景示例：

```cpp
void UMyAttributeSet::OnAttributeAggregatorCreated(const FGameplayAttribute& Attribute, UAttributeAggregator* Aggregator)
{
    Super::OnAttributeAggregatorCreated(Attribute, Aggregator);

    // 如果是生命值（Health）的聚合器被创建
    if (Attribute == GetHealthAttribute())
    {
        // 配置聚合器的数值精度（保留1位小数）
        Aggregator->SetPrecision(1);

        // 绑定聚合器的数值更新事件，同步到UI
        Aggregator->OnAggregatedValueUpdated.AddUObject(this, &UMyAttributeSet::OnHealthUpdated);
    }
}

// 聚合器数值更新时的回调
void UMyAttributeSet::OnHealthUpdated(float NewValue)
{
    // 更新UI显示
    UpdateHealthUI(NewValue);
}
```

#### 总结：

`OnAttributeAggregatorCreated()` 是属性聚合器创建后的初始化回调，用于对属性的 “数值计算管理器” 进行自定义配置、事件绑定或辅助数据初始化。它通常用于处理需要精细控制属性计算逻辑或同步策略的场景，是 GAS 中高级属性管理的重要工具。

## 4.5 GameplayEffect

### 4.5.1 定义GameplayEffect

#### 文中原话：

`GameplayEffect`一般是不实例化的, 当Ability或`ASC`想要应用`GameplayEffect`时, 其会从`GameplayEffect`的`ClassDefaultObject`创建一个`GameplayEffectSpec`, 之后成功应用的`GameplayEffectSpec`会被添加到一个名为`FActiveGameplayEffect`的新结构体, 其是`ASC`在名为`ActiveGameplayEffect`的特殊结构体容器中追踪的内容.  

#### 逐句解析：

1. **“GameplayEffect 一般是不实例化的”**
   `GameplayEffect`在项目中通常以 “资源文件”（`.uasset`）的形式存在，相当于一个 “效果模板”（比如 “5 秒内每秒回血 10 点”“增加 20 点攻击力”）。它本身不会被直接 “激活”，而是作为 “蓝图” 被反复使用，不会为每个应用场景创建一个独立的 GE 实例。
2. **“当 Ability 或 ASC 想要应用 GameplayEffect 时，会从 GameplayEffect 的 ClassDefaultObject 创建一个 GameplayEffectSpec”**
   当需要实际应用这个 GE 时（比如技能触发回血效果），不会直接使用 GE 模板，而是基于它的 “默认对象（ClassDefaultObject）” 创建一个`GameplayEffectSpec`（简称 Spec，可理解为 “具体执行订单”）。
   - `ClassDefaultObject`：GE 模板的默认实例，存储了 GE 的基础配置（如持续时间、修改器规则等）。
   - `GameplayEffectSpec`：基于模板创建的 “带参数的具体请求”，可以临时修改一些参数（比如原本模板是 “回血 10 点”，Spec 可以改成 “回血 20 点”，或指定只对特定目标生效）。
     简单说：GE 是 “标准菜谱”，Spec 是 “根据客人要求调整后的具体点餐单”。
3. **“成功应用的 GameplayEffectSpec 会被添加到一个名为 FActiveGameplayEffect 的新结构体”**
   当 Spec 被成功应用到目标（如角色）身上后，它会被包装成`FActiveGameplayEffect`结构体（可理解为 “正在执行的任务”）。
   这个结构体不仅包含 Spec 的所有信息，还会记录效果的 “当前状态”：比如已经生效了多久、剩余持续时间、是否被暂停、叠加层数等。
   例如：“5 秒回血” 的 GE，`FActiveGameplayEffect`会记录 “已生效 2 秒，剩余 3 秒，当前已回血 20 点”。
4. **“其是 ASC 在名为 ActiveGameplayEffect 的特殊结构体容器中追踪的内容”**
   所有`FActiveGameplayEffect`（正在生效的效果）都会被存放在`ASC`（AbilitySystemComponent）的`ActiveGameplayEffect`容器中，由 ASC 统一管理：
   - 每帧更新持续效果的状态（如每秒执行一次回血）；
   - 当效果到期（如 5 秒结束）或被移除（如驱散技能）时，从容器中删除；
   - 处理效果的叠加、刷新、优先级等逻辑（如多个加速效果同时生效时，取最高值）。

#### 总结流程：

`GameplayEffect`（模板） → 基于模板创建`GameplayEffectSpec`（带参数的请求） → 成功应用后转为`FActiveGameplayEffect`（正在执行的效果） → 由 ASC 的`ActiveGameplayEffect`容器管理生命周期。

这个流程的核心是 “模板复用 + 实例化管理”：通过 GE 模板定义基础规则，用 Spec 灵活调整参数，最终用 FActiveGameplayEffect 追踪实际生效状态，既保证了效率，又能灵活应对各种场景。

### 4.5.2 应用GameplayEffect

### 4.5.3 移除GameplayEffect

### 4.5.4 GameplayEffectModifier

#### 通俗解释与使用场景：

##### 1. `ScalableFloat`（可缩放浮点型）

**核心特点**：数值来自 “数据表（Data Table）” 或 “固定值”，可随等级 / 变量动态变化。

- 像一个 “动态数值模板”：数据表按 “变量 + 等级” 二维排列（比如行是 “伤害”“防御”，列是 1 级、2 级...），它会根据技能当前等级自动读取对应行的值。
- 也可以不依赖数据表，直接硬编码一个固定值（此时默认系数为 1，等价于 “固定数值”）。
- 读取后还能通过 “系数” 进一步调整（比如基础值 ×1.5）。

**举例**：

- 技能 “火球术” 的伤害随等级变化：1 级时 100 点，2 级时 150 点... 可以用`ScalableFloat`关联数据表，自动根据等级取对应值。

##### 2. `AttributeBased`（基于属性型）

**核心特点**：数值来自 “源对象” 或 “目标对象” 的其他属性值（如攻击力、防御力）。

- 像一个 “属性联动器”：比如 “伤害 = 攻击者攻击力 ×0.8”，这里 “攻击者攻击力” 就是 “源属性（Source Attribute）”；或者 “治疗量 = 目标最大生命值 ×0.2”，这里 “目标最大生命值” 是 “目标属性（Target Attribute）”。
- 支持 “快照（Snapshotting）”：创建效果时就固定属性值（比如技能释放时记录当前攻击力，后续即使攻击力变化，伤害也不变）；
- 或 “非快照”：应用效果时才读取属性值（比如技能命中时才取当前攻击力，更实时）。

**举例**：

- 物理攻击的伤害公式：`伤害 = 自身攻击力 × 0.5 - 目标防御力 × 0.2`，可以用`AttributeBased`分别关联 “自身攻击力” 和 “目标防御力”。

##### 3. `CustomCalculationClass`（自定义计算类）

**核心特点**：通过自定义 C++/ 蓝图类实现复杂计算，灵活性最高。

- 像一个 “高级计算器”：当数值计算逻辑复杂（比如涉及多个属性、条件判断、概率等），前两种方式不够用时，就用它。
- 需要继承`UMultiplierMagnitudeCalculation`类，重写计算函数，自由编写逻辑（比如 “暴击时伤害翻倍，否则按基础公式计算”）。

**举例**：

- 复杂伤害公式：`最终伤害 = (攻击力 × 1.2 + 技能等级 × 10) × (1 + 暴击率 × 暴击倍率) - 目标抗性 × 0.3`，这种多条件组合的计算适合用自定义类。

##### 4. `SetByCaller`（调用者设置型）

**核心特点**：数值在运行时由 “创建者”（如技能、效果）动态设置，而非预先定义。

- 像一个 “临时输入框”：需要根据实时操作动态调整数值时使用（比如蓄力技能，蓄力越久伤害越高）。
- 本质是存在于`GameplayEffectSpec`中的 “标签 - 数值映射表”（`TMap<FGameplayTag, float>`），设置时用特定`GameplayTag`作为键，修改器通过该标签读取值。
- **注意**：如果运行时没设置对应标签的数值，会报错并返回 0（可能导致除零等问题）。

**举例**：

- 蓄力技能 “重击”：蓄力 1 秒伤害 100，蓄力 2 秒伤害 200... 技能释放时根据蓄力时间，通过`SetByCaller`设置伤害值，再应用到效果中。

##### 总结：

四种`Modifier`的核心区别在于 “数值来源”：

- `ScalableFloat` → 数据表 / 固定值（适合随等级变化的静态数值）；
- `AttributeBased` → 其他属性（适合与源 / 目标属性联动的动态数值）；
- `CustomCalculationClass` → 自定义逻辑（适合复杂计算）；
- `SetByCaller` → 运行时动态设置（适合实时交互决定的数值）

#### 4.5.4.1 Multiply和Divide Modifier

默认情况下, 所有的`Multiply`和`Divide`Modifier在对`Attribute`的BaseValue乘除前都会先加到一起. 

这部分知识着重解释了乘除运算的运算规律以及相关问题，暂时不需要深究。

#### 4.5.4.2 Modifier的GameplayTag

解释文中本节内容。

这段话详细解释了`Modifier`（修改器）中`SourceTag`、`TargetTag`以及`Attribute Based Modifier`特有的`SourceTagFilter`、`TargetTagFilter`的作用机制，核心是**通过 “标签过滤” 控制修改器是否生效或参与计算**。我们可以分两部分理解：

##### 1. `Modifier`的`SourceTag`和`TargetTag`：控制修改器是否生效的 “条件标签”

它们的作用是**在`GameplayEffect`应用后，进一步限制 “该修改器是否实际生效”**，类似 “二次筛选条件”。

**具体规则**：

1. **作用类似 “应用后条件”**
   不同于`GameplayEffect`的`Application Tag requirements`（这是 “Effect 能否被应用” 的前置条件），`SourceTag`和`TargetTag`是 “Effect 已经成功应用后”，判断 “这个 Modifier 是否真的要生效” 的条件：

   - `SourceTag`：要求**效果的源对象（如技能释放者）必须包含这些标签**，否则 Modifier 不生效；
   - `TargetTag`：要求**效果的目标对象（如被攻击的敌人）必须包含这些标签**，否则 Modifier 不生效。

   举例：
   一个 “对燃烧目标额外造成伤害” 的 Modifier，`TargetTag`设为`"Status.Burning"`。当 Effect 应用到目标后，只有目标身上有 “燃烧” 标签时，这个额外伤害的 Modifier 才会生效。

2. **检查时机的特殊性**

   - 对于**瞬时 Effect**：应用时检查一次标签，符合则 Modifier 生效，否则不生效；
   - 对于**周期性无限 Effect**（如 “持续存在，每 3 秒触发一次”）：**只在第一次应用 Effect 时检查标签**，后续周期触发时不再检查。
     举例：
     一个 “对中毒目标每 3 秒造成伤害” 的无限周期性 Effect，第一次应用时目标有 “中毒” 标签（`TargetTag`匹配），则后续每 3 秒都会触发伤害，即使中途目标解毒（标签消失），伤害仍会继续。

##### 2. `Attribute Based Modifier`的`SourceTagFilter`和`TargetTagFilter`：筛选参与计算的修改器

这两个过滤器专门用于`Attribute Based Modifier`（基于属性的修改器），作用是**在计算 “源属性（Backing Attribute）的数值” 时，筛选掉不符合标签条件的其他 Modifier**，确保只有特定 Modifier 参与计算。

**具体逻辑**：

1. **筛选 “影响源属性的 Modifier”**
   `Attribute Based Modifier`的数值依赖于 “源对象或目标对象的某个属性（如攻击力、防御力）”，而这个属性本身可能被多个 Modifier 影响（比如 “攻击力” 可能被 “武器加成”“ buff 加成” 等多个 Modifier 修改）。
   `SourceTagFilter`和`TargetTagFilter`会筛选：**只有那些源 / 目标包含过滤器指定标签的 Modifier，才能参与源属性的数值计算**。

   举例：
   一个基于 “源对象攻击力” 的伤害 Modifier，`SourceTagFilter`设为`"Buff.AttackUp"`。则计算 “源对象攻击力” 时，只会包含那些 “源对象有`Buff.AttackUp`标签” 的攻击力 Modifier（如 “狂暴 buff 的攻击力加成”），其他无标签的 Modifier（如基础攻击力）会被排除。

2. **标签的 “捕获时机”**
   过滤器检查的标签不是实时的，而是 “提前捕获” 的：

   - 源对象（Source）的标签：在`GameplayEffectSpec`创建时（如技能释放瞬间）被捕获并保存；
   - 目标对象（Target）的标签：在`GameplayEffect`执行时（如效果应用到目标瞬间）被捕获并保存。

   对于**无限（Infinite）或持续（Duration）Effect**：每次检查 Modifier 是否符合条件（聚合器条件）时，会用这些 “捕获的标签” 与过滤器比对，而非实时标签。

##### 总结：

- `SourceTag`/`TargetTag`：控制 Modifier 在 Effect 应用后是否生效，检查时机有限制（周期性 Effect 只初始检查）；
- `SourceTagFilter`/`TargetTagFilter`：仅用于基于属性的 Modifier，筛选哪些 Modifier 能参与源属性的数值计算，依赖提前捕获的标签。

这些标签机制让 Modifier 的生效逻辑更灵活，能实现 “只对特定状态（标签）的对象生效”“只计算特定来源的属性加成” 等复杂需求。

### 4.5.5 GameplayEffect堆栈

在解释文章内容之前，需要明确的是这里所说的堆栈的概念不是编写代码时程序员口中的那个堆栈，而是一种用来表达`GameplayEffect`堆叠层数的堆栈。

下面是对文章中本节内容的解释。

这段话描述的是 GAS 中`GameplayEffect`（GE）的 **“堆栈（Stack）机制”**—— 简单说，就是当多次应用同一个持续 / 无限 GE 时，不创建多个独立实例，而是通过 “叠加层数” 来管理，类似游戏中 “中毒叠加 3 层”“ buff 叠加 5 层” 的效果。而两种堆栈类型的核心区别在于**“如何区分不同来源的叠加”**。

#### 先通俗理解 “堆栈（Stack）”：

默认情况下，多次应用同一个 GE 会创建多个独立实例（比如两次 “5 秒中毒” 会同时存在两个中毒效果，各自计时）。而 “堆栈” 机制让它们合并成一个 “叠加效果”：

- 比如 “每层中毒每秒掉 10 血，最多 3 层”，第一次应用叠加到 1 层（每秒 10 血），第二次应用叠加到 2 层（每秒 20 血），第三次到 3 层（每秒 30 血），而不是同时存在 3 个独立中毒效果。
- 堆栈只对**持续（Duration）** 或**无限（Infinite）** GE 有效（瞬时 GE 应用后立即消失，无需叠加）。

#### 两种堆栈类型的区别：

##### 1. `Aggregate by Source`（按来源聚合）

**核心特点**：目标身上的堆栈 “按来源分开计算”，每个来源（如不同攻击者、不同技能）有自己独立的堆栈。

- 举例：
  游戏中有两个敌人 A 和 B，都能给玩家上 “减速” GE（持续 10 秒，每层减速 10%，最多 3 层），堆栈类型设为`Aggregate by Source`。
  - 敌人 A 攻击玩家：玩家身上出现 “A 的减速堆栈”，层数 1（减速 10%）；
  - 敌人 A 再次攻击：“A 的减速堆栈” 升到 2 层（减速 20%）；
  - 敌人 B 攻击玩家：玩家身上新增 “B 的减速堆栈”，层数 1（此时玩家同时受 A 的 2 层 + B 的 1 层，共减速 30%）；
  - 每个来源的堆栈独立计时、独立叠加（A 的堆栈不会影响 B 的）。
- 适用场景：需要区分 “效果来源” 的叠加，比如 “不同玩家造成的 debuff 分开计算层数”“同一个技能的多次命中单独叠加”。

##### 2. `Aggregate by Target`（按目标聚合）

**核心特点**：目标身上只有 “一个共享堆栈”，所有来源的叠加都计入这个堆栈，受 “共享层数上限” 限制。

- 举例：
  同上 “减速” GE，堆栈类型设为`Aggregate by Target`，共享上限 3 层。
  - 敌人 A 攻击：共享堆栈层数 1（减速 10%）；
  - 敌人 B 攻击：共享堆栈层数 2（减速 20%）；
  - 敌人 A 再攻击：共享堆栈层数 3（减速 30%，达到上限）；
  - 此时无论 A 或 B 再攻击，层数都不会超过 3 层，且整个堆栈只有一个（共享计时，比如持续 10 秒从最后一次叠加开始刷新）。
- 适用场景：所有来源的效果 “共同叠加”，比如 “任何攻击都能增加同一个 buff 的层数”“团队中多个队友的增益效果共享叠加上限”。

#### 总结

两种堆栈类型的核心区别是 “是否区分效果来源”：

- `Aggregate by Source`：来源不同则堆栈独立（各算各的层数）；
- `Aggregate by Target`：无视来源，所有叠加计入同一个共享堆栈（共算一层数）。

通过选择堆栈类型，可以灵活实现 “按来源独立叠加” 或 “全来源共同叠加” 的效果，满足不同的游戏设计需求（如区分敌人 debuff、团队共享 buff 等）。

### 4.5.6 授予Ability

### 4.5.7 GameplayEffect标签

**对文中本节内容的解释**：

`GameplayEffect`（GE）中的这些`GameplayTagContainer`（标签容器）是 GE 与其他系统（如技能、状态、交互逻辑）通信的 “语言”，不同类别的标签容器承担着不同的功能，核心是通过 “标签” 控制 GE 的应用条件、状态维持、效果联动等。

#### 先理解基础概念：`Added`和`Removed`标签

GE 可以继承父类 GE 的标签配置，`Added`标签是当前 GE“新增的、父类没有的标签”；`Removed`标签是 “父类有但当前 GE 去掉的标签”。最终生效的是`Combined GameplayTagContainer`（合并后的标签容器），即父类标签减去 Removed 标签，再加上 Added 标签，确保标签继承的正确性。

#### 各分类标签容器的具体作用：

##### 1. `Gameplay Effect Asset Tags`（GE 资源标签）

**核心作用：给 GE 本身打 “描述性标签”，无实际功能，仅用于分类和识别**。

- 这些标签属于 GE 自身（类似 “标签属性”），不会影响游戏逻辑，仅用于开发者筛选、查询或编辑器内的分类。
- 举例：给所有 “回血类 GE” 打上`"Type.Heal"`标签，所有 “中毒类 GE” 打上`"Type.Poison"`标签，方便在编辑器中按类型搜索 GE 资源。

##### 2. `Granted Tags`（授予标签）

**核心作用：当 GE 应用到目标时，自动给目标的`ASC`添加这些标签；当 GE 被移除时，自动移除这些标签**。

- 仅对**持续（Duration）** 或**无限（Infinite）** GE 有效（瞬时 GE 应用后立即消失，无需授予标签）。
- 这些标签是目标 “当前状态” 的标识，其他系统可通过检测这些标签判断目标状态。
- 举例：
  一个 “燃烧” GE 的`Granted Tags`设为`"Status.Burning"`。当 GE 应用到敌人时，敌人的 ASC 会添加`"Status.Burning"`标签（其他技能 / 逻辑可检测该标签，比如 “对燃烧目标造成额外伤害”）；当燃烧 GE 结束，该标签会自动从敌人 ASC 中移除。

##### 3. `Ongoing Tag Requirements`（持续标签需求）

**核心作用：GE 应用后，需要目标持续满足这些标签条件才能 “激活运行”；不满足则暂时关闭，满足后重新激活**。

- 仅对**持续 / 无限 GE**有效。GE 不会被移除，只是暂时不执行其功能（如停止属性修改、暂停周期性效果）。
- 举例：
  一个 “潜行” GE 的`Ongoing Tag Requirements`设为`"Status.NotDetected"`（未被发现）。当目标被敌人发现（失去`"Status.NotDetected"`标签），潜行 GE 会关闭（隐身效果消失、不再获得潜行加成）；当目标重新隐藏（恢复标签），GE 会重新激活（隐身和加成恢复）。

##### 4. `Application Tag Requirements`（应用标签需求）

**核心作用：GE 能否应用到目标的 “前置条件”—— 目标必须拥有这些标签，否则 GE 无法应用**。

- 这是 GE 应用的 “门槛”，在应用前检查目标的标签，不满足则直接失败。
- 举例：
  一个 “对机械敌人生效” 的 GE，`Application Tag Requirements`设为`"Enemy.Type.Mechanical"`。当尝试将其应用到非机械敌人（无该标签）时，GE 会应用失败；只有目标是机械敌人（有标签）时，才能成功应用。

##### 5. `Remove Gameplay Effects with Tags`（移除带特定标签的 GE）

**核心作用：当前 GE 成功应用到目标后，会自动移除目标身上所有 “带有指定标签” 的其他 GE**。

- 用于 “效果冲突” 或 “净化” 场景，比如 “驱散 debuff”“新 buff 覆盖旧 buff”。
- 举例：
  一个 “净化” GE 的`Remove Gameplay Effects with Tags`设为`"Status.Poison"`（中毒标签）。当该 GE 应用到目标后，会自动移除目标身上所有 “Asset Tags 或 Granted Tags 包含`"Status.Poison"`” 的 GE（比如各种中毒效果）。

#### 总结：

这些标签容器是 GE 与游戏世界交互的 “规则引擎”：

- `Asset Tags`：描述 GE 自身，方便管理；
- `Granted Tags`：给目标打状态标签，供其他系统识别；
- `Ongoing Tag Requirements`：控制 GE 应用后的激活状态；
- `Application Tag Requirements`：限制 GE 的应用目标；
- `Remove...Tags`：让 GE 应用后清理冲突效果。

通过组合这些标签容器，能灵活实现复杂的效果逻辑（如 “只对特定敌人生效，应用后给目标上状态，状态存续依赖条件，且能驱散旧效果”）。

### 4.5.8 免疫

`GameplayEffect`的这种 “基于 GameplayTag 的免疫机制”，核心是让目标（拥有免疫的一方）能够**主动阻止特定来源或特定类型的 GameplayEffect（GE）应用**，并且在阻止时能触发回调（委托），比单纯用`Application Tag Requirements`（被动检查条件）更灵活，还能提供反馈。

#### 通俗理解核心作用：

想象游戏中 “玩家获得了‘免疫中毒’状态”，此时所有 “中毒类 GE” 都无法应用到玩家身上，且系统会通知 “中毒被免疫了”（比如播放免疫特效）。这种 “主动挡掉特定 GE + 触发反馈” 的机制，就是这里要讲的免疫系统。

#### 两种具体免疫方式的区别：

##### 1. `GrantedApplicationImmunityTags`（基于源标签的免疫）

**核心逻辑：通过 “源对象的标签” 来免疫其所有 GE**。

- 目标（被免疫方）的

  ```
  ASC
  ```

  中如果有

  ```
  GrantedApplicationImmunityTags
  ```

  ，就会检查 “要应用 GE 的源对象（如释放技能的敌人、物品等）” 的标签：

  - 源对象的`ASC`标签，或源对象使用的`Ability`的标签（如果有），只要包含`GrantedApplicationImmunityTags`中的任何标签，那么这个源对象的所有 GE 都无法应用到目标身上。

- 这是一种 “大范围免疫”：一旦源对象带有被免疫的标签，其所有 GE 都会被目标挡掉，无需逐个检查 GE 本身。

**举例**：

- 目标（玩家）的`GrantedApplicationImmunityTags`设为`"Source.Enemy.Dragon"`（免疫龙族敌人的所有效果）。
- 当龙族敌人（源对象）释放任何 GE（如火焰吐息、震慑）时，由于源对象的`ASC`带有`"Source.Enemy.Dragon"`标签，这些 GE 都会被玩家免疫，无法应用。

##### 2. `Granted Application Immunity Query`（基于 GE Spec 的免疫查询）

**核心逻辑：通过 “GE 实例的具体信息” 来精准免疫特定 GE**。

- 它不会笼统地看 “源对象的标签”，而是检查 “即将应用的`GameplayEffectSpec`（GE 的具体实例）” 是否匹配预设的 “查询条件”（比如 GE 的标签、修改器类型、持续时间等）。
- 如果匹配，就阻止这个 GE 应用；不匹配则允许。这是一种 “精准免疫”，可以针对特定 GE（而非整个源对象的所有 GE）。

**举例**：

- 目标的免疫查询条件设为 “GE 的 Asset Tags 包含`"Type.Poison"`”（只免疫中毒类 GE）。
- 当任何源对象释放 “中毒 GE”（其 Spec 符合查询条件），会被免疫；但释放 “减速 GE”（不符合条件）则可以正常应用。

#### 免疫机制的额外价值：`OnImmunityBlockGameplayEffectDelegate`委托

当免疫成功阻止 GE 应用时，`ASC`会触发这个委托。开发者可以绑定回调函数，做一些 “免疫反馈” 逻辑：

- 播放免疫特效（如目标身上闪过护盾光效）；
- 播放免疫音效（如 “格挡” 音效）；
- 记录日志（如 “玩家成功免疫了龙族的火焰吐息”）。

#### 与`Application Tag Requirements`的区别：

- `Application Tag Requirements`是 GE 自身的 “应用门槛”（比如 “只能应用到带 XX 标签的目标”），不满足则 GE 自己放弃应用，**不会触发免疫相关的委托**；
- 这里的免疫机制是目标主动 “挡掉” GE，**会触发免疫委托**，且能更灵活地控制 “哪些来源 / 哪些类型的 GE 被挡掉”。

#### 总结：

这种免疫机制通过`GrantedApplicationImmunityTags`（挡掉特定源的所有 GE）和`Granted Application Immunity Query`（挡掉特定类型的 GE），实现了对 GE 应用的主动拦截，同时通过委托提供了免疫反馈的入口，让 “免疫” 这一游戏逻辑既灵活又有表现力。

### 4.5.9 GameplayEffectSpec

这段话详细解释了 `GameplayEffectSpec`（简称 GESpec）的本质、创建方式和核心内容，它是 GAS 中连接 “静态 GameplayEffect（GE 模板）” 与 “动态实际生效效果” 的关键桥梁。我们可以拆解为 “是什么、怎么来、有什么用” 三个维度理解：

#### 一、先搞懂核心定位：GESpec 是 GE 的 “动态实例化订单”

`GameplayEffect` 本身是 “静态模板”（比如 “5 秒内每秒回血 10 点” 的配置文件），无法直接应用；而 `GESpec` 是基于这个模板创建的 “动态实例”—— 相当于 “根据模板生成的具体执行订单”，可以在应用前灵活修改参数，最终决定 “这次要怎么生效”。

- 通俗类比：
  GE 是餐厅的 “菜单模板”（写着 “番茄炒蛋，默认微辣、分量中”）；
  GESpec 是顾客下单时的 “个性化订单”（可以改成 “番茄炒蛋，免辣、分量大”）；
  最终厨房按 “个性化订单”（GESpec）做菜，而不是按 “菜单模板”（GE）直接做。

#### 二、GESpec 的创建方式：从模板到订单的生成过程

GESpec 只能通过 `UAbilitySystemComponent`（ASC）的接口创建，核心是 `MakeOutgoingSpec()` 函数（蓝图 / 代码均可调用）：

1. 传入 “要基于的 GE 模板”（比如 “回血 GE”）；
2. 可选传入 “等级、创建者信息” 等初始参数；
3. 函数返回一个可修改的 `GESpec` 实例。

- 关键特点：**创建后不必立即应用**。
  比如技能释放时生成 “伤害 GESpec”，先传给投掷物（如子弹），等子弹击中目标后，再用这个 GESpec 应用伤害效果 —— 实现 “延迟生效” 的逻辑。

#### 三、GESpec 的核心内容：决定 “效果怎么生效” 的关键参数

这些内容是 GESpec 区别于 GE 模板的核心，每一项都能动态调整，最终影响效果的实际表现：

| 核心内容                        | 作用与通俗解释                                               |
| ------------------------------- | ------------------------------------------------------------ |
| **关联的 GE 类**                | 记录这个 GESpec 是基于哪个 GE 模板创建的（比如 “回血 GE”“中毒 GE”），确保基础逻辑一致。 |
| **等级**                        | 可自定义，不必和创建它的 Ability 等级一致。比如 Ability 是 2 级，但 GESpec 强制按 1 级效果生效（如 “技能等级衰减”）。 |
| **持续时间 / 周期**             | 覆盖 GE 模板的默认值。比如 GE 模板是 “持续 5 秒”，GESpec 可改成 “持续 3 秒”（如 “效果被削弱”）；周期性 GE（如每秒伤害）的周期也能改（如从 1 秒 / 次改成 0.5 秒 / 次）。 |
| **当前堆栈数**                  | 记录这次应用要叠加的层数，上限由 GE 模板的堆栈设置决定。比如 GE 最多叠 3 层，GESpec 可指定 “这次直接叠 2 层”。 |
| **GameplayEffectContextHandle** | 记录 “谁创建了这个 GESpec”（如释放技能的玩家、触发效果的物品），用于追溯效果来源（比如 “伤害是谁造成的”）。 |
| **Snapshot 捕获的 Attribute**   | 创建 GESpec 时，会 “快照”（固定）相关属性值（如释放技能时的攻击力），后续即使属性变化，GESpec 也用快照值计算（比如 “技能释放后攻击力变化，不影响已生成的伤害 GESpec”）。 |
| **DynamicGrantedTags**          | 除了 GE 模板自带的 `Granted Tags`（如 “燃烧” 标签），GESpec 可额外动态添加标签（比如 “这次中毒额外加‘减速’标签”），应用后会和模板标签一起授予目标。 |
| **DynamicAssetTags**            | 类似 `DynamicGrantedTags`，但作用于 GESpec 自身的描述标签（如给这次 GESpec 加 “Critical” 标签，标记为 “暴击效果”），不影响目标，仅用于逻辑判断。 |
| **SetByCaller TMaps**           | 存储运行时动态设置的数值（键是 GameplayTag，值是浮点数），供 GE 的 `SetByCaller Modifier` 使用。比如蓄力技能的 GESpec，通过这个 Map 传入 “蓄力时间对应的伤害值”。 |

#### 四、GESpec 的最终归宿：从订单到实际效果

当 GESpec 通过 `ASC->ApplyGameplayEffectSpecToTarget()` 等接口成功应用后：

1. GAS 会将 GESpec 转换为 `FActiveGameplayEffect`（正在生效的效果结构体）；
2. 这个结构体被加入 ASC 的 `ActiveGameplayEffect` 容器，由 GAS 自动管理生命周期（更新、结束、移除）。

#### 总结：GESpec 的核心价值

GESpec 是 GE 模板的 “动态化工具”，解决了 “模板复用” 与 “灵活调整” 的矛盾：

- 基于 GE 模板保证基础逻辑统一（如 “回血” 的核心是加生命值）；
- 通过 GESpec 的动态参数（等级、时间、堆栈、自定义标签等），实现 “同一种效果，不同场景下有不同表现”（如 “正常回血”“暴击回血”“削弱后回血”）；
- 支持延迟应用、来源追溯、快照固定等高级逻辑，是 GAS 实现复杂效果的核心载体。

#### 4.5.9.1 SetByCaller

**对文中本节内容的解释**：

这段话详细解释了 GAS 中 `SetByCaller` 的本质、存储方式和使用规则，核心是它作为 **“GameplayEffectSpec（GESpec）的动态数值容器”**，能在运行时灵活传递浮点值，解决 “效果参数需要根据实时逻辑动态生成” 的问题（比如蓄力技能的伤害、根据距离变化的治疗量）。

##### 先通俗理解 `SetByCaller`：

把 `SetByCaller` 看作 GESpec 自带的 “临时记事本”—— 上面可以记一些 “标签 / 名称” 和对应的数字，比如 “蓄力时间 = 2 秒”“伤害倍率 = 1.5”。这些数字既可以给 GE 的 Modifier 用（比如 “伤害 = 基础值 × 蓄力倍率”），也可以给其他计算逻辑用（比如自定义伤害计算），不用提前在 GE 模板里写死，而是运行时临时填。

##### 一、核心本质：GESpec 上的 “键值对数值存储”

`SetByCaller` 的数值存在 GESpec 的两个 `TMaps`（键值对表）里，对应两种 “索引方式”：

| 存储容器                    | 索引类型（键）                                        | 核心特点                                                     |
| --------------------------- | ----------------------------------------------------- | ------------------------------------------------------------ |
| `TMap<FGameplayTag, float>` | GameplayTag（如 `"Ability.Charge.DamageMultiplier"`） | 用于给 **GE 的 Modifier** 调用（必须提前在 GE 里定义对应 Tag） |
| `TMap<FName, float>`        | FName（如 `"ChargeTime"`）                            | 用于 **通用数值传递**（无需提前定义，灵活传递给自定义计算）  |

简单说：Tag 索引的是 “给 Modifier 用的专用数值”，FName 索引的是 “给其他逻辑用的通用数值”。

##### 二、两种使用场景的规则差异

这是 `SetByCaller` 最关键的部分 —— 给 Modifier 用，和给其他逻辑用，规则完全不同，直接影响使用时是否会报错。

###### 1. 场景一：给 GE 的 `Modifier` 用（最常见）

**核心规则：必须提前定义，缺失会报错**

- 步骤：
  1. 先在 GE 模板中，把某个 Modifier 的类型设为 `SetByCaller`，并指定对应的 `GameplayTag`（比如 Modifier 是 “伤害”，Tag 设为 `"Damage.SetByCaller.Value"`）；
  2. 生成 GESpec 时，通过代码 / 蓝图给 GESpec 的 `TMap<FGameplayTag, float>` 中，添加 “该 Tag + 具体数值”（比如 Tag 对应的值设为 200，代表这次伤害是 200）；
  3. 应用 GESpec 时，Modifier 会自动读取这个 Tag 对应的数值，用于计算属性修改。
- 关键风险：
  如果 GE 里定义了 `SetByCaller` Modifier，但生成 GESpec 时没加对应的 Tag 和数值，游戏会**抛出运行时错误**，并默认返回 0。
  → 比如 Modifier 是 “伤害 = SetByCaller 值 ÷ 2”，若没传值，会变成 “0 ÷ 2”，虽然不会崩溃，但逻辑会出错（伤害为 0）。
- 举例：
  蓄力技能 “重击”：
  - GE 中定义 “伤害 Modifier” 为 `SetByCaller`，Tag 是 `"Ability.Smash.Damage"`；
  - 技能释放时，根据蓄力时间计算伤害（蓄力 2 秒 → 伤害 300），把 `("Ability.Smash.Damage", 300)` 存入 GESpec 的 Tag 映射表；
  - 应用 GESpec 时，Modifier 读取 300 这个值，最终造成 300 点伤害。

###### 2. 场景二：给其他逻辑用（通用数值传递）

**核心规则：无需提前定义，缺失返回默认值**

- 用途：传递给 `GameplayEffectExecutionCalculations`（执行计算）或 `ModifierMagnitudeCalculations`（修改器数值计算）等自定义逻辑，比如传递 “攻击距离”“暴击倍率” 等辅助参数。
- 步骤：
  1. 生成 GESpec 时，直接给 `TMap<FName, float>` 加 “FName + 数值”（比如 `("AttackDistance", 500.0f)`，代表攻击距离 500 单位）；
  2. 在自定义计算类中，读取 GESpec 中这个 FName 对应的数值（如果没找到，返回开发者设置的默认值，比如 0，并可选输出警告日志）。
- 优势：灵活，不用在 GE 里提前配置，适合临时传递逻辑参数。
- 举例：
  自定义伤害计算需要 “根据攻击距离衰减伤害”：
  - 生成 GESpec 时，把实际攻击距离（如 300 单位）存入 FName 映射表：`("Distance", 300.0f)`；
  - 在 `ModifierMagnitudeCalculations` 类中，读取 `"Distance"` 对应的值，计算衰减后的伤害（比如距离越远，伤害越低）。

##### 三、总结：`SetByCaller` 的核心价值与使用注意

- **核心价值**：打破 GE 模板的 “静态限制”，让效果参数能根据运行时逻辑动态生成（如蓄力、距离、暴击等），是连接 “技能逻辑” 和 “效果计算” 的关键桥梁。

- 使用注意

  ：

  1. 给 Modifier 用：必须提前在 GE 里定好 Tag，GESpec 必须传对应值，否则报错；
  2. 给其他逻辑用：用 FName 自由传值，缺失时处理好默认值，避免逻辑异常。

简单说：`SetByCaller` 就是 GESpec 的 “动态钱包”—— 给 Modifier 花的钱要提前报备（定 Tag），给其他地方花的钱可以灵活支配（FName）。

### 4.5.10 GameplayEffectContext

要理解 `GameplayEffectContext`（简称 GEC），可以先把它看作 **`GameplayEffectSpec`（GESpec）的 “身份档案 + 数据快递袋”**：既记录效果的 “来源” 和 “目标” 这些核心身份信息，又能灵活装自定义数据，在 GAS 各个模块（如伤害计算、特效触发）之间传递。下面分 “核心作用”“为什么要继承”“继承步骤”“实际案例” 四部分拆解：

#### 一、先搞懂：`GameplayEffectContext` 到底是干什么的？

`FGameplayEffectContext` 是 GESpec 自带的一个结构体，核心作用有两个：

1. **记录 “效果的基础身份信息”（必带）**
   - **创建者（Instigator）**：记录是谁创建了这个 GESpec（比如释放火球术的玩家、触发陷阱的敌人），用于追溯 “效果来源”（比如 “伤害是谁造成的”）。
   - **目标数据（TargetData）**：记录效果要作用的目标信息（比如单个敌人的位置、多个敌人的列表），是 “命中检测” 和 “效果应用” 的关键依据。
2. **作为 “通用数据传递载体”（可扩展）**
   默认的 GEC 只有基础信息，但游戏中常需要传递更多自定义数据（比如 “技能是否暴击”“伤害来源的武器 ID”“霰弹枪的多个命中点”）。此时就需要 **继承 GEC**，给它加新的字段，让它能装这些自定义数据，在 `ModifierMagnitudeCalculation`（伤害计算）、`GameplayCue`（特效触发）、`AttributeSet`（属性修改）等模块之间流转。

#### 二、为什么需要继承 `GameplayEffectContext`？

默认的 `FGameplayEffectContext` 就像一个 “简易快递袋”，只能装 “寄件人（创建者）” 和 “收件人地址（TargetData）”；但实际游戏中，我们需要 “更大的快递箱”，装更多东西：

- 比如霰弹枪射击：需要传递 “5 个命中敌人的位置 + 每个位置的伤害衰减值”，默认 GEC 存不下这么多 TargetData；
- 比如暴击伤害：需要传递 “暴击倍率（1.5x）+ 暴击触发的特效 ID”，默认 GEC 没有这些字段；
- 比如武器专属效果：需要传递 “武器类型（弓箭 / 枪械）+ 武器等级”，用于计算不同武器的伤害加成。

这时就必须继承 GEC，自定义一个 “扩展版快递箱”，添加这些新字段。

#### 三、继承 `GameplayEffectContext` 的 6 个关键步骤（缺一不可）

继承 GEC 不是简单加个字段就行，需要让虚幻引擎 “认识” 这个自定义结构体，确保它能正常复制、网络同步、被各个模块调用。具体步骤如下：

| 步骤 | 操作内容                                                     | 核心目的                                                     |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1    | 继承 `FGameplayEffectContext`                                | 基于父类扩展，保留基础功能（创建者、TargetData），新增自定义字段（如 `bIsCritical` 标记暴击、`MultipleHitData` 存多命中点）。 |
| 2    | 重写 `GetScriptStruct()`                                     | 告诉引擎 “这个自定义结构体的类型是什么”，让引擎能识别它（比如在蓝图中显示、在网络同步时判断类型）。 示例：返回自定义结构体的 `StaticStruct()`（如 `FMyGameplayEffectContext::StaticStruct()`）。 |
| 3    | 重写 `Duplicate()`                                           | GEC 在传递过程中（如从技能传到抛射物）会被复制，`Duplicate()` 用于 “深拷贝”—— 不仅复制父类的基础信息，还要复制自定义字段（比如把 `MultipleHitData` 完整复制给新实例）。 |
| 4    | （可选）重写 `NetSerialize()`                                | 如果自定义数据需要**网络同步**（比如服务器计算的暴击信息要同步给客户端播特效），必须重写这个函数，定义 “如何把自定义字段转换成网络数据”（序列化）和 “如何从网络数据还原”（反序列化）。 如果数据只在本地用（如单机游戏），可跳过。 |
| 5    | 实现 `TStructOpsTypeTraits`                                  | 这是虚幻的 “结构体操作 trait”，用于告诉引擎这个自定义结构体支持哪些功能（比如是否支持序列化、是否支持复制）。 必须按父类 `FGameplayEffectContext` 的写法，给自定义结构体添加 `TStructOpsTypeTraits` 特化，确保引擎能正确处理它。 |
| 6    | 在 `AbilitySystemGlobals` 中重写 `AllocGameplayEffectContext()` | `AbilitySystemGlobals` 是 GAS 的 “全局配置类”，`AllocGameplayEffectContext()` 用于创建 GEC 实例。重写它后，当 GAS 生成 GESpec 时，会自动创建我们自定义的 GEC（而不是默认的），确保所有 GESpec 都用扩展版的 “数据快递袋”。 |

#### 四、实际案例：GASShooter 中的自定义 GEC

GASShooter 是 Epic 官方的 GAS 示例项目，它继承 `FGameplayEffectContext` 做了一件关键的事：**扩展 TargetData 以支持霰弹枪的多目标命中**。

##### 场景需求：

霰弹枪射击时，会同时命中多个敌人（比如 3 个），需要：

1. 给每个命中的敌人都应用伤害效果；
2. 在每个命中位置都播放 “命中特效”（GameplayCue）。

##### 默认 GEC 的问题：

默认 `FGameplayEffectContext` 的 TargetData 只能存 “单个目标” 或 “简单的目标列表”，无法精确记录 “每个目标的命中位置、伤害衰减” 等细节，导致无法针对性触发特效和计算伤害。

##### 自定义 GEC 的解决思路：

1. 继承 `FGameplayEffectContext`，新增字段 `TArray<FHitResult> MultipleHitResults`（存所有命中点的详细信息，包括位置、命中的敌人）；
2. 重写 `Duplicate()` 和 `NetSerialize()`，确保多命中数据能复制和网络同步；
3. 在 `AbilitySystemGlobals` 中让 GAS 自动创建这个自定义 GEC；
4. 当霰弹枪命中时，把所有 `FHitResult` 存入自定义 GEC 的 `MultipleHitResults`；
5. 在 `GameplayCue` 中读取这个字段，遍历每个命中点，分别播放特效；
6. 在伤害计算（`ModifierMagnitudeCalculation`）中，根据每个命中点的距离，计算伤害衰减（比如离枪口越远，伤害越低）。

#### 五、总结：`GameplayEffectContext` 的核心价值

- **基础层**：记录效果的 “来源” 和 “目标”，是 GAS 追溯责任、应用效果的基础；
- **扩展层**：通过继承成为 “万能数据袋”，解决 “跨模块传递自定义数据” 的需求（如多目标、暴击信息、武器参数）；
- **关键意义**：让 GAS 的各个模块（技能、效果、计算、特效）能 “共享数据”，避免硬编码，让逻辑更灵活可扩展。

简单说，GEC 就像 GAS 系统中的 “快递系统”—— 基础版负责送 “身份信息”，扩展版能送 “各种自定义包裹”，是连接不同模块的关键数据纽带

### 4.5.11 Modifier Magnitude Calculation

**对文中内容的解释**：

要理解 **ModifierMagnitudeCalculations（简称 MMC）**，核心是抓住它的定位：**为 GameplayEffect（GE）的 Modifier 计算 “具体生效数值” 的 “可预测工具”**。它不像 ExecutionCalculation（EC）那样能处理复杂逻辑（如多属性联动、伤害结算），但胜在稳定可预测，且能灵活读取 GAS 核心数据（属性、标签、SetByCaller 等），是 GE 实现 “动态数值调整” 的核心组件。

#### 一、MMC 的核心特性与关键概念

在看代码示例前，先理清 MMC 最关键的 3 个特性，这是理解后续流程的基础：

##### 1. 核心作用：只做一件事 —— 返回浮点值

MMC 的唯一核心逻辑在 `CalculateBaseMagnitude_Implementation()` 函数中，最终返回一个浮点值（比如 “减少 40 点魔法值”“增加 20% 攻击力”）。这个值会被 GE 的 Modifier 进一步处理（比如乘以 Modifier 自带的 “系数”），最终作用于目标的 Attribute（属性，如魔法值、攻击力）。

##### 2. 关键能力：捕获 Attribute（属性）

MMC 可以读取 **源（Source，如技能释放者）** 或 **目标（Target，如被攻击的敌人）** 的 Attribute（如当前魔法值、最大生命值），但需要先 “声明要捕获的属性”，否则会报错。
捕获属性时有个核心开关：**是否 Snapshot（快照）**—— 决定属性值 “何时被固定” 以及 “是否随后续变化更新”。

之前的表格可以用更通俗的语言翻译：

| Snapshot（快照） | 捕获对象（源 / 目标） | 属性值 “何时被记录” | 后续属性变化时是否自动更新 | 典型场景                                                     |
| ---------------- | --------------------- | ------------------- | -------------------------- | ------------------------------------------------------------ |
| 是               | 源（Source）          | GE Spec 创建时      | 否（固定初始值）           | 技能伤害基于 “释放时的攻击力”（即使后续攻击力变化，伤害不变） |
| 是               | 目标（Target）        | GE 应用时           | 否（固定应用时的值）       | debuff 基于 “被命中时的最大生命值”（后续目标血量变化不影响 debuff 强度） |
| 否               | 源（Source）          | GE 应用时           | 是（实时更新）             | 光环效果基于 “当前释放者的智力”（释放者智力变化，光环强度同步变） |
| 否               | 目标（Target）        | GE 应用时           | 是（实时更新）             | 毒伤基于 “目标当前魔法值”（目标用掉魔法后，毒伤会变弱）      |

##### 3. 数据来源：完全访问 GameplayEffectSpec（GE Spec）

GE Spec 是 GE 的 “实例化蓝图”，包含了 GE 的所有临时数据。MMC 可以从 Spec 中读取：

- 捕获的 Attribute 值（上面说的 “源 / 目标属性”）；
- SetByCaller 传递的自定义数值（比如技能等级对应的伤害系数）；
- 源 / 目标的 GameplayTag（比如 “是否对毒素弱点” 标签）。

#### 二、结合代码示例：拆解 MMC 的完整工作流程

以文档中 “毒系减少魔法值” 的 `UPAMMC_PoisonMana` 为例，一步步看 MMC 如何工作，以及它和其他 GAS 组件的交互。

##### 场景背景

我们要实现一个 “毒系效果”：对目标施加持续掉魔法值的 debuff，掉血幅度受两个条件影响：

1. 目标当前魔法值占最大魔法值的比例（超过 50% 则掉血翻倍）；
2. 目标是否有 “对毒系魔法弱点” 标签（有则再翻倍）。

##### 步骤 1：MMC 构造函数 —— 声明 “要捕获的属性”

在 MMC 的构造函数中，必须明确告诉 GAS：“我要读取哪些属性”，否则后续获取属性时会报 “缺失 Spec” 错误。

```c++
UPAMMC_PoisonMana::UPAMMC_PoisonMana()
{
    // 1. 定义要捕获的“目标当前魔法值（Mana）”
    ManaDef.AttributeToCapture = UPAAttributeSetBase::GetManaAttribute(); // 来自 AttributeSet
    ManaDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target; // 捕获“目标”的属性
    ManaDef.bSnapshot = false; // 不快照：实时读取目标当前的 Mana
    
    // 2. 定义要捕获的“目标最大魔法值（MaxMana）”
    MaxManaDef.AttributeToCapture = UPAAttributeSetBase::GetMaxManaAttribute(); // 来自 AttributeSet
    MaxManaDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target; // 捕获“目标”的属性
    MaxManaDef.bSnapshot = false; // 不快照：实时读取目标当前的 MaxMana
    
    // 3. 把两个属性添加到“待捕获列表”（关键！不添加会报错）
    RelevantAttributesToCapture.Add(ManaDef);
    RelevantAttributesToCapture.Add(MaxManaDef);
}
```

##### 这里涉及的 GAS 组件：

- **AttributeSet（UPAAttributeSetBase）**：提供要捕获的属性（Mana、MaxMana）—— 属性的定义和存储都在 AttributeSet 中，MMC 只是 “读取” 它们。
- **FGameplayEffectAttributeCaptureDefinition**：相当于 “属性捕获的配置单”，明确 “读哪个属性、读谁的、是否快照”。

##### 步骤 2：核心计算函数 —— 根据条件返回最终数值

`CalculateBaseMagnitude_Implementation()` 是 MMC 的核心，在这里完成 “读取数据 → 判断条件 → 计算数值” 的逻辑。

```c++
float UPAMMC_PoisonMana::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec & Spec) const
{
    // 1. 从 GE Spec 中读取“源/目标的 GameplayTag 集合”（用于判断弱点标签）
    const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags(); // 源（比如施法者）的标签
    const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags(); // 目标的标签
    
    // 2. 准备“属性评估参数”——传递标签信息，确保计算时能识别标签影响
    FAggregatorEvaluateParameters EvaluationParameters;
    EvaluationParameters.SourceTags = SourceTags;
    EvaluationParameters.TargetTags = TargetTags;
    
    // 3. 读取捕获的“目标当前 Mana”（并做安全校验：不能为负）
    float Mana = 0.f;
    GetCapturedAttributeMagnitude(ManaDef, Spec, EvaluationParameters, Mana); // 从 Spec 中获取之前声明的 Mana
    Mana = FMath::Max<float>(Mana, 0.0f); // 防止出现负魔法值
    
    // 4. 读取捕获的“目标最大 Mana”（安全校验：避免除零）
    float MaxMana = 0.f;
    GetCapturedAttributeMagnitude(MaxManaDef, Spec, EvaluationParameters, MaxMana);
    MaxMana = FMath::Max<float>(MaxMana, 1.0f); // 最大 Mana 至少为 1，防止后续除法出错
    
    // 5. 基础掉魔法值：-20（负号表示“减少”）
    float Reduction = -20.0f;
    
    // 6. 条件 1：目标当前 Mana 超过最大 Mana 的 50% → 掉血翻倍
    if (Mana / MaxMana > 0.5f)
    {
        Reduction *= 2; // 变成 -40
    }
    
    // 7. 条件 2：目标有“Status.WeakToPoisonMana”标签 → 再翻倍
    if (TargetTags->HasTagExact(FGameplayTag::RequestGameplayTag(FName("Status.WeakToPoisonMana"))))
    {
        Reduction *= 2; // 变成 -80
    }
    
    // 8. 返回最终数值（比如 -80，表示“每次生效减少 80 点魔法值”）
    return Reduction;
}
```

##### 这里涉及的 GAS 组件：

- **FGameplayEffectSpec（Spec）**：MMC 的 “数据仓库”—— 提供标签（SourceTags/TargetTags）、捕获的属性值（通过 `GetCapturedAttributeMagnitude` 读取），甚至可以读取 SetByCaller 数值（比如技能等级对应的系数）。
- **GameplayTag**：用于条件判断（目标是否弱点）—— 标签是 GAS 中 “状态 / 特性” 的统一标识，MMC 可以通过 Spec 读取源 / 目标的标签集合。

##### 步骤 3：MMC 与 GameplayEffect（GE）的结合

MMC 本身不直接生效，必须 “挂载” 到 GE 的 Modifier 上，才能将计算出的数值作用于目标。

##### 具体操作（蓝图 / C++）：

1. 创建一个 **持续型 GE**（比如 “毒系掉蓝 debuff”，持续 5 秒，每 1 秒生效一次）；
2. 在 GE 中添加一个 **Modifier**（类型为 “修改 Attribute”，目标属性是 “Mana”）；
3. 在 Modifier 的 “Magnitude Calculation” 中，选择我们定义的 `UPAMMC_PoisonMana`；
4. （可选）在 Modifier 中设置 “系数（Coefficient）”—— 比如设置为 1.5，那么 MMC 返回的 -80 会变成 -120（最终掉蓝 120）。

##### 步骤 4：最终生效流程（完整链路）

1. **触发 GE**：比如玩家释放 “毒系技能”，通过 Ability 生成该 GE 的 Spec，并应用到目标身上；

2. MMC 执行

   ：GE 每次生效（比如每 1 秒）时，会调用 MMC 的

    

   ```
   CalculateBaseMagnitude
   ```

   ：

   - MMC 从 Spec 中读取目标当前的 Mana/MaxMana（因 `bSnapshot=false`，实时读取）；
   - MMC 检查目标是否有弱点标签，计算出最终掉蓝值（比如 -80）；

3. **Modifier 处理**：GE 的 Modifier 接收 MMC 的 -80，若 Modifier 系数为 1.5，则最终数值为 -120；

4. **更新 Attribute**：Modifier 将 -120 作用于目标的 AttributeSet 中的 Mana 属性，目标的当前魔法值减少 120。

#### 三、MMC 的优势与注意事项

##### 优势

1. **可预测性**：逻辑仅依赖输入数据（属性、标签、SetByCaller），无副作用，适合用于需要稳定数值的场景（如 debuff、光环）；
2. **灵活性**：支持所有类型的 GE（即刻、持续、无限、周期性），且能读取 Spec 的几乎所有数据；
3. **低耦合**：MMC 是独立类，可复用（比如多个毒系 GE 都用同一个 MMC，只需调整 Modifier 系数）。

##### 注意事项

1. **必须声明捕获属性**：如果要读取 Attribute，必须在 MMC 构造函数中将 `FGameplayEffectAttributeCaptureDefinition` 添加到 `RelevantAttributesToCapture`，否则会报 “缺失 Spec” 错误；
2. **需手动处理数值安全**：GAS 不会自动执行 AttributeSet 的 `PreAttributeChange()`（比如 “魔法值不能低于 0” 的限制），所以 MMC 中要自己做 Clamp（如 `Mana = FMath::Max(Mana, 0)`）；
3. **Snapshot 影响实时性**：若 `bSnapshot=true`，属性值会固定在 “捕获时”，后续目标属性变化不会影响 MMC 结果（比如用快照捕获目标最大生命值后，目标血量被打低，debuff 强度仍不变）。

通过这个例子，你可以清晰看到 MMC 在 GAS 数值链路中的核心作用：它是 “GE 数值” 的 “计算器”，连接了 AttributeSet（属性源）、GameplayTag（状态判断）和 GameplayEffect（最终生效），是实现动态、灵活数值逻辑的关键组件。

### 4.5.12 Gameplay Effect Execution Calculation

`GameplayEffectExecutionCalculation`（简称`ExecCalc`）是 Unreal Engine 中`GameplayEffect`修改`AbilitySystemComponent`（ASC）的核心机制之一，其设计聚焦于灵活性和强大的定制能力。以下从核心特性、使用规则、实现细节和应用场景展开说明：

#### 一、核心特性与定位

- **强定制性**：与`ModifierMagnitudeCalculation`（MMC）相比，`ExecCalc`不仅能捕获属性（`Attribute`）并选择性创建快照（Snapshot），还支持同时修改**多个属性**，理论上可实现任何程序员需要的逻辑（如复杂伤害计算、多属性联动修改等）。
- **技术限制**：作为代价，`ExecCalc`必须在**C++ 中实现**，且因其逻辑灵活性，无法被引擎预测（`Unpredictable`），这对网络同步设计有一定影响。

#### 二、适用场景与限制

- **仅支持两种`GameplayEffect`**：`ExecCalc`只能被**即刻生效（Instant）** 和**周期性生效（Periodic）** 的`GameplayEffect`使用。插件中所有含 “Execute” 的逻辑（如`ExecuteGameplayEffect`）通常均与此相关。
- **网络调用规则**：对于`Local Predicted`、`Server Only`和`Server Initiated`类型的`GameplayAbility`，`ExecCalc`**仅在服务端调用**，客户端无需重复执行，需注意网络同步逻辑设计。

#### 三、属性捕获与快照（Snapshot）机制

`ExecCalc`通过 “捕获属性” 获取源（Source）或目标（Target）的`Attribute`值，捕获时机与 “是否启用快照”“是源还是目标” 相关，具体规则如下：

| 快照启用？ | 源 / 目标      | 捕获时机（基于`GameplayEffectSpec`） |
| ---------- | -------------- | ------------------------------------ |
| 是         | 源（Source）   | `GameplayEffectSpec`**创建时**       |
| 是         | 目标（Target） | `GameplayEffectSpec`**应用时**       |
| 否         | 源（Source）   | `GameplayEffectSpec`**应用时**       |
| 否         | 目标（Target） | `GameplayEffectSpec`**应用时**       |

**关键注意点**：

- 捕获属性时，引擎会基于 ASC 中现有修饰器（`Modifier`）重新计算`Attribute`的`CurrentValue`，但**不会触发**`AbilitySet`中的`PreAttributeChange()`回调。因此，所有属性限制逻辑（如数值 clamping）必须在`ExecCalc`中手动实现。

#### 四、实现方式：属性捕获的结构体定义

为设置属性捕获规则，需遵循 Epic 在 ActionRPG 样例中的实践：**为每个`ExecCalc`定义一个专属结构体**，用于声明如何捕获属性，并在结构体构造函数中创建副本（Copy）。

- **命名要求**：每个结构体必须有**唯一名称**（因共享命名空间），否则会导致属性捕获错误（如读取到错误的属性值）。

- 示例结构（伪代码）：

  ```cpp
  // 为"伤害计算"定义的捕获结构体
  struct FExecCalc_DamageCaptureDefinition {
      FExecCalc_DamageCaptureDefinition() {
          // 声明需要捕获的属性（如源的攻击力、目标的防御力）
          RelevantAttributesToCapture.Add(UStrengthAttribute::GetStaticFName());
          RelevantAttributesToCapture.Add(UDefenseAttribute::GetStaticFName());
          // 设置是否启用快照
          bSnapshotSource = true;
          bSnapshotTarget = false;
      }
  };
  ```

#### 五、典型应用场景

`ExecCalc`最常见的用途是实现**复杂伤害 / 治疗公式**，尤其当计算逻辑依赖多个源和目标的属性时：

- 例如：从`GameplayEffectSpec`的`SetByCaller`中读取基础伤害，结合源的 “暴击倍率” 属性和目标的 “护盾值” 属性，最终计算实际伤害值（如`实际伤害 = 基础伤害 × 暴击倍率 - 目标护盾值`）。
- 样例参考：ActionRPG 项目中，`ExecCalc_Damage`通过捕获目标的护盾属性，对`SetByCaller`传入的基础伤害进行减免计算，最终应用到目标的生命值属性上。

#### 总结

`ExecCalc`是处理复杂属性修改逻辑的 “终极工具”，其灵活性使其成为实现伤害、治疗、多属性联动等核心玩法的首选，但需注意 C++ 实现成本、网络调用规则及属性捕获的细节（如手动处理数值限制）。

### 4.5.13 自定义应用需求

### 4.5.14 花费(Cost)GameplayEffect

#### 关于Cost GE

`Cost GE` 并非 Unreal Engine 中 `GameplayAbility`（GA）的 “特殊内置选项”，而是**设计师 / 开发者自定义的 `GameplayEffect`（GE）**，只是因其功能（作为能力激活的 “花费消耗”）而被赋予了 “Cost GE” 这个约定俗成的名称。

##### 一、本质：Cost GE 是 “功能特定的自定义 GE”

`GameplayAbility` 类中确实有专门关联 “花费” 的接口（如 `GetCostGameplayEffect()`、`ApplyCost()` 等），但这些接口本质上是**对 “一个普通 GE” 的调用逻辑**。这个被关联的 GE 本身没有任何 “特殊标记”，它之所以被称为 “Cost GE”，是因为开发者将其**设计为 “仅用于实现能力激活消耗” 的功能**。

例如：

- 一个普通的即刻 GE，若其 Modifier 是 “法力值 -= 10”，并被配置到某个 GA 的 “Cost” 选项中，它就成为了该 GA 的 Cost GE。
- 若将同一个 GE 配置到 “Cooldown” 选项中，它就会被当作 “冷却 GE”（尽管功能上不匹配，仅举例说明逻辑）。

##### 二、为何需要 “专门的 Cost GE”？

`GameplayAbility` 激活时，引擎会自动检查并应用关联的 Cost GE（通过 `CheckCost()` 和 `ApplyCost()` 等函数），这要求 Cost GE 满足一些 “功能约定”（非引擎强制，但实践中必须遵守）：

1. **类型必须是 “即刻（Instant）”**：确保激活时立即扣除属性（如法力值、体力），而非持续或无限期生效。
2. **包含 “属性减值 Modifier”**：通常是对资源类属性（如 `Mana`、`Stamina`）的减法操作（如 `Additive` 类型的负值：`-10`）。
3. **可预测性**：推荐使用 MMC 而非 ExecCalc，确保客户端能预测消耗，避免网络同步问题。

这些约定是开发者为了实现 “能力花费” 功能而设计的，而非引擎对 Cost GE 有特殊定义。

##### 三、与 GA 的关联方式

`GameplayAbility` 通过以下方式关联 Cost GE（以 C++ 为例）：

1. **直接指定 GE 模板**：在 GA 子类中定义 `UPROPERTY` 引用一个 GE 资源，作为默认 Cost GE：

   ```cpp
   UPROPERTY(EditDefaultsOnly, Category = "Cost")
   TSubclassOf<UGameplayEffect> CostGameplayEffect;
   ```

   然后重写 `GetCostGameplayEffect()` 返回该模板，引擎会在激活时实例化并应用这个 GE。

2. **动态生成 GE**：重写 `GetCostGameplayEffect()` 在运行时创建 GE 实例（如根据等级动态调整消耗值），如前文提到的 “复用 Cost GE” 技巧。

##### 总结

- **Cost GE 是 “用途定义的 GE”**：它是普通的 `GameplayEffect`，只因被用于实现 “能力激活消耗” 而得名。
- **引擎提供关联逻辑，但不限制 GE 本身**：`GameplayAbility` 有专门的接口处理 Cost GE 的检查和应用，但 Cost GE 的具体功能（消耗什么属性、消耗多少）完全由开发者定义。

这种设计体现了 GAS 的灵活性：通过 “通用组件 + 功能约定” 的方式，让开发者可以自由定制能力系统的各种机制（包括花费、冷却、效果等）

#### **GameplayAbility 中的 Cost GameplayEffect（花费效果）详解**

在 Unreal Engine 的 Gameplay Ability System（GAS）中，`GameplayAbility`（GA）的 “花费（Cost）” 是控制能力激活的核心机制之一，通过关联一个`GameplayEffect`（GE）实现。以下从基础概念、特性及复用技巧展开说明：

##### 一、基础概念与核心特性

- **定义**：Cost GE 是`GameplayAbility`的可选配置，用于指定激活该能力所需消耗的`Attribute`（如法力值、体力值等）。若 GA 未配置有效的 Cost GE，则该能力**无法被激活**。
- **GE 类型要求**：Cost GE 必须是**即刻生效（Instant）** 的`GameplayEffect`，且需包含一个或多个对`Attribute`的 “减值 Modifier”（如 “法力值 -= 10”）。
- **可预测性**：默认情况下，Cost GE 是可预测的（Predictable），这对网络同步至关重要。因此，**不建议在 Cost GE 中使用`ExecutionCalculations`**（因其不可预测），而推荐使用`ModifierMagnitudeCalculation`（MMC）—— MMC 既能处理复杂计算，又能保持可预测性。

##### 二、Cost GE 的复用技巧

初期使用时，通常为每个有花费的 GA 配置独立的 Cost GE，但进阶用法中可通过复用 Cost GE 减少冗余，尤其适用于**实例化（Instanced）的 Ability**。核心思路是：多个 GA 共享同一个 Cost GE，通过动态修改`GameplayEffectSpec`中的数据（花费值）适配不同 GA 的需求。

###### 技巧 1：使用 MMC 从 GA 实例中读取花费值

这是最简单的复用方式。创建一个 MMC，使其从`GameplayEffectSpec`关联的`GameplayAbility`实例中读取具体花费值，步骤如下：

1. **在 GA 子类中定义花费值**：
   在自定义的`GameplayAbility`子类（如`UPGGameplayAbility`）中添加可配置的花费属性（通常用`FScalableFloat`支持不同等级的花费调整）：

   ```cpp
   UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cost")
   FScalableFloat Cost; // 可在蓝图/编辑器中配置不同等级的花费值
   ```

2. **实现 MMC 逻辑**：
   创建一个 MMC 子类（如`UPGMMC_HeroAbilityCost`），重写计算方法，从`GameplayEffectSpec`中获取 GA 实例，进而读取其`Cost`值：

   ```cpp
   float UPGMMC_HeroAbilityCost::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec& Spec) const
   {
       // 从Spec的上下文获取关联的GA实例（非复制版本）
       const UPGGameplayAbility* Ability = Cast<UPGGameplayAbility>(Spec.GetContext().GetAbilityInstance_NotReplicated());
       
       if (!Ability)
       {
           return 0.0f; // 若获取失败，返回0（不消耗）
       }
       
       // 根据GA的当前等级返回对应的花费值
       return Ability->Cost.GetValueAtLevel(Ability->GetAbilityLevel());
   }
   ```

3. **关联 Cost GE 与 MMC**：
   在复用的 Cost GE 中，将其 Modifier 的 “Magnitude Calculation” 指定为上述 MMC。这样，不同 GA 实例激活时，Cost GE 会自动通过 MMC 读取各自配置的`Cost`值，实现 “一个 GE 适配多个 GA”。

###### 技巧 2：重写`GetCostGameplayEffect()`动态创建 GE

通过重写`UGameplayAbility`的`GetCostGameplayEffect()`函数，在运行时动态生成 Cost GE，并根据当前 GA 的配置设置花费值。这种方式更灵活，但实现复杂度略高：

```cpp
const UGameplayEffect* UPGGameplayAbility::GetCostGameplayEffect() const
{
    // 1. 检查是否已缓存动态生成的Cost GE，避免重复创建
    if (CachedCostGE)
    {
        return CachedCostGE;
    }
    
    // 2. 动态创建GameplayEffect（或从模板复制）
    UGameplayEffect* CostGE = NewObject<UGameplayEffect>(GetTransientPackage());
    CostGE->DurationPolicy = EGameplayEffectDurationType::Instant; // 确保是即刻生效
    
    // 3. 根据当前GA的花费配置，添加减值Modifier
    FGameplayModifierInfo Modifier;
    Modifier.Attribute = UMyAttributes::GetManaAttribute(); // 例如消耗法力值
    Modifier.ModifierOp = EGameplayModOp::Additive; // 减值（用负值实现）
    Modifier.Magnitude.Value = -Cost.GetValueAtLevel(GetAbilityLevel()); // 花费值取负
    
    CostGE->Modifiers.Add(Modifier);
    
    // 4. 缓存并返回
    CachedCostGE = CostGE;
    return CachedCostGE;
}
```

这种方式的核心是：在运行时根据 GA 的具体配置（如`Cost`值）动态构建 Cost GE，无需预先定义多个 GE 模板。

##### 总结

Cost GE 是控制能力激活的关键机制，其设计需兼顾可预测性（优先用 MMC）和复用性。通过 “MMC 读取 GA 实例属性” 或 “动态创建 GE” 两种技巧，可有效减少冗余 GE 配置，尤其适合大型项目中多能力共享相似花费逻辑的场景。

### 4.5.15 冷却(Cooldown)GameplayEffect

**对文中内容的解释**：

`Cooldown GameplayEffect`（简称`Cooldown GE`）是控制技能冷却逻辑的核心机制，用于限制技能激活后的再次使用时机。其设计与`Cost GE`有相似之处，但核心逻辑依赖`GameplayTag`而非属性修改，以下从基础概念、特性及复用技巧展开说明：

#### 一、基础概念与核心特性

- **定义**：`Cooldown GE`是`GameplayAbility`的可选配置，用于指定技能激活后必须等待的冷却时间。若技能处于冷却中（即对应的`Cooldown Tag`存在），则该技能**无法被激活**。
- **GE 类型要求**：`Cooldown GE`必须是**持续型（Duration）** 的`GameplayEffect`，且**不包含任何属性 Modifier**。其核心作用是通过 “持续时间内授予`Cooldown Tag`” 来标识冷却状态。
- **冷却标识Cooldown Tag**：`Cooldown GE`的`GrantedTags`中必须包含独一无二的`GameplayTag`（称为`Cooldown Tag`）。技能是否处于冷却中，本质是检查该 Tag 是否存在于`ASC`中。
  - 若多个技能共享冷却（如同一插槽的可切换技能），可共用同一个`Cooldown Tag`。
- **可预测性**：默认情况下`Cooldown GE`是可预测的（Predictable），建议保留这一特性（避免使用`ExecutionCalculations`），`MMC`是复杂冷却时间计算的推荐方案。

#### 二、Cooldown GE 的复用技巧

初期通常为每个技能配置独立的`Cooldown GE`，但进阶用法中可通过复用`Cooldown GE`减少冗余，尤其适用于**实例化（Instanced）的 Ability**。核心思路是：多个技能共享同一个`Cooldown GE`模板，通过动态修改`GameplayEffectSpec`中的`Cooldown Tag`和冷却时间（从技能自身配置读取）实现适配。

##### 技巧 1：使用 SetByCaller 动态设置冷却时间

通过`SetByCaller`在运行时为`Cooldown GE`传入冷却时间，步骤如下：

1. **在 GA 子类中定义配置项**：
   包含冷却时间（支持随等级变化）、专属`Cooldown Tag`，以及临时标签容器（用于合并标签）：

   ```cpp
   UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cooldown")
   FScalableFloat CooldownDuration; // 冷却时间（可随等级配置）
   
   UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cooldown")
   FGameplayTagContainer CooldownTags; // 技能专属的Cooldown Tag
   
   // 临时容器：用于存储"Cooldown Tags"与GE自身标签的并集
   UPROPERTY()
   FGameplayTagContainer TempCooldownTags;
   ```

2. **重写`GetCooldownTags()`合并标签**：
   技能检查冷却时会读取`GetCooldownTags()`返回的标签，需将技能专属`Cooldown Tags`与`Cooldown GE`自身的标签合并（避免冲突）：

   ```cpp
   const FGameplayTagContainer* UPGGameplayAbility::GetCooldownTags() const
   {
       FGameplayTagContainer* MutableTags = const_cast<FGameplayTagContainer*>(&TempCooldownTags);
       // 合并父类（如默认GE）的标签
       const FGameplayTagContainer* ParentTags = Super::GetCooldownTags();
       if (ParentTags)
       {
           MutableTags->AppendTags(*ParentTags);
       }
       // 合并技能专属的Cooldown Tags
       MutableTags->AppendTags(CooldownTags);
       return MutableTags;
   }
   ```

3. **重写`ApplyCooldown()`注入标签和冷却时间**：
   在应用冷却时，动态为`Cooldown GE`的`Spec`添加`Cooldown Tag`，并通过`SetByCaller`传入冷却时间：

   ```cpp
   void UPGGameplayAbility::ApplyCooldown(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo) const
   {
       UGameplayEffect* CooldownGE = GetCooldownGameplayEffect();
       if (CooldownGE)
       {
           // 创建GE Spec（基于复用的Cooldown GE模板）
           FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(CooldownGE->GetClass(), GetAbilityLevel());
           // 注入技能专属的Cooldown Tag
           SpecHandle.Data.Get()->DynamicGrantedTags.AppendTags(CooldownTags);
           // 通过SetByCaller设置冷却时间（使用约定的Tag如"Data.Cooldown"）
           FGameplayTag CooldownTag = FGameplayTag::RequestGameplayTag(FName("Data.Cooldown"));
           SpecHandle.Data.Get()->SetSetByCallerMagnitude(CooldownTag, CooldownDuration.GetValueAtLevel(GetAbilityLevel()));
           // 应用GE到技能所有者
           ApplyGameplayEffectSpecToOwner(Handle, ActorInfo, ActivationInfo, SpecHandle);
       }
   }
   ```

4. **配置复用的 Cooldown GE**：
   在`Cooldown GE`的 “Duration Policy” 中，将持续时间设置为`SetByCaller`，并指定`Data Tag`为步骤 3 中约定的 Tag（如`Data.Cooldown`）。

##### 技巧 2：使用 MMC 计算冷却时间

与`SetByCaller`类似，但冷却时间通过`MMC`从技能配置中读取，无需手动设置`SetByCaller`，步骤如下：

1. **定义与技巧 1 相同的 GA 配置项**：
   包含`CooldownDuration`、`CooldownTags`和`TempCooldownTags`（与技巧 1 完全一致）。

2. **重写`GetCooldownTags()`**：
   代码与技巧 1 完全相同，目的是合并标签。

3. **重写`ApplyCooldown()`注入标签**：
   无需设置`SetByCaller`，只需注入`Cooldown Tag`（冷却时间由 MMC 计算）：

   ```cpp
   void UPGGameplayAbility::ApplyCooldown(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo) const
   {
       UGameplayEffect* CooldownGE = GetCooldownGameplayEffect();
       if (CooldownGE)
       {
           FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(CooldownGE->GetClass(), GetAbilityLevel());
           // 仅注入Cooldown Tag，冷却时间由MMC从技能中读取
           SpecHandle.Data.Get()->DynamicGrantedTags.AppendTags(CooldownTags);
           ApplyGameplayEffectSpecToOwner(Handle, ActorInfo, ActivationInfo, SpecHandle);
       }
   }
   ```

4. **实现 MMC 计算冷却时间**：
   创建`MMC`子类（如`UPGMMC_HeroAbilityCooldown`），从技能实例中读取`CooldownDuration`：

   ```cpp
   float UPGMMC_HeroAbilityCooldown::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec& Spec) const
   {
       // 从Spec中获取技能实例
       const UPGGameplayAbility* Ability = Cast<UPGGameplayAbility>(Spec.GetContext().GetAbilityInstance_NotReplicated());
       if (!Ability)
       {
           return 0.0f; // 无效时返回0（无冷却）
       }
       // 返回当前等级对应的冷却时间
       return Ability->CooldownDuration.GetValueAtLevel(Ability->GetAbilityLevel());
   }
   ```

5. **配置复用的 Cooldown GE**：
   在`Cooldown GE`的 “Duration Policy” 中，将持续时间设置为`Custom Calculation`，并指定为上述 MMC。

#### 三、核心注意事项

1. **Cooldown Tag 的唯一性**：每个技能（或共享冷却的技能组）的`Cooldown Tag`必须唯一，否则会导致冷却状态误判（如技能 A 的冷却影响技能 B）。
2. **实例化 Ability 限制**：复用技巧仅适用于 “实例化的 Ability”，因为每个实例可独立存储`CooldownDuration`和`CooldownTags`。
3. **网络同步**：`Cooldown Tag`的授予和移除由`Cooldown GE`自动处理，因`Cooldown GE`可预测，客户端与服务器的冷却状态会自动同步。

#### 总结

`Cooldown GE`通过 “持续授予独特 Tag” 实现冷却逻辑，其复用核心是动态注入`Cooldown Tag`和冷却时间（通过`SetByCaller`或`MMC`）。相比为每个技能创建独立 GE，复用方式可大幅减少资源冗余，尤其适合技能数量多的项目。需注意 Tag 的唯一性和实例化 Ability 的限制，以确保冷却逻辑准确可靠。

#### 4.5.15.1 获取Cooldown GameplayEffect的剩余时间

这段代码实现了一个关键功能：**查询与指定标签相关的冷却效果（Cooldown GameplayEffect）的剩余时间和总持续时间**，通常用于 UI 显示技能冷却进度（如技能图标上的倒计时）。以下是详细解析：

##### 一、函数作用与参数

```cpp
bool APGPlayerState::GetCooldownRemainingForTag(
    FGameplayTagContainer CooldownTags,  // 要查询的冷却标签集合
    float & TimeRemaining,              // 输出：剩余冷却时间（秒）
    float & CooldownDuration            // 输出：总冷却持续时间（秒）
)
```

- **返回值**：`true` 表示查询到有效冷却效果，`false` 表示未查询到。
- **核心目的**：通过冷却标签（`CooldownTags`）找到对应的激活中（Active）的 `Cooldown GE`，并返回其剩余时间和总时长。

##### 二、核心逻辑拆解

###### 1. 前置检查

```cpp
if (AbilitySystemComponent && CooldownTags.Num() > 0)
```

- 确保 `ASC`（AbilitySystemComponent）存在（技能系统的核心组件，管理所有 GE）。
- 确保传入的 `CooldownTags` 不为空（至少有一个标签需要查询）。

###### 2. 构建查询条件

```cpp
FGameplayEffectQuery const Query = FGameplayEffectQuery::MakeQuery_MatchAnyOwningTags(CooldownTags);
```

- `FGameplayEffectQuery` 是用于筛选 `ASC` 中激活的 `GameplayEffect` 的查询条件。

- ```
  MakeQuery_MatchAnyOwningTags(CooldownTags)
  ```

   

  表示：

  查询所有 “拥有（Owning）`CooldownTags` 中任意一个标签” 的激活中 GE

  。

  - 这里的 “拥有标签” 指 GE 的 `GrantedTags` 或 `DynamicGrantedTags` 中包含目标标签（即 `Cooldown GE` 生效时授予的冷却标签）。

###### 3. 获取符合条件的 GE 的时间信息

```cpp
TArray< TPair<float, float> > DurationAndTimeRemaining = 
    AbilitySystemComponent->GetActiveEffectsTimeRemainingAndDuration(Query);
```

- `GetActiveEffectsTimeRemainingAndDuration(Query)` 是 `ASC` 的接口，作用是：
  对所有符合查询条件（`Query`）的**激活中 GE**，返回它们的 “剩余时间” 和 “总持续时间”，存储为 `TPair<float, float>`（Key = 剩余时间，Value = 总时长）。

###### 4. 筛选 “最长剩余时间” 的 GE

```cpp
if (DurationAndTimeRemaining.Num() > 0)
{
    int32 BestIdx = 0;
    float LongestTime = DurationAndTimeRemaining[0].Key;
    for (int32 Idx = 1; Idx < DurationAndTimeRemaining.Num(); ++Idx)
    {
        if (DurationAndTimeRemaining[Idx].Key > LongestTime)
        {
            LongestTime = DurationAndTimeRemaining[Idx].Key;
            BestIdx = Idx;
        }
    }
    TimeRemaining = DurationAndTimeRemaining[BestIdx].Key;
    CooldownDuration = DurationAndTimeRemaining[BestIdx].Value;
    return true;
}
```

- 当存在多个符合条件的 GE（例如一个技能同时受多个冷却标签影响），代码会选择

  剩余时间最长

  的那个 GE 的时间信息作为结果。

  - 原因：技能实际可用时间由 “最后结束的冷却” 决定，因此取最长剩余时间更符合直观感受（例如 “技能自身冷却 10 秒” 和 “全局共享冷却 5 秒” 同时生效时，技能需等待 10 秒后才能使用）。

###### 5. 未查询到结果

```cpp
return false;
```

- 若没有符合条件的激活中 GE（即技能不在冷却中），返回 `false`，`TimeRemaining` 和 `CooldownDuration` 保持初始值（0）。

##### 三、关键注意事项

1. **客户端查询的前提**：
   代码注释提到 “客户端查询依赖 `ASC` 的同步模式”，原因是：
   - `Cooldown GE` 的状态（是否激活、剩余时间）需要从服务器同步到客户端。
   - 若 `ASC` 的同步模式（`ReplicationMode`）配置为 “不同步 GE”，客户端将无法获取准确的冷却时间，导致查询结果错误。
2. **标签匹配逻辑**：
   函数查询的是 “拥有 `CooldownTags` 中任意标签” 的 GE（`MatchAnyOwningTags`），这与冷却检查逻辑一致（只要有一个冷却标签存在，技能就不可用）。
3. **适用场景**：
   通常在 UI 类（如技能面板）中调用，用于更新技能图标的冷却进度条、显示剩余秒数等，提升玩家体验。

##### 总结

这个函数通过 “标签查询” 定位技能的冷却效果，核心是利用 `ASC` 的接口筛选激活中的 `Cooldown GE`，并返回最长剩余时间（确保符合玩家对 “冷却结束时间” 的预期）。在实际使用中，需注意客户端与服务器的 GE 状态同步，以保证 UI 显示的准确性。

#### 4.5.15.2 监听冷却开始和结束

在 Gameplay Ability System（GAS）中，监听冷却（Cooldown）的开始与结束需要结合 `AbilitySystemComponent`（ASC）的事件委托和标签事件，选择合适的监听方式能避免网络同步（如客户端预测与服务端校正）带来的逻辑错误。以下是具体解析：

##### 一、监听冷却的开始（Cooldown Start）

冷却的 “开始” 本质是 `Cooldown GE` 被应用到 ASC 并激活，或其对应的 `Cooldown Tag` 被添加到 ASC 中。有两种核心监听方式：

###### 1. 监听 `Cooldown GE` 被添加（推荐）

通过绑定 `ASC->OnActiveGameplayEffectAddedDelegateToSelf` 委托，在 `Cooldown GE` 被应用到 ASC 时触发回调。

**核心代码示例**：

```cpp
// 在初始化时绑定委托
AbilitySystemComponent->OnActiveGameplayEffectAddedDelegateToSelf.AddUObject(this, &ThisClass::OnCooldownGEAdded);

// 回调函数：处理 Cooldown GE 被添加的事件
void ThisClass::OnCooldownGEAdded(UAbilitySystemComponent* SourceASC, const FGameplayEffectSpec& Spec, FActiveGameplayEffectHandle Handle)
{
    // 检查该 GE 是否是冷却效果（通过标签筛选）
    if (Spec.Def->GrantedTags.HasTag(FGameplayTag::RequestGameplayTag(FName("Ability.Cooldown"))))
    {
        // 冷却开始：可通过 Spec 获取详细信息
        float CooldownDuration = Spec.GetDuration(); // 冷却总时长
        bool bIsPredicted = Spec.IsPredicted(); // 判断是客户端预测的还是服务端校正的
        
        // 执行冷却开始的逻辑（如更新UI显示）
        UpdateCooldownUIStart(CooldownDuration, bIsPredicted);
    }
}
```

**优势**：

- 可直接访问 `FGameplayEffectSpec`（`Spec`），获取冷却的详细信息（如总时长、是否为客户端预测的 GE 等）。
- 能区分 “客户端预测的冷却” 和 “服务端校正的冷却”，避免网络同步导致的 UI 闪烁或错误。

###### 2. 监听 `Cooldown Tag` 被添加

通过 `ASC->RegisterGameplayTagEvent` 监听 `Cooldown Tag` 的添加事件（`EGameplayTagEventType::NewOrRemoved`）。

**核心代码示例**：

```cpp
// 注册标签事件（监听 CooldownTag 的添加/移除）
AbilitySystemComponent->RegisterGameplayTagEvent(CooldownTag, EGameplayTagEventType::NewOrRemoved).AddUObject(this, &ThisClass::OnCooldownTagChanged);

// 回调函数：处理标签变化
void ThisClass::OnCooldownTagChanged(const FGameplayTag Tag, int32 NewCount)
{
    if (NewCount > 0) // 标签被添加（冷却开始）
    {
        // 执行冷却开始逻辑
    }
}
```

**局限性**：

- 仅能知道 “标签被添加”，无法获取冷却的具体信息（如总时长、是否为预测的 GE 等）。
- 无法区分标签是来自客户端预测的 GE 还是服务端校正的 GE，可能导致重复处理。

**推荐选择**：优先监听 `Cooldown GE` 被添加，因为其能获取更完整的上下文信息，尤其适合处理网络同步场景。

##### 二、监听冷却的结束（Cooldown End）

冷却的 “结束” 本质是 `Cooldown GE` 从 ASC 中移除，或其对应的 `Cooldown Tag` 从 ASC 中移除。有两种核心监听方式：

###### 1. 监听 `Cooldown Tag` 被移除（推荐）

通过 `ASC->RegisterGameplayTagEvent` 监听 `Cooldown Tag` 的移除事件（`EGameplayTagEventType::NewOrRemoved`）。

**核心代码示例**：

```cpp
// 同上，注册标签事件
AbilitySystemComponent->RegisterGameplayTagEvent(CooldownTag, EGameplayTagEventType::NewOrRemoved).AddUObject(this, &ThisClass::OnCooldownTagChanged);

// 回调函数：处理标签变化
void ThisClass::OnCooldownTagChanged(const FGameplayTag Tag, int32 NewCount)
{
    if (NewCount == 0) // 标签被移除（冷却结束）
    {
        // 执行冷却结束逻辑（如隐藏UI冷却进度）
        UpdateCooldownUIEnd();
    }
}
```

**优势**：

- 标签的移除**唯一对应冷却的真正结束**。即使服务端校正时移除了客户端预测的 `Cooldown GE`（此时标签可能仍存在，因为服务端的 GE 会重新添加标签），也不会误判为冷却结束。

**为什么这很重要？**
客户端会先预测性地应用 `Cooldown GE`（显示冷却），但服务端可能稍后发送 “校正后的 GE”（如调整冷却时长），此时客户端会先移除预测的 GE，再添加服务端的 GE—— 这一过程中 `Cooldown Tag` 会短暂保留（不会被移除），因此监听标签移除能避免将 “GE 替换” 误判为 “冷却结束”。

###### 2. 监听 `Cooldown GE` 被移除

通过绑定 `ASC->OnAnyGameplayEffectRemovedDelegate` 委托，在 `Cooldown GE` 从 ASC 中移除时触发回调。

**局限性**：

- 服务端校正时，客户端会先移除 “预测的 `Cooldown GE`”，再添加 “服务端的 `Cooldown GE`”—— 此时 `OnAnyGameplayEffectRemovedDelegate` 会被触发，但冷却并未真正结束（标签仍存在），导致误判（如 UI 错误地隐藏冷却进度）。

**推荐选择**：优先监听 `Cooldown Tag` 被移除，因为其能准确反映冷却的实际结束，避免网络同步中的 “GE 替换” 导致的逻辑错误。

##### 三、关键注意事项

1. **客户端同步依赖 ASC 模式**：
   客户端要监听 `Cooldown GE` 或 `Cooldown Tag` 的变化，必须确保 ASC 的 **`ReplicationMode`** 配置为同步 GE（如 `Full` 或 `Mixed`）。若同步模式为 `None`，客户端无法收到服务端的 GE 同步信息，监听会失效。
2. **标签的唯一性**：
   确保 `Cooldown Tag` 唯一对应一个技能（或技能组），避免监听时受到其他冷却的干扰。
3. **网络预测与校正的兼容**：
   客户端预测的 `Cooldown GE` 和服务端校正的 `Cooldown GE` 可能存在差异（如时长不同），监听时需通过 `Spec.IsPredicted()` 区分，确保 UI 显示与服务端最终状态一致。

##### 总结

- **冷却开始**：推荐监听 `OnActiveGameplayEffectAddedDelegateToSelf`（`Cooldown GE` 添加），可获取 `Spec` 信息，兼容网络预测。
- **冷却结束**：推荐监听 `RegisterGameplayTagEvent`（`Cooldown Tag` 移除），能准确反映实际冷却结束，避免服务端校正导致的误判。

#### 4.5.15.3 预测冷却时间

冷却时间的 “可预测性” 问题本质上是**网络同步与客户端预测之间的矛盾**—— 客户端为了提升操作流畅度会提前预测冷却状态，而服务端作为权威最终判定实际冷却时间，这种差异在高延迟场景下会直接影响玩家体验。以下是具体解析：

##### 一、冷却时间 “不可真正预测” 的核心原因

GAS 中，客户端会基于本地逻辑 “预测性” 地应用 `Cooldown GE`（例如技能激活后立即显示冷却），但服务端才是最终的 “权威”—— 服务端会根据自身逻辑计算实际冷却时间，并将结果同步给客户端。两者的差异主要来自：

1. **网络延迟**：客户端预测与服务端判定之间存在时间差（如玩家延迟 200ms，服务端的冷却校正信息需要 200ms 才能传到客户端）。
2. **数据不一致**：服务端可能因各种原因（如技能等级、 buff 影响）计算出与客户端预测不同的冷却时间（例如客户端预测冷却 2 秒，服务端实际计算为 2.5 秒）。

这会导致一种典型问题：

- 客户端预测冷却已结束（UI 显示技能可用），但服务端的冷却仍在进行。此时玩家点击技能，客户端会尝试激活，但服务端会拒绝（因冷却未结束），最终技能无法使用，造成 “操作失效” 的挫败感。

##### 二、样例项目的折中解决方案

为缓解这种矛盾，样例项目（如 ActionRPG）采用 “**先视觉反馈，后权威校正**” 的策略：

1. **客户端预测阶段**：技能激活时，客户端立即灰化技能图标（提供即时视觉反馈，让玩家知道技能进入冷却），但不启动精确的冷却计时器。
2. **服务端校正阶段**：当服务端的 `Cooldown GE`（权威冷却）同步到客户端后，客户端才基于服务端的实际冷却时间启动计时器，更新 UI 进度。

这种方式的核心是：**先用视觉反馈告知玩家 “技能已进入冷却”，再通过服务端数据确保后续进度显示的准确性**，减少 “客户端误判可用但实际不可用” 的落差感。

##### 三、高延迟玩家的劣势与 Fortnite 的应对

高延迟玩家在短冷却技能（如 1-2 秒冷却）上的劣势尤为明显：

- 客户端预测冷却结束 → 玩家点击技能 → 服务端仍处于冷却 → 技能被拒绝，而这段时间（延迟时长）足以让玩家错过操作窗口。

Fortnite 为避免这种问题，采用了**自定义冷却管理（Bookkeeping）**，而非依赖 GAS 原生的 `Cooldown GE`：

- 不通过 `GameplayEffect` 同步冷却状态，而是自己维护一套客户端与服务端的冷却逻辑，通过更精细的网络同步策略（如预测性激活允许、后续服务端校验补偿）让客户端与服务端的冷却状态更一致。
- 本质是 “弱化 GE 同步的依赖”，用更灵活的自定义逻辑适配快节奏游戏对低延迟体验的要求。

### 4.5.16 修改已激活GameplayEffect的持续时间

用到再学

### 4.5.17 运行时创建动态GameplayEffect

在运行时创建动态`GameplayEffect`是一个高阶技术, 你不必经常使用它

只有`即刻(Instant)GameplayEffect`可以在运行时由C++创建, `持续(Duration)`和`无限(Infinite)`GameplayEffect不能在运行时动态创建, 因为它们在同步时会寻找并不存在的`GameplayEffect`类定义. 

### 4.5.18 GameplayEffect Containers

简化一些`GE`相关操作的自定义结构体，该部分不属于原生`GAS`，有需要再研究。

## 4.6 Gameplay Abilities

### 4.6.1 GameplayAbility定义

#### 4.6.1.1 Replication Policy

文中极不推荐使用`GA`中的该功能。

##### Replication Policy（复制策略）：“名字骗人，改了也白改”

先直白说它的问题：**你以为它能控制技能（GA）的同步逻辑，其实它根本起不到你想要的作用，纯属 “伪选项”** 。

举个例子：
比如你做一个多人游戏，想让 “玩家 A 放的火球技能” 同步给其他玩家（比如玩家 B 看到玩家 A 的火球动画）。你看到 “Replication Policy” 这个名字，以为改它就能控制 “火球技能的状态同步到其他玩家的屏幕上”—— 但实际上：

- UE 里的 `GameplayAbility`（GA）有个核心规则：**它只在 “技能所属的客户端”（比如玩家 A 自己的客户端）和 “服务端” 运行，从不在 “模拟角色”（比如玩家 B 屏幕上看到的玩家 A 的角色）上运行**。
- 技能的视觉效果（比如火球动画、特效），本来就不是靠 GA 本身同步的，而是靠 `AbilityTask`（技能任务，比如播放动画的任务）或 `GameplayCue`（技能提示，比如特效触发）同步到其他玩家的屏幕上。

你改 “Replication Policy”，既不能让 GA 跑到模拟角色上，也不能优化视觉同步 —— 相当于 “对着空气调参数”，不仅没用，还可能因为误解它的作用（以为改了能解决同步问题），反而绕远路、甚至误改出其他 bug（比如打乱默认的 GA Spec 同步逻辑）。

加上 Epic 都打算未来删掉它，现在用就是给自己留坑。

#### 4.6.1.2 Server Respects Remote Ability Cancellation

文中极不推荐使用该选项。

##### Server Respects Remote Ability Cancellation：“客户端一喊停，服务端就瞎停，容易出‘不同步 bug’”

这个设置的意思是：**客户端说 “我的技能结束了 / 取消了”，服务端就必须立刻停掉这个技能**—— 听起来好像 “客户端和服务端保持一致”，但实际会因为 “网络延迟” 和 “客户端预测” 搞砸。

举个典型坑场景：
你做一个 “蓄力火球” 技能：玩家按按键蓄力（GA 运行中），松开按键发射（GA 结束）。

- 玩家是 “高延迟网络”（比如玩外服）：他松开按键时，客户端先 “预测” 技能结束了（屏幕上显示火球发出去），然后才把 “技能取消” 的消息发给服务端。
- 但因为延迟，服务端此时还在处理 “蓄力到一半” 的逻辑（比如计算火球的伤害值），突然收到客户端的 “停手” 指令，就强制结束了技能 —— 结果就是：客户端显示 “火球发出去了”，但服务端没算伤害（因为被强制停了），出现 “打空了” 的 bug；
- 反过来也可能：客户端卡了一下，没及时发 “取消” 指令，服务端还在运行技能，导致玩家明明取消了，角色还在蓄力，出现 “技能卡住” 的问题。

简单说：这个设置把 “技能是否结束” 的决定权交给了客户端，但服务端才是最终的 “裁判”（负责伤害、状态等关键逻辑），强制让裁判听客户端的，必然会因为延迟导致 “两边对不上”，体验崩了。

#### 4.6.1.3 Replicate Input Directly

##### Replicate Input Directly：“疯狂发无用数据，网络扛不住，还容易出 bug”

这个设置的意思是：**客户端会把 “技能输入的按下 / 抬起”（比如按 Q 激活技能）一直同步给服务端**—— 比如你按着 Q 不松，客户端会不停发 “我按 Q 了”“还按着 Q”“我松 Q 了”，没完没了。

不推荐的核心原因是 **“性能浪费” 和 “可靠性差”** ，举两个实际坑：

1. **性能坑：网络流量爆炸**
   比如你做一个 MOBA 游戏，10 个玩家同时玩，每个玩家的技能输入（比如按 W、E、R）都用这个设置 “一直同步”—— 哪怕玩家只是误触了一下技能键（没激活成功），客户端也会发一堆 “按下 / 抬起” 的消息。这些无用数据会占满网络带宽，导致游戏卡顿、延迟飙升，尤其是手机端或低网速玩家，体验直接崩。
2. **可靠性坑：输入顺序乱了**
   自己同步输入很容易出 “顺序错” 的问题：比如玩家快速按 “Q（按下）→ Q（抬起）”，因为网络延迟，服务端可能先收到 “抬起” 的消息，再收到 “按下” 的消息 —— 结果就是：服务端以为玩家 “没按过 Q”，技能根本没激活，客户端却显示 “按了 Q”，出现 “技能按了没反应” 的 bug。

而 Epic 推荐的 “`AbilityTask` 里的 `Generic Replicated Event`”（通用同步事件）是优化过的：它不会 “一直同步”，只在 “关键节点”（比如技能真的激活时）同步输入，还会自动处理 “输入顺序” 和 “延迟补偿”—— 相当于 Epic 已经给你搭好了 “安全的桥”，你偏要自己踩 “不稳的独木桥”，没必要。

### 4.6.2 绑定输入到ASC



#### 4.6.2.1 绑定输入时不激活Ability

### 4.6.3 授予Ability

这段代码展示了向 `AbilitySystemComponent`（ASC）授予 `GameplayAbility`（GA）的核心逻辑，涉及 “服务器权威授予”“能力规格（Spec）创建”“自动同步到客户端” 等关键机制。我们可以从 “为什么只在服务器授予”“`GameplayAbilitySpec` 是什么”“授予流程的核心细节” 三个层面拆解：

#### 一、为什么 “只在服务器授予 GA”？

代码第一行就明确限制：`if (Role != ROLE_Authority)` 则不执行授予逻辑 —— 这是 GAS 多人游戏的核心原则：**服务器拥有 “授予能力” 的唯一权威**，客户端不能自主授予 GA。

原因有二：

1. **防止客户端作弊**：如果客户端能自己授予 GA（比如给自己刷出 “无敌技能”），游戏逻辑会完全失控。服务器作为 “裁判”，必须控制所有 GA 的授予（如玩家升级解锁技能、完成任务获得技能）。
2. **保证数据一致性**：服务器授予 GA 后，会自动将 “能力规格（`GameplayAbilitySpec`）” 同步给 “所属客户端”（即技能拥有者的客户端），其他客户端（如旁观玩家）不会收到 —— 这样既保证 “拥有者能看到自己的技能”，又避免 “无关客户端浪费性能同步不必要的数据”。

#### 二、`FGameplayAbilitySpec`：GA 的 “实例化配置单”

授予 GA 时，并不是直接把 `UGameplayAbility` 类扔进 ASC，而是创建一个 `FGameplayAbilitySpec`（简称 Spec）—— 它相当于 GA 的 “实例化配置单”，记录了这个 GA 的 “个性化参数”。

代码中 `FGameplayAbilitySpec` 的构造参数详解：

```cpp
FGameplayAbilitySpec(
    StartupAbility,                  // 1. GA的类（如“火球术”的蓝图类）
    GetAbilityLevel(...),            // 2. 技能等级（影响伤害/范围等）
    static_cast<int32>(...AbilityInputID),  // 3. 绑定的输入枚举值（如“Ability1”对应左键）
    this                             // 4. SourceObject（技能的来源对象，通常是角色自身）
)
```

这四个参数决定了 GA 的 “实例化状态”：

- **参数 1（GA 类）**：指定要授予的具体技能（如 `UGA_Fireball`）；
- **参数 2（等级）**：同一 GA 可以有不同等级（如 1 级火球术伤害 50，2 级 100），通过 `GetAbilityLevel` 动态获取；
- **参数 3（输入 ID）**：将 GA 与之前绑定的输入枚举关联（如绑定 “Ability1”，则按左键会激活该 GA）；
- **参数 4（SourceObject）**：记录 “谁拥有这个技能”（通常是角色自身 `this`），用于技能逻辑中定位来源（如伤害计算时识别 “攻击者是谁”）。

#### 三、授予流程的核心细节

##### 1. `AddCharacterAbilities` 函数的执行时机

通常在角色初始化时调用（如 `BeginPlay` 或玩家出生时），用于授予 “初始技能”（如默认攻击、跳跃）。代码中的 `CharacterAbilitiesGiven` 是个布尔值标记，确保技能只被授予一次（避免重复授予导致多个相同技能）。

##### 2. `CharacterAbilities` 数组的作用

`TArray<TSubclassOf<UGDGameplayAbility>> CharacterAbilities` 是角色类中定义的 “初始技能列表”，开发者可以在蓝图 / 代码中配置（如给 “战士” 角色配置 `GA_SwordAttack`、`GA_ShieldBlock`，给 “法师” 配置 `GA_Fireball`、`GA_IceShield`）。
遍历这个数组并调用 `GiveAbility`，就能批量授予角色的所有初始技能。

##### 3. 授予后发生了什么？

- **服务器端**：ASC 将 Spec 加入 `ActivatableAbilities` 列表，该 GA 变为 “可激活状态”（满足标签条件、冷却等时可被输入触发）；
- **网络同步**：服务器自动将 Spec 同步给 “所属客户端”（角色自己的客户端），客户端的 ASC 会创建对应的本地 Spec 副本，确保 “客户端能预测技能表现”（如本地播放技能动画）；
- **其他客户端**：不会收到这个 Spec，因为他们不需要执行该技能的逻辑，只需通过 `AbilityTask` 或 `GameplayCue` 同步视觉效果（如看到别人放火球）。

#### 四、扩展：动态授予技能的场景

除了初始技能，游戏中还需要 “动态授予”（如升级解锁、拾取道具获得），逻辑与初始授予一致，只需在服务器端调用 `GiveAbility` 即可：

```cpp
// 示例：玩家升级后授予新技能
void AMyPlayerState::OnLevelUp(int32 NewLevel)
{
    if (NewLevel == 5 && AbilitySystemComponent)
    {
        // 服务器授予“终极技能”
        FGameplayAbilitySpec UltimateSpec(UGA_Ultimate::StaticClass(), 1, static_cast<int32>(EGDAbilityInputID::Ability5), this);
        AbilitySystemComponent->GiveAbility(UltimateSpec);
    }
}
```

#### 总结

向 ASC 授予 GA 的核心逻辑是：**服务器通过 `GiveAbility` 函数，将 GA 包装成 `FGameplayAbilitySpec` 并添加到 ASC 中**，同时自动同步给所属客户端。这个过程保证了 “技能授予的权威性” 和 “网络数据的一致性”，是 GAS 中 “技能管理” 的基础。

记住：所有 GA 的授予必须在服务器端执行，客户端只负责接收同步后的 Spec 并处理本地表现（如动画、特效），不参与授权逻辑。

### 4.6.4 激活Ability

在 GAS 中，`GameplayAbility`（GA）除了 “输入自动激活”，还能通过 `ASC` 提供的 4 种主动激活方式灵活触发 —— 这些方式覆盖了 “批量激活”“精准激活”“带数据激活” 等场景，是实现被动技能、剧情触发、联动技能的核心工具。下面分 “每种激活方式的作用 + 参数 + 场景” 拆解，重点讲易踩坑的 “Event 激活” 细节：

#### 一、先明确：主动激活的核心价值

输入自动激活适合 “玩家主动按键触发”（如 Q 键放技能），但游戏中还需要 **“非玩家触发” 或 “带条件 / 数据的触发”**：

- 比如 “战斗开始时自动激活所有‘战斗准备’技能”（批量激活）；
- 比如 “剧情触发特定‘大招’”（精准激活某类技能）；
- 比如 “触发治疗技能时，传递‘治疗目标 + 治疗量’”（带数据激活）。
  这 4 种主动激活方式就是为解决这些需求设计的。

#### 二、4 种主动激活方式详解

##### 1. 按 `GameplayTag` 激活：`TryActivateAbilitiesByTag`

**核心作用**：批量激活 ASC 中 “带有指定标签” 且满足激活条件（冷却、标签需求等）的所有 GA。
可以理解为 “给所有带某标签的技能发‘激活指令’”。

##### 关键细节：

- **函数原型**：

  ```cpp
  bool TryActivateAbilitiesByTag(const FGameplayTagContainer& GameplayTagContainer, bool bAllowRemoteActivation = true);
  ```

  - `GameplayTagContainer`：要匹配的标签（如 `GameState.BattleStart`“战斗开始”、`Buff.DefenseUp`“防御提升”）；
  - `bAllowRemoteActivation`：是否允许 “客户端触发，服务器验证”（通常设为 `true`，但敏感技能如 “无敌” 可设为 `false`，强制仅服务器激活，防作弊）。

- **适用场景**：需要批量激活 “同类技能” 的场景。
  举例：
  游戏进入战斗时，调用：

  ```cpp
  // 定义“战斗开始”标签
  FGameplayTagContainer BattleStartTags;
  BattleStartTags.AddTag(FGameplayTag::RequestGameplayTag("GameState.BattleStart"));
  // 激活所有带该标签的技能（如战士的“战斗怒吼”、法师的“元素唤醒”）
  ASC->TryActivateAbilitiesByTag(BattleStartTags);
  ```

  此时，所有 “标签需求包含 `GameState.BattleStart`” 且满足条件的 GA 会被批量激活。

##### 2. 按 `GA 类` 激活：`TryActivateAbilityByClass`

**核心作用**：精准激活 ASC 中 “指定类” 的 GA（若该类有多个实例，激活优先级最高的那个）。
适合 “明确知道要激活哪个类的技能”，但不关心具体实例的场景。

##### 关键细节：

- **函数原型**：

  ```cpp
  bool TryActivateAbilityByClass(TSubclassOf<UGameplayAbility> InAbilityToActivate, bool bAllowRemoteActivation = true);
  ```

  - `InAbilityToActivate`：要激活的 GA 类（如 `UGA_Fireball`“火球术”、`UGA_Heal`“治疗术”）。

- **适用场景**：剧情触发、任务奖励等 “明确激活某类技能” 的场景。
  举例：
  剧情对话结束后，强制激活玩家的 “终极技能”：

  ```cpp
  // 激活“终极技能”类的 GA
  ASC->TryActivateAbilityByClass(UGA_Ultimate::StaticClass());
  ```

  若玩家已拥有该类 GA 且满足条件（无冷却、有能量），则直接激活。

##### 3. 按 `GameplayAbilitySpecHandle` 激活：`TryActivateAbility`

**核心作用**：激活 ASC 中 “特定实例” 的 GA（通过 `FGameplayAbilitySpecHandle` 定位）。
适合 “同一 GA 有多个实例” 的场景（如同一技能有不同等级 / 配置的 `Spec`，需激活其中一个）。

##### 关键细节：

- **函数原型**：

  ```cpp
  bool TryActivateAbility(FGameplayAbilitySpecHandle AbilityToActivate, bool bAllowRemoteActivation = true);
  ```

  - `AbilityToActivate`：`FGameplayAbilitySpec` 的唯一标识（`Handle`）—— 授予 GA 时，`ASC->GiveAbility()` 会返回该 `Handle`，需提前保存。

- **适用场景**：同一 GA 有多个实例，需精准激活某一个。
  举例：
  玩家有两个 “火球术” 实例（1 级和 2 级），激活 2 级实例：

  ```cpp
  // 授予 2 级火球术时，保存其 SpecHandle
  FGameplayAbilitySpec FireballSpec2(UGA_Fireball::StaticClass(), 2); // 2 级
  FGameplayAbilitySpecHandle FireballHandle2 = ASC->GiveAbility(FireballSpec2);
  
  // 后续激活 2 级火球术（而非 1 级）
  ASC->TryActivateAbility(FireballHandle2);
  ```

##### 4. 按 `GameplayEvent` 激活：`TriggerAbilityFromGameplayEvent`

**核心作用**：通过 “发送 `GameplayEvent`” 激活 GA，且能传递 **数据负载（Payload）**（如目标信息、伤害值、触发来源）—— 这是唯一支持 “带数据激活” 的方式，灵活性最高。

##### 关键细节（分 3 步：配置 GA→发送 Event→激活技能）：

##### 步骤 1：给 GA 配置 “Event 触发条件”

要让 GA 能被 Event 激活，必须先在 GA 中设置 **`Trigger`（触发器）**：

1. 在 GA 的蓝图 / 代码中，找到 `Trigger` 配置（蓝图中在 “Class Defaults” 的 “Ability Trigger Data” 里）；
2. 新增一个 Trigger，选择 **`GameplayEvent`** 类型；
3. 给 Trigger 分配一个 **`GameplayTag`**（如 `Event.Activate.Heal`“激活治疗”）—— 这个标签是 “Event 与 GA 的绑定钥匙”。

##### 步骤 2：发送 `GameplayEvent` 并传递 Payload

通过 `UAbilitySystemBlueprintLibrary::SendGameplayEventToActor` 发送 Event，同时用 `FGameplayEventData` 传递数据（Payload）：

```cpp
// 1. 创建要传递的 Payload（数据负载）
FGameplayEventData EventData;
EventData.Instigator = PlayerCharacter; // 触发者（玩家）
EventData.TargetData = TargetDataHandle; // 目标数据（如治疗目标的位置/引用）
EventData.EventMagnitude = 200.0f; // 额外数值（如治疗量 200）

// 2. 定义 Event 标签（必须和 GA 的 Trigger 标签一致）
FGameplayTag ActivateHealTag = FGameplayTag::RequestGameplayTag("Event.Activate.Heal");

// 3. 给目标 ASC 发送 Event（激活治疗技能）
UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(
    TargetCharacter, // 技能要作用的目标（如队友）
    ActivateHealTag, // 绑定的 Event 标签
    EventData // 传递的 Payload
);
```

##### 步骤 3：GA 中接收 Payload 并执行逻辑

GA 激活时，必须通过 `ActivateAbilityFromEvent` 函数接收 Payload（而非标准的 `ActivateAbility`）：

- 蓝图注意事项：蓝图中不能有 “标准 `ActivateAbility` 节点”，否则会优先调用它，导致`ActivateAbilityFromEvent`不执行；必须用 “`ActivateAbilityFromEvent`” 节点，从`EventData`中读取 Payload

- 代码示例：

  ```cpp
  void UGA_Heal::ActivateAbilityFromEvent(const FGameplayEventData& EventData)
  {
      Super::ActivateAbilityFromEvent(EventData);
  
      // 从 Payload 中读取数据
      AActor* HealTarget = EventData.TargetData.Get(0)->GetActor(); // 治疗目标
      float HealAmount = EventData.EventMagnitude; // 治疗量 200
  
      // 执行治疗逻辑（应用治疗 GE）
      ApplyHealEffect(HealTarget, HealAmount);
  
      // 技能结束，调用 EndAbility（非无限技能必须调用）
      EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
  }
  ```

##### 适用场景：需要传递数据的激活（如治疗、伤害、目标联动）。

比如：“队友倒地时，触发‘救援治疗’技能，传递‘倒地队友的引用’和‘救援治疗量’”。

#### 三、特殊激活方式：`GiveAbilityAndActivateOnce`

**核心作用**：“授予 GA + 立即激活一次” 二合一，激活后该 GA 会被自动移除（一次性技能）。
适合 “临时触发一次的技能”（如剧情奖励的 “临时无敌”、道具触发的 “一次性爆发”）。

```cpp
// 创建一次性技能的 Spec（如“临时无敌”，等级 1）
FGameplayAbilitySpec InvincibilitySpec(UGA_TempInvincibility::StaticClass(), 1);
// 授予并立即激活，激活后自动移除
ASC->GiveAbilityAndActivateOnce(InvincibilitySpec);
```

#### 四、关键注意事项

1. **`EndAbility` 必须调用**：
   除了 “无限持续的被动技能”（如永久加防御的被动），所有主动激活的 GA 都必须在逻辑结束后调用 `EndAbility()`—— 否则 GA 会一直处于 “激活中” 状态，占用资源且无法再次激活。
2. **`bAllowRemoteActivation` 的安全意义**：
   对于 “伤害、无敌、治疗” 等敏感技能，建议设为 `false`，强制 **“仅服务器激活”**（客户端触发请求，服务器验证后执行），避免客户端作弊（如本地触发无敌但服务器不认可）。
3. **蓝图激活 Event 技能的坑**：
   若在蓝图中用 Event 激活 GA，**必须使用 “ActivateAbilityFromEvent” 节点**，且蓝图中不能有 “标准 ActivateAbility” 节点 —— 否则标准节点会优先执行，导致 Payload 无法接收。

#### 五、总结：如何选择激活方式？

| 激活方式                   | 核心优势                | 适用场景                               |
| -------------------------- | ----------------------- | -------------------------------------- |
| 按 Tag 激活                | 批量激活同类技能        | 战斗开始、场景切换等批量触发           |
| 按 Class 激活              | 精准激活特定类技能      | 剧情触发、任务奖励等明确类的技能       |
| 按 SpecHandle 激活         | 激活特定实例技能        | 同一技能多实例（如不同等级）的精准触发 |
| 按 Event 激活              | 支持传递数据（Payload） | 治疗、伤害、目标联动等需额外数据的场景 |
| GiveAbilityAndActivateOnce | 临时一次性技能          | 道具触发、临时 buff 等一次性效果       |

根据 “是否需要批量激活”“是否需要传递数据”“是否是一次性技能” 三个维度，就能快速选择合适的激活方式，避免冗余逻辑或踩坑。

#### 4.6.4.1 被动Ability

这段内容详细讲解了 **被动 `GameplayAbility`（被动技能）** 的核心实现逻辑 —— 重点是 “如何自动激活” 和 “为何选择 `OnAvatarSet` 作为初始化入口”，同时涉及被动技能的网络策略设计。我们可以从 “被动技能的特殊需求”“`OnAvatarSet` 的作用”“代码逻辑拆解”“网络策略” 四个层面彻底理解：

##### 一、先明确：被动技能的核心需求

被动技能（如 “永久增加 10% 防御”“每 5 秒恢复 5 点生命值”）与主动技能（如 “火球术”“闪避”）的核心差异在于：

1. **自动激活**：不需要玩家按键触发，授予后立即激活，且通常持续运行；
2. **生命周期绑定**：与 “角色（Avatar）” 绑定 —— 当技能的 `AvatarActor`（技能作用的角色）被设置后（如玩家出生、角色切换），被动技能需要立即初始化并生效；
3. **无手动终止**：除非技能被移除（如角色死亡、被动被驱散），否则不会主动结束，无需调用 `EndAbility()`。

这些需求决定了被动技能不能用主动技能的 “输入激活” 或 “Event 激活”，必须依赖 `OnAvatarSet` 这个特殊时机。

##### 二、关键入口：`OnAvatarSet` 函数的作用

`UGameplayAbility::OnAvatarSet` 是 GAS 框架为 “技能与角色绑定” 设计的 **初始化回调函数**，其触发时机和作用完美匹配被动技能的需求：

###### 1. 触发时机（为什么选它？）

`OnAvatarSet` 仅在以下两个条件同时满足时触发：

- 技能已通过 `ASC->GiveAbility()` 授予（加入 `ActivatableAbilities` 列表）；
- 技能的 **`AvatarActor` 被设置**（即技能明确知道 “要作用于哪个角色”）。

典型场景：

- 玩家出生时，`PlayerState` 的 ASC 授予被动技能，同时 `AvatarActor` 设为玩家的 `Character` → `OnAvatarSet` 触发；
- 角色切换时（如从 “战士” 切换到 “法师”），新角色的 ASC 授予对应被动技能，`AvatarActor` 设为新角色 → `OnAvatarSet` 触发。

这个时机确保被动技能 “只在角色准备就绪后才初始化”，避免出现 “技能已授予但不知道作用于谁” 的空指针问题。

###### 2. 框架定位（Epic 为什么推荐它？）

Epic 明确说明 `OnAvatarSet` 是 **被动技能的正确初始化位置**，相当于技能的 “`BeginPlay`”—— 主动技能的初始化通常在 `ActivateAbility` 中，而被动技能因为 “自动激活”，需要更早的初始化入口，`OnAvatarSet` 恰好填补了这个空白。

##### 三、代码逻辑拆解：被动技能自动激活的实现

以样例代码 `UGDGameplayAbility::OnAvatarSet` 为例，逐行解析被动技能 “自动激活” 的核心逻辑：

```cpp
void UGDGameplayAbility::OnAvatarSet(const FGameplayAbilityActorInfo * ActorInfo, const FGameplayAbilitySpec & Spec)
{
    Super::OnAvatarSet(ActorInfo, Spec); // 调用父类实现，确保框架基础逻辑执行

    // ActivateAbilityOnGranted：自定义布尔值，标记“该技能是否在授予时自动激活”
    if (ActivateAbilityOnGranted)
    {
        // 调用 ASC 的 TryActivateAbility，通过 Spec.Handle 激活当前被动技能
        bool ActivatedAbility = ActorInfo->AbilitySystemComponent->TryActivateAbility(Spec.Handle, false);
    }
}
```

###### 1. 核心变量：`ActivateAbilityOnGranted`

这是自定义 `UGameplayAbility` 子类（如 `UGDGameplayAbility`）中添加的布尔值变量（通常用 `UPROPERTY` 暴露给蓝图）：

- 设为 `true`：表示该技能是被动技能，授予后自动激活（如 “永久防御加成”）；
- 设为 `false`：表示该技能是主动技能，需要手动触发（如 “火球术”）。

通过这个变量，可以在同一个父类中区分 “被动” 和 “主动” 技能，复用初始化逻辑，无需为被动技能单独写一套类。

###### 2. 激活逻辑：`TryActivateAbility(Spec.Handle, false)`

- **`Spec.Handle`**：`FGameplayAbilitySpec` 的唯一标识 —— 每个授予的技能都有一个独立的 `Spec`，通过 `Handle` 可以精准激活 “当前这个被动技能实例”（避免激活其他同名技能）；
- **第二个参数 `false`**：对应 `bAllowRemoteActivation`，设为 `false` 表示 “不允许客户端远程激活”—— 被动技能的激活逻辑必须由服务器控制（防作弊，避免客户端自己激活非法被动），客户端仅同步结果（如显示被动特效）。

###### 3. 被动技能的 `ActivateAbility` 实现（补充）

`OnAvatarSet` 触发 `TryActivateAbility` 后，会调用被动技能的 `ActivateAbility` 函数。由于被动技能 “持续运行”，`ActivateAbility` 中通常不需要复杂的异步逻辑（如 `AbilityTask`），只需初始化 “持续效果”（如应用永久 `GameplayEffect`）：

```cpp
void UGA_PassiveDefense::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    if (!HasAuthority()) return; // 仅服务器执行权威逻辑

    // 1. 应用“永久增加10%防御”的 GameplayEffect（无限持续）
    if (ActorInfo->AbilitySystemComponent)
    {
        // 创建 GE Spec（防御加成 GE）
        FGameplayEffectSpecDef DefenseGESpecDef;
        DefenseGESpecDef.Def = UGE_PassiveDefenseBoost::StaticClass(); // 防御加成 GE 类
        DefenseGESpecDef.Level = 1; // 技能等级

        // 应用无限持续的 GE（持续时间设为无限）
        ActorInfo->AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(
            MakeGameplayEffectSpec(DefenseGESpecDef),
            Handle,
            ActorInfo->OwnerActor.Get()
        );
    }

    // 2. 被动技能无需调用 EndAbility()（持续运行，直到被移除）
}
```

##### 四、被动技能的网络策略：为什么是 “仅服务器（Server Only）”？

样例中提到 “被动技能一般有一个仅服务器的网络执行策略”，核心原因是 **“避免客户端作弊” 和 “保证数据一致性”**：

###### 1. 网络执行策略（Net Execution Policy）的作用

`GameplayAbility` 的网络执行策略决定了 “技能的逻辑在哪些端（服务器 / 客户端）执行”，被动技能选择 “Server Only” 意味着：

- 技能的 `ActivateAbility`、`OnAvatarSet` 等核心逻辑 **仅在服务器执行**；
- 客户端不执行任何逻辑，仅通过服务器同步的 `GameplayEffect`（如防御加成）和 `GameplayCue`（如被动特效）显示结果。

###### 2. 为什么不能让客户端执行？

- **防作弊**：如果客户端能执行被动技能逻辑（如自己加防御），作弊者可以修改客户端代码，给自己叠加无限防御，服务器无法验证；
- **数据一致性**：被动技能的效果（如防御值、回血速度）是游戏的 “权威数据”，必须由服务器统一计算 —— 如果客户端和服务器都执行，可能因延迟或计算差异导致 “客户端显示防御 100，服务器实际算 50” 的不一致问题。

##### 五、总结：被动技能的完整实现链路

1. **技能配置**：在自定义 `UGameplayAbility` 子类中添加 `ActivateAbilityOnGranted` 布尔值，设为 `true`；将网络执行策略设为 “Server Only”；
2. **授予技能**：服务器通过 `ASC->GiveAbility()` 授予被动技能（如玩家出生时）；
3. **触发初始化**：当技能的 `AvatarActor` 被设置时，`OnAvatarSet` 自动触发；
4. **自动激活**：`OnAvatarSet` 中调用 `TryActivateAbility`，激活被动技能；
5. **持续生效**：`ActivateAbility` 中应用无限持续的 `GameplayEffect`，技能持续运行，无需调用 `EndAbility()`；
6. **客户端同步**：服务器同步 `GameplayEffect` 效果和 `GameplayCue` 到客户端，客户端显示被动效果（如角色身上的防御光环）。

这套逻辑的核心是利用 `OnAvatarSet` 精准匹配被动技能 “与角色绑定即激活” 的需求，同时通过 “Server Only” 策略保证数据安全和一致性，是 GAS 中实现被动技能的标准范式。

### 4.6.5 取消Ability

在 GAS 中，“取消 `GameplayAbility`（GA）” 的核心是**终止技能的运行逻辑、清理资源，并同步状态**—— 内部取消（技能自己停）和外部取消（ASC 强制停）的逻辑不同，且需要注意 “非实例化 GA” 的特殊坑。下面结合函数用途、场景和注意事项，把取消逻辑讲透：

#### 一、先明确：取消 GA 的核心目的

无论是内部还是外部取消，最终都是为了：

1. 终止技能的异步逻辑（如 `AbilityTask` 播放的动画、投射物飞行）；
2. 清理技能占用的资源（如定时器、临时数据）；
3. 同步技能状态（告知服务器 / 客户端 “技能已取消”，避免两边状态不一致）；
4. 允许技能重新激活（未正确取消的 GA 会一直处于 “激活中”，无法再次触发）。

#### 二、内部取消：`CancelAbility()`—— 技能自己 “喊停”

`CancelAbility()` 是 `UGameplayAbility` 类的成员函数，**只能在 GA 内部调用**（如技能逻辑中判断 “满足取消条件” 时主动触发）。

##### 核心逻辑：

调用 `CancelAbility()` 后，会自动执行两步：

1. 调用 `EndAbility()` 函数，并将 `WasCancelled` 参数设为 `true`（标记 “技能是被取消的，而非正常结束”）；
2. 终止 GA 关联的所有 `AbilityTask`（如播放动画的 `UAT_PlayMontageAndWait`、等待延迟的 `UAT_WaitDelay`），避免异步逻辑继续执行。

##### 适用场景：

技能内部的 “主动取消”，比如：

- 蓄力技能（按住蓄力，松开前按 ESC 取消）：在 `UAT_WaitInputRelease` 的 “取消回调” 中调用 `CancelAbility()`；
- 引导技能（持续施法，按右键中断）：在 “中断输入” 的响应逻辑中调用 `CancelAbility()`。

##### 代码示例（蓄力技能取消）：

```cpp
void UGA_ChargeFireball::ActivateAbility(...)
{
    Super::ActivateAbility(...);

    // 创建“等待输入松开”的 Task，同时监听“取消输入”
    UAT_WaitInputRelease* WaitReleaseTask = UAT_WaitInputRelease::Create(this);
    // 正常松开：执行发射逻辑
    WaitReleaseTask->OnRelease.AddUObject(this, &UGA_ChargeFireball::OnInputReleased);
    // 取消输入（如按 ESC）：调用 CancelAbility()
    WaitReleaseTask->OnCancel.AddUObject(this, &UGA_ChargeFireball::OnInputCancelled);
    WaitReleaseTask->ReadyForActivation();
}

void UGA_ChargeFireball::OnInputCancelled()
{
    // 内部取消技能，自动调用 EndAbility(..., true)
    CancelAbility();
}

void UGA_ChargeFireball::EndAbility(...)
{
    if (WasCancelled)
    {
        // 处理取消后的逻辑（如停止蓄力动画、返还部分蓝量）
        StopChargeMontage();
        RefundMana(50); // 返还50%蓝量
    }
    else
    {
        // 正常结束的逻辑（如发射火球）
        SpawnFireball();
    }

    Super::EndAbility(...);
}
```

#### 三、外部取消：ASC 的 5 个函数 —— 强制终止技能

当需要从 “技能外部” 取消 GA 时（如角色被眩晕、场景切换、技能优先级冲突），需调用 `AbilitySystemComponent`（ASC）提供的取消函数。这 5 个函数覆盖了 “按类取消”“按实例取消”“批量取消” 等场景，各有明确用途。

##### 1. 按 GA 类取消：`CancelAbility(UGameplayAbility* Ability)`

- **作用**：取消 ASC 中所有 “指定类” 的激活中 GA（比如取消所有 `UGA_Fireball` 类的技能）。
- **关键细节**：参数是 GA 的**类对象**（如 `UGA_Fireball::StaticClass()`），会取消该类的所有激活实例（若同一类有多个激活的 GA，会全部取消）。
- **适用场景**：需要 “批量终止某类技能”，比如 “角色被眩晕时，取消所有正在施法的攻击技能”。

**代码示例（眩晕取消攻击技能）：**

```cpp
// 角色被眩晕时，调用该函数
void AMyCharacter::OnStunned()
{
    if (AbilitySystemComponent)
    {
        // 取消所有“攻击类 GA”（假设 UGA_Attack 是所有攻击技能的父类）
        AbilitySystemComponent->CancelAbility(UGA_Attack::StaticClass());
    }
}
```

##### 2. 按 SpecHandle 取消：`CancelAbilityHandle(const FGameplayAbilitySpecHandle& AbilityHandle)`

- **作用**：取消 ASC 中 “特定实例” 的 GA—— 通过 `FGameplayAbilitySpecHandle`（GA 实例的唯一标识）精准定位，只取消目标实例，不影响同类其他实例。
- **关键细节**：`AbilityHandle` 是授予 GA 时 `ASC->GiveAbility()` 返回的标识，需提前保存（比如存在角色的变量中）。
- **适用场景**：同一类 GA 有多个激活实例，需精准取消某一个，比如 “玩家同时激活两个 `UGA_Shield`（小盾和大盾），只取消小盾”。

**代码示例（精准取消小盾技能）：**

```cpp
// 授予小盾技能时，保存其 SpecHandle
FGameplayAbilitySpec SmallShieldSpec(UGA_SmallShield::StaticClass());
FGameplayAbilitySpecHandle SmallShieldHandle = AbilitySystemComponent->GiveAbility(SmallShieldSpec);
SavedShieldHandle = SmallShieldHandle; // 保存到角色的变量中

// 后续需要取消小盾时
void AMyCharacter::CancelSmallShield()
{
    if (AbilitySystemComponent && SavedShieldHandle.IsValid())
    {
        AbilitySystemComponent->CancelAbilityHandle(SavedShieldHandle);
    }
}
```

##### 3. 按标签批量取消：`CancelAbilities(const FGameplayTagContainer* WithTags=nullptr, const FGameplayTagContainer* WithoutTags=nullptr, UGameplayAbility* Ignore=nullptr)`

- 作用：按`GameplayTag`筛选并取消激活中的 GA，支持 “包含指定标签”“排除指定标签”“忽略某个 GA” 三种筛选规则。
  - `WithTags`：只取消 “带有这些标签” 的 GA（如取消所有带 `Tag.Ability.Type.Attack` 的攻击技能）；
  - `WithoutTags`：取消 “不带有这些标签” 的 GA（如取消除带 `Tag.Ability.Type.Defense` 外的所有技能）；
  - `Ignore`：取消时跳过这个 GA（如取消所有技能，但保留 “无敌技能”）。
- **适用场景**：需要 “按标签分类取消”，比如 “场景切换时，取消所有非被动技能（带 `Tag.Ability.Type.Active`），保留被动技能”。

**代码示例（场景切换取消主动技能）：**

```cpp
void AMyPlayerController::OnLevelChanged()
{
    if (AMyCharacter* MyChar = GetPawn<AMyCharacter>())
    {
        if (UAbilitySystemComponent* ASC = MyChar->GetAbilitySystemComponent())
        {
            // 定义“主动技能”标签
            FGameplayTagContainer ActiveAbilityTags;
            ActiveAbilityTags.AddTag(FGameplayTag::RequestGameplayTag("Ability.Type.Active"));

            // 取消所有带“主动技能”标签的 GA，忽略“无敌技能”
            ASC->CancelAbilities(&ActiveAbilityTags, nullptr, UGA_GodMode::StaticClass());
        }
    }
}
```

##### 4. 取消所有技能：`CancelAllAbilities(UGameplayAbility* Ignore=nullptr)`

- **作用**：取消 ASC 中所有激活中的 GA，可选 “忽略某个 GA”（如取消所有技能，但保留 “复活技能”）。
- **关键坑点**：对 **非实例化 GA（Non-Instanced GA）** 支持不好 —— 如样例项目中的 “跳跃技能”（通常设为非实例化，因为不需要多实例），`CancelAllAbilities` 可能会 “命中该 GA 后停止遍历”，导致其他技能没被取消。
- **适用场景**：需要 “一刀切” 取消所有技能，且不涉及非实例化 GA 的场景，比如 “角色死亡时，取消所有正在运行的技能”。

**代码示例（角色死亡取消所有技能）：**

```cpp
void AMyCharacter::OnDeath()
{
    if (AbilitySystemComponent)
    {
        // 取消所有技能，忽略“死亡动画技能”（确保死亡动画能正常播放）
        AbilitySystemComponent->CancelAllAbilities(UGA_DeathAnimation::StaticClass());
    }
}
```

##### 5. 彻底销毁激活状态：`DestroyActiveState()`

- **作用**：比 `CancelAllAbilities` 更彻底 —— 不仅取消所有激活中的 GA，还会销毁 ASC 中所有 “实例化 GA” 的状态（如清除 `ActivatableAbilities` 列表中的临时数据），相当于 “重置 ASC 的技能激活状态”。
- **适用场景**：需要 “完全重置技能系统” 的极端场景，比如 “角色重生时，彻底清理上一轮的技能状态，避免残留逻辑影响新生命周期”。

**代码示例（角色重生重置技能状态）：**

```cpp
void AMyCharacter::OnRespawn()
{
    if (AbilitySystemComponent)
    {
        // 彻底销毁所有激活状态，重置技能系统
        AbilitySystemComponent->DestroyActiveState();

        // 重新授予初始技能
        AddCharacterAbilities();
    }
}
```

#### 四、关键注意事项：非实例化 GA 的取消坑

样例中提到 “`CancelAllAbilities` 对非实例化 GA 支持不好”，需要重点理解：

##### 1. 什么是 “非实例化 GA（Non-Instanced GA）”？

GA 的 “实例化” 指：授予 GA 时，是否为每个授予操作创建独立的 `FGameplayAbilitySpec` 实例。

- **实例化 GA**：每次 `GiveAbility()` 都会创建新的 `Spec`（如 “火球术”，可同时激活多个实例）；
- **非实例化 GA**：所有 `GiveAbility()` 共享同一个 `Spec`（如 “跳跃”“冲刺”，同一时间只能激活一个，无需多实例）。

##### 2. `CancelAllAbilities` 的坑点：

`CancelAllAbilities` 遍历 ASC 的激活中 GA 时，遇到非实例化 GA 可能会 “提前终止遍历”（引擎底层逻辑对非实例化 GA 的处理优先级特殊），导致后续的实例化 GA 没被取消。

##### 3. 解决方案：

对非实例化 GA，优先用 `CancelAbility()`（按类取消）或 `CancelAbilityHandle()`（按实例取消），避免用 `CancelAllAbilities`。
例如，取消 “跳跃”（非实例化 GA）：

```cpp
// 正确：用 CancelAbility 按类取消非实例化 GA
AbilitySystemComponent->CancelAbility(UGA_Jump::StaticClass());

// 错误：用 CancelAllAbilities 可能无法取消，还会影响其他技能
// AbilitySystemComponent->CancelAllAbilities();
```

#### 五、总结：如何选择取消方式？

| 取消场景                          | 推荐函数                         | 注意事项                                   |
| --------------------------------- | -------------------------------- | ------------------------------------------ |
| 技能内部主动取消                  | `CancelAbility()`（GA 成员函数） | 自动调用 `EndAbility(true)`，清理 Task     |
| 外部精准取消某一个 GA 实例        | `CancelAbilityHandle()`          | 需提前保存 `SpecHandle`                    |
| 外部批量取消某类 GA               | `CancelAbility()`（ASC 函数）    | 按类取消，覆盖所有实例                     |
| 外部按标签批量筛选取消            | `CancelAbilities()`              | 灵活用 `WithTags`/`WithoutTags` 筛选       |
| 外部取消所有技能（无需求实例 GA） | `CancelAllAbilities()`           | 避开非实例化 GA，否则用 `CancelAbility` 补 |
| 外部彻底重置技能激活状态          | `DestroyActiveState()`           | 仅用于角色重生、场景切换等极端场景         |

核心原则：**精准取消优先用 `CancelAbilityHandle`，批量取消优先用 `CancelAbilities`（按标签），非实例化 GA 避免用 `CancelAllAbilities`**，确保取消逻辑无遗漏、不影响其他技能。

### 4.6.6 获取激活的Ability

**对文中该部分代码的解释**：

`UAbilitySystemComponent::GetActivatableGameplayAbilitySpecsByAllMatchingTags` 是 GAS 中一个用于**按标签筛选 “可激活的 `GameplayAbility` 规格（`FGameplayAbilitySpec`）”** 的核心函数，它能帮助你快速定位符合特定标签条件的技能，在批量技能管理、逻辑联动等场景中非常实用。

#### 一、函数作用与参数解析

##### 核心作用：

从 `ASC` 的 `ActivatableAbilities` 列表中，筛选出**所有标签完全匹配指定标签容器**的 `FGameplayAbilitySpec`（技能规格），并返回这些规格的指针数组。
简单说：“找出所有带有我要的标签的可激活技能规格”。

##### 参数详解：

```cpp
void GetActivatableGameplayAbilitySpecsByAllMatchingTags(
    const FGameplayTagContainer& GameplayTagContainer,  // 1. 要匹配的标签容器
    TArray<struct FGameplayAbilitySpec*>& MatchingGameplayAbilities,  // 2. 输出参数：匹配的技能规格数组
    bool bOnlyAbilitiesThatSatisfyTagRequirements = true  // 3. 是否仅包含满足标签需求的技能
)
```

1. **`GameplayTagContainer`**：
   筛选的 “标签条件”—— 只有**同时包含该容器中所有标签**的技能规格才会被匹配（“全包含” 逻辑，而非 “任一包含”）。
   例如：容器包含 `Tag.Ability.Type.Attack` 和 `Tag.Ability.Damage.Fire`，则只会匹配同时带有这两个标签的技能。
2. **`MatchingGameplayAbilities`**：
   输出参数（引用传递）—— 函数执行后，会将所有匹配的 `FGameplayAbilitySpec*` 存入该数组，供后续逻辑使用（如激活、取消、查询状态等）。
3. **`bOnlyAbilitiesThatSatisfyTagRequirements`**：
   可选参数（默认 `true`）—— 若为 `true`，则仅返回 “满足自身标签需求（`TagRequirements`）” 的技能规格。
   - 技能的 `TagRequirements` 是在 `UGameplayAbility` 中定义的 “激活前置标签条件”（如 “必须带有 `Tag.State.Alive` 才能激活”）；
   - 设为 `true` 时，即使技能带有目标标签，但不满足自身 `TagRequirements`，也不会被纳入结果（确保筛选出的技能 “真的可以被激活”）。

#### 二、典型使用场景

##### 1. 批量激活符合标签的技能

例如：“战斗开始时，自动激活所有带有 `Tag.Ability.BattleStart` 和 `Tag.Ability.Passive` 的被动技能”。

```cpp
void AMyCharacter::OnBattleStart()
{
    if (!AbilitySystemComponent) return;

    // 1. 定义要匹配的标签（战斗开始 + 被动技能）
    FGameplayTagContainer BattlePassiveTags;
    BattlePassiveTags.AddTag(FGameplayTag::RequestGameplayTag("Ability.BattleStart"));
    BattlePassiveTags.AddTag(FGameplayTag::RequestGameplayTag("Ability.Passive"));

    // 2. 筛选匹配的技能规格
    TArray<FGameplayAbilitySpec*> MatchingSpecs;
    AbilitySystemComponent->GetActivatableGameplayAbilitySpecsByAllMatchingTags(
        BattlePassiveTags, 
        MatchingSpecs, 
        true  // 仅包含满足标签需求的技能
    );

    // 3. 批量激活这些技能
    for (FGameplayAbilitySpec* Spec : MatchingSpecs)
    {
        if (Spec && Spec->Ability)
        {
            AbilitySystemComponent->TryActivateAbility(Spec->Handle);
        }
    }
}
```

##### 2. 检查是否存在符合标签的激活中技能

例如：“释放大招前，检查是否有带有 `Tag.Ability.Buff.DamageBoost` 的增益技能在激活中，若有则叠加伤害”。

```cpp
float UGA_Ultimate::CalculateUltimateDamage()
{
    float BaseDamage = 1000.0f;
    if (!CurrentActorInfo || !CurrentActorInfo->AbilitySystemComponent) return BaseDamage;

    // 1. 定义要匹配的标签（伤害增益 buff）
    FGameplayTagContainer DamageBoostTags;
    DamageBoostTags.AddTag(FGameplayTag::RequestGameplayTag("Ability.Buff.DamageBoost"));

    // 2. 筛选匹配的技能规格
    TArray<FGameplayAbilitySpec*> MatchingSpecs;
    CurrentActorInfo->AbilitySystemComponent->GetActivatableGameplayAbilitySpecsByAllMatchingTags(
        DamageBoostTags, 
        MatchingSpecs, 
        true
    );

    // 3. 若存在激活中的增益技能，增加伤害
    for (FGameplayAbilitySpec* Spec : MatchingSpecs)
    {
        if (Spec && Spec->Ability && Spec->Ability->IsActive())
        {
            BaseDamage *= 1.5f; // 伤害提升 50%
            break; // 假设只需要一个增益即可
        }
    }

    return BaseDamage;
}
```

##### 3. 批量取消带有特定标签的技能

例如：“角色被沉默时，取消所有带有 `Tag.Ability.Type.Active` 和 `Tag.Ability.RequiresCast` 的主动施法技能”。

```cpp
void AMyCharacter::OnSilenced()
{
    if (!AbilitySystemComponent) return;

    // 1. 定义要匹配的标签（主动技能 + 需要施法）
    FGameplayTagContainer CastingAbilityTags;
    CastingAbilityTags.AddTag(FGameplayTag::RequestGameplayTag("Ability.Type.Active"));
    CastingAbilityTags.AddTag(FGameplayTag::RequestGameplayTag("Ability.RequiresCast"));

    // 2. 筛选匹配的技能规格
    TArray<FGameplayAbilitySpec*> MatchingSpecs;
    AbilitySystemComponent->GetActivatableGameplayAbilitySpecsByAllMatchingTags(
        CastingAbilityTags, 
        MatchingSpecs, 
        true
    );

    // 3. 批量取消这些技能
    for (FGameplayAbilitySpec* Spec : MatchingSpecs)
    {
        if (Spec && Spec->Handle.IsValid())
        {
            AbilitySystemComponent->CancelAbilityHandle(Spec->Handle);
        }
    }
}
```

#### 三、关键注意事项

1. **“全匹配” 逻辑**：
   函数要求技能规格**同时包含 `GameplayTagContainer` 中的所有标签**，而非 “包含任一标签”。若需要 “任一匹配”，应使用 `GetActivatableGameplayAbilitySpecsByAnyMatchingTags`（注意函数名中的 “Any”）。
2. **`FGameplayAbilitySpec` 与 `UGameplayAbility` 的区别**：
   函数返回的是 `FGameplayAbilitySpec*`（技能规格，包含实例化信息如等级、输入绑定），而非 `UGameplayAbility*`（技能类本身）。若需获取技能类，可通过 `Spec->Ability` 访问。
3. **性能考量**：
   频繁调用（如每帧）可能导致性能开销，尤其是当 `ActivatableAbilities` 列表庞大时。建议：
   - 对高频逻辑，缓存筛选结果（如战斗开始时缓存一次，战斗结束后清空）；
   - 减少标签容器中的标签数量（标签越多，匹配越慢）。
4. **标签来源**：
   技能规格的标签来自两个地方：
   - 技能类（`UGameplayAbility`）中定义的 `AbilityTags`（静态标签）；
   - 授予技能时通过 `FGameplayAbilitySpec` 动态添加的标签（动态标签）。
     函数会同时检查这两类标签，确保匹配准确性。

#### 总结

`GetActivatableGameplayAbilitySpecsByAllMatchingTags` 是 GAS 中**按标签批量管理技能**的核心工具，它通过 “全标签匹配” 逻辑精准筛选可激活的技能规格，支持批量激活、取消、状态查询等场景。使用时需注意 “全匹配” 与 “任一匹配” 的区别，以及性能优化，确保在复杂技能系统中高效运行。

### 4.6.7 实例化策略

`GameplayAbility`（GA）的实例化策略是 GAS 中**控制技能实例生命周期和状态管理**的核心机制，直接影响技能的性能、状态存储能力和使用限制。三种策略的本质区别在于 “是否创建实例”“实例的复用方式” 以及 “能否存储动态状态”，下面结合实际开发场景深入解析：

#### 一、按 Actor 实例化（Instanced Per Actor）：“单实例复用，状态持久化”

##### 核心特点：

- **实例生命周期**：每个 `AbilitySystemComponent`（ASC）中，该 GA 只会创建**一个实例**，并在多次激活之间复用（不会销毁）。
- **状态管理**：支持存储动态状态（如变量值、冷却进度、累计数据等），且状态会在 “激活 - 结束” 循环中保留（除非手动重置）。
- **性能**：中等（仅初始化一次实例，后续激活无实例创建开销）。

##### 适用场景：

这是最常用的策略，适合**需要在多次激活之间保留状态**的技能 —— 几乎所有 “有记忆性” 的技能都适用：

- **角色核心技能**：如 “火球术”（需要记录 “已击中敌人次数” 来触发暴击加成）、“闪避”（需要记录 “上次闪避时间” 来计算冷却）。
- **带成长 / 累计效果的技能**：如 “连击技能”（第一次攻击造成 100 伤害，第二次 150，需要累计连击数）。
- **需要手动重置状态的技能**：如 “蓄力攻击”（松开按键后重置蓄力值，但下次激活时保留 “最大蓄力上限” 配置）。

##### 注意事项：

- 由于实例复用，技能的成员变量（如 `CurrentChargeValue`）会在多次激活之间保留，需在 `EndAbility` 或 `ActivateAbility` 中手动重置不需要的状态（避免影响下次激活）。
- 每个 ASC 仅一个实例，因此同一角色的该技能多次激活会共享状态，但不同角色（不同 ASC）的实例相互独立（不冲突）。

#### 二、按操作实例化（Instanced Per Execution）：“多实例隔离，状态重置”

##### 核心特点：

- **实例生命周期**：**每次激活技能时，都会创建全新的 GA 实例**，技能结束后实例被销毁（不会复用）。
- **状态管理**：每次激活的状态（变量值、任务委托等）完全隔离，新实例的变量会使用默认值（自动重置）。
- **性能**：较差（每次激活都有实例创建 / 销毁的开销，频繁激活会增加内存压力）。

##### 适用场景：

适合**每次激活需要 “全新状态” 且无需复用数据**的技能，尤其是 “一次性效果” 或 “并行激活” 的技能：

- **独立触发的一次性技能**：如 “治疗脉冲”（每次激活生成一个独立的治疗圈，圈的范围、持续时间互不影响）。
- **可叠加的瞬时技能**：如 “多重箭”（同时激活 3 次，每次生成一个独立的箭实例，各自计算飞行轨迹）。
- **需要完全隔离逻辑的技能**：如 “召唤物技能”（每个召唤物的攻击技能实例独立，不会相互干扰）。

##### 注意事项：

- 由于每次激活都是新实例，无需手动重置状态（默认值自动生效），但会增加性能开销（不适合高频激活的技能，如每秒触发的普攻）。
- 样例项目通常不使用这种策略，除非有明确的 “多实例隔离” 需求。

#### 三、非实例化（Non-Instanced）：“无实例，直接操作 CDO，性能最优”

##### 核心特点：

- **实例生命周期**：**不创建任何实例**，所有激活逻辑直接操作 GA 的 “类默认对象（CDO）”（`UGameplayAbility::GetDefaultObject()`）。
- **状态管理**：**完全不能存储动态状态**（CDO 是全局共享的默认对象，修改其变量会影响所有使用该 GA 的对象），也无法绑定 `AbilityTask` 的委托（委托需要实例上下文）。
- **性能**：最优（无实例创建 / 销毁开销，直接执行逻辑）。

##### 适用场景：

适合**逻辑简单、无状态存储需求、且需要高频激活**的技能 —— 核心是 “无动态数据，每次激活逻辑完全一致”：

- **高频简单技能**：如 MOBA 小兵的基础攻击（仅播放动画 + 应用伤害，无需记录状态）、角色的跳跃（仅触发物理力 + 播放动画，无复杂逻辑）。
- **无变量依赖的瞬时技能**：如 “嘲讽”（仅广播一个事件 + 应用标签，不需要存储任何数据）。

##### 注意事项：

- **严禁存储动态状态**：任何对 CDO 变量的修改（如 `bIsJumping = true`）会影响所有使用该 GA 的角色（因为 CDO 全局共享）。
- **无法使用 `AbilityTask` 委托**：`AbilityTask` 的回调（如动画结束、延迟完成）需要绑定到实例，非实例化 GA 没有实例，会导致委托失效或崩溃。
- 样例项目的 “跳跃技能” 用这种策略，正因为其逻辑简单（仅触发物理跳跃）、无状态、高频使用（性能优先）。

#### 四、三种策略的核心对比与选择依据

| 维度                  | 按 Actor 实例化       | 按操作实例化                 | 非实例化                     |
| --------------------- | --------------------- | ---------------------------- | ---------------------------- |
| **实例数量**          | 每个 ASC 1 个（复用） | 每次激活 1 个（新实例）      | 0 个（直接用 CDO）           |
| **状态存储**          | 支持（激活间保留）    | 支持（每次激活独立）         | 不支持（CDO 共享，禁止修改） |
| **性能**              | 中等                  | 较差（高频激活压力大）       | 最优                         |
| **`AbilityTask`支持** | 完全支持              | 完全支持                     | 不支持（无实例绑定委托）     |
| **典型场景**          | 核心技能、带状态技能  | 一次性叠加技能、独立实例技能 | 高频简单技能（普攻、跳跃）   |

#### 五、选择策略的决策流程

1. **是否需要存储动态状态？**
   - 是 → 排除 “非实例化”；
   - 否 → 优先考虑 “非实例化”（性能最优）。
2. **是否需要每次激活的状态完全隔离？**
   - 是 → 选择 “按操作实例化”（适合一次性、叠加技能）；
   - 否 → 选择 “按 Actor 实例化”（适合复用状态的技能）。
3. **激活频率如何？**
   - 高频激活（如每秒多次）→ 排除 “按操作实例化”（性能差）；
   - 低频激活 → 可考虑 “按操作实例化”（隔离性优先）。

#### 总结

实例化策略的本质是**在 “性能”“状态管理”“逻辑隔离性” 之间做权衡**：

- 大多数技能用 “按 Actor 实例化”（平衡状态需求和性能）；
- 简单高频技能用 “非实例化”（性能优先，无状态）；
- 少数需要完全隔离的技能用 “按操作实例化”（隔离性优先，容忍性能开销）。

理解这三种策略的核心差异，才能避免 “非实例化技能尝试存储状态导致全局错误”“高频技能用按操作实例化导致卡顿” 等常见问题。

### 4.6.8 网络执行策略(Net Execution Policy)

文中的本节内容暂时没有看不懂的地方，有需要再补充。

### 4.6.9 Ability标签

文中的本节内容暂时没有看不懂的地方，有需要再补充。

### 4.6.10 Gameplay Ability Spec

文中的本节内容暂时没有看不懂的地方，有需要再补充。

### 4.6.11 传递数据到Ability

需要时再看

### 4.6.12 Ability花费(Cost)和冷却(Cooldown)

主要讲了`GA`激活之前会检查`Cost`和`Cooldown`是否支持当前`GA`的激活。

### 4.6.13 升级Ability

根据是否需要在升级`GA`时中断当前`GA`而选择性的使用文中提到的两种升级方式。

### 4.6.14 Ability集合

`UDataAsset`类的`GameplayAbilitySet`类，作用就是便利`GA`的统一管理使用。

### 4.6.15 Ability批处理

需要时再使用。

### 4.6.16 网络安全策略(Net Security Policy)

需要时再使用。

## 4.7 Ability Tasks

### 4.7.1 AbilityTask定义

`Ability Task`可以让`GA`不仅仅在一帧中执行任务，实现随时间推移而触发或响应一段时间后触发的委托操作。

`UAbilityTask`的构造函数中强制硬编码允许最多1000个同时运行的`AbilityTask`, 当设计那些同时拥有数百个Character的游戏(像RTS)的`GameplayAbility`时要注意这一点.  

### 4.7.2 自定义AbilityTask

看文中内容即可。

### 4.7.3 使用AbilityTask

看文中内容即可。

### 4.7.4 Root Motion Source Ability Task

GAS自带的`AbilityTask`可以使用挂载在`CharacterMovementComponent`中的`Root Motion Source`随时间推移而移动`Character`, 像击退, 复杂跳跃, 吸引和猛冲。

## 4.8 Gameplay Cues

### 4.8.1 GameplayCue定义

GameplayCue（简称 GC）是 Unreal Engine（UE）中**Gameplay Abilities 系统**的重要组成部分，专注于处理非核心游戏逻辑的表现层功能（如音效、粒子、镜头反馈等），同时具备同步性和可预测性，是实现 “功能与表现分离” 的关键工具。

#### 一、GameplayCue 的核心定位与特性

GC 的核心价值在于**解耦 “游戏规则逻辑”（如伤害计算、状态生效）与 “表现层反馈”（如受击火花、眩晕特效）**，让开发者无需在`GameplayEffect`（GE）或`GameplayAbility`（GA）中硬编码表现逻辑，而是通过 “标签订阅” 的方式触发效果。

##### 关键特性

1. **可同步性**：默认支持服务端 - 客户端同步（除非在客户端明确指定仅本地执行）；
2. **可预测性**：遵循 UE 网络预测逻辑，避免客户端表现与服务端状态脱节；
3. **标签驱动**：必须通过**以 “GameplayCue” 为父级的 GameplayTag**触发（如`GameplayCue.Character.Hit`），标签是 GC 的唯一标识；
4. **订阅机制**：通过`GameplayCueNotify`对象或实现`IGameplayCueInterface`的 Actor 订阅 GC 事件，无需硬编码关联。

#### 二、GameplayCue 的触发方式

GC 无法主动执行，必须通过`AbilitySystemComponent`（ASC，能力系统组件）触发，核心是 “发送带 GC 标签的事件”，具体步骤如下：

1. **确定 GC 标签**：创建一个有效的 GC 标签（必须以`GameplayCue`开头，例如`GameplayCue.Weapon.Shoot`）；

2. 选择事件类型

   ：GC 支持 3 种核心事件类型，对应不同的表现需求：

   - `Execute`：一次性触发（如子弹发射音效、单次伤害火花）；
   - `Add`：持续生效时激活（如眩晕状态的粒子特效启动）；
   - `Remove`：持续生效结束时关闭（如眩晕状态解除，粒子停止）；

3. **通过 ASC 触发**：在代码或蓝图中，通过 ASC 调用`GameplayCueManager`的对应接口（如`ExecuteGameplayCue`、`AddGameplayCue`），传入 GC 标签和触发参数（如位置、目标 Actor）。

#### 三、GameplayCueNotify：GC 的事件响应载体

`GameplayCueNotify`是 UE 提供的 “GC 事件处理器” 基类，分为两种核心类型，分别对应不同的使用场景，需根据 “效果是否需要持续实例化” 选择。

##### 两种 GameplayCueNotify 类对比

| 类名（Class）                         | 支持的事件  | 适配的 GameplayEffect 类型               | 核心特点                                                     | 适用场景                                                     |
| ------------------------------------- | ----------- | ---------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **GameplayCueNotify_Static**（静态）  | Execute     | Instant（瞬时）、Periodic（周期性）      | - 无实例化，直接操作`ClassDefaultObject`（CDO） - 性能开销低，仅触发一次 | 一次性表现（如：子弹命中火花、单次伤害音效、暴击镜头抖动）   |
| **GameplayCueNotify_Actor**（实例化） | Add、Remove | Duration（持续时长）、Infinite（无限期） | - 触发 Add 时生成实例，触发 Remove 时销毁（需手动配置） - 支持实例化状态管理（如粒子播放进度） - 可限制同时存在的实例数量 | 持续性表现（如：角色奔跑时的脚步声循环、眩晕状态的环绕粒子、中毒持续掉血的特效） |

##### 关键使用注意事项

- **GameplayCueNotify_Actor 必须勾选 “Auto Destroy on Remove”**：若不勾选，Remove 事件触发后实例不会销毁，后续再调用 Add 时无法生成新实例，导致表现失效；
- **事件重写**：两种类均需在蓝图 / 代码中重写对应事件（如`OnExecute`、`OnAdd`、`OnRemove`），在事件中编写具体表现逻辑（如 Spawn 粒子、播放 Sound）；
- **标签绑定**：每个`GameplayCueNotify`需在细节面板（Details）中绑定对应的 “GameplayCueTag”，才能接收该标签的 GC 事件。

#### 四、网络同步与客户端优化

GC 的同步行为与 ASC 的**同步模式（Replication Mode）** 强相关，需注意服务端与客户端的触发次数差异，避免表现重复或缺失。

##### 1. 同步模式对 GC 的影响

当 ASC 使用非 “Full” 同步模式（如 “Minimal” 或 “None”）时：

- 服务端（Listen Server，监听服务器）：`Add`和事件会触发两次：
  1. 第一次：服务端本地应用 GE 时触发；
  2. 第二次：通过 “Minimal NetMultiCast” 同步到客户端时，服务端自身也会响应；
- **客户端**：所有 GC 事件（Execute/Add/Remove）仅触发**一次**；
- **例外**：`WhileActive`事件（持续生效中的帧更新）在服务端和客户端均仅触发一次。

#### 2. 客户端本地触发优化（关键性能点）

对于 “客户端本地可见、无需服务端同步” 的表现（如角色自身的呼吸粒子、本地 UI 反馈），无需通过服务端 GE 触发 GC，可直接在客户端调用`ASC->ExecuteGameplayCueLocal`（本地执行），避免网络开销。

**示例场景**：

- 错误做法：服务端发送 “呼吸 GC” 标签，同步到客户端触发粒子；
- 正确做法：客户端检测角色是否存活，直接本地调用 “呼吸 GC”，无需服务端参与。

#### 五、样例项目参考（官方 / 常见实践）

UE 官方样例项目（如`ActionRPG`、`LyraStarterGame`）中，GC 的使用符合上述规范，典型案例如下：

| 样例功能     | GC 类型                  | 触发方式                           | 核心逻辑                                                     |
| ------------ | ------------------------ | ---------------------------------- | ------------------------------------------------------------ |
| 眩晕状态特效 | GameplayCueNotify_Actor  | GE（Duration 类型）触发 Add/Remove | - Add 事件：Spawn 眩晕环绕粒子、播放耳鸣音效 - Remove 事件：停止粒子、恢复音效 |
| 枪支伤害反馈 | GameplayCueNotify_Static | GA 触发 Execute 事件               | - 子弹命中时，服务端调用 Execute，同步触发火花粒子 + 撞击音效 - 客户端本地补充镜头微抖（无需同步） |
| 奔跑脚步声   | GameplayCueNotify_Actor  | 角色移动状态触发 Add/Remove        | - Add（奔跑时）：循环播放脚步声，限制 1 个实例 - Remove（停止奔跑）：停止音效，销毁实例 |

#### 六、总结与核心避坑点

1. **标签合法性**：GC 标签必须以`GameplayCue`开头，否则无法触发（如`Cue.Character.Hit`是无效标签）；
2. **实例销毁**：GameplayCueNotify_Actor 务必勾选 “Auto Destroy on Remove”，否则实例残留导致后续 Add 失效；
3. **网络次数**：Listen Server 的 Add/Remove 事件会触发两次，需在代码中判断`IsLocalController()`，避免重复表现；
4. **本地优先**：非关键同步的表现（如 UI 反馈、本地粒子）优先用`Local`接口，减少网络开销。

通过合理使用 GC，可让游戏的 “表现层逻辑” 更模块化、易维护，同时兼顾网络同步的稳定性与客户端性能。

### 4.8.2 触发GameplayCue

下面是对文中本节内容**手动调用GC**这一部分所示代码的解释：

这些函数是手动控制 GC 的主要接口，补充了通过`GameplayEffect`（GE）间接触发 GC 的方式。

#### 函数功能解析

##### 1. 一次性触发类（Execute）

```cpp
// 执行一次性GC，可传递效果上下文（如命中结果、源/目标信息）
void ExecuteGameplayCue(const FGameplayTag GameplayCueTag, 
                       FGameplayEffectContextHandle EffectContext = FGameplayEffectContextHandle());

// 执行一次性GC，通过专用参数结构体传递更详细的配置（如位置、旋转、缩放）
void ExecuteGameplayCue(const FGameplayTag GameplayCueTag, 
                       const FGameplayCueParameters& GameplayCueParameters);
```

- **用途**：触发`GameplayCueNotify_Static`的`OnExecute`事件，适合一次性表现（如子弹命中火花、技能释放音效）。
- **特点**：执行后无残留，不占用 GC 实例，参数可灵活传递触发时的上下文信息（如伤害位置、击中的 Actor）。

##### 2. 持续性添加类（Add）

```cpp
// 添加持续性GC，可传递效果上下文
void AddGameplayCue(const FGameplayTag GameplayCueTag, 
                   FGameplayEffectContextHandle EffectContext = FGameplayEffectContextHandle());

// 添加持续性GC，通过专用参数结构体配置
void AddGameplayCue(const FGameplayTag GameplayCueTag, 
                   const FGameplayCueParameters& GameplayCueParameters);
```

- **用途**：触发`GameplayCueNotify_Actor`的`OnAdd`事件，用于启动持续性表现（如角色进入潜行状态时的隐身粒子）。
- **特点**：会在 ASC 中注册该 GC 为 “活跃状态”，直到调用`RemoveGameplayCue`才会触发`OnRemove`事件。同一标签可被多次添加（但可通过`GameplayCueNotify_Actor`的 “实例限制” 设置控制是否重复生成）。

##### 3. 持续性移除类（Remove）

```cpp
// 移除指定标签的持续性GC
void RemoveGameplayCue(const FGameplayTag GameplayCueTag);

// 移除所有通过手动添加（非GE触发）的持续性GC
void RemoveAllGameplayCues();
```

- **用途**：终止持续性 GC 的表现（如角色退出潜行状态，停止隐身粒子）。
- 注意：
  - `RemoveGameplayCue`需与`AddGameplayCue`配对使用，否则可能导致`GameplayCueNotify_Actor`实例残留。
  - `RemoveAllGameplayCues`仅移除 “手动添加” 的 GC，通过 GE 自动触发的 GC 仍由 GE 的生命周期管理（GE 过期时自动移除）。

#### 关键参数说明

- **`FGameplayTag GameplayCueTag`**：
  必须是`GameplayCue.XXX`格式的有效标签（如`GameplayCue.Ability.Fireball.Explode`），用于定位对应的`GameplayCueNotify`对象。
- **`FGameplayEffectContextHandle`**：
  通常用于传递与游戏逻辑相关的上下文（如源 Actor、目标 Actor、命中结果`FHitResult`），可在`GameplayCueNotify`的事件中解析这些信息（如根据命中位置生成粒子）。
- **`FGameplayCueParameters`**：
  更专注于表现层的参数结构体，包含：
  - 空间信息（`Location`、`Rotation`、`Scale`）：控制粒子 / 音效的生成位置和朝向；
  - 源 / 目标 Actor 引用：用于绑定特效到特定 Actor（如角色身上的光环粒子）；
  - 自定义数据（`OptionalObject`、`OptionalFloat`等）：传递额外配置（如粒子颜色、音效音量）。

#### 典型使用场景

1. **手动触发一次性特效**：
   在角色跳跃时，直接调用`ExecuteGameplayCue`播放跳跃音效和地面灰尘粒子：

   ```cpp
   FGameplayTag JumpCueTag = UGameplayTagsManager::Get().RequestGameplayTag(FName("GameplayCue.Character.Jump"));
   FGameplayCueParameters Params;
   Params.Location = GetActorLocation(); // 以角色位置为中心
   ASC->ExecuteGameplayCue(JumpCueTag, Params);
   ```

2. **手动控制持续性状态**：
   角色进入 “狂暴模式” 时添加 GC（播放怒吼音效 + 红光粒子），退出时移除：

   ```cpp
   // 进入狂暴
   FGameplayTag BerserkCueTag = UGameplayTagsManager::Get().RequestGameplayTag(FName("GameplayCue.State.Berserk"));
   ASC->AddGameplayCue(BerserkCueTag);
   
   // 退出狂暴
   ASC->RemoveGameplayCue(BerserkCueTag);
   ```

#### 与 GE 触发的区别

- **手动调用**：**这些函数是 “主动触发”，无需依赖 GE**，适合非技能 / 状态类的表现（如交互反馈、环境音效）。
- **网络同步**：手动在客户端调用时，仅影响本地表现；在服务端调用时，会根据 ASC 同步模式同步到客户端（可能触发服务端两次执行，如前文所述）。
- **生命周期独立**：手动添加的 GC 需手动移除，而 GE 触发的 GC 会随 GE 的生效 / 失效自动管理。

通过这些接口，开发者可以灵活控制 GC 的触发时机，实现 “逻辑与表现” 的精细化分离。

### 4.8.3 客户端GameplayCue

文中主要讲的还是`GC`的触发方式，只不过这里是只在客户端触发的函数，函数名尾部都会多一个`Local`，注意文中的提示：**如果某个`GameplayCue`是客户端添加的, 那么它也应该自客户端移除. 如果它是通过同步添加的, 那么它也应该通过同步移除。**  

### 4.8.4 GameplayCue参数

文中主要介绍了参数如何传递给手动触发的GC和通过GE触发的GC。

这些参数常常用于确定声音、特效等播放或生成的位置、方向等信息。

### 4.8.5 Gameplay Cue Manager

#### 核心逻辑解析

默认情况下，`GameplayCueManager`的行为是：

1. 扫描`DefaultGame.ini`中`GameplayCueNotifyPaths`指定的路径，找到所有`GameplayCueNotify`类；
2. 自动异步加载这些类及其引用的所有资源（粒子、音效等），无论它们是否会在游戏中实际使用。

这种机制在小型项目中没问题，但在《Paragon》这类大型游戏中会导致：

- 内存浪费：数百兆未使用的资源驻留内存；
- 启动卡顿：大量资源异步加载导致启动时无响应；
- 性能冗余：预加载的资源可能从不会被触发。

#### 优化方案：按需加载`GameplayCue`

通过自定义`UGameplayCueManager`子类并修改加载策略，可实现 “仅加载实际触发的`GameplayCue`”：

##### 1. 步骤拆解

- **指定自定义`GameplayCueManager`**：
  在`DefaultGame.ini`中配置，让引擎使用你的子类而非默认类：

  ```ini
  [/Script/GameplayAbilities.AbilitySystemGlobals]
  GlobalGameplayCueManagerClass="/Script/你的模块名.你的子类名"
  ```

- **重写加载控制函数**：
  在子类中重写`ShouldAsyncLoadRuntimeObjectLibraries()`，返回`false`以禁用默认的 “全量异步加载”：

  ```cpp
  virtual bool ShouldAsyncLoadRuntimeObjectLibraries() const override
  {
      return false; // 关闭默认的全量异步加载
  }
  ```

- **效果**：
  引擎只会在`GameplayCue`首次被触发时，才加载对应的`GameplayCueNotify`及其关联资源（粒子、音效等），而非启动时全部加载。

##### 2. 优缺点与适用场景

| 优势                   | 潜在问题                                                     | 最佳适用场景                                                 |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 大幅减少启动时内存占用 | HDD 上首次触发可能有延迟（SSD 无此问题）                     | 大型项目（如 MOBA、开放世界） 含大量`GameplayCue`但实际使用分散的场景 |
| 加快游戏启动速度       | 编辑器中可能因粒子编译产生轻微卡顿（打包后消失）             | 资源总量大、品类多的项目                                     |
| 避免无用资源加载       | 需要确保关键`GameplayCue`在触发前预加载（如角色出生必用特效） | 对内存敏感的平台（如主机、移动端）                           |

#### 补充建议

- **关键资源预加载**：对于游戏初期必然触发的`GameplayCue`（如角色出生特效），可在关卡加载时手动调用异步加载接口（如`UGameplayStatics::LoadAsset`），避免首次触发延迟。
- **结合 SSD 优势**：如你所述，SSD 的高速随机读写可掩盖首次加载延迟，因此该优化在 HDD 平台上收益更明显。
- **路径配置精细化**：通过`GameplayCueNotifyPaths`限制扫描路径（如只扫描`/Game/Characters`而非整个项目），进一步减少`GameplayCueManager`的扫描和管理开销。

这种按需加载的思路，本质是 “空间换时间” 与 “时间换空间” 的权衡，在大型项目中是提升性能的重要实践。

### 4.8.6 阻止GameplayCue响应

### 4.8.7 GameplayCue批处理

文中该部分内容主要介绍了GC批处理时的优化方案

#### 4.8.7.1 手动RPC

#### 4.8.7.2 GameplayEffect中的多个GameplayCue

## 4.9 Ability System Globals

### 4.9.1 InitGlobalData()

#### 一、问题本质：为什么必须调用`UAbilitySystemGlobals::InitGlobalData()`？

在 UE 4.24 之前，`AbilitySystem`的核心数据（如`TargetData`的`ScriptStruct`、属性配置表）会自动初始化；但从 4.24 开始，引擎将这部分初始化逻辑剥离为显式调用的`InitGlobalData()`，目的是让开发者更灵活地控制初始化时机（避免与其他系统加载顺序冲突）。

如果不调用该函数，会触发两个核心问题：

1. **`ScriptStructCache`错误**：`TargetData`依赖的`ScriptStruct`（如`FGameplayAbilityTargetData_SingleTargetHit`）无法被缓存到`ScriptStructCache`中，导致代码中访问`TargetData`时找不到对应的结构定义；
2. **客户端断连**：服务端与客户端的`TargetData`序列化 / 反序列化依赖`ScriptStructCache`的一致性，缓存缺失会导致网络数据解析失败，最终客户端从服务端断开。

#### 二、`UAbilitySystemGlobals::InitGlobalData()`的正确调用方式

该函数只需在项目生命周期中**调用一次**，但调用位置需根据项目架构和加载顺序选择，不同方案各有优劣，以下是主流实践对比：

| 调用位置                                  | 实现方式                                                     | 优点                                                         | 适用场景                                                     | 注意事项                                                     |
| ----------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **`UEngineSubsystem::Initialize()`**      | 1. 创建自定义`EngineSubsystem`（继承`UEngineSubsystem`）； 2. 在`Initialize()`重写函数中调用`UAbilitySystemGlobals::InitGlobalData()`； 3. 确保`Subsystem`被引擎自动加载（在`.uplugin`中配置`LoadingPhase=PreDefault`）。 | - 加载时机早，能覆盖大多数系统的初始化需求； - 符合 UE “Subsystem” 的模块化设计，代码整洁。 | 大多数普通项目（无特殊`AssetManager`或`GlobalAttributeDefaultsTable`依赖）； 样例项目推荐方案。 | 若项目使用`GlobalAttributeDefaultsTables`（全局属性默认表），可能因`Subsystem`加载顺序早于`EditorSubsystem`导致崩溃（下文会解决）。 |
| **`UEngine::Init()`**                     | 1. 自定义`Engine`类（继承`UEngine`）； 2. 在`Init()`函数中调用`UAbilitySystemGlobals::InitGlobalData()`； 3. 在`DefaultEngine.ini`中指定自定义`Engine`类： `[/Script/Engine.Engine] EngineClass=/Script/你的模块名.你的Engine类`。 | - 初始化时机极早，能确保在所有`AbilitySystem`相关逻辑前执行； 适合需要深度定制引擎初始化流程的项目。 | Paragon 等大型项目； 需自定义引擎逻辑的场景。                | 实现成本较高（需重写`Engine`类），普通项目无需使用。         |
| **`AssetManager::StartInitialLoading()`** | 1. 自定义`AssetManager`（继承`UAssetManager`）； 2. 在`StartInitialLoading()`（资源加载起始函数）中调用`UAbilitySystemGlobals::InitGlobalData()`； 3. 在`DefaultEngine.ini`中指定自定义`AssetManager`： `[/Script/Engine.Engine] AssetManagerClassName=/Script/你的模块名.你的AssetManager类`。 | - 能确保在`GlobalAttributeDefaultsTables`（依赖资源加载）初始化前执行； 解决`Subsystem`加载顺序导致的崩溃问题。 | Fortnite 等依赖`AssetManager`管理全局资源的项目； 使用`GlobalAttributeDefaultsTables`后出现崩溃的项目。 | 依赖`AssetManager`的加载逻辑，若项目未自定义`AssetManager`，需先实现基础框架。 |
| **`GameInstance::Init()`**                | 1. 自定义`GameInstance`（继承`UGameInstance`）； 2. 在`Init()`函数中调用`UAbilitySystemGlobals::InitGlobalData()`； 3. 在`DefaultEngine.ini`中指定自定义`GameInstance`： `[/Script/Engine.Engine] GameInstanceClass=/Script/你的模块名.你的GameInstance类`。 | - 加载时机晚于`Subsystem`，但能确保`EditorSubsystem`已初始化； 适合解决 “`Subsystem`加载顺序导致的`GlobalAttributeDefaultsTables`崩溃”。 | 项目使用`GlobalAttributeDefaultsTables`且在`Subsystem`中调用崩溃时； 依赖`GameInstance`初始化流程的项目。 | 若项目有早于`GameInstance`初始化的`AbilitySystem`逻辑（如引擎启动阶段的测试代码），可能因初始化晚导致错误。 |

#### 三、崩溃解决方案：`GlobalAttributeDefaultsTables`引发的加载顺序问题

当使用`AbilitySystemGlobals`的`GlobalAttributeSetDefaultsTableNames`（全局属性默认表名列表）时，若在`UEngineSubsystem::Initialize()`中调用`InitGlobalData()`，可能因 **`EngineSubsystem`加载早于`EditorSubsystem`** 导致崩溃 —— 核心原因是：
`GlobalAttributeDefaultsTables`需要`EditorSubsystem`（编辑器子系统）来绑定`InitGlobalData()`中的委托（如属性表加载完成后的回调），而`EngineSubsystem`的初始化时机早于`EditorSubsystem`，导致委托绑定失败，触发空指针崩溃。

##### 解决步骤：

1. **确认崩溃原因**：若崩溃调用栈中包含`UAbilitySystemGlobals::InitGlobalData()`和`GlobalAttributeDefaultsTables`相关代码，即可判定为加载顺序问题；

2. 更换调用位置：将`UAbilitySystemGlobals::InitGlobalData()`从`UEngineSubsystem::Initialize()`迁移到以下位置（二选一）：

   - 方案 1：`AssetManager::StartInitialLoading()`

     （推荐，贴合 Fortnite 实践）：

     ```cpp
     void UMyAssetManager::StartInitialLoading()
     {
         Super::StartInitialLoading();
         // 初始化AbilitySystem全局数据
         UAbilitySystemGlobals::InitGlobalData();
     }
     ```

   - 方案 2：`GameInstance::Init()`：

     ```cpp
     void UMyGameInstance::Init()
     {
         Super::Init();
         // 初始化AbilitySystem全局数据
         UAbilitySystemGlobals::InitGlobalData();
     }
     ```

3. **验证加载顺序**：确保`AssetManager`或`GameInstance`的初始化时机晚于`EditorSubsystem`（UE 默认如此，无需额外配置），此时`GlobalAttributeDefaultsTables`能正常绑定委托，崩溃问题解决。

#### 四、模板代码：直接复用的初始化实现

为避免重复踩坑，以下提供两种场景的模板代码，可直接复制到项目中使用：

##### 场景 1：普通项目（无`GlobalAttributeDefaultsTables`）——`EngineSubsystem`实现

```cpp
// 1. 头文件（MyAbilitySystemSubsystem.h）
#include "CoreMinimal.h"
#include "Subsystems/EngineSubsystem.h"
#include "MyAbilitySystemSubsystem.generated.h"

UCLASS()
class YOUR_PROJECT_NAME_API UMyAbilitySystemSubsystem : public UEngineSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
};

// 2. 源文件（MyAbilitySystemSubsystem.cpp）
#include "MyAbilitySystemSubsystem.h"
#include "AbilitySystemGlobals.h"

void UMyAbilitySystemSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    // 初始化AbilitySystem全局数据（UE 4.24+必需）
    UAbilitySystemGlobals::InitGlobalData();
}

void UMyAbilitySystemSubsystem::Deinitialize()
{
    Super::Deinitialize();
    // 无需额外清理，引擎自动处理
}

// 3. 配置.uplugin文件（确保Subsystem被加载）
"Modules": [
    {
        "Name": "YOUR_PROJECT_NAME",
        "Type": "Runtime",
        "LoadingPhase": "PreDefault",
        "Subsystems": [
            {
                "Class": "MyAbilitySystemSubsystem",
                "AutoCreate": true
            }
        ]
    }
]
```

##### 场景 2：使用`GlobalAttributeDefaultsTables`——`AssetManager`实现

```cpp
// 1. 头文件（MyAssetManager.h）
#include "CoreMinimal.h"
#include "AssetManager/AssetManager.h"
#include "MyAssetManager.generated.h"

UCLASS()
class YOUR_PROJECT_NAME_API UMyAssetManager : public UAssetManager
{
    GENERATED_BODY()

public:
    virtual void StartInitialLoading() override;
};

// 2. 源文件（MyAssetManager.cpp）
#include "MyAssetManager.h"
#include "AbilitySystemGlobals.h"

void UMyAssetManager::StartInitialLoading()
{
    Super::StartInitialLoading();
    // 初始化AbilitySystem全局数据（解决GlobalAttributeDefaultsTables崩溃）
    UAbilitySystemGlobals::InitGlobalData();
}

// 3. 配置DefaultEngine.ini（指定自定义AssetManager）
[/Script/Engine.Engine]
AssetManagerClassName="/Script/YOUR_PROJECT_NAME.MyAssetManager"
```

#### 总结

UE 4.24 + 中`UAbilitySystemGlobals::InitGlobalData()`是`TargetData`和`AbilitySystem`正常工作的 “必选项”，核心要点如下：

1. **调用时机**：只需一次，但需根据是否使用`GlobalAttributeDefaultsTables`选择位置（`Subsystem`或`AssetManager/GameInstance`）；
2. **崩溃规避**：若因`GlobalAttributeDefaultsTables`崩溃，优先迁移到`AssetManager`中调用；
3. **模板复用**：直接使用上述模板代码，可避免 90% 以上的`TargetData`相关错误（如`ScriptStructCache`、客户端断连）。

这一配置是`AbilitySystem`模块从 “自动初始化” 到 “显式控制” 的关键过渡，理解加载顺序逻辑后，能更灵活地适配不同项目架构。

## 4.10 预测(Prediction)

### 4.10.1

### 4.10.2

### 4.10.3

### 4.10.4

### 4.10.5

## 4.11 Targeting

### 4.11.1 Target Data

`FGameplayAbilityTargetData` 和 `FGameplayAbilityTargetDataHandle` 是 GAS 中**跨网络传递目标数据**的核心工具，解决了 “技能目标信息（如选中的敌人、点击的位置）在客户端与服务器间可靠同步” 的关键问题。下面从 “核心作用”“结构设计”“使用流程” 三个维度解析，帮你彻底理解其工作原理和实践价值：

#### 一、核心价值：解决 “目标数据跨网络同步” 的痛点

在多人游戏中，技能的 “目标选择” 通常在客户端完成（如玩家点击敌人或地面），但技能的实际生效（如造成伤害、施加效果）需要在服务器执行。这就需要一种机制：

1. 客户端将 “目标信息”（如敌人引用、位置坐标）打包；
2. 可靠地同步到服务器；
3. 服务器解析并使用这些数据执行技能逻辑。

`FGameplayAbilityTargetData`（简称 `TargetData`）及其容器 `FGameplayAbilityTargetDataHandle`（简称 `TargetDataHandle`）就是为此设计的 —— 它们提供了**类型安全、支持多态、可网络序列化**的目标数据传递方案。

#### 二、`FGameplayAbilityTargetData`：目标数据的 “原子单元”

##### 1. 基础特性：

- **抽象基类**：不能直接实例化，必须使用其派生类（GAS 提供了多个开箱即用的派生类，也可自定义）。
- **网络序列化**：内置 `NetSerialize` 方法，确保数据能在客户端与服务器间正确同步。
- **多态支持**：可存储多种类型的目标信息（实体、位置、碰撞结果等），通过继承扩展自定义数据。

##### 2. 常用派生类（GAS 内置）：

GAS 在 `GameplayAbilityTargetTypes.h` 中提供了几种常用实现，覆盖大部分基础需求：

| 派生类                                       | 用途                                | 核心数据示例                             |
| -------------------------------------------- | ----------------------------------- | ---------------------------------------- |
| `FGameplayAbilityTargetData_Actor`           | 存储单个 Actor 目标（如选中的敌人） | `TWeakObjectPtr<AActor> TargetActor`     |
| `FGameplayAbilityTargetData_LocationInfo`    | 存储位置信息（如地面点击位置）      | `FVector Location`、`FRotator Direction` |
| `FGameplayAbilityTargetData_SingleTargetHit` | 存储碰撞结果（如射线检测命中信息）  | `FHitResult HitResult`                   |
| `FGameplayAbilityTargetData_MultiTargetHit`  | 存储多个碰撞结果（如范围检测命中）  | `TArray<FHitResult> HitResults`          |

##### 3. 自定义 `TargetData`（扩展需求）：

当内置类型不够用时（如需要传递 “目标优先级”“伤害类型” 等自定义数据），可继承 `FGameplayAbilityTargetData` 实现：

```cpp
// 1. 定义自定义 TargetData（需继承并实现序列化）
USTRUCT()
struct FMyCustomTargetData : public FGameplayAbilityTargetData
{
    GENERATED_BODY()

    // 自定义数据：目标 Actor + 伤害倍率
    UPROPERTY()
    TWeakObjectPtr<AActor> TargetActor;
    
    UPROPERTY()
    float DamageMultiplier = 1.0f;

    // 必须实现网络序列化（关键！确保跨网同步）
    virtual bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess) override
    {
        Super::NetSerialize(Ar, Map, bOutSuccess);
        Ar << TargetActor;
        Ar << DamageMultiplier;
        bOutSuccess = true;
        return true;
    }

    // 其他必要实现（如复制、比较）
    virtual FMyCustomTargetData* Duplicate() const override
    {
        return new FMyCustomTargetData(*this);
    }
};
// 注册为网络可序列化类型（关键！）
template<>
struct TStructOpsTypeTraits<FMyCustomTargetData> : public TStructOpsTypeTraitsBase2<FMyCustomTargetData>
{
    enum { WithNetSerializer = true }; // 启用网络序列化
};
```

#### 三、`FGameplayAbilityTargetDataHandle`：`TargetData` 的 “容器与管理者”

`TargetData` 通常不会单独传递，而是通过 `FGameplayAbilityTargetDataHandle` 管理，原因是：

##### 1. 核心作用：

- **多目标支持**：内部包含 `TArray<FGameplayAbilityTargetData*>`，可存储多个 `TargetData`（如范围技能选中的多个敌人）。
- **多态安全**：通过指针数组存储不同派生类的 `TargetData`，确保类型信息在传递和解析时不丢失。
- **便捷操作**：提供复制、合并、网络序列化等工具方法（如 `Append` 合并多个目标，`NetSerialize` 统一同步）。

##### 2. 常用操作：

```cpp
// 1. 创建 Handle 并添加目标数据
FGameplayAbilityTargetDataHandle CreateTargetHandle(AActor* TargetActor, float DamageMulti)
{
    FGameplayAbilityTargetDataHandle Handle;

    // 添加自定义 TargetData
    TSharedPtr<FMyCustomTargetData> CustomData = MakeShareable(new FMyCustomTargetData());
    CustomData->TargetActor = TargetActor;
    CustomData->DamageMultiplier = DamageMulti;
    Handle.Add(CustomData);

    // 也可添加内置 TargetData（如位置信息）
    TSharedPtr<FGameplayAbilityTargetData_LocationInfo> LocationData = MakeShareable(new FGameplayAbilityTargetData_LocationInfo());
    LocationData->Location = FVector(100, 200, 0); // 目标位置
    Handle.Add(LocationData);

    return Handle;
}

// 2. 解析 Handle 中的数据（服务器端）
void ProcessTargetHandle(const FGameplayAbilityTargetDataHandle& Handle)
{
    // 遍历所有 TargetData
    for (const auto& DataPtr : Handle.Data)
    {
        if (!DataPtr.IsValid()) continue;

        // 区分不同类型的 TargetData（多态解析）
        if (const auto* CustomData = static_cast<FMyCustomTargetData*>(DataPtr.Get()))
        {
            // 处理自定义数据（如对目标应用带倍率的伤害）
            AActor* Target = CustomData->TargetActor.Get();
            float FinalDamage = 100 * CustomData->DamageMultiplier;
            ApplyDamage(Target, FinalDamage);
        }
        else if (const auto* LocationData = static_cast<FGameplayAbilityTargetData_LocationInfo*>(DataPtr.Get()))
        {
            // 处理位置数据（如在该位置生成特效）
            SpawnEffectAtLocation(LocationData->Location);
        }
    }
}
```

#### 四、典型使用流程（客户端→服务器同步目标数据）

以 “玩家点击地面释放范围技能” 为例，完整流程如下：

##### 1. 客户端收集目标数据：

```cpp
// 在 AbilityTask 中（如 WaitTargetData）收集玩家输入的目标位置
void UGA_AreaBlast::ActivateAbility(...)
{
    Super::ActivateAbility(...);

    // 创建“等待目标数据”的 Task（如玩家点击地面）
    UAT_WaitTargetData* WaitTargetTask = UAT_WaitTargetData::Create(this, FName("AreaBlastTarget"));
    WaitTargetTask->OnTargetDataReady.AddUObject(this, &UGA_AreaBlast::OnTargetDataReady);
    WaitTargetTask->ReadyForActivation();
}

// 玩家选择目标后回调
void UGA_AreaBlast::OnTargetDataReady(const FGameplayAbilityTargetDataHandle& ClientHandle)
{
    if (!ClientHandle.IsValid()) return;

    // 客户端发送目标数据到服务器（通过 ServerRPC）
    Server_ExecuteAreaBlast(ClientHandle);
}
```

##### 2. 服务器接收并解析数据：

```cpp
// 服务器 RPC 处理
UFUNCTION(Server, Reliable, WithValidation)
void Server_ExecuteAreaBlast(const FGameplayAbilityTargetDataHandle& ServerHandle);

void UGA_AreaBlast::Server_ExecuteAreaBlast_Implementation(const FGameplayAbilityTargetDataHandle& ServerHandle)
{
    // 解析 Handle 中的位置数据
    for (const auto& DataPtr : ServerHandle.Data)
    {
        if (const auto* LocationData = static_cast<FGameplayAbilityTargetData_LocationInfo*>(DataPtr.Get()))
        {
            // 在目标位置应用范围伤害 GE
            ApplyAreaDamageGE(LocationData->Location, 500.0f); // 半径 500 的范围伤害
        }
    }

    // 技能结束
    EndAbility(...);
}
```

##### 3. 数据在其他系统中的传递：

`TargetDataHandle` 还会被存入 `FGameplayEffectContext`（GE 上下文），使后续系统（如伤害计算、特效触发）能访问目标数据：

```cpp
// 在 GameplayEffect 的 Execution 中获取 TargetData
void UDamageExecution::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& Params)
{
    const FGameplayEffectContext* Context = Params.GetContext();
    const FGameplayAbilityTargetDataHandle& TargetHandle = Context->GetTargetData();

    // 使用 TargetData 计算伤害（如根据目标距离调整伤害）
    float DistanceFactor = CalculateDistanceFactor(TargetHandle);
    float FinalDamage = BaseDamage * DistanceFactor;
    // ... 应用伤害
}
```

#### 五、关键注意事项

1. **网络同步的可靠性**：
   `TargetData` 的网络序列化依赖 `NetSerialize` 实现，自定义时必须正确重写该方法，否则会出现 “客户端数据与服务器不一致” 的问题。
2. **避免存储大量数据**：
   `TargetData` 主要用于传递 “目标定位信息”，不应存储大体积数据（如纹理、复杂模型），否则会增加网络带宽消耗。
3. **多态解析的安全性**：
   解析时需用 `static_cast` 并检查指针有效性（`DataPtr.IsValid()`），避免因类型不匹配导致崩溃。
4. **与 Target Actor 配合使用**：
   复杂目标选择（如扇形检测、障碍物过滤）通常通过 `Target Actor`（如 `ATargetActor_SphereTrace`）完成，其 `GetTargetData()` 方法会直接返回 `TargetDataHandle`，无需手动创建。

#### 总结

`FGameplayAbilityTargetData` 和 `FGameplayAbilityTargetDataHandle` 是 GAS 中**目标数据传递的 “标准协议”**：

- `TargetData` 负责 “存储具体目标信息”，支持多态和网络同步；
- `TargetDataHandle` 负责 “管理多个目标数据”，提供便捷的容器操作；
- 二者配合，解决了 “客户端选目标、服务器执行” 的核心同步问题，是实现技能目标逻辑的基础。

理解并正确使用这两个结构，能让你的技能系统在多人环境下更可靠、更灵活。

### 4.11.2 Target Actor

`TargetActor`（`AGameplayAbilityTargetActor`）是 GAS 中**可视化目标选择过程、捕获目标数据**的核心组件，它与 `WaitTargetData` 系列 `AbilityTask` 配合，解决了 “技能如何让玩家直观选择目标（如地面区域、敌人）并同步数据” 的问题。下面从 “核心作用”“使用流程”“关键机制”“优化方案” 四个维度，结合实际场景解析：

#### 一、核心作用：连接 “玩家交互” 与 “目标数据”

`TargetActor` 的本质是 “目标选择的中间层”，主要做三件事：

1. **可视化反馈**：通过静态网格、贴花、光标（Reticle）等组件，让玩家直观看到 “当前选中的目标范围 / 位置”（如陨石技能的地面伤害区域贴花）；
2. **目标数据捕获**：通过射线检测、范围重叠检测（Overlap）等逻辑，实时获取目标信息（如选中的敌人、点击的地面坐标）；
3. **交互响应**：根据玩家输入（确认 / 取消）或预设规则，将捕获的目标数据打包为 `TargetData`，传递给 `GameplayAbility` 或 `GameplayEffect`。

#### 二、基础使用流程：从创建到数据传递

以 “玩家点击地面释放陨石技能” 为例，`TargetActor` 与 `WaitTargetData` 的配合流程如下：

##### 1. 选择 / 创建 `TargetActor` 子类

GAS 提供了多个开箱即用的 `TargetActor`，覆盖常见场景：

- **`AGameplayAbilityTargetActor_GroundTrace`**：地面射线检测，适合 “地面区域技能”（如陨石、治疗阵），自带地面贴花可视化；
- **`AGameplayAbilityTargetActor_ActorTrace`**：实体射线检测，适合 “单体目标技能”（如狙击、指向性伤害）；
- **`AGameplayAbilityTargetActor_SphereTrace`**：球形范围检测，适合 “范围选中多个敌人”（如 AOE 伤害）。

若默认类型不满足需求（如自定义检测形状），可继承 `AGameplayAbilityTargetActor` 实现自定义逻辑（如胶囊体检测、扇形检测）。

##### 2. 在 `GameplayAbility` 中通过 `WaitTargetData` 调用

`WaitTargetData` 是触发 `TargetActor` 的 `AbilityTask`，核心是 “生成 `TargetActor` 实例 → 等待玩家交互 → 接收目标数据”。

**蓝图示例**（陨石技能）：

1. 在 GA 蓝图的 `ActivateAbility` 中，添加 **`Wait Target Data`** 节点；
2. 节点参数配置：
   - **`Target Actor Class`**：选择 `AGameplayAbilityTargetActor_GroundTrace`；
   - **`Confirmation Type`**：选择 `UserConfirmed`（需要玩家点击确认目标）；
   - **`Reticle Class`**：选择自定义光标（如 “陨石落点光标”，可选）；
   - **`Filter`**：设置过滤规则（如忽略友方单位，可选）；
3. 绑定 `On Target Data Ready` 回调（接收目标数据）和 `On Target Data Cancelled` 回调（处理取消逻辑）。

**代码示例**：

```cpp
void UGA_Meteor::ActivateAbility(const FGameplayAbilitySpecHandle& Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo& ActivationInfo, const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    // 1. 创建 WaitTargetData Task
    UAT_WaitTargetData* WaitTargetTask = UAT_WaitTargetData::Create(
        this, 
        FName("MeteorTarget"),  // Task 实例名
        AGameplayAbilityTargetActor_GroundTrace::StaticClass()  // TargetActor 类
    );

    // 2. 配置 TargetActor 参数（如地面贴花、检测距离）
    if (WaitTargetTask->TargetActor)
    {
        AGameplayAbilityTargetActor_GroundTrace* GroundTargetActor = Cast<AGameplayAbilityTargetActor_GroundTrace>(WaitTargetTask->TargetActor);
        if (GroundTargetActor)
        {
            GroundTargetActor->SetTraceDistance(5000.0f);  // 射线检测最大距离
            GroundTargetActor->SetDecalMaterial(MeteorDecalMaterial);  // 地面贴花材质
            GroundTargetActor->SetDecalSize(FVector(300.0f));  // 贴花大小（伤害范围）
        }
    }

    // 3. 绑定回调：目标确认时执行
    WaitTargetTask->OnTargetDataReady.AddUObject(this, &UGA_Meteor::OnTargetConfirmed);
    // 绑定回调：目标取消时执行
    WaitTargetTask->OnTargetDataCancelled.AddUObject(this, &UGA_Meteor::OnTargetCancelled);

    // 4. 激活 Task
    WaitTargetTask->ReadyForActivation();
}

// 目标确认：接收 TargetData 并执行技能
void UGA_Meteor::OnTargetConfirmed(const FGameplayAbilityTargetDataHandle& TargetDataHandle)
{
    if (!HasAuthority()) return;

    // 解析 TargetData 中的地面位置
    if (const auto* LocationData = static_cast<FGameplayAbilityTargetData_LocationInfo*>(TargetDataHandle.Data[0].Get()))
    {
        FVector MeteorLocation = LocationData->Location;
        // 在目标位置生成陨石（Spawn Actor + 应用伤害 GE）
        SpawnMeteor(MeteorLocation);
    }

    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, false, false);
}

// 目标取消：终止技能
void UGA_Meteor::OnTargetCancelled(const FGameplayAbilityTargetDataHandle& TargetDataHandle)
{
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}
```

##### 3. 玩家交互与数据同步

根据 `Confirmation Type` 的不同，玩家交互方式和数据同步逻辑也不同（核心机制见下文）：

- 若为 `UserConfirmed`：玩家点击鼠标确认后，`TargetActor` 生成 `TargetData`，通过 `WaitTargetData` 传递给 GA；
- 若为 `Instant`：无需玩家确认，`TargetActor` 立即生成 `TargetData` 并销毁（适合 “瞬发指向性技能”，如普攻射线检测）。

#### 三、关键机制解析

##### 1. 确认类型（`EGameplayTargetingConfirmation::Type`）：控制玩家交互方式

`Confirmation Type` 决定了 “何时确认目标并生成 `TargetData`”，是 `TargetActor` 交互逻辑的核心：

| 确认类型        | 触发时机                                                     | 适用场景                                     |
| --------------- | ------------------------------------------------------------ | -------------------------------------------- |
| `Instant`       | 无需交互，`TargetActor` 生成后立即生成 `TargetData` 并销毁，不执行 `Tick` | 瞬发无交互技能（如自动瞄准的普攻、范围检测） |
| `UserConfirmed` | 玩家按下 “确认键”（如鼠标左键）或调用 `ASC->TargetConfirm()` 时触发 | 需要玩家选择目标的技能（如陨石、治疗阵）     |
| `Custom`        | 由 GA 手动调用 `ConfirmTaskByInstanceName()` 触发            | 自定义交互逻辑（如长按蓄力后确认）           |
| `CustomMulti`   | 支持多次确认（不销毁 `TargetActor`），需手动调用确认 / 取消  | 多目标选择技能（如连续标记多个敌人）         |

**注意**：部分 `TargetActor` 不支持所有确认类型（如 `AGameplayAbilityTargetActor_GroundTrace` 不支持 `Instant`），需提前测试。

##### 2. 网络同步逻辑：客户端与服务器的目标数据一致性

`TargetActor` 默认**不可同步**，但可通过配置实现 “客户端选目标、服务器校验 / 生成数据”，避免作弊：

| 同步模式（`ShouldProduceTargetDataOnServer`） | 逻辑流程                                                     | 优缺点                                                       |
| --------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `false`（客户端生成数据）                     | 1. 客户端 `TargetActor` 捕获数据； 2. 玩家确认后，客户端通过 RPC 将 `TargetData` 发送到服务器； 3. 服务器接收并校验数据（防作弊）。 | 优点：客户端响应快； 缺点：需服务器校验（如判断目标是否在合理范围）。 |
| `true`（服务器生成数据）                      | 1. 客户端 `TargetActor` 仅做可视化，不捕获数据； 2. 玩家确认后，客户端发送 “确认 RPC” 到服务器； 3. 服务器执行检测生成 `TargetData` 并执行技能。 | 优点：数据绝对可靠（无作弊）； 缺点：客户端可能有延迟（预测误差）。 |

**防作弊建议**：对 “伤害、治疗” 等敏感技能，优先用 `ShouldProduceTargetDataOnServer = true`，或在服务器端对客户端发送的 `TargetData` 做严格校验（如 “目标是否在技能射程内”“是否有障碍物遮挡”）。

##### 3. 可视化组件：Reticle 与 Decal

`TargetActor` 通过两种方式提供可视化反馈：

- **`GameplayAbilityWorldReticle`（光标）**：跟随鼠标 / 视野的动态光标（如 “准星”“选中框”），通过 `Reticle Class` 参数配置，适合 “单体目标选中”；
- **`Decal`（贴花）**：静态贴在地面 / 物体表面的纹理（如陨石伤害范围、治疗圈），默认 `GroundTrace` 类自带，可通过 `SetDecalMaterial` 自定义，适合 “区域目标可视化”。

若需更复杂的可视化（如动态粒子效果），可在自定义 `TargetActor` 中添加 `UParticleSystemComponent` 并在 `Tick` 中更新位置。

#### 四、性能优化与进阶技巧

##### 1. 优化 `TargetActor` 的 `Tick` 开销

`Non-Instant` 类型的 `TargetActor` 会在 `Tick` 中频繁执行检测（如射线 / Overlap），高频调用可能导致性能问题，优化方案：

- **降低 `Tick` 频率**：在自定义 `TargetActor` 中重写 `Tick`，通过定时器控制检测间隔（如每 0.05 秒检测一次，而非每帧）；
- **减少检测复杂度**：简化检测形状（如用射线代替球形 Overlap）、缩小检测范围、减少过滤逻辑；
- **复用 `TargetActor`**：默认 `WaitTargetData` 每次调用都会创建 / 销毁 `TargetActor`，高频技能（如全自动射击）可自定义 `AbilityTask` 复用实例（参考 GASShooter 的 `WaitTargetDataWithReusableActor`）。

##### 2. 自定义 `TargetActor`：实现特殊需求

当默认 `TargetActor` 不满足需求时（如扇形检测、持久化目标），可继承 `AGameplayAbilityTargetActor` 扩展：

**示例：持久化目标（记住最后有效目标）**
默认 `TargetActor` 会在目标离开检测范围后失效，若需 “记住最后选中的目标”（如制导导弹），可在自定义类中添加变量存储：

```cpp
class AMyPersistentTargetActor : public AGameplayAbilityTargetActor
{
    GENERATED_BODY()

public:
    virtual void Tick(float DeltaSeconds) override
    {
        Super::Tick(DeltaSeconds);

        // 执行检测
        TArray<FHitResult> Hits;
        PerformTrace(Hits);

        // 若检测到有效目标，更新持久化目标
        if (!Hits.IsEmpty())
        {
            PersistentTargetHit = Hits[0];
            bHasPersistentTarget = true;
        }
        // 若目标已销毁，清空持久化目标
        else if (bHasPersistentTarget && !PersistentTargetHit.GetActor())
        {
            bHasPersistentTarget = false;
        }
    }

    // 获取持久化目标数据
    virtual FGameplayAbilityTargetDataHandle GetTargetData() const override
    {
        FGameplayAbilityTargetDataHandle Handle;
        if (bHasPersistentTarget)
        {
            auto* HitData = new FGameplayAbilityTargetData_SingleTargetHit();
            HitData->HitResult = PersistentTargetHit;
            Handle.Add(HitData);
        }
        return Handle;
    }

private:
    FHitResult PersistentTargetHit;  // 持久化目标的碰撞结果
    bool bHasPersistentTarget = false;  // 是否有持久化目标
};
```

##### 3. 目标过滤（Filter）：精准筛选有效目标

`TargetActor` 的 `Filter` 参数用于过滤无效目标（如忽略友方、忽略障碍物），常用过滤规则：

- **类过滤**：仅允许选中 `APawn` 或 `AEnemyCharacter` 类；
- **标签过滤**：仅允许选中带有 `Tag.Team.Enemy` 的目标；
- **距离过滤**：仅允许选中距离小于 1000cm 的目标。

可通过 `FGameplayTargetDataFilterHandle` 配置复杂过滤逻辑，确保 `TargetData` 仅包含有效目标。

#### 五、总结

`TargetActor` 是 GAS 中 “目标选择体验” 的核心，其价值在于：

- **可视化**：让玩家直观感知目标范围，提升操作体验；
- **数据捕获**：标准化目标数据格式，为后续技能逻辑（如伤害、治疗）提供输入；
- **灵活性**：支持自定义检测逻辑、交互方式和网络同步策略，适配不同类型的技能（单体 / 范围、瞬发 / 蓄力）。

使用时需重点关注：

1. 根据技能类型选择合适的 `TargetActor` 子类和确认类型；
2. 针对网络同步场景，选择 “客户端生成 + 服务器校验” 或 “服务器生成” 模式，确保数据可靠；
3. 优化 `Tick` 检测频率，避免性能开销；
4. 复杂需求通过自定义 `TargetActor` 实现，如持久化目标、特殊检测形状。

掌握 `TargetActor` 后，就能实现从 “玩家交互” 到 “技能生效” 的完整目标选择链路，让技能逻辑更直观、更可靠。

### 4.11.3 TargetData过滤器

用到在学

### 4.11.4 Gameplay Ability World Reticles

多用于通过显式图标告诉玩家当前的目标定位，看文章即可。

### 4.11.5 Gameplay Effect Containers Targeting

`GameplayEffectContainer`（简称 “GE 容器”）是 GAS 中一种**轻量、高效的目标定位方案**，专为 “无需玩家交互、即时生效” 的技能设计（如自动攻击、范围 AOE）。它通过预定义的定位规则快速生成 `TargetData`，避免了 `TargetActor` 的实例创建 / 销毁开销，是提升技能性能的重要工具。

#### 一、核心价值：高效生成目标数据，无需玩家交互

与 `TargetActor` 相比，`GameplayEffectContainer` 的设计目标完全不同：

- **无需可视化**：不提供光标、贴花等玩家反馈（因为无需交互）；
- **即时执行**：调用后立即通过预设规则（如射线检测、范围 Overlap）生成 `TargetData`；
- **无实例开销**：基于 CDO（类默认对象）运行，不创建 `AActor` 实例，性能更优；
- **双向生成**：客户端和服务器各自独立生成 `TargetData`（不同步），适合 “两边都需要快速计算” 的场景（如客户端预测、服务器验证）。

#### 二、核心组成：`GameplayEffectContainer` 与 `TargetType`

`GameplayEffectContainer` 的工作依赖两个关键部分：

##### 1. `FGameplayEffectContainer` 结构体（容器定义）

用于描述 “要应用的 GE” 和 “如何定位目标”，包含两个核心字段：

- **`TargetTypes`**：定位规则数组（`TArray<TSubclassOf<UGameplayAbilityTargetType>>`），每个元素是一个 `TargetType` 类，定义一种定位逻辑；
- **`TargetGameplayEffects`**：要应用到目标的 GE 数组（`TArray<FGameplayEffectSpecHandle>`），定位完成后会将这些 GE 应用到目标身上。

##### 2. `UGameplayAbilityTargetType` 类（定位规则）

这是定位逻辑的实际实现者，GAS 提供了基础类，你可以通过继承扩展自定义规则。其核心函数是 `GetTargets()`，用于生成 `TargetData`：

```cpp
// 基类虚函数：返回目标数据
virtual void GetTargets(
    const UGameplayAbility* Ability,
    const FGameplayAbilityTargetDataHandle& PreFilteredData,
    const FGameplayTagContainer* SourceTags,
    const FGameplayTagContainer* TargetTags,
    FGameplayTagContainer* OptionalRelevantTags,
    TArray<FGameplayAbilityTargetDataHandle>& OutTargetData,
    TArray<FHitResult>& OutHitResults,
    TArray<AActor*>& OutActors) const;
```

**常用内置 / 示例 `TargetType`**：

- **`URPGTargetType_Self`**（Action RPG 样例）：定位技能拥有者自身（如给自己上 buff）；
- **`URPGTargetType_UseEventData`**：从触发技能的事件中提取目标（如碰撞事件中的被击者）；
- **`URPGTargetType_SphereOverlap`**：以拥有者为中心做球形范围检测（如 AOE 技能）；
- **`URPGTargetType_LineTrace`**：从枪口发射射线检测（如自动步枪普攻）。

#### 三、使用流程：从定义到应用

以 “自动攻击（射线检测命中敌人）” 为例，使用 `GameplayEffectContainer` 的步骤如下：

##### 1. 定义 `GameplayEffectContainer`

在 `GameplayAbility` 中定义容器，指定定位规则（`TargetTypes`）和要应用的 GE（如伤害 GE）：

**蓝图示例**：

- 在 GA 的 “Details” 面板中添加 `Gameplay Effect Containers` 数组；
- 新增一个容器，`Target Types` 选择自定义的 `UTargetType_LineTrace`（射线检测）；
- `Target Gameplay Effects` 添加伤害 GE（如 `GE_AttackDamage`）。

**代码示例**：

```cpp
// 在 GA 类中定义容器
UPROPERTY(EditDefaultsOnly, Category = "Attack")
FGameplayEffectContainer AttackContainer;

// 容器初始化（在构造函数或 BeginPlay 中）
void UGA_AutoAttack::InitContainer()
{
    // 设置定位类型为射线检测
    AttackContainer.TargetTypes.Add(UTargetType_LineTrace::StaticClass());
    
    // 添加伤害 GE
    FGameplayEffectSpecHandle DamageGESpec = MakeOutgoingGameplayEffectSpec(UGE_AttackDamage::StaticClass());
    AttackContainer.TargetGameplayEffects.Add(DamageGESpec);
}
```

##### 2. 实现 `TargetType` 定位逻辑

继承 `UGameplayAbilityTargetType`，重写 `GetTargets()` 实现射线检测：

```cpp
// 射线检测定位类型
UCLASS()
class MYGAME_API UTargetType_LineTrace : public UGameplayAbilityTargetType
{
    GENERATED_BODY()

public:
    virtual void GetTargets(const UGameplayAbility* Ability, const FGameplayAbilityTargetDataHandle& PreFilteredData, 
        const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, 
        FGameplayTagContainer* OptionalRelevantTags, TArray<FGameplayAbilityTargetDataHandle>& OutTargetData, 
        TArray<FHitResult>& OutHitResults, TArray<AActor*>& OutActors) const override
    {
        Super::GetTargets(Ability, PreFilteredData, SourceTags, TargetTags, OptionalRelevantTags, OutTargetData, OutHitResults, OutActors);

        if (!Ability || !Ability->GetCurrentActorInfo()->AvatarActor.IsValid())
            return;

        // 1. 获取射线起点（如枪口）和方向（如视线方向）
        AActor* Avatar = Ability->GetCurrentActorInfo()->AvatarActor.Get();
        FVector StartLocation = Avatar->GetMesh()->GetSocketLocation("Muzzle"); // 枪口位置
        FVector EndLocation = StartLocation + Avatar->GetActorForwardVector() * 1000.0f; // 10米射程

        // 2. 执行射线检测
        FCollisionQueryParams Params;
        Params.AddIgnoredActor(Avatar); // 忽略自身
        FHitResult HitResult;
        if (GetWorld()->LineTraceSingleByChannel(HitResult, StartLocation, EndLocation, ECC_Visibility, Params))
        {
            // 3. 将命中结果打包为 TargetData
            TSharedPtr<FGameplayAbilityTargetData_SingleTargetHit> HitData = MakeShareable(new FGameplayAbilityTargetData_SingleTargetHit());
            HitData->HitResult = HitResult;
            OutTargetData.Add(FGameplayAbilityTargetDataHandle(HitData));

            // 4. 记录命中的Actor（供后续应用GE）
            if (HitResult.GetActor())
                OutActors.Add(HitResult.GetActor());
        }
    }
};
```

##### 3. 在技能中应用容器

通过 `ApplyGameplayEffectContainer` 函数触发定位并应用 GE：

```cpp
void UGA_AutoAttack::ActivateAbility(...)
{
    Super::ActivateAbility(...);

    // 应用 GE 容器：自动执行定位并给目标上伤害 GE
    FGameplayAbilityTargetDataHandle TargetDataHandle;
    FGameplayTagContainer OptionalTags;
    ApplyGameplayEffectContainer(AttackContainer, TargetDataHandle, OptionalTags);

    EndAbility(...);
}
```

**蓝图中**：使用 **`Apply Gameplay Effect Container`** 节点，直接传入定义好的容器即可。

#### 四、与 `TargetActor` 的核心差异

| 特性         | `GameplayEffectContainer`            | `TargetActor`                                  |
| ------------ | ------------------------------------ | ---------------------------------------------- |
| **玩家交互** | 无（自动定位，无需确认）             | 有（支持玩家确认 / 取消，依赖输入）            |
| **可视化**   | 无（纯逻辑计算）                     | 有（光标、贴花等反馈）                         |
| **性能**     | 高（无实例创建，基于 CDO）           | 中 / 低（需创建 Actor，`Tick`检测可能开销大）  |
| **网络同步** | 客户端和服务器独立生成数据（不同步） | 可通过 RPC 同步客户端数据到服务器              |
| **适用场景** | 自动攻击、范围 AOE、即时 buff/debuff | 需玩家选择目标的技能（如陨石落点、指向性技能） |

#### 五、最佳实践与注意事项

1. **优先用于 “无交互” 技能**：
   如自动攻击、周期性 AOE、被动技能触发的效果等，这些场景不需要玩家选择目标，用 `GameplayEffectContainer` 可显著提升性能。
2. **避免复杂定位逻辑**：
   虽然可以在 `TargetType` 中实现复杂检测，但过度复杂的逻辑（如多层嵌套 Overlap、大量过滤）会抵消其性能优势，建议保持定位逻辑简洁。
3. **网络一致性处理**：
   由于客户端和服务器独立生成 `TargetData`，可能出现 “客户端预测命中但服务器未命中” 的情况（如网络延迟导致敌人位置不同步）。解决方案：
   - 以服务器结果为准，客户端预测错误时回滚；
   - 对关键效果（如伤害），仅在服务器应用，客户端只做视觉反馈。
4. **复用 `TargetType`**：
   将通用定位逻辑（如 “以自身为中心的球形检测”）封装为可复用的 `TargetType` 子类，避免重复开发。

#### 总结

`GameplayEffectContainer` 是 GAS 中 **高效处理 “无交互目标定位”** 的最佳方案，其核心优势在于 “无实例开销” 和 “即时执行”。对于自动攻击、范围 AOE 等无需玩家选择目标的技能，它比 `TargetActor` 更轻量、更高效。使用时需注意网络一致性问题，并根据技能逻辑设计合适的 `TargetType` 定位规则，平衡性能与功能需求。

# 5. 常用的Abilty和Effect

## 5.1

## 5.2

## 5.3

## 5.4

## 5.5

## 5.6

## 5.7

## 5.8

## 5.9