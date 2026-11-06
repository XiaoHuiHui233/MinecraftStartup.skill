# MinecraftStartup.skill

**你的下一款启动器，何必是启动器。**

加载 [SKILL.md](SKILL.md)，让具备本地工具的 Agent 按固定 Step 管理 Minecraft。目标平台为 **Windows、Linux、macOS**；公共流程统一，系统工具按实际 OS 适配，不依赖某一平台独有命令。一个入口按目标加载模块，正常路径照步骤执行，缺选择就问，异常才深入诊断；不要求安装 Prism，不另发布启动器程序。

## 快速启动优先

直接说“MC启动”或指定任意实例时，Agent 优先读取该实例缓存的 Java 指令，最小核对后注入本次会话并启动，不再逐个读取管理手册、扫描所有 Java 或重走安装流程。每次配置变更后更新受影响实例的缓存；只有缺失/失效才定点重建。缓存不保存令牌，也不代替玩家确认主菜单。见 [最高优先级入口](SKILL.md#最高优先级直接启动走缓存) 与 [缓存维护](references/launch-cache.md)。

本地整合包按 [Prism 式解包](references/prism-archive.md) 用已有归档工具先检查路径并读取 manifest；不强制额外流式配额/Unicode 归一化引擎，复用包内版本与 loader；CF 默认按 Prism 的官方 API 解析、官方文件下载和 size/hash 校验链路执行，不强制玩家申请个人 API Key；维护者确认已获 Prism 官方使用该应用 Key 的授权，授权部署按 [应用 Key 接入步骤](references/curseforge-import.md#已授权-prism-应用-key) 使用，不再要求玩家另行申请；Key 不嵌入文档，其他部署不由此自动获得授权。官方 downloadUrl 缺失/下载受限时，默认允许 [Modrinth 官方 SHA-1 同字节兜底](references/curseforge-import.md#解析与恢复)，不重复询问同范围授权；必须匹配 CF 权威 SHA-1+size、验证合法公开 URL，分别保留 CF 身份与实际来源，不绕过撤回或已知禁止分发限制。仅在玩家另选其他第三方来源时使用 [CFWidget + Curse Maven 无个人 Key 导入](references/curseforge-public-import.md)；后者明确记录第三方来源和非官方哈希验证等级，受限/缺失项不跳过。现代 Forge/NeoForge 使用已确认的官方安装器与元数据固定 Wrapper；导入默认新实例、不自动启动。见 [整合包步骤](references/modpacks.md) 与 [loader 步骤](references/forge-install.md)。

## 管理模块

| 目标 | 操作手册 |
|---|---|
| 目录、配置、Java、账号 | [本地文件](references/local-files.md) · [配置](references/configuration.md) · [Java](references/java-management.md) · [账号](references/accounts.md) |
| 实例与版本/加载器 | [实例创建/选择/复制/删除](references/instances.md) · [版本与加载器](references/loaders.md) |
| 启动、停止与诊断 | [完整启动流程](references/launch.md) · [command 构造](references/launch-command.md) · [日志诊断](references/diagnostics.md) |
| Mod 和游戏资源 | [Mod](references/mods.md) · [资源包/Shader/数据包](references/resources.md) |
| 世界、服务器与截图 | [世界](references/worlds.md) · [客户端服务器列表](references/servers.md) · [截图](references/screenshots.md) |
| 整合包和独立备份 | [整合包](references/modpacks.md) · [备份恢复](references/backups.md) |

## 共用能力

- [操作契约](references/operations.md)：输入、路径与运行门禁、授权、锁、备份、提交、失败恢复、网络策略及持久字段。
- [Prism 组件与依赖](references/prism-components.md)、[游戏/Java 下载](references/downloads.md)、[版本参数解析](references/version-profile.md)、[微软设备授权与刷新](references/microsoft-login.md)。
- [初始配置模板](templates/config.example.json)：只提供空数据结构，不是实际玩家配置。

配置持久化于 `~/.minecraftstartup/config.json`。首次写操作需明确选择数据根目录；只读查询不触发登录/下载/初始化。实例游戏数据隔离，覆盖/删除前备份，恢复和内容修改需停机确认。创建实例确认包含该实例所需游戏下载，不重复问同范围授权。

默认微软公开 OAuth client ID 来自 PrismLauncher，可覆盖，不表示上游背书；登录和启动可用现有 HTTP、系统凭据命令及普通后台进程工具完成，不要求专用秘密中转、私密参数注入或流式脱敏接口；不向聊天展示令牌。外部内容、第三方 API 与文件格式按模块明确支持范围，不绕过受限分发。Qt UI、自更新器和 Prism Java bridge 不作为本 Skill 强依赖。

只发布 Skill 指令与声明式模板。玩家运行禁止 Python/Node.js/Ruby/Perl 等额外运行时、临时脚本及运行时/库安装，不借 Java 自写 helper 绕过；允许已有专用工具与目标 OS 原生命令（如系统原生 Windows PowerShell），不依赖跨平台脚本。Java 作为游戏依赖的授权安装及合法游戏下载仍允许。禁止 Computer Use/桌面控制/模拟键鼠/GUI 自动截图；主菜单、进入世界与画面生效由玩家操作确认，日志不冒充画面证据。详见 [玩家运行能力契约](SKILL.md#玩家运行能力契约)。NBT、图像、协议等缺已有合格能力时只阻断依赖项，不临时安装、不声称全功能已支持。文档与示例检查不代表所有模块、服务和游戏版本均已端到端实测。

## 开源许可与致谢

本项目采用 [GPL-3.0-only](LICENSE)，适用范围与分发说明见 [COPYING.md](COPYING.md)。

感谢 [Prism Launcher](https://prismlauncher.org/) 及 MultiMC、PolyMC 贡献者的开放实现。上游许可与致谢见 [NOTICE.md](NOTICE.md)。本项目独立开发，非 Prism Launcher 官方项目，未获其认可或关联背书。
