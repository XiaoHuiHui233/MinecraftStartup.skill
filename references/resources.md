# 资源包、旧材质包、Shader 与世界数据包

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

范围是 Java Edition 普通实例内的文件资源，不是世界商店、全局数据包框架或 Bedrock Marketplace。调用前提为文件/ZIP/JSON/hash/HTTP/进程工具；要验证游戏内效果还需玩家提供对应 UI/命令证据或授权可观察的游戏工具。缺工具返回具体 BLOCKED，不把可执行步骤改成抽象“安装资源”。

## 目录与子流程契约

路径均相对实例 gameRoot（实例自有 `.minecraft/`）：

| kind | 唯一目标范围 | 格式及限制 | 游戏侧生效入口 |
| --- | --- | --- | --- |
| resourcepack | `resourcepacks/<文件.zip或目录>` | 根有 pack.mcmeta，读取 pack 描述/格式及 assets；按实际 MC 支持的格式范围核对 | Options → Resource Packs，选入并确认顺序，Done 等重载 |
| texturepack | `texturepacks/<文件.zip或目录>` | Java 1.6 之前的旧材质布局如 pack.txt/terrain.png；不把现代 pack 自动转换 | 对应旧客户端 Texture Packs 菜单选中 |
| shader | `shaderpacks/<文件.zip或目录>` | 根有 shaders/ 或提供方明确的合法结构；不是 `.jar` Mod | 已安装兼容 Iris/OptiFine 等的 Shader 菜单选择并启用 |
| datapack | `saves/<已选世界目录>/datapacks/<文件.zip或目录>` | 根有 pack.mcmeta 和 data/；要求该 MC 支持数据包 | 世界创建时数据包 UI；已有世界用 `/datapack`/`/reload` 并验证 |

`<世界目录>`来自实际 saves 直接子目录及 level.dat，只用稳定实际路径，不用世界显示名拼路径；必须选唯一世界，不创建同名空世界。不存在的目录/链接逃逸 BLOCKED。无原生全局 datapacks；Prism 的 GlobalDataPacksEnabled 页面不证明 vanilla 会加载任意实例级 datapacks。玩家请求全局时要求具体框架及可核实流程，否则 BLOCKED: GLOBAL_DATAPACK_UNSUPPORTED。pack_format/支持范围随 MC 变化，读取实际包声明并与该 MC 可复核规范核对；不能凭记忆硬编码新版本格式，未知标 unverified，不静默接受警告。

所有操作先执行 [操作上下文](operations.md#输入与结果约定)，返回对应 Step 的路径定位处；所有实例文件写入执行完整 [写入门禁](operations.md#写入前门禁)，返回调用方下一步，再执行 [提交与恢复](operations.md#提交与恢复)。同一 operation 仅申请一次共享锁，备份子流程复用当前锁且禁止递归备份。复用 mods 子流程时只调用其规划/已持锁提交部分，不能重新门禁。网络先执行 [网络与来源](operations.md#网络与来源)，返回当前 HTTP 步骤；离线仅允许本地扫描/复制。

content.entries 使用公共 schema，kind 按上表，datapack 的 relativePath 包含 saves/世界/datapacks 前缀；provider/projectId/versionId/hashes/updateLocked 与 [Mod 来源锁](mods.md#调用契约与来源锁) 相同，sidecar 保存在 instanceRoot/content-state/，同样参与备份/提交。enabled 沿用布尔磁盘状态：已合法落盘且未隔离的资源填 true，仅表示可供游戏扫描，不表示选中/生效；已明确移到实例受控隔离区的文件才填 false。本模块不提供以改后缀隔离这些资源的操作。游戏选择/渲染/数据包启用证据保存在来源 sidecar 的非秘密数据中，未观察时为 unknown（含证据来自文件、命令或 UI，不能用磁盘存在充当激活）。缺省不修改 options.txt、level.dat、Iris/OptiFine 配置；不向公共 schema 擅自加入可空 enabled 或 activation 字段。

## 扫描资源

### Step 1：确定类型与世界

输入实例与 kind 或 all；上下文定位后扫描表中存在的目录。datapack 必须选世界；all 可列出每个真实世界的资源但不得把多个世界聚合成可批量写目标。不登记扫描结果、不联网、不切换当前实例；目录未建立显示 empty，实例/世界原路径丢失则 blocked。

### Step 2：读结构与来源

用文件工具列直接子项；ZIP 安全只读最多 20,000 成员、单个元数据 4 MiB，计算文件 sha256，拒绝穿越和链接成员，不执行内容。目录包只在授权资源根内按路径顺序列普通文件和逐文件 hash；符号/硬链接、设备或过大目录停止该项。读取 pack.mcmeta 的 pack 字段与格式范围、shader 结构、旧材质文件；存在多层外壳时标 nested_invalid，不在扫描时自动解包修复。把资源自身数据、来源锁和本地快照分列。

### Step 3：区分存在与选中

每项显示 present/missing/invalid、受管/未知来源、项目版本及更新锁；resourcepack 的 options.txt 选择或 Shader 配置只能作为静态提示，不能证明本次重载/渲染成功。datapack 不能从文件名推断 enabled，level.dat 不随便写。`.disabled` 对这些类型不统一可靠；未知后缀/目录不判定禁用。游戏正在运行时标时间点快照。

### Step 4：验收返回

用工具汇总扫描数、异常数和作用域，说明未检查的世界/目录及游戏内状态 unknown/有证据。无配置或根目录时不初始化玩家数据。需要导入/下载/移除转各操作 Step 1，不在扫描后自动修改。

## 本地导入或在线下载

### Step 1：选择具体目标

输入 kind、实例、datapack 的世界、文件路径或具体 Modrinth 项目/版本。上下文核对目录；Shader 安装不自动安装 Iris/OptiFine，缺渲染器可明确“只存文件”或等待玩家请求 [Mod 管理](mods.md#选择版本与依赖) 安装兼容渲染器；旧材质只对真实旧客户端处理，现代 MC 请求 texturepack 时让玩家改选 resourcepack 或提供转换后的合法资源，不悄悄改类型。

### Step 2：取得文件计划

本地：只读校验结构、hash，目录包可用普通复制到 staging，逐文件清单禁止链接；不执行脚本，不随意移除外壳。在线：执行公共网络流程，复用 [Modrinth 搜索](mods.md#modrinth-搜索) 的 GET /search、编码与 offset 分页，替换 facets/版本加载器：resourcepack 用 project_type:resourcepack 与版本 loaders=["minecraft"]；shader 用 project_type:shader，GET `/tag/loader` 检查 name/supported_project_types 后按实际渲染器选择（如 iris/optifine，不照抄实例 fabric）；datapack 使用服务实际支持的 datapack 项目筛选及 loaders=["datapack"]。若 v2 服务仅按 mod 分类数据包，允许明确展示 categories:datapack 候选后逐版本核实 loaders，而不能把 mod JAR 装进世界。schema/标签不支持则 BLOCKED，不猜 API。texturepack 无独立通用 Modrinth 类型，默认本地导入；在线仅在具体项目明确提供此旧版合法 ZIP 且能核验时使用，不以资源包命中冒充。

### Step 3：核验版本与依赖

GET `/project/<ID>/version`，game_versions 为所选 MC 的 JSON 数组，loaders 按 Step 2；选具体 ID 后 GET `/version/<ID>`。检查 project_id/id/game_versions/loaders/version_type/date_published/files 的 filename/url/size/hashes/primary；执行 [版本与依赖](mods.md#选择版本与依赖) 的受限图规划规则（深度20/节点200/请求400、required 纳入、incompatible 检查、optional 不默认装），但资源目标路由必须保持本表 kind。跨 kind 的 required 需独立可证明类型和授权；需要改世界/安装渲染器时不得隐式扩大计划。多主文件 WAIT_INPUT；不能把 sources/dev JAR当资源。默认 release，SHA-512优先/有 SHA-1也核验；无上游 hash 阻断在线安装。本地 hash只是快照。

### Step 4：门禁与暂存

展示新增/覆盖路径、目标世界、文件大小/来源、附加依赖和“落盘不等于启用”。同名不同 hash/同项目旧版本先确认替换；不同目录包不合并覆盖。写入门禁返回本 Step 的下载/复制暂存处。HTTP下载调用 [单文件下载](downloads.md#单文件下载)，目标为 operation staging，返回本 Step 校验处；本地普通复制也进入 staging。核对完整性和根结构，再写来源 sidecar，初次 enabled=true 表示磁盘可扫描、sidecar 的激活证据为 unknown。没有游戏内启用权限不阻断明确的仅文件导入。

### Step 5：提交并验收

执行提交与恢复，登记公共 content 字段和 sidecar，目录包按逐文件清单验证，hashes 为空或仅用于已有文件摘要，不伪造“目录文件 hash”（完整目录证据在 sidecar）。回到本 Step 重扫：路径/结构/大小/hash、来源锁、未覆盖其他世界均匹配。返回 files_installed、activation=unverified，下一步只有玩家要求才到“游戏内启用与停用”；不自动启动 MC。

## 检查更新或指定改版

### Step 1：核验锁与当前字节

输入 content ID、check/apply/change-version、明确 versionId（改版时）和 channel。上下文后重扫；updateLocked 项跳过，单次越过锁须玩家明确允许，保留原锁值。文件 hash 不符来源锁则 source_stale；未知来源只有玩家批准才按 [Mod 更新识别](mods.md#更新或指定改版) 的 SHA-512 `/version_file` 流程识别，不能按文件名猜。目录包/改过的文件默认不自动更新。

### Step 2：规划替换与激活差异

执行“本地导入或在线下载” Step 2–3 的 API/类型校验取得候选，指定 ID精确选择，否则同 MC/类型/渲染器/channel 按真实发布时间推荐。列出旧新路径、hash、依赖、文件选中状态可能失效；check 在此返回，不能落盘。resourcepack/texturepack 文件名改变要重新在游戏菜单选中；Shader 若有 `<old.zip>.txt` 配置可提出复制到新 ZIP 的 `.txt`，只有目标不存在、玩家确认且格式兼容才做，绝不覆盖新配置或保证新版本兼容。datapack 在同一世界改名可能改变 `file/...` 标识与顺序，须列重新启用步骤；不能把改名说成自动迁移 enabled。

### Step 3：一次门禁完成替换

确认计划后写入门禁，返回本 Step 暂存处，只拿一次锁。按下载子流程把新包校验进 staging，再执行提交与恢复；新文件通过前不删除旧包。同名替换原子替换单文件，目录包及 sidecar 是逐项事务记录不是整体原子。保留 content ID/updateLocked 与磁盘 enabled，游戏选中状态无法重新证明时将 sidecar 激活证据标 unknown，并保留历史证据，不假报已启用。

### Step 4：验收与失败出口

读回每项新锁/文件及世界边界；报告已更新/无候选/被锁跳过/失败，activation 待复核。提交部分失败走公共恢复；外部改动触发 RECOVERY_CONFLICT。涉及 datapack 已实际执行并修改世界后，回退 ZIP 不等于回退世界，必须通过 [恢复备份](backups.md#恢复备份) 的完整世界恢复计划；不以一次 `/reload` 保证撤销副作用。

## 游戏内启用与停用

### Step 1：取得独立生效授权

输入具体资源、目标世界/实例、enable/disable 和当前游戏状态。只落盘的请求不推断启用授权。展示数据包可能运行函数、修改世界的风险，确认世界备份（游戏退出下用 [创建备份](backups.md#创建备份)；该备份请求自身持锁一次，完成释放后再游戏操作）。游戏 UI/命令属于本节允许的游戏内变更；不持实例写锁同时改磁盘，不在运行中用文件工具修改 options.txt/level.dat。需启动而未运行时交 [启动流程](launch.md) 并返回本 Step；本模块不替玩家静默启动。缺游戏工具或玩家协助返回 WAIT_INPUT: ACTIVATION_EVIDENCE，而非 enabled=true。

### Step 2：执行类型对应动作

resourcepack：在 Options → Resource Packs 找确切包，enable 移到 Selected 并确定优先级，disable 移回 Available，点 Done 等重载；未知格式警告先暂停确认，不能自动接受。texturepack：在实际旧版 Texture Packs 菜单选包，停用改选 Default；没有该菜单说明类型/版本不匹配。shader：打开已安装渲染器的 Shader Packs 菜单，选择目标并启用；停用用 OFF/关闭 Shaders，不能只给 ZIP 添加 `.disabled`。菜单因渲染器版本不同则根据可见 UI 操作，找不到返回 BLOCKED: RENDERER_UI_UNKNOWN，不猜配置键。

数据包：在确切目标世界、具有命令权限时先 `/datapack list available` 和 `/datapack list enabled`，用实际返回的 `file/<真实包标识>`，enable 执行 `/datapack enable "<实际ID>"`（默认顺序不改；玩家要求时才用 first/last/before/after），disable 执行 `/datapack disable "<实际ID>"`；需要应用磁盘新内容时在授权下 `/reload`。只在服务器/世界明确授权范围执行，不私自开作弊、开 LAN 或授 OP。命令权限不足/包格式不兼容/feature flag缺失返回 BLOCKED；不要直接编辑 NBT 绕过。命令参数用游戏命令字符串转义，不能把不可信包名当 shell 命令。

### Step 3：观察生效而非推测

resourcepack/texturepack：等待重载完成，核对 Selected/当前包、顺序与实际可见变化/无重载错误。Shader：核对菜单开关/选中名、渲染画面和错误日志；菜单选中但编译失败只报 selected_not_rendered。datapack：重查 `/datapack list enabled`/available，核对目标ID在期望集合、reload无错误，再按该包明确的非破坏验证方式观察功能；enabled 列表通过不代表包全部逻辑正确。没有截图/命令实际输出只能报告 player_action_pending，不捏造观察结果。

### Step 4：持久化有证据状态

记录玩家确认或工具实际观察的证据与局限；要保存 sidecar 激活证据时待游戏正常退出后运行一次写入门禁及提交与恢复，返回本 Step 读回核对。游戏仍运行可先返回游戏侧成功、record_pending，不能为写配置谎称游戏停止。content.enabled 保持磁盘可扫描状态，不随菜单停用而改写为磁盘隔离；sidecar 激活证据表示最近观察，不保证下一次运行保持。失败说明选中/渲染/命令分别到哪一步，不自动停用其他资源“修复”。

## 移除资源文件

### Step 1：选择范围并提示副作用

输入唯一 content ID或明确相对路径；上下文核对 kind/世界。列文件/目录清单，区分卸载文件与游戏侧停用；若想仅停用，转上节，不改后缀伪装。删除数据包不能撤销已修改的世界；玩家需要撤销效果时先要求独立世界恢复计划。

### Step 2：退出并备份

先建议按游戏 UI/commands 停用并取得证据；玩家明确仅删文件可跳过但报告残留选择可能造成警告。写入门禁要求游戏退出并备份目标及将处理的关联 Shader `.txt`；datapack 风险下建议完整世界备份，未授权不扩大删除。目录包只移除已验证授权的该目录，不跟随链接，不删除整个 resourcepacks/shaderpacks/datapacks。

### Step 3：提交并验证

执行提交与恢复，移走对应文件/目录、sidecar、条目；Shader 配置默认保留，除非计划明确包括。读回路径已不存在、其他包/世界未变、备份可校验；记录 file_removed，不声称游戏状态已刷新。若仍需游戏侧验证，释放写锁后返回“游戏内启用与停用” Step 1，不递归持锁执行或备份。

## 协议与固定源码依据

API 使用 [Modrinth 搜索](https://docs.modrinth.com/api/operations/searchprojects/)、[版本/文件](https://docs.modrinth.com/api/operations/getprojectversions/)、[loader 标签及支持类型](https://docs.modrinth.com/api/operations/loaderlist/)，HTTP/JSON细节见 [Mod 版本协议](mods.md#选择版本与依赖)。游戏侧可复核资料：[资源包](https://minecraft.wiki/w/Resource_pack)、[旧 Java 材质包](https://minecraft.wiki/w/Texture_pack_(Java_Edition))、[datapack命令与权限](https://minecraft.wiki/w/Commands/datapack)。根据实际 MC版本核对，不以最新资料保证旧版语法。

固定源码基线 `899d2f3957215a09fe398a8a8cbb8a8a9c619968`：
- [按资源类型路由和四类下载入口](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/ui/dialogs/ResourceDownloadDialog.cpp#L252-L287)、[各 provider 页面](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/ui/dialogs/ResourceDownloadDialog.cpp#L407-L488)。
- [Shader 更新入口](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/ui/pages/instance/ShaderPackPage.cpp#L48-L77)、[Shader旧配置迁移](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/ResourceDownloadTask.cpp#L117-L127)。
- [世界内 datapacks](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/ui/pages/instance/WorldListPage.cpp#L276-L304)、[全局页面设置门槛](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/ui/pages/instance/DataPackPage.cpp#L290-L292)。Prism文件管理能力不等于自动选中资源包/安装渲染器；本节游戏内流程是显式 Skill行为，不冒充上游自动完成。
