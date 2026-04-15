## 虚幻-steam网络游戏开发流程

完全可以先在虚幻引擎中完成游戏的所有核心功能，最后再接入 Steamworks SDK 实现多人联机功能。这种开发流程非常合理，能有效降低开发复杂度，推荐按以下步骤推进：

### 一、前期开发：无需 Steamworks SDK 即可完成的工作

1. **单机核心功能开发**

   - 实现下棋的基础逻辑：棋盘交互、落子规则、胜负判定、UI 界面（棋盘、回合提示、胜负弹窗等）。
   - 测试单机模式流畅性，确保棋子放置、动画、音效等功能无 BUG。

2. **基于 “局域网” 测试网络逻辑**
   虚幻引擎自带局域网联机功能，无需 Steamworks SDK 即可验证网络同步逻辑，步骤：

   - 在项目设置中，将 “Default Platform Service” 设为 “Null”（不依赖第三方服务）。
   - 用虚幻的基础网络节点（`Create Session`、`Find Sessions`、`Join Session`）实现局域网联机，此时`bUseLAN`参数设为`true`。
   - 测试核心网络功能：
     - 两人局域网内创建 / 加入房间。
     - 落子操作通过 RPC 同步到双方。
     - 回合切换、胜负判定在服务器端正确执行。

   这一步可验证所有网络逻辑（变量复制、RPC 调用、服务器权威），与 Steam 联机的核心代码完全兼容，只是连接范围限于局域网。

### 二、后期接入 Steamworks SDK 的注意事项

当核心功能稳定后，再接入 Steamworks SDK，只需修改少量配置和代码，具体：

1. **替换网络服务为 Steam**
   - 启用 “Online Subsystem Steam” 插件，项目设置中 “Default Platform Service” 切换为 “Steam”。
   - 修改会话创建参数：将`bUseLAN`设为`false`，填入 Steam App ID（测试阶段用 480）。
2. **适配 Steam 的会话接口**
   虚幻的`Create Session`、`Find Sessions`等节点在底层会自动适配 Steam 服务，无需重写逻辑，只需注意：
   - Steam 会话名称需唯一，避免与其他公开房间冲突。
   - 查找会话时，Steam 会返回互联网上的公开房间（而非仅局域网）。
3. **测试流程**
   - 先用 Steam 测试 ID（480）打包，在两台登录 Steam 账号的电脑上测试互联网联机。
   - 确认功能正常后，替换为正式 Steam App ID，提交到 Steam 平台。

### 三、优势与注意事项

- **优势**：
  - 前期专注于游戏核心体验，避免被网络服务细节干扰。
  - 局域网测试与 Steam 联机的核心逻辑一致，后期接入成本低。
- **注意事项**：
  - 网络逻辑需严格遵循 “服务器权威” 原则（所有操作由服务器验证），确保后期接入 Steam 时无需重构代码。
  - 预留 UI 位置显示 Steam 相关信息（如玩家昵称、在线状态），后期只需添加 Steam 用户信息获取节点（`Get Steam ID`、`Get Player Name`）。

这种 “先核心后网络” 的开发流程是行业常见做法，完全可行。你可以先集中精力完成下棋游戏的单机和局域网联机功能，待所有功能稳定后，再花 1-2 天时间接入 Steamworks SDK 并测试互联网联机即可。

## 虚幻游戏发布Steam步骤（未验证是否正确）

在虚幻引擎中制作双人网络下棋游戏并发布到 Steam，核心是通过 **Steamworks 集成** 和虚幻的 **网络框架（Netcode）** 实现玩家连接。以下是详细实现步骤：

### 一、前期准备：启用 Steam 网络插件

1. **启用 Online Subsystem Steam 插件**
   - 打开项目 → 编辑（Edit）→ 插件（Plugins）→ 搜索 **“Online Subsystem Steam”** 和 **“Online Subsystem”**，勾选启用，重启引擎。
   - 这两个插件是虚幻对接 Steam 网络功能的核心（会话创建、查找、连接等）。
2. **配置 Steamworks SDK（可选，打包时必需）**
   - 下载 [Steamworks SDK](https://partner.steamgames.com/doc/sdk)，解压后将 `sdk` 文件夹放入项目 `ThirdParty` 目录（如无则创建）。
   - 项目设置（Project Settings）→ Platforms → Windows → Steam SDK Path：填写 SDK 路径（如 `../../ThirdParty/steamworks/sdk`）。

### 二、配置项目网络参数

1. **设置默认在线系统为 Steam**

   - 项目设置 → 引擎（Engine）→ 在线服务（Online Services）→ Default Platform Service：选择 **“Steam”**。

2. **设置 Steam App ID**

   - 开发测试阶段：使用 Steam 的测试 ID **480**（通用测试 ID）。

   - 正式发布：需在 [Steamworks 后台](https://partner.steamgames.com/) 注册项目获取专属 App ID。

   - 配置方式：

     - 项目设置 → 引擎 → 在线服务 → Steam → Steam App ID：填入 `480`（测试）或你的专属 ID。

     - 或在

        

       ```
       DefaultEngine.ini
       ```

        

       中添加：

       ini

       

       

       

       

       

       ```ini
       [OnlineSubsystemSteam]
       bEnabled=true
       SteamAppId=480
       ```

3. **设置网络模式**

   - 项目设置 → 引擎 → 网络（Networking）→ 网络模式（Net Mode）：默认设为 **“Play As Client”**（便于测试）。
   - 确保项目支持 **“专用服务器”** 或 **“客户端 / 服务器”** 架构（双人游戏通常用 “客户端 - 服务器” 模式，一方作为主机）。

### 三、实现核心网络连接功能（蓝图）

#### 1. 会话（Session）管理（核心！）

会话是 Steam 网络中玩家连接的基础，需实现 **创建会话（主机）**、**查找会话（客户端）**、**加入会话** 功能。

##### （1）创建会话（主机创建房间）

- 使用 **“Create Session”** 节点（蓝图 → 网络 → Online Session）。

- 关键参数：

  - **Num Public Connections**：设为 2（双人游戏）。
  - **Session Name**：房间名称（可选，用于显示）。
  - **bUseLAN**：是否仅限局域网（Steam 联机需设为 `false`）。

- 流程：

  plaintext

  

  

  

  

  

  ```plaintext
  玩家点击“创建房间” → 调用“Create Session”（参数如上） → 成功则加载游戏地图
  ```

##### （2）查找会话（客户端搜索房间）

- 使用 **“Find Sessions”** 节点，搜索 Steam 上的公开会话。

- 配合 **“On Find Sessions Complete”** 回调事件，获取会话列表。

- 流程：

  plaintext

  

  

  

  

  

  ```plaintext
  玩家点击“查找房间” → 调用“Find Sessions” → 搜索完成后，用“Get Search Results”获取会话列表 → 显示到UI（如房间名称、当前人数）
  ```

##### （3）加入会话（客户端加入房间）

- 从会话列表中选择目标会话，调用 **“Join Session”** 节点。

- 成功后自动连接到主机并加载相同地图。

- 流程：

  plaintext

  

  

  

  

  

  ```plaintext
  玩家选择房间 → 调用“Join Session”（传入目标会话） → 成功则加载游戏地图
  ```

#### 2. 网络同步：棋子操作同步

下棋游戏需确保 **落子、棋子位置** 在两台客户端同步，需利用虚幻的网络同步机制：

##### （1）设置棋子 Actor 为网络可复制

- 在棋子 Actor 的蓝图中：
  - 细节面板 → 网络（Replication）→ 勾选 **“Replicates”**（允许该 Actor 在网络中复制）。
  - 若棋子位置需要同步，勾选 **“Replicate Movement”**。

##### （2）使用 RPC（远程过程调用）处理落子逻辑

- 落子操作需由服务器权威处理（避免客户端作弊），流程：
  1. 客户端点击棋盘 → 调用 **“Server RPC”**（服务器远程调用），发送落子位置。
  2. 服务器验证落子合法性（如是否在空位）→ 若合法，修改服务器端棋子位置。
  3. 服务器端棋子位置变化后，自动同步到所有客户端（因勾选了 “Replicates”）。
- 蓝图节点：
  - 右键 → 网络 → **“Create Server RPC”**（创建服务器 RPC 函数，命名为 “Server_PlacePiece”）。
  - 在该函数中编写落子逻辑（修改棋子位置、判断胜负等）。

#### 3. 连接状态处理

- 显示连接中 / 失败提示

  ：

  - “Create Session”/“Join Session” 失败时，通过 **“On Failure”** 引脚触发 UI 提示（如 “创建房间失败”）。

- 断开连接

  ：

  - 使用 **“Leave Session”** 节点退出当前会话，返回主菜单。

- 检测玩家离开

  ：

  - 监听 **“On Player Left Game”** 事件，处理对手断线逻辑（如显示 “对手已离开”）。

### 四、测试网络连接

1. **本地测试**：
   - 启动两个游戏实例（编辑器中点击 **“Play” 旁的下拉箭头 → 选择 “Standalone Game”**，启动两个窗口）。
   - 一个实例创建房间，另一个查找并加入，测试落子是否同步。
2. **Steam 联机测试**：
   - 打包游戏（文件 → 打包项目 → Windows）。
   - 确保测试机器均登录 Steam 账号，运行打包后的游戏，测试跨机连接。

### 五、发布到 Steam 的网络相关配置

1. **申请 Steamworks App ID**

   - 登录 [Steamworks 后台](https://partner.steamgames.com/)，创建新项目，获取专属 App ID。
   - 将项目设置中的 Steam App ID 替换为你的专属 ID。

2. **配置 Steamworks VDF 文件**

   - 在项目打包目录中，创建 `steam_appid.txt` 文件，内容为你的 App ID（用于 Steam 客户端识别）。

   - 确保

      

     ```
     DefaultEngine.ini
     ```

      

     中 Steam 配置正确：

     ini

     

     

     

     

     

     ```ini
     [OnlineSubsystem]
     DefaultPlatformService=Steam
     
     [OnlineSubsystemSteam]
     bEnabled=true
     SteamAppId=你的AppID
     ```

3. **测试多人功能**

   - 通过 Steamworks 的 **“Steamworks Test App”** 测试联机功能，确保会话创建、加入、同步正常。

### 关键注意事项

- **网络权限**：所有影响游戏状态的操作（如落子、胜负判断）必须在服务器端执行，客户端仅处理输入和显示。
- **会话可见性**：创建会话时确保 `bIsPublic` 设为 `true`，否则其他玩家无法搜索到。
- **防火墙 / 端口**：测试时确保防火墙允许游戏程序的网络访问（Steam 默认端口无需额外配置）。

通过以上步骤，即可实现基于 Steam 的双人下棋游戏网络连接功能。核心是利用虚幻的 Online Subsystem Steam 插件管理会话，配合网络同步机制确保棋子操作一致

## 打包注意事项

在虚幻引擎中打包游戏（尤其是计划发布到 Steam 的双人网络下棋游戏）时，需要注意以下关键事项，以确保打包过程顺利且最终可正常运行：

### 一、打包前的基础检查

1. **项目设置配置**

   - 平台兼容性

     ：在「项目设置 > 平台」中，确认目标平台（如 Windows）的基础配置正确：

     - 架构选择：根据目标用户群体选择 32 位（x86）或 64 位（x64，推荐）。
     - 渲染设置：确保默认渲染器（如 DirectX 11/12）与目标平台兼容。

   - **游戏名称与图标**：在「项目设置 > 应用程序 > 显示」中设置游戏名称、图标（.ico 格式）、启动画面，避免使用默认名称。

2. **资源检查**

   - 清理冗余资源：通过「编辑 > 参考查看器」删除未使用的资产（模型、材质、蓝图等），减小包体大小。
   - 资源压缩：确保纹理、音频等资源已启用压缩（在资源细节面板中勾选 “Compress”）。
   - 检查缺失资源：打包前会自动检测缺失资源，在「内容浏览器 > 过滤器 > 显示缺失资产」中修复（如重新导入、删除无效引用）。

3. **蓝图与代码检查**

   - 编译所有蓝图：在内容浏览器中右键「重新编译所有蓝图」，确保无编译错误。
   - 关闭调试节点：移除蓝图中用于测试的 “Print String”“Draw Debug” 等调试节点，避免影响性能或显示无关信息。
   - 网络逻辑验证：针对多人下棋游戏，重点测试服务器 / 客户端逻辑（如 RPC 调用、变量复制），确保打包后网络同步正常。

### 二、Steam 相关打包配置（核心！）

1. **Steamworks SDK 路径正确**

   - 确保「项目设置 > 平台 > Windows > Steam SDK Path」指向正确的 Steamworks SDK 路径（如`../../ThirdParty/steamworks/sdk`），否则打包时会提示 “找不到 Steam SDK”。
   - 推荐将 SDK 放入项目目录（如`ProjectName/ThirdParty/steamworks/sdk`），避免依赖外部路径导致打包失败。

2. **替换正式 Steam App ID**

   - 开发测试时使用的

     ```
     480
     ```

     （测试 ID）需替换为你在 Steamworks 后台申请的

     正式 App ID

     ：

     - 「项目设置 > 引擎 > 在线服务 > Steam > Steam App ID」填入正式 ID。
     - 在打包输出目录（如`Saved/StagedBuilds/Windows/GameName`）中创建`steam_appid.txt`，内容为正式 App ID（确保 Steam 客户端能识别游戏）。

3. **Steamworks 配置文件**

   - 若使用 Steamworks 额外功能（如成就、云存档），需在项目中包含`steam_appid.txt`和`appinfo.vdf`（从 Steamworks 后台下载），并放在打包后的游戏根目录。

### 三、打包过程设置

1. **打包选项选择**
   - 路径设置：在「文件 > 打包项目 > 打包设置」中，选择输出目录（避免中文路径，否则可能出现权限问题）。
   - 打包类型：
     - 测试打包：选择「开发包（Development）」，包含调试信息，便于排查问题。
     - 发布打包：选择「Shipping」，移除调试信息，优化性能，适合正式发布。
   - 勾选「Cook Content for Windows」（针对 Windows 平台），确保资源被正确烘焙。
2. **分平台打包注意事项**
   - **Windows**：确保安装对应 Visual Studio 版本（如 VS2019/2022）及 Windows SDK，否则可能出现编译错误。
   - **其他平台**（如 macOS/Linux）：需安装对应平台的 SDK，并在「项目设置 > 平台」中配置编译工具路径。
3. **打包后验证**
   - 打包完成后，先运行输出目录中的`GameName.exe`，测试基础功能（如启动、主菜单、音效）。
   - 重点测试网络功能：启动两个打包后的程序，测试 “创建房间”“加入房间”“落子同步” 是否正常（需登录 Steam 账号）。

### 四、包体优化与兼容性

1. **减小包体大小**
   - 启用「共享材质实例」和「纹理组压缩」，在「项目设置 > 渲染 > 纹理」中设置合理的压缩等级。
   - 移除打包目录中不必要的文件（如`Saved`、`Intermediate`文件夹，仅保留`Binaries`、`Content`、`Engine`等必要目录）。
2. **兼容性测试**
   - 在不同配置的电脑上测试（如低配置、不同操作系统版本），检查是否有崩溃、卡顿、渲染错误（如下棋界面显示异常）。
   - 测试防火墙 / 杀毒软件影响：确保游戏程序能正常访问网络（尤其是 Steam 联机需要的端口）。
3. **错误日志排查**
   - 若打包失败或运行崩溃，查看日志文件：
     - 打包日志：`Saved/Logs/ProjectName.log`
     - 运行日志：`Saved/Logs/ProjectName.log`（在打包后的游戏目录中）
   - 常见错误：资源路径错误、Steam SDK 缺失、网络权限未开启（如蓝图中未启用 “Replicates”）。

### 五、发布到 Steam 的额外准备

1. **打包格式**
   - 按照 Steamworks 要求，将打包后的游戏文件整理为`Steamworks SDK/tools/ContentBuilder/content`目录下的结构，便于通过 Steamworks 上传工具（SteamPipe）提交。
2. **测试分支设置**
   - 在 Steamworks 后台创建测试分支，先上传开发包供内部测试，确认无问题后再发布到正式分支。
3. **法律信息与版权**
   - 确保游戏中使用的资源（模型、音效、字体等）无版权问题，必要时添加版权声明（可在游戏启动画面或设置界面显示）。

### 总结

打包的核心原则是：**先配置后打包，先测试后发布**。尤其对于网络下棋游戏，需重点验证 Steam 联机功能、棋子同步逻辑和跨设备兼容性。打包前多做几次开发包测试，修复所有报错和逻辑问题，再用 Shipping 包发布到 Steam，可大幅降低发布风险。