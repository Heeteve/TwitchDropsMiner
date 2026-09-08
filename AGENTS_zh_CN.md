# TwitchDropsMiner 项目检查说明

检查日期：2026-09-04

本文档根据当前仓库源码、依赖配置、构建脚本和测试目录整理，用于帮助后续开发或代码代理快速理解项目。

## 1. 项目主要用途

TwitchDropsMiner 是一个自动获取 Twitch 限时 Drops 奖励的工具。它会：

- 登录并维护 Twitch 账号会话；
- 获取账号已关联的 Drops 活动、游戏、奖励和当前进度；
- 根据游戏优先级、排除列表、活动剩余时间和频道状态选择目标频道；
- 通过请求 Twitch 的流元数据或 Spade 观看事件来模拟观看，不下载实际的视频和音频内容，从而降低带宽占用；
- 通过 Twitch WebSocket 事件接收频道上下线、标题或游戏变化、Drops 进度和奖励领取事件；
- 在频道下线、活动切换或更高优先级频道出现时自动切换频道；
- 在奖励达到领取条件后自动领取，并持续刷新库存和进度；
- 通过界面展示登录状态、频道列表、WebSocket 状态、活动进度、库存和配置。

项目同时提供两种运行入口：

- `main.py`：原始 tkinter 桌面 GUI；
- `main_webui.py`：基于 NiceGUI 的浏览器 WebUI，可从本机或局域网访问，并支持可选的认证、HTTPS 和健康检查接口。

项目的目标是专注于 Twitch Drops，不负责下载视频、不挖掘频道积分，也不支持未关联账号的活动。Drops 活动通常仍需要用户在 Twitch 侧完成账号关联。

## 2. 主要目录和文件职责

### 2.1 主要目录

| 目录 | 职责 |
| --- | --- |
| `webui/` | 本分支新增的浏览器 WebUI。包含 NiceGUI 管理器、GUI 适配器、各页面组件、WebUI 认证、TLS、翻译和前端辅助代码。 |
| `webui/adapters/` | GUI 兼容适配层。让 `twitch.py` 继续使用类似 `gui.status`、`gui.channels`、`gui.inv` 的接口，而无需了解 NiceGUI。 |
| `webui/components/` | NiceGUI 页面和面板。包含主页面、库存、设置、帮助、顶部导航，以及主页面和设置页面的细分 Section。 |
| `webui/components/main/` | 主页面的频道、控制台、Drops 进度、登录和 WebSocket 状态组件。 |
| `webui/components/settings/` | 设置页面的通用设置、游戏优先级列表和排除列表组件。 |
| `webui/lang/` | WebUI 专用或覆盖用的多语言 JSON 文件；基础翻译仍来自根目录的 `lang/`。 |
| `tests/` | pytest 测试，包括 WebUI 与 tkinter GUI 的接口一致性、库存 HTML 渲染、Drops 倒计时和翻译键检查。 |
| `lang/` | 原始 GUI 和核心业务使用的多语言 JSON 文件。 |
| `icons/` | 应用图标、状态图标和 WebUI 使用的图标资源。 |
| `appimage/` | Linux AppImage 构建配置和相关资源。 |
| `.github/workflows/` | GitHub Actions 工作流，负责测试、构建、发布开发版本以及定期合并上游仓库。 |
| `.vscode/` | VS Code 的启动和构建任务配置。 |

以下目录主要由运行或构建过程生成，通常不是业务源码：

- `config/`：运行时配置目录，保存 `settings.json`、Twitch `cookies.jar`、WebUI 认证数据库和可选 TLS 证书；
- `cache/`：图片缓存及图片哈希映射；
- `env/`：本地虚拟环境；
- `build/`、`dist/`：PyInstaller 构建过程中产生的中间文件和产物。

### 2.2 根目录重要文件

| 文件 | 职责 |
| --- | --- |
| `main.py` | tkinter 入口，初始化日志、设置、锁文件和 `Twitch` 客户端，并运行桌面 GUI。 |
| `main_webui.py` | NiceGUI 入口，注入 `gui` 模块替身，将 `Twitch` 后端与 WebUI 放入同一个 asyncio 事件循环。 |
| `twitch.py` | Twitch Drops 的主控制器和状态机，负责认证、请求、库存、活动、频道、观看和事件协调。 |
| `inventory.py` | Drops 领域模型，负责活动、掉落、奖励、进度、前置条件、资格判断和领取。 |
| `channel.py` | 频道和直播流模型，负责频道上下线、流元数据、Drops 可用性和观看事件请求。 |
| `websocket.py` | Twitch WebSocket 连接池，负责连接、订阅主题、心跳、重连和事件分发。 |
| `gui.py` | 原始 tkinter GUI 及其组件，同时定义核心业务依赖的 GUI 接口形状。 |
| `settings.py` | 从命令行参数和 `config/settings.json` 读取、修改和保存配置。 |
| `constants.py` | 状态枚举、Twitch GraphQL 查询、WebSocket 主题、路径、限制和时间间隔等常量。 |
| `utils.py` | JSON 序列化、文件锁、退避重试、限流、异步值、游戏对象等通用工具。 |
| `translate.py` | 基础翻译加载、语言切换和翻译键访问。 |
| `exceptions.py` | 登录、请求、GraphQL、WebSocket、退出和重载等异常类型。 |
| `cache.py` | 原始 tkinter 库使用的图片缓存，减少库存页面图片重复下载。 |
| `pyproject.toml`、`requirements*.txt`、`uv.lock` | Python 版本和依赖声明、可选依赖组以及锁定版本。 |
| `build.spec`、`Dockerfile.alpine`、`build.*` | PyInstaller、Alpine Linux 和跨平台构建配置。 |

## 3. 核心业务模块

### 3.1 直接参与 Drops 业务的模块

#### `twitch.py`

项目的核心编排模块，`Twitch` 类集中管理：

- Twitch 会话、Cookie 和 OAuth 登录状态；
- HTTP 请求和 GraphQL 请求，包括重试、限流和代理；
- Drops 库存、活动、掉落和目标游戏；
- 频道集合及当前观看频道；
- WebSocket 连接池和用户/频道主题；
- 观看任务、维护任务以及应用生命周期。

主状态机使用 `constants.State`，主要流程为：

`INVENTORY_FETCH → GAMES_UPDATE → CHANNELS_CLEANUP → CHANNELS_FETCH → CHANNEL_SWITCH → IDLE`

在运行过程中也可能进入 `RESTART` 或 `EXIT`。频道事件和 Drops 事件会触发重新选台、刷新库存或领取奖励。

#### `inventory.py`

实现 Drops 领域模型：

- `Benefit`：奖励信息，例如物品、徽章或表情；
- `BaseDrop`：掉落的通用属性、前置条件、领取 ID 和领取逻辑；
- `TimedDrop`：观看分钟数、进度、剩余时间、可领取状态和进度更新；
- `DropsCampaign`：活动时间、游戏、账号关联、允许频道、掉落集合、资格和整体进度。

该模块决定活动或掉落是否可以赚取、是否能够在指定时间内完成，以及在收到进度后如何更新本地状态。

#### `channel.py`

实现 `Channel` 和 `Stream`：

- 表示频道身份、显示名称、上下线和待上线状态；
- 保存当前直播游戏、标题、观众数和 Drops 可用性；
- 根据 Twitch GraphQL 数据创建或更新流信息；
- 在频道状态变化时通知 `Twitch` 重新评估观看目标；
- 生成并发送用于推进 Drops 的 Spade 观看事件。

#### `websocket.py`

实现 Twitch WebSocket 的单连接和连接池：

- 使用 `Websocket` 管理单条连接；
- 使用 `WebsocketPool` 按主题数量分片管理多条连接；
- 发送 `LISTEN`、`UNLISTEN`、`PING`；
- 处理频道上下线、直播更新、Drops 进度、通知和重连事件；
- 对连接异常执行退避重连。

当前常量限制为最多 `8` 条连接、每条连接最多 `50` 个主题，因此理论上最多跟踪 `199` 个频道。

### 3.2 配置和基础支撑模块

- `settings.py`：管理语言、深色模式、代理、游戏优先级、排除列表、优先级模式、连接质量和 Drops 检查选项等持久化设置。
- `constants.py`：集中定义 Twitch GraphQL 持久化查询、WebSocket 主题、客户端类型、运行状态、路径和各种轮询限制。
- `utils.py`：提供 `Game`、`RateLimiter`、`ExponentialBackoff`、`AwaitableValue`、JSON 读写、文件锁和异步任务辅助工具。
- `exceptions.py`：通过异常区分登录失败、请求失败、需要验证码、退出、重载和 WebSocket 关闭等控制流。
- `translate.py`：为核心 GUI 和业务日志提供多语言文本。
- `cache.py`：缓存库存页面使用的图片；主要服务于 tkinter GUI。

### 3.3 界面和边界模块

- `gui.py`：原始桌面界面。它不是 Drops 业务核心，但其公开属性和方法构成 `twitch.py` 依赖的 GUI 接口。
- `webui/manager.py`：`WebUIManager`，实现与 `GUIManager` 兼容的顶层接口，并注册 NiceGUI 页面。
- `webui/adapters/`：将核心对 GUI 的调用转发到 WebUI。
- `webui/components/`：持有浏览器页面状态和 NiceGUI 控件。
- `webui/auth.py`：WebUI 登录认证，不等同于 Twitch 账号登录。
- `webui/patches.py`：只在 WebUI 入口加载的补丁，用于扩展 WebUI 专属配置或资格逻辑，尽量避免修改核心文件。

`main_webui.py` 会在导入 `twitch.py` 前将 `sys.modules["gui"]` 指向一个只提供 `WebUIManager` 的替身模块，因此核心模块仍然可以使用 `from gui import GUIManager`，但实际运行时不会加载 tkinter。

## 4. 当前项目可能使用的技术栈

### 4.1 语言与运行时

- Python `>=3.10`；
- `asyncio` 异步事件循环；
- 标准库中的 `dataclasses`、`enum`、`logging`、`sqlite3`、`pathlib`、`json`、`typing` 等；
- 以类型注解和少量 `TypedDict`、泛型、异步任务为主的 Python 应用结构。

### 4.2 网络和 Twitch 集成

- `aiohttp`：异步 HTTP 客户端和 WebSocket 客户端；
- `yarl`：URL 类型和 URL 处理；
- `truststore`：将系统信任库注入 Python SSL；
- Twitch OAuth 设备码登录和 Cookie 会话恢复；
- Twitch GraphQL 持久化查询，用于获取库存、活动、频道目录、直播信息、当前掉落和领取结果；
- Twitch WebSocket PubSub 风格主题，用于接收用户和频道事件；
- Twitch Spade 观看事件接口，用于推进 Drops 进度；
- 内置代理、重试退避、连接超时和 GraphQL 限流处理。

### 4.3 图形界面和 Web 技术

- tkinter/ttk：原始桌面 GUI；
- Pillow：桌面 GUI 图片处理和图片缓存；
- Windows 条件依赖 `pywin32`，并通过 `winreg` 实现 Windows 注册表相关功能；
- NiceGUI `>=3.11.1,<4.0.0`：本分支的浏览器 UI；
- NiceGUI 底层使用 FastAPI 及其 WebSocket/HTTP 服务能力，页面代码使用 Quasar/Vue 组件和 CSS 工具类；
- 浏览器端通过 JavaScript 实现 favicon、弹窗、通知、Tab URL 同步等交互；
- WebUI 认证使用 SQLite、`pwdlib[argon2]`、PyJWT 和 `slowapi`；
- WebUI 可调用 OpenSSL 生成自签名证书并启用 HTTPS。

### 4.4 数据存储

- JSON：设置文件、翻译文件、图片缓存映射和调试 dump；
- `aiohttp` CookieJar：Twitch 登录 Cookie，持久化为 `config/cookies.jar`；
- SQLite：仅在启用 WebUI 认证时保存用户密码哈希和 JWT 签名密钥；
- 本地文件锁：避免同一应用被重复启动。

### 4.5 测试、环境和发布

- pytest：当前测试覆盖接口一致性、库存 HTML 渲染、倒计时、翻译键和 WebUI 翻译覆盖；
- uv：Python 环境和依赖同步工具，仓库包含 `uv.lock`；
- PyInstaller：打包 Windows、Linux 和 macOS 可执行程序；
- Docker + Alpine Linux：构建 Linux musl 版本，支持 `tkinter` 或 `nicegui` 构建参数；
- AppImage Builder：构建 Linux AppImage；
- GitHub Actions：执行验证、测试、跨平台构建、开发版发布和上游同步。

## 5. 核心运行链路

1. 入口文件解析命令行参数，加载 `Settings`，配置日志并创建 `Twitch`。
2. `Twitch` 初始化认证、HTTP 会话、库存、频道集合、观看任务和 WebSocket 连接池。
3. 通过 OAuth 设备码流程或持久化 Cookie 完成 Twitch 登录。
4. `fetch_inventory()` 获取进行中的活动、可用活动和活动详情，构造 `DropsCampaign` 与 `TimedDrop`。
5. `GAMES_UPDATE` 根据关联状态、设置中的优先级/排除列表和活动可完成性生成目标游戏列表，并领取已经准备好的奖励。
6. `CHANNELS_CLEANUP` 和 `CHANNELS_FETCH` 清理旧频道、发现直播频道、检查在线状态、按优先级和观众数排序，并订阅频道主题。
7. `CHANNEL_SWITCH` 选择合适频道；`_watch_loop` 周期性发送观看事件，必要时通过 GraphQL 或本地估算补齐进度。
8. WebSocket 收到频道或 Drops 事件后更新领域对象，并触发 UI 更新、重新选台、领取奖励或重新获取库存。
9. tkinter 或 WebUI 只负责展示和接收用户操作，核心业务通过兼容的 GUI 接口向界面发送状态变化。

维护时应优先在 `webui/` 内通过适配器、页面组件或 WebUI 补丁实现 WebUI 功能；只有在核心业务确实无法通过现有接口完成时，才考虑修改根目录核心模块。
