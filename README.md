# MinecraftStartup.skill

**你的下一款启动器，何必是启动器。**

加载 [SKILL.md](SKILL.md)，让具备本地工具的 Agent 按固定 Step 管理 Minecraft。一个入口按目标加载模块，正常路径照步骤执行，缺选择就问，异常才深入诊断；不要求安装 Prism，不另发布启动器程序。

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

默认微软公开 OAuth client ID 来自 PrismLauncher，可覆盖，不表示上游背书；秘密由宿主安全能力处理。外部内容、第三方 API 与文件格式按模块明确支持范围，不绕过受限分发。Qt UI、自更新器和 Prism Java bridge 不作为本 Skill 强依赖。

只发布 Skill 指令与声明式模板；所需工具由宿主提供，缺能力/未知格式必须准确阻断。文档与示例检查不代表所有模块、服务和游戏版本均已端到端实测。

## 开源许可与致谢

本项目采用 [GPL-3.0-only](LICENSE)，适用范围与分发说明见 [COPYING.md](COPYING.md)。

感谢 [Prism Launcher](https://prismlauncher.org/) 及 MultiMC、PolyMC 贡献者的开放实现。上游许可与致谢见 [NOTICE.md](NOTICE.md)。本项目独立开发，非 Prism Launcher 官方项目，未获其认可或关联背书。
