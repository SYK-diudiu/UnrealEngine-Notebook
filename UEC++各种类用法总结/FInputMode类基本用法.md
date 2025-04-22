# FInputMode类基本用法

[TOC]

## 前言

本文分享`FInputMode`类的基本用法以及我使用`FInputMode`类的一些心得。

在虚幻引擎（Unreal Engine）中，**`FInputMode`** 是一组用于管理玩家输入模式的类，主要用于控制玩家与游戏世界和UI界面的交互方式。以下是其核心用法和常见子类的详细说明：

---

## 1. `FInputMode` 的常见子类
`FInputMode` 本身是一个抽象基类，实际使用中主要通过以下具体子类配置输入模式：

| **子类**                  | **功能描述**                                                 |
| ------------------------- | ------------------------------------------------------------ |
| **`FInputModeGameOnly`**  | 纯游戏输入模式：玩家控制角色/物体，鼠标通常隐藏且锁定到视口（如FPS游戏）。 |
| **`FInputModeUIOnly`**    | 纯UI输入模式：仅响应UI交互（如菜单界面），鼠标可见且锁定到视口。 |
| **`FInputModeGameAndUI`** | 混合模式：同时支持游戏操作和UI交互（如RPG游戏中打开背包时仍可移动视角）。 |

我们一般都是直接使用这些子类定义临时的变量，经过一些设置之后，通过`PlayerController`的`SetInputMode`函数将设置好的`InputMode`添加进控制器中来达到效果。

因此我们可以预见到的是，这些操作我们最好直接写在`Player Controller`类的派生类中，也就是写在我们为角色自建的控制器类中，因为这样我们可以很方便的使用控制器的成员函数，也省去了获取控制器的一些代码操作，同时代码之间的逻辑也会更加清晰。

---

## 2. 基本用法步骤
### **(1) 创建输入模式实例**
根据需求选择合适的子类实例化：
```cpp
// 纯游戏输入模式
FInputModeGameOnly GameMode;

// 纯UI输入模式
FInputModeUIOnly UIMode;

// 混合模式（Game + UI）
FInputModeGameAndUI GameAndUIMode;
```

### **(2) 配置输入参数（可选）**
通过方法调整输入行为：
```cpp
// 示例：混合模式下允许鼠标移出窗口
GameAndUIMode.SetLockMouseToViewportBehavior(EMouseLockMode::DoNotLock);

// 在捕获输入时保持光标可见（如拖动UI时不隐藏）
GameAndUIMode.SetHideCursorDuringCapture(false);

// 设置UI焦点控件（确保输入事件传递到指定UI）
GameAndUIMode.SetWidgetToFocus(MyWidget->TakeWidget());
```

### **(3) 应用输入模式到玩家控制器**

如果我们的代码直接写在控制器类中的话，不需要这一步中的获取控制器`PC`的代码，可以直接使用代码中有注释的部分。

```cpp
APlayerController* PC = GetWorld()->GetFirstPlayerController();
if (PC) {
    PC->SetInputMode(GameAndUIMode); // 应用输入模式
    PC->bShowMouseCursor = true;     // 显示光标（UI模式需显式设置）
}
```

---

## 3. 不同输入模式的典型场景
### **场景1：第一人称射击（`FInputModeGameOnly`）**
```cpp
// 进入战斗状态时应用
FInputModeGameOnly GameMode;
GameMode.SetConsumeCaptureMouseDown(true); // 点击鼠标时捕获输入
PC->SetInputMode(GameMode);
PC->bShowMouseCursor = false; // 隐藏光标
```

### **场景2：主菜单界面（`FInputModeUIOnly`）**
```cpp
// 打开菜单时切换
FInputModeUIOnly UIMode;
// 锁定鼠标到窗口
UIMode.SetLockMouseToViewportBehavior(EMouseLockMode::LockAlways); 
PC->SetInputMode(UIMode);
PC->bShowMouseCursor = true; // 显示光标
```

### **场景3：游戏内交互界面（`FInputModeGameAndUI`）**
```cpp
// 打开背包时允许同时操作角色和UI
FInputModeGameAndUI GameAndUIMode;
GameAndUIMode.SetHideCursorDuringCapture(false); // 操作时不隐藏光标
GameAndUIMode.SetWidgetToFocus(BackpackWidget->TakeWidget()); // 焦点在背包UI
PC->SetInputMode(GameAndUIMode);
PC->bShowMouseCursor = true;
```

---

## 4. 关键方法详解

### **通用配置方法**
- **`SetLockMouseToViewportBehavior(EMouseLockMode)`**  
  控制鼠标是否锁定到游戏窗口：
  - `DoNotLock`：允许鼠标移出窗口。
  - `LockAlways`：始终锁定（默认）。
  - `LockInFullscreen`：仅全屏时锁定。

- **`SetHideCursorDuringCapture(bool)`**  
  控制输入捕获（如点击、拖动）时是否隐藏光标。

### **混合模式专用方法**
- **`SetWidgetToFocus(TSharedPtr<SWidget>)`**  
  指定UI控件为输入焦点，确保键盘/手柄事件优先传递到该控件。

---

## 5. 注意事项
1. **光标可见性**  
   
   - `FInputModeUIOnly` 和 `FInputModeGameAndUI` 需显式设置 `bShowMouseCursor = true`。
   - `FInputModeGameOnly` 通常隐藏光标（`bShowMouseCursor = false`）。
   
2. **输入优先级**  
   - 在混合模式下，UI事件（如按钮点击）会优先于游戏输入（如角色移动）。

3. **多平台适配**  
   - 主机平台（如Xbox/PS）需禁用鼠标逻辑，切换为手柄输入模式：
     ```cpp
     #if !PLATFORM_DESKTOP
     PC->SetInputMode(FInputModeGameOnly());
     #endif
     ```

4. **网络游戏同步**  
   - 输入模式通常只需在客户端设置，服务器无需处理。

---

## 6. 动态切换输入模式示例
```cpp
// 打开菜单时切换为UI模式
void AMyCharacter::OpenMenu() {
    APlayerController* PC = GetController<APlayerController>();
    if (PC && MenuWidget) {
        FInputModeUIOnly UIMode;
        UIMode.SetWidgetToFocus(MenuWidget->TakeWidget());
        PC->SetInputMode(UIMode);
        PC->bShowMouseCursor = true;
    }
}

// 关闭菜单恢复游戏模式
void AMyCharacter::CloseMenu() {
    APlayerController* PC = GetController<APlayerController>();
    if (PC) {
        FInputModeGameOnly GameMode;
        PC->SetInputMode(GameMode);
        PC->bShowMouseCursor = false;
    }
}
```

---

## 总结
`FInputMode` 类通过其子类提供了灵活的输入管理方案，开发者可根据游戏状态（战斗、菜单、交互）动态切换模式，确保玩家输入行为符合设计预期。合理配置鼠标锁定、光标可见性和焦点控件，是实现流畅交互体验的关键。