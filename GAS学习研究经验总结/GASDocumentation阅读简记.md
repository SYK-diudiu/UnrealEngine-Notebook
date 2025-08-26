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

### 4.5.17 运行时创建动态GameplayEffect

### 4.5.18 GameplayEffect Containers