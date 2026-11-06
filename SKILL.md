---
name: minecraft-startup
description: Manage Minecraft, instances, accounts and game content.
version: 0.1.0
license: GPL-3.0-only
---

# MinecraftStartup

你的下一款启动器，何必是启动器。

GPL-3.0-only。许可全文见 [LICENSE](LICENSE)，分发说明见 [COPYING.md](COPYING.md)，Prism Launcher / MultiMC / PolyMC 的来源与声明见 [NOTICE.md](NOTICE.md)。无担保；“许可/关于/致谢”请求只展示这些材料，不初始化或启动游戏。

这是可执行操作手册，不是需求清单或另一套启动器程序。一个主入口按玩家目标加载所需管理模块；共用配置、网络、备份和提交契约，不把 Prism 的 Qt UI 或 Java bridge 当必装依赖。正常路径按 Step 执行，未覆盖异常才深入诊断。

## 使用条件与边界

宿主需提供真实本地文件/JSON/归档/hash、HTTP、受控进程、秘密存储及必要的画面/日志工具，可使用同等能力的一次性可审阅任务。某项操作缺工具时只阻断该项；不因要列截图就要求登录。项目只发布指令与声明式模板，不捆绑运行时代码。

不运行服务端托管、不替玩家购买游戏、不读取其他启动器凭据，不把“列命令”当已执行。初始结构见 [配置模板](templates/config.example.json)，不含任何真实玩家配置。

## 固定执行规则

1. 每操作依次执行 Step 1 → 2 → …；子流程完成后带结果返回明确调用点，遇 WAIT_INPUT/BLOCKED 停止依赖它的后续步骤。
2. 复用已明确的实例/版本/账号/Java/来源选择；无文档默认值则询问，不能猜用户要原版、离线或最新版本。
3. 终点：DONE=请求结果已验证；WAIT_INPUT=仅询问当前缺少选择/授权；BLOCKED=具体前提/工具/服务失败。查询、下载、落盘、可启动和进入主菜单分别验收。
4. 所有新管理写入共用 [门禁](references/operations.md#写入前门禁) 与 [提交恢复](references/operations.md#提交与恢复)。已持锁子流程复用 operation，不循环申请锁/备份；所有写者和一致性快照先独占取得父级稳定操作锁，持有到验收；子流程复用，不以单次检查替代互斥。
5. 世界、servers.dat、版本/loader、运行内容、恢复及实例删除必须停机确认；禁止只凭旧 PID/lastLaunch 判断。覆盖/删除先独立备份，链接副本不是独立备份。
6. 只改授权的实例/配置/导出目标，不改 PATH/系统默认 Java；不跟随根外链接，不自动清空文件，不运行导入包里的命令/脚本。
7. 全部网络（包括元数据、认证刷新、Java 下载、状态 ping、上传）先过 [网络策略](references/operations.md#网络与来源)。离线身份不等于禁联网；offline=true 时不能偷偷发请求。换源、执行第三方程序与外发需独立授权。
8. 微软设备流的 user_code、官方 verification_uri 和有效期可以向当前发起玩家显示；后台 device_code、access/refresh token、密码/MFA 仅留在本地秘密上下文，不回显模型、聊天、普通配置或日志。公开 OAuth client ID 不是秘密。
9. 不把文件禁用/Shader 下载/世界恢复称成游戏内已生效；无真实兼容性或画面证据就明确未验证。

## 主入口

### Step 1：识别操作与范围

提取明确 action、实例、资源类型、版本、路径、账号/Java/来源以及读写意图。复合请求按依赖执行，例如先创建实例再安装 Mod；未知目标只问当前缺项。将“查看/列举/诊断”与“安装/更新/删除/恢复/上传”区分，不从查询推导写入授权。

### Step 2：读取配置和必要前提

执行 [配置读取](references/configuration.md#读取配置)。缺配置的只读请求返回未初始化/空列表，不建目录或登录；需要持久化时才执行 [首次初始化](references/local-files.md#首次初始化)，根目录必须明确问玩家。损坏配置保留原件，不用模板覆盖。

首次完整“启动 MC”在目录就绪后先完成 [账号选择/绑定](references/accounts.md#查看与选择)，再选实例；微软必须实际授权并取得权利/角色，不能只存 client ID 就称登录。复用等待中的设备授权任务，不重复发码。单独管理目录、Java、资源、世界、备份、截图不强制登录。

### Step 3：加载一个目标模块

| 玩家目标 | 执行模块 |
|---|---|
| 启动/再次启动/停止 MC | [启动管理](references/launch.md) |
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
| 搜索/导入/导出/更新整合包 | [整合包管理](references/modpacks.md) |
| 创建/列出/校验/恢复/删除备份 | [备份恢复](references/backups.md) |
| 日志/启动预检/崩溃诊断/摘要或确认后上传 | [日志诊断](references/diagnostics.md) |

下载、元数据和认证是共享能力，不要求玩家先手工调用：组件走 [Prism 链路](references/prism-components.md)，文件/Java 走 [下载](references/downloads.md)，微软走 [设备流与刷新](references/microsoft-login.md)。只读模式先做模块只读步骤，不强制走写入门禁。

### Step 4：执行、持久化与返回

先加载 [公共操作契约](references/operations.md)，再按目标模块全部 Step 执行。创建实例继续沿用 Prism 式名称/MC/loader/loader 版本选择，确认即自动下载所需游戏材料，不重复问同范围许可；额外 Java、换源、执行外部代码另确认。

启动按 [完整启动步骤](references/launch.md)，命令按 [唯一 command 构造规则](references/launch-command.md#拼接-command)：`[javaExecutable, *jvmArgv, mainClass, *gameArgv]`，`cwd=gameRoot`，`shell=false`；真实在线 argv 不回显。此直接主类方案不是 Prism EntryPoint/stdin 协议的完整复制，legacy/traits/quick-play 未覆盖时明确阻断。

变更只通过 [配置保存](references/configuration.md#保存配置) 合并负责字段并读回；不覆盖其他模块、未知字段或别的任务结果。内容来源/版本/hash、备份和阶段日志按公共 schema 登记；配置是磁盘文件，不是对话记忆。

### Step 5：按证据报告

DONE 仅报告完成操作、目标和验收。WAIT_INPUT 提当前问题；BLOCKED 提失败阶段、工具证据、已应用部分及备份/恢复入口。游戏区分 spawned/running/ready/exited；主菜单须实际画面或玩家确认，历史 ready 不覆盖当前退出状态。

## 数据与安全约定

唯一全局配置为当前用户的 `~/.minecraftstartup/config.json`，旧配置只补缺省，不重置已有数据。默认根下 instances/mods/java/skins/downloads；实例位于 `instances/<稳定ID>/.minecraft/`，游戏库/assets/mods/config/saves 全隔离，Java 可按 ID 共享；根 mods 仅是文件库。

默认公开微软 client ID 来自 PrismLauncher，可覆盖，不声称拥有该应用或获得上游背书；认证准入必须实测。API key/token 只用宿主安全句柄。外发前展示并复核脱敏内容；隐私过滤不能保证所有第三方内容安全。

## 验证

每模块都要检查真实文件/来源/hash、授权范围、持久化读回与失败恢复；不能用目录/PID/HTTP 请求成功替代最终验收。未支持格式、受限平台服务、缺失依赖或未覆盖版本立即给出明确 BLOCKED，不猜协议、不伪造成功。文档结构检查不等于所有玩家环境已实测。
