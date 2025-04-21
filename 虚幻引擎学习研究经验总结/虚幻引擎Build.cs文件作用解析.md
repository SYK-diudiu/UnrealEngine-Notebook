# 虚幻引擎Build.cs文件作用解析

[TOC]

## 常见`.Build.cs`文件代码解析

```csharp
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;

public class SYKSampleDemo : ModuleRules
{
	public SYKSampleDemo(ReadOnlyTargetRules Target) : base(Target)
	{
        /*
		 * ReadOnlyTargetRules Target 是 ModuleRules 类构造函数的一个参数，它在模块构建配置方面发挥着关键作用。
		 * 
		 * ReadOnlyTargetRules 是一个结构体，
		 * 其中包含了与目标平台、配置和构建相关的规则及信息。
		 * Target 作为这个结构体的实例，提供了一系列只读属性和方法，
		 * 可用于在构建模块时获取目标平台、配置类型等信息，
		 * 从而依据不同的目标进行针对性的配置。
		 * 
		 * 借助 ReadOnlyTargetRules Target，
		 * 开发者能够根据不同的目标平台（如 Windows、Linux、iOS 等）、
		 * 配置类型（如 Debug、Development、Shipping 等）
		 * 以及其他构建相关的设置，对模块的构建过程进行定制。
		 * 这有助于实现跨平台开发和不同构建配置下的灵活处理。
		 */

        PCHUsage = ModuleRules.PCHUsageMode.UseExplicitOrSharedPCHs;
		/*
		 * PCH 全称为 Pre-Compiled Header，即预编译头。它是一种在编译过程中用于提高编译效率的技术手段.
		 * 
		 * 在编译程序时，编译器会对每个源文件进行独立处理。很多源文件都会包含相同的头文件，
		 * 例如标准库头文件、项目公共头文件等。每次编译源文件时，重复处理这些头文件会消耗大量时间。
		 * 预编译头技术就是把这些频繁使用且不常变动的头文件预先编译成一个文件，
		 * 后续编译源文件时，编译器可以直接使用这个预编译好的文件，
		 * 从而避免对头文件的重复编译，显著提升编译速度。
		 * 
		 * 在虚幻引擎的 .Build.cs 文件里，可以对预编译头的使用模式进行配置，常见的模式有以下几种：
		 * 1、UseExplicitOrSharedPCHs：模块必须指定一个预编译头文件，或者继承引擎的共享预编译头。
		 * 2、UseSharedPCHs：强制使用引擎的 Engine.h 作为预编译头。
		 * 3、NoPCHs：禁用预编译头，这种模式适用于第三方库模块或者不稳定的模块。
		 */

		PublicIncludePaths.AddRange(
			new string[] {
				//"SYKThirdPartyLibrary/Public",
				"SYKSampleDemo",
			}
			);

        PrivateIncludePaths.AddRange(
            new string[] {
				// ... add other private include paths required here ...
			}
            );
        /*
		 * PublicIncludePaths 和 PrivateIncludePaths 是用于配置头文件搜索路径的重要属性。
		 * 
		 * 在编译过程中，编译器需要知道在哪里查找头文件。
		 * 通过设置 PublicIncludePaths 和 PrivateIncludePaths，
		 * 可以告诉编译器在哪些目录下搜索头文件，从而顺利完成编译。
		 * 
		 * 1、PublicIncludePaths：这里配置的路径是公共的，
		 * 意味着不仅当前模块可以使用这些路径下的头文件，依赖于当前模块的其他模块也可以使用。
		 * 通常用于存放对外公开的头文件，比如模块的接口定义、公共工具类等。
		 * 2、PrivateIncludePaths：此路径下的头文件仅供当前模块内部使用，其他模块无法访问。
		 * 一般用于存放模块的私有实现细节、内部使用的工具类等，这些内容不希望被外部模块直接引用。
		 * 
		 * AddRange函数接受一个string类型的集合，即字符串集合collection
		 * collection：代表要添加的路径列表。
		 * 因此我们只需要在函数参数列表中new一个string数组
		 * 在编译时会自动转换为集合
		 */
        PublicDependencyModuleNames.AddRange(new string[] { 
			"Core", 
			"CoreUObject", 
			"Engine", 
			"InputCore", 
			"EnhancedInput" ,
			//"SYKThirdPartyLibrary"
		});

        /*
		 * 1、Core：它是虚幻引擎的核心模块，提供了基础的功能和数据结构，像内存管理、日志记录、字符串处理、容器类等。几乎所有的模块都会依赖 Core 模块。
		 * 2、CoreUObject：该模块实现了虚幻引擎的对象系统，包含 UObject 类及其相关功能。UObject 是虚幻引擎中很多类的基类，提供了反射、序列化、垃圾回收等重要特性。
		 * 3、Engine：这是虚幻引擎的核心引擎模块，涵盖了游戏引擎的主要功能，例如场景管理、渲染、物理模拟、AI 等。大多数游戏模块都会依赖 Engine 模块。
		 * 4、InputCore：负责处理基本的输入事件，比如键盘、鼠标、游戏手柄等输入设备的输入。通过这个模块，你可以监听和处理各种输入事件。
		 * 5、EnhancedInput：这是虚幻引擎中用于处理输入的增强型模块，它提供了更灵活和强大的输入处理机制，支持输入映射、输入动作、输入条件等功能，方便开发者实现复杂的输入逻辑。
		 */


        PrivateDependencyModuleNames.AddRange(
            new string[]
            {
				// ... add private dependencies that you statically link with here ...	
			}
            );

        /*
		 * PublicDependencyModuleNames.AddRange 和 PrivateDependencyModuleNames.AddRange 
		 * 是用来配置模块依赖关系的关键方法。
		 * 在虚幻引擎中，一个模块可能会依赖其他模块来实现特定功能。
		 * 这些依赖关系需要在 .Build.cs 文件里进行明确配置，
		 * 如此编译器才能正确地链接这些模块。
		 * 
		 * 1、PublicDependencyModuleNames.AddRange：该函数用于添加公共依赖模块。
		 * 公共依赖意味着依赖模块的接口对当前模块的使用者是可见的。
		 * 也就是说，不仅当前模块可以使用这些依赖模块的功能，依赖于当前模块的其他模块也能使用。
		 * 2、PrivateDependencyModuleNames.AddRange：此函数用于添加私有依赖模块。
		 * 私有依赖表示依赖模块的接口仅对当前模块内部可见，
		 * 其他依赖于当前模块的模块无法直接访问这些私有依赖模块的功能。
		 * 
		 * 避免循环依赖：要确保依赖关系不会形成循环，否则会导致编译错误。
		 * 例如，模块 A 依赖模块 B，而模块 B 又依赖模块 A，这就是循环依赖。
		 */

        /*
         * 对与（Public/Private)IncludePaths和（Public/Private)DependencyModuleNames的区别
         * IncludePaths是头文件路径配置
         * DependencyModuleNames是模块依赖配置
         * 
		 * 头文件路径配置：解决的是编译时头文件的查找问题，通过指定路径让编译器找到所需的头文件。
		 * 模块依赖配置：解决的是链接时模块代码的整合问题，通过指定依赖模块让编译器将相关模块的代码正确链接在一起。
		 * 
		 * 二者一个作用的领域不同，一个是编译期、另一个是链接期
		 * 在模块构建过程中分别承担着不同的职责，二者相互配合，才能确保模块的正确编译和链接。
		 * 
		 * 如果我的PublicIncludePaths只包含了当前模块的头文件，那么PublicDependencyModuleNames 不需要添加当前模块
		 * 一旦我使用了其它模块的头文件，
		 * 我就需要在（Public/Private)IncludePaths中添加对应的路径，
		 * 同时还要在（Public/Private)DependencyModuleNames中添加对应模块
		 */

        DynamicallyLoadedModuleNames.AddRange(
            new string[]
            {
				// ... add any modules that your module loads dynamically here ...
			}
            );

        /*
		 * 模块的加载方式主要有静态加载和动态加载。
		 * 静态加载是在编译时就确定模块之间的依赖关系，
		 * 将所有依赖模块的代码链接到可执行文件或库中；
		 * 而动态加载则是在程序运行时根据需要加载模块。
		 * DynamicallyLoadedModuleNames.AddRange 就是用来配置动态加载模块的。
		 * 
		 * 对于动态加载的模块，
		 * 写在这里的模块名只是告诉引擎，我的某某模块是动态加载的，你小心点
		 * 仅写在这里是不够的，这里相当于一个动态加载模块的声明
		 * 我们还需要在功能代码中，在你觉得功能需要的合适时机
		 * 通过代码，实现模块的加载、使用模块功能、卸载等一系列工作
		 */
    }
}

```

## 问题汇总

### NO.1

为什么有时候我只使用PublicDependencyModuleNames添加了外部模块，没有使用PublicIncludePaths添加头文件路径，却也可以编译通过？

#### 1、模块内部已配置头文件路径

每个模块都有自己的 `.Build.cs` 文件，在其中可以对公共和私有头文件路径进行配置。当你在当前模块里使用 `PublicDependencyModuleNames` 添加了某个外部模块时，这个外部模块自身可能已经在其 `.Build.cs` 文件中正确配置了公共头文件路径。

例如，有一个外部模块 `ExternalModule`，其 `.Build.cs` 文件可能如下配置：

```csharp
using System.IO;
using UnrealBuildTool;

public class ExternalModule : ModuleRules
{
    public ExternalModule(ReadOnlyTargetRules Target) : base(Target)
    {
        PublicIncludePaths.AddRange(
            new string[] {
                Path.Combine(ModuleDirectory, "Public")
            }
        );
    }
}
```

当你在当前模块中添加对 `ExternalModule` 的依赖时，编译器能够根据 `ExternalModule` 自身配置的公共头文件路径找到所需的头文件，所以即使你没有在当前模块中使用 `PublicIncludePaths` 添加该外部模块的头文件路径，编译依然可以通过。

#### 2. 全局头文件搜索路径

虚幻引擎可能存在一些全局的头文件搜索路径配置。这些全局路径会在编译时被编译器自动搜索，若外部模块的头文件位于这些全局路径中，那么即使你没有显式地使用 `PublicIncludePaths` 添加该模块的头文件路径，编译器也能找到这些头文件。

#### 3. 头文件的相对位置

如果外部模块的头文件与当前模块的源文件或头文件处于相对合适的位置，编译器可能会根据相对路径找到这些头文件。例如，外部模块的头文件与当前模块的源文件在同一目录或其子目录下，编译器可以直接找到这些头文件，从而使编译通过。

#### 4. 预编译头文件的影响

预编译头文件（PCH）在编译过程中可能会包含一些常用的头文件。如果外部模块的头文件被包含在预编译头文件中，那么编译器在编译时会自动获取这些头文件，无需额外配置 `PublicIncludePaths`。不过，这种方式可能会导致编译时的头文件依赖关系不够清晰，建议尽量避免过度依赖预编译头文件来包含头文件。

### NO.2

