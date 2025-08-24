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

### 4.5.6

### 4.5.7

### 4.5.8

### 4.5.9

### 4.5.10

### 4.5.11

### 4.5.12

### 4.5.13

### 4.5.14

### 4.5.15

### 4.5.16

### 4.5.17

### 4.5.18