---
name: minecraft-startup
description: Manage Minecraft, instances, accounts and game content.
metadata:
  version: "0.1.0"
license: GPL-3.0-only
---

# MinecraftStartup

你的下一款启动器，何必是启动器。

## 最高优先级：直接启动走缓存

“MC启动 / 启动 Minecraft / 启动某实例 / 再次启动”先执行下面 Step 1～5，**不要先读其他 references、扫描全部 Java、重走元数据链或全量校验 assets**。此入口对每个 instance 都适用，不限当前实例或上次启动的实例；安全、授权和玩家运行能力边界仍适用。只有缓存缺失/失效或明确请求安装、改配置、修复、进入指定世界/服务器时，才按失败原因加载对应模块。缓存维护细节仅在变更/重建时读取 [启动缓存](references/launch-cache.md)。

### Step 1：直接定位实例和缓存

读当前用户 `~/.minecraftstartup/config.json`；显式实例 ID/唯一名称优先，其次 `instances.selectedId`，最后唯一实例，有歧义只问实例。读取该实例 `<dataRoot>/<folders.instances>/<instanceId>/launch-cache.json`，绝不能用全局 lastLaunch 的指令替代另一实例。配置不存在才转首次初始化；损坏保留原件。缓存路径不接受根外链接，缓存只是 JSON 数据，不执行其中的 shell 代码。

### Step 2：最小核对，不重做安装

检查缓存 schemaVersion=1、state=ready、目标 instanceId/OS/架构/cwd/executable 与当前配置及 cache.configBinding 一致，配置绑定按缓存协议的固定投影比较，不信任缓存自选绑定范围。核对 sources 中 profile/组件清单的小文件 hash、Java 的文件身份/可执行性、requiredFiles 中主 JAR/库/日志配置/natives 的存在与 size/mtime 身份；只对变化项复核已有预期 hash，无法确认或缺文件就失效。复用已经验证、仍存在且未变化的 nativeDir 和安装收据；不每次重解压 natives、不全量散列 assets、不联网更新元数据。此为缓存快速检查，不宣称本次全量完整性复验。当前 network.offline 仍实时生效；已有工具能力未变不重复探测。

缓存固定绑定投影为 `{dataRoot, folders, instance, java, account, clientId}`：instance 为目标实例完整记录去掉 status/installationError，java 为该 javaId 对应完整记录去掉 status，account 为当前实际账号完整非秘密记录去掉 status，clientId 为 accounts.microsoft.clientId（缺省 null）。按结构化值比较；不绑定 lastLaunch、其他实例选择或生命周期记录。argvTemplate 必须为字符串数组且不含 executable；dynamicBindings 的唯一零基 index 必须落在数组范围，valueTemplate 必须与该元素一致且仅允许 `${auth_access_token}` / `${auth_session}`，其他未解析占位符/未知 schema 一律失效。不得执行带真实凭据或未验证来源的缓存。

### Step 3：准备会话，再互斥注入

先只读核对该实例实际进程，已运行就复用，不以另一 Java/实例正在运行阻断。会话核对/必要刷新在取得实例锁之前完成，避免刷新账号后持单实例锁扩展到其他实例。读取当前选定账号本项目凭据，核对 clientId/期限；在线会话有效且授权/角色非秘密验证记录仍在有效期内时直接复用，不每次重新 entitlement/profile 请求。过期/失效才加载微软刷新流程；全流程离线策略禁止刷新，不默转离线。仅微软账号的缓存 authValidation 必须匹配当前 accountId/clientId/uuid/playerName、canPlay=true 且当前时间早于 validUntil；validUntil 不超过对应 MC 会话到期时间，验证记录必须属于当前会话（安全存储内标记的 savedAt/sessionRevision 与缓存记录一致，不能靠旧账号 ready 状态推定）。会话就绪后，用独占创建取得 `<instancesRoot>/.management-publish-<instanceId>.lock` 父级稳定操作锁，再取得 `<instanceRoot>/.management/lock` 实例锁（本次 UUID/nonce，同一顺序）；已有非本任务锁则停止，不自动删锁。持锁重读 Step 2 绑定、当前会话修订号和目标进程，已运行就复用，变化则退出重建，不在锁内刷新账号或反序扩锁。按缓存 dynamicBindings 仅在内存注入当前 token/session，离线身份按已保存类型填值；argvTemplate 已分词，替换后不再 split。每次生成新 taskId/PID/日志，不缓存凭据或复用旧任务身份。

### Step 4：直接执行缓存 Java argv

通过已有后台进程工具执行 `[cache.executable, *resolvedArgv]`，cwd=cache.cwd，优先 shell=false；仅有原生 shell 时逐元素正确转义，不 eval、不执行缓存脚本。不额外加载完整 command 手册来重复拼 classpath。stdin 关闭、stdout/stderr 写本次私有本地日志；记录实际 Java 子进程 PID/开始身份，监督进程 wait 取得真实退出码和退出时间。用 config.lock 独占、重读、journal 负责字段 before/after、同目录暂存原子合并和读回保存 lastLaunch，保留未知字段，随后释放自己拥有的锁；不覆盖较新任务。启动参数未变的生命周期保存复用缓存，不递归重建。能力缺失如实 BLOCKED，不安装额外运行时、不生成 helper、不用 GUI 自动化。

### Step 5：简短确认结果

短时观察，进程仍活为 running，提前退出读真实 exitCode；主菜单仍需玩家确认，不能凭缓存历史 ready 或日志冒称就绪。结束回调只在 lastLaunch.taskId 仍匹配时锁定合并 exited/exitCode，保留就绪证据，否则只记本任务 sidecar。内部通知静默维护。成功命中直接返回，不再执行下面的普通主入口。缓存失效只重建必要部分，成功保存后回 Step 2；同一请求最多重建一次，仍失败明确 BLOCKED，不无限循环。

GPL-3.0-only。许可全文见 [LICENSE](LICENSE)，分发说明见 [COPYING.md](COPYING.md)，Prism Launcher / MultiMC / PolyMC 的来源与声明见 [NOTICE.md](NOTICE.md)。无担保；“许可/关于/致谢”请求只展示这些材料，不初始化或启动游戏。

这是可执行操作手册，不是需求清单或另一套启动器程序。一个主入口按玩家目标加载所需管理模块；共用配置、网络、备份和提交契约，不把 Prism 的 Qt UI 或 Java bridge 当必装依赖。正常路径按 Step 执行，未覆盖异常才深入诊断。

## 使用条件与边界

宿主需提供已有专用文件/JSON/归档/hash、HTTP、后台进程和凭据存储工具，或符合下述契约的目标 OS 原生命令。某项操作缺工具时只阻断该项；不因要列截图就要求登录。项目只发布指令与声明式模板，不捆绑运行时代码。

不运行服务端托管、不替玩家购买游戏、不读取其他启动器凭据，不把“列命令”当已执行。初始结构见 [配置模板](templates/config.example.json)，不含任何真实玩家配置。

## 玩家运行能力契约

本 Skill 的正式目标为 **Windows、Linux、macOS**：公共流程保持平台无关，原生命令按实际执行 OS 明确分支；不能把开发机命令、绝对路径或单平台验收当作三端实现。本契约适用于 README、所有 references 和模板的玩家执行阶段；子模块不能放宽。先探测宿主**已有**专用工具的真实接口、输入、输出、限额和失败行为，或目标 OS 原生命令，再执行操作；工具名只是能力类别，不是凭空存在的命令。

- 禁止依赖或调用 Python/Node.js/Ruby/Perl 等额外编程运行时，禁止生成/执行临时程序或脚本、安装运行时或库、借 Java 编译/执行自写 helper 绕过。宿主专用工具内部如何实现不属于玩家依赖，但不得通过 terminal、通用代码执行器或所谓“一次性宿主任务”偷偷运行这些代码。没有能力就 BLOCKED，不索要安装授权来解除本禁令。
- 允许已有专用文件、结构化 JSON/TOML、HTTP、hash、归档、进程和安全存储工具，以及目标 OS 原生 shell/命令；Windows PowerShell 仅在系统原生可用时使用。原生 shell 只调度已核实的原生命令，不生成跨平台脚本依赖、不用其手拼 NBT/图像/网络协议；非原生 pwsh 等也不能临时安装。调用前核对真实版本/参数，不能杜撰工具名或跨 OS 照抄。
- **Java 是 Minecraft 必需依赖，仍允许按 Java/下载模块明确授权安装或下载，游戏、库、assets、Mod 的合法下载也仍允许。** 禁的是额外编程能力与自写 helper，不是正常 Java 探测和游戏 mainClass 执行；官方 loader installer、元数据固定的 ForgeWrapper 及其声明的 processors 是游戏依赖，按 [现代 Forge/NeoForge](references/forge-install.md) 披露精确执行链并取得针对该链的确认；未知 installer/processor 仍阻断，不允许自写 helper。
- 禁止 Computer Use、桌面控制、模拟鼠标键盘、GUI 自动截图、浏览器驱动游戏 UI 及 GUI 验收兜底。主菜单、进入世界/服务器、菜单选择和渲染效果由玩家亲自操作并确认；无确认返回 WAIT_INPUT，当前进程只能报告 spawned/running/exited。日志只证明其对应事件，不冒充画面或玩家确认。已有 PNG 的文件管理/经授权专用图像工具读取不等于允许抓屏。
- 整合包归档按 [Prism 式解包](references/prism-archive.md) 使用已有 libarchive/系统 bsdtar 等经核实接口；不因缺全 Unicode 归一化、流式配额或专用 JSON 内存接口阻断正常包。缺专用 NBT、图像解码/预览、DNS/Java SLP、实际路径防护或原子写/锁能力时，依赖它的操作返回 `BLOCKED: CAPABILITY_MISSING (<具体能力>)`，保留不依赖它的只读候选/其他模块。不删除复杂流程假称全支持；typed NBT 无损、归档安全、真实协议等验收保持不变。
- 登录和启动使用现有 HTTP、系统凭据存储、原生命令及普通后台进程工具即可；不要求专用秘密中转、安全编排、私密 argv 注入或流式日志脱敏接口，不因缺少这些接口阻断正常使用。可在本地读取本项目凭据、传递 token 并构造实际启动参数；避免无关回显，不把令牌写进聊天、普通配置或提交到仓库。设备授权由玩家亲自在官方网页完成，不控制其桌面。

能力探测只读、不安装、不联网认证；实际网络与写入仍走相应门禁。具体工具的路径/链接防护、互斥、原子提交须按目标模块核实，不能只以“系统自带”推定满足；整合包归档的预检告警与用户明确要求的硬限额须区分，不把不存在于 Prism 流程的流式限额器作为默认门槛。

## 固定执行规则

1. 每操作依次执行 Step 1 → 2 → …；子流程完成后带结果返回明确调用点，遇 WAIT_INPUT/BLOCKED 停止依赖它的后续步骤。
2. 复用已明确的实例/版本/账号/Java/来源选择；无文档默认值则询问，不能猜用户要原版、离线或最新版本。
3. 终点：DONE=请求结果已验证；WAIT_INPUT=仅询问当前缺少选择/授权；BLOCKED=具体前提/工具/服务失败。查询、下载、落盘、可启动和进入主菜单分别验收。
4. 所有新管理写入共用 [门禁](references/operations.md#写入前门禁) 与 [提交恢复](references/operations.md#提交与恢复)。已持锁子流程复用 operation，不循环申请锁/备份；所有写者和一致性快照先独占取得父级稳定操作锁，持有到验收；子流程复用，不以单次检查替代互斥。
5. 世界、servers.dat、版本/loader、运行内容、恢复及实例删除必须停机确认；禁止只凭旧 PID/lastLaunch 判断。覆盖/删除先独立备份，链接副本不是独立备份。
6. 只改授权的实例/配置/导出目标，不改 PATH/系统默认 Java；不跟随根外链接，不自动清空文件，不运行导入包里的命令/脚本。
7. 全部网络（包括元数据、认证刷新、Java 下载、状态 ping、上传）先过 [网络策略](references/operations.md#网络与来源)。离线身份不等于禁联网；offline=true 时不能偷偷发请求。换源、执行第三方程序与外发需独立授权；唯一默认来源例外是 [CF 精确同字节兜底](references/curseforge-import.md#解析与恢复)：官方 downloadUrl 缺失/下载受限时允许 Modrinth 官方 SHA-1 查询，仅接受 CF 权威 SHA-1+size 完全一致及合法公开 URL，不重复询问同范围授权，不绕过撤回或已知禁止分发限制。
8. 微软设备流的 user_code、官方 verification_uri 和有效期可以向当前发起玩家显示；后台 device_code、access/refresh token 供授权工具链按需使用，不在聊天或普通配置展示；密码/MFA 由玩家自行输入，不在聊天收集。公开 OAuth client ID 不是秘密。
9. 不把文件禁用/Shader 下载/世界恢复称成游戏内已生效；无真实兼容性证据或玩家生效确认就明确未验证。

## 主入口

### Step 1：识别操作与范围

提取明确 action、实例、资源类型、版本、路径、账号/Java/来源以及读写意图。复合请求按依赖执行，例如先创建实例再安装 Mod；未知目标只问当前缺项。将“查看/列举/诊断”与“安装/更新/删除/恢复/上传”区分，不从查询推导写入授权。

### Step 2：读取配置和必要前提

玩家已给本地整合包时，先转 [导入整合包](references/modpacks.md#导入整合包) Step 1～3 的只读识别分支，安全读取 manifest，得到名称、包版本、MC 和 loader 精确版本后回本 Step。此读取不依赖配置初始化、账号、Java 或 CF key；没有配置也先报告包事实与计划。只有在线搜索/挑选版本才走搜索流程，不能让本地包用户重选一轮已声明版本。

执行 [配置读取](references/configuration.md#读取配置)。缺配置的只读请求返回未初始化/空列表，不建目录或登录；需要持久化时才执行 [首次初始化](references/local-files.md#首次初始化)，根目录必须明确问玩家。损坏配置保留原件，不用模板覆盖。

首次完整“启动 MC”在目录就绪后先完成 [账号选择/绑定](references/accounts.md#查看与选择)，再选实例；微软必须实际授权并取得权利/角色，不能只存 client ID 就称登录。复用等待中的设备授权任务，不重复发码。单独管理目录、Java、资源、世界、备份、截图不强制登录。

### Step 3：加载一个目标模块

| 玩家目标 | 执行模块 |
|---|---|
| 启动/再次启动 MC | 优先执行本文顶部快速启动；仅缓存缺失/失效加载 [启动缓存](references/launch-cache.md) / [启动管理](references/launch.md) |
| 停止 MC | [启动管理](references/launch.md#停止游戏) |
| 初始化/查看/更换数据目录 | [本地文件](references/local-files.md) |
| 刷新/列出/增删改查 Java | [Java 管理](references/java-management.md) |
| 微软登录/离线身份/切换或删除账号 | [账号管理](references/accounts.md) |
| 创建/选择/修改/复制/删除实例 | [实例管理](references/instances.md) |
| 查询/安装/切换/移除加载器，变更 MC 版本 | [版本与加载器](references/loaders.md) |
| 列 Mod、本地导入、搜索安装、启禁用、改版、更新/锁定、移除 | [Mod 管理](references/mods.md) |
| 管理资源包/旧材质包/Shader/世界数据包 | [内容资源管理](references/resources.md) |
| 列世界、复制/导入导出、改名、删除、进入世界 | [世界管理](references/worlds.md) |
| 列/编辑客户端服务器、查询状态、启动并加入 | [服务器列表](references/servers.md) |
| 查看/预览/导出/重命名/删除/分享截图 | [截图管理](references/screenshots.md) |
| 已给本地 ZIP/mrpack，导入整合包 | [本地先读清单](references/modpacks.md#导入整合包)，复用包内名称/版本/MC/loader |
| 在线搜索/选择整合包，导出/更新 | [整合包管理](references/modpacks.md) |
| 创建/列出/校验/恢复/删除备份 | [备份恢复](references/backups.md) |
| 日志/启动预检/崩溃诊断/摘要或确认后上传 | [日志诊断](references/diagnostics.md) |

下载、元数据和认证是共享能力，不要求玩家先手工调用：组件走 [Prism 链路](references/prism-components.md)，文件/Java 走 [下载](references/downloads.md)，微软走 [设备流与刷新](references/microsoft-login.md)。只读模式先做模块只读步骤，不强制走写入门禁。

### Step 4：执行、持久化与返回

先加载 [公共操作契约](references/operations.md)，再按目标模块全部 Step 执行。创建实例继续沿用 Prism 式名称/MC/loader/loader 版本选择，确认即自动下载所需游戏材料，不重复问同范围许可；额外 Java、换源、执行外部代码另确认。

启动先走本文顶部快速启动；仅首次/失效重建按 [完整启动步骤](references/launch.md)，命令按 [唯一 command 构造规则](references/launch-command.md#拼接-command)：`[javaExecutable, *jvmArgv, mainClass, *gameArgv]`，`cwd=gameRoot`，优先 `shell=false`；仅有原生 shell 调度时按 command Step 9 正确转义，真实在线 argv 不向聊天展示。此直接主类方案不是 Prism EntryPoint/stdin 协议的完整复制，legacy/traits/quick-play 未覆盖时明确阻断。

变更只通过 [配置保存](references/configuration.md#保存配置) 合并负责字段并读回；不覆盖其他模块、未知字段或别的任务结果。内容来源/版本/hash、备份和阶段日志按公共 schema 登记；配置是磁盘文件，不是对话记忆。

### Step 5：按证据报告

DONE 仅报告完成操作、目标和验收。WAIT_INPUT 提当前问题；BLOCKED 提失败阶段、工具证据、已应用部分及备份/恢复入口。游戏区分 spawned/running/ready/exited；主菜单须玩家明确确认（记录来源为 player_confirmation），历史 ready 不覆盖当前退出状态。

## 数据与安全约定

唯一全局配置为当前用户的 `~/.minecraftstartup/config.json`，旧配置只补缺省，不重置已有数据。默认根下 instances/mods/java/skins/downloads；实例位于 `instances/<稳定ID>/.minecraft/`，游戏库/assets/mods/config/saves 全隔离，Java 可按 ID 共享；根 mods 仅是文件库。

默认公开微软 client ID 来自 PrismLauncher，可覆盖，不声称拥有该应用或获得上游背书；认证准入必须实测。本地 CF 整合包不强制玩家申请个人 Key：默认优先 Prism 官方解析/下载/校验链路，本项目维护者已确认 Prism 应用 Key 使用授权，按 [授权接入](references/curseforge-import.md#已授权-prism-应用-key) 读取并请求，不再仅因其来自 Prism 阻断；实际缺来源或服务拒绝才报告具体问题，不自动降级或任意换源；官方 downloadUrl 缺失/下载受限项默认允许 Modrinth 官方 SHA-1 精确同字节兜底，严格复核 CF 权威 SHA-1+size、合法公开 URL 并分别记录 CF 身份与实际来源，不重复询问同范围授权；CFWidget/Curse Maven 等其他第三方适配仍仅玩家另行选择来源时按 [公开服务导入](references/curseforge-public-import.md) 确认后继续；不能冒称 Prism 使用匿名官方 API。API key/token 优先存系统凭据存储，通过 credentialRef 定位；不要求专用秘密句柄接口。外发前展示并复核脱敏内容；隐私过滤不能保证所有第三方内容安全。

## 验证

每模块都要检查真实文件/来源/hash、授权范围、持久化读回与失败恢复；不能用目录/PID/HTTP 请求成功替代最终验收。未支持格式、受限平台服务、缺失依赖或未覆盖版本立即给出明确 BLOCKED，不猜协议、不伪造成功。文档结构检查不等于所有玩家环境已实测。
