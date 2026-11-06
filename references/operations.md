# 管理操作公共契约

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

执行前遵循 [玩家运行能力契约](../SKILL.md#玩家运行能力契约)：只用已有专用工具或目标 OS 原生命令；缺能力 BLOCKED，不安装额外编程运行时/库（游戏 Java/材料下载例外）、不写临时脚本、不用 Computer Use 或 GUI 自动截图。游戏内操作和视觉验收由玩家完成确认。

本文件是管理模块共用子流程，不是独立启动器。用宿主已有结构化文件/HTTP/归档/进程工具或经核实的目标 OS 原生命令执行；不要求安装 Prism，不在项目内生成运行时代码。任何模块不能用“调用管理器”代替具体步骤。所有路径相对当前玩家配置解析，不引用开发机器路径。

## 按模块探测能力

在调用方读取真实输入后、任何依赖动作前逐项探测；下表是接口验收契约，不是工具注册表或命令别名。记录真实工具名称/版本、可接受输入、限额和已核实能力；不要把“有 terminal/HTTP”推定为满足全部能力。缺项只阻断依赖该项的步骤，保留 partial/unknown 与具体原因，禁止自写代码/安装依赖补齐。

| 模块 | 真实输入与必须已有的能力 | 缺项边界 |
|---|---|---|
| 目录/配置 | 当前用户主目录、明确 dataRoot、实际 JSON；路径/链接检查、结构化合并、UUID、独占锁/暂存、原子替换/读回 | 查询可只读；无可靠写入/锁则不初始化或提交 |
| Java | PATH 全部目录/手动路径、实测 OS/架构；受控 argv 执行版本探测 | 可按授权走 Java 下载，不自写 helper，不把版本不符当不可执行 |
| 实例/loader/Prism/profile | 精确 ID/MC/loader、真实组件字节及索引 hash；JSON、hash、受限依赖/版本规则处理、文件隔离、参数模板分词 | 未知版本比较/trait/processor 阻断；不能用语言运行时补实现或静默换 Mojang 源 |
| 下载/Java 安装 | metadata URL/大小/hash、目标根及 OS；安全 HTTP、流式 hash、受限归档、权限/安装目录内链接检查、进程探测 | 普通下载成功不代表安装；缺归档/权限工具不注册 available |
| 账号/微软 | 确认名称或配置 client ID；离线精确 MD5 UUID、微软普通 HTTP 请求/轮询、系统凭据读写和参数传递 | 不要求专用安全编排或私密注入；仅实际工具/服务失败时报告，不索取聊天 token、不默转离线 |
| Mod/资源 | 真实 JAR/ZIP/目录、MC/loader、API 返回版本；有界归档、JSON/TOML、版本约束/hash/安全提交 | 不支持格式标 unknown/blocked，不按文件名猜；游戏激活由玩家操作确认 |
| 整合包/备份 | 真实清单及源树、冻结路径/hash；整合包走 [Prism 式归档](prism-archive.md)，备份保持自身严格清单契约；路径范围/实际冲突检查、独立复制、三方差异/事务 | 缺安全归档、差异或互斥即阻断依赖操作，不用空包/覆盖混合伪装恢复 |
| 世界/服务器 NBT | 实际 level.dat/servers.dat、锁后 SHA-256；专用有界 typed NBT 解析/无损修改/读回比较 | 缺解析可列文件候选但不称有效清单；修改需完整无损能力，绝不手拼格式 |
| 服务器状态 | 明确 host/port、授权 DNS/IP；已有 DNS 与 Java SLP 工具，固定审定 IP 并保留握手 host | HTTP/端口开放不等于 SLP；无协议/地址固定能力则 BLOCKED |
| 已有截图 | 真实 PNG 路径/hash；文件能力，按操作另需有界图像元信息/解码/专用展示及安全 multipart HTTP | 不抓屏；无解码不称 decoded，无预览/分享验证能力不称完成，外发先确认 |
| 启动/停止/诊断 | 真实 profile/Java/文件及 taskId/PID/开始时间；普通后台进程、本地日志重定向、只读进程身份/日志 | 不因缺私密 argv 或流式脱敏接口阻断启动/读取；菜单/世界/服务器状态等玩家确认，不做 GUI 兜底 |

## 输入与结果约定

### Step 1：解析操作

提取 action、显式实例 ID/唯一名称、资源类型/路径/项目、版本及玩家约束；操作不明确返回 WAIT_INPUT。只读查询不触发登录、初始化数据目录、下载、备份或启动。读取 [配置](configuration.md#读取配置)；不存在时只读查询返回空/未初始化，需要写入才返回主入口初始化。

### Step 2：解析作用域

实例操作复用 [实例选择](instances.md#查看与选择) 的定位规则，但纯查询不改变 selectedId；显式实例优先，其次已有选择，最后唯一实例，否则询问。返回 instanceRoot、gameRoot、minecraftVersion、loader 和配置指纹；缺失路径进入 BLOCKED，不创建空目录冒充原实例。全局文件库操作只定位配置根 mods，不能把它当所有实例共享运行目录。

### Step 3：返回统一上下文

返回 operation/action、目标稳定 ID、已解析路径、输入指纹和授权范围；可用工具能力缺失返回 BLOCKED: CAPABILITY_MISSING 并说清具体能力。DONE 只表示调用方指定结果验收通过；下载完成、文件落盘、游戏内生效分开报告。

## 写入前门禁

### Step 1：验证路径

对源和目标规范化并 resolve 所有已存在的祖先；用路径组件包含关系而非字符串 startswith 判断是否落在允许根内。拒绝 ..、绝对归档成员、驱动器/UNC 逃逸、符号链接/junction/reparse-point 逃逸及特殊设备。目标是玩家明确授权的实例、备份目录或导出文件；不写另一实例。对原地覆盖的已有文件检查 hardlink 数，存在共享链接则停止请求先解除耦合，不能改动其他实例。

### Step 2：确认停止

凡写运行时内容、world、servers.dat、版本/loader、恢复、整包更新、删除实例，均确认目标实例没有在运行。综合当前受管任务、实际 Java/游戏进程与其 cwd/参数指向，不只读历史 lastLaunch 或进程名；PID 必须复核身份，不能因复用而杀错进程。状态无法确认返回 BLOCKED: RUN_STATE_UNKNOWN。不要擅自停止，玩家明确要求停止才调用 [停止游戏](launch.md#停止游戏)。只读清单与文件导出也注明运行中快照可能不一致，不能称可恢复备份。

### Step 3：形成计划并确认

列出将新增/覆盖/移除的具体相对路径、来源、版本和外发目标。玩家明确的单项安装/禁用/删除请求覆盖该单项目标；依赖扩大、批量删除、覆盖冲突、世界删除、整包替换/恢复、外部程序执行、换源、外发必须先确认。未选定 provider/项目/版本不能把搜索意图当安装许可。完整实例创建确认继续沿用原有“确认即游戏下载”规则，不重复问同范围授权；换源确认的唯一默认例外是 [CF 精确同字节兜底](curseforge-import.md#解析与恢复)，仅允许官方 URL 缺失/下载受限项经 Modrinth 官方 SHA-1 查询且通过 CF 权威 SHA-1+size、合法公开 URL 和分发限制检查，不另问同范围来源授权。

### Step 4：锁定并备份

所有写入、一致性备份、复制和恢复都先以独占创建取得 `<instancesRoot>/.management-publish-<instanceId>.lock` 父级稳定操作锁，并持有到整个操作验收/安全退出；不是“看一眼不存在”后放行。已有锁仅同一 operation/nonce 的子流程可复用，否则等待或 BLOCKED。多实例操作按稳定 ID 字典序取得所有父锁，防止反向死锁。取得父锁后才在 `<instanceRoot>/.management/lock` 独占建实例锁；取得实例锁后再验证父锁仍由本 operation/nonce 持有，否则释放自己刚建的实例锁并退出。实例目录移动不移动父锁；恢复重建新目录锁前也不释放父锁；用真实 UUID 分配 operation ID，在 dataRoot 下创建 `.management/journals/<opId>.json`（不是 instanceRoot，journalPath 为此根相对路径，不能随实例删除/恢复消失），记录计划、目标旧指纹及每项阶段，登记 operations.entries 后才改文件，不写 token。已有锁时检查拥有任务，不因时间旧就自动删；确认任务已退出后才经玩家确认清理。锁只互斥遵守本协议的 Agent，不能约束 Minecraft 或外部编辑器，因此提交前仍复核进程与文件指纹。备份覆盖/删除前的文件，执行 [创建备份](backups.md#创建备份)，首次全新文件可以只保留下载暂存。备份子流程复用当前 operation/锁，禁止递归申请锁或备份；备份失败就停止原操作。

### Step 5：返回许可

返回已确认计划、锁、备份 ID 和目标指纹。之后只执行此计划；发现目标/版本/来源变化先停止并重新确认，不扩大授权。

## 提交与恢复

### Step 1：写入暂存并验证

在目标同文件系统的 operation 专属 staging 写入，禁止直接截断原文件。按格式验证 JSON/NBT/JAR/ZIP 与预期 hash；来源 hash 和本地快照 hash 分开记录。新下载调用 [单文件下载](downloads.md#单文件下载)，包解压限制见 [整合包管理](modpacks.md#导入整合包)。不自动执行归档中的 installer、脚本、钩子或 wrapper；独立取得且经 metadata 固定的官方 loader 链按 [现代 Forge/NeoForge](forge-install.md) 确认后执行，不使用整合包捎带的二进制替代。

### Step 2：复核并提交

重读当前配置及所有将覆盖文件的指纹，重新检查游戏已停止。冲突返回 BLOCKED: TARGET_CHANGED，保留原文件与本操作 staging。同目录单文件替换可用原子 rename/replace；多文件操作不是整体原子事务，逐项记录 planned/applied/verified/failed 及旧/新 hash，不宣称全包原子成功。

### Step 3：持久化

用 [保存配置](configuration.md#保存配置) 合并本操作负责记录，不替换未知字段或其他任务的整个列表。重新扫描文件后登记真实状态；文件已改而保存/读回失败时报告 BLOCKED: CONFIG_WRITE_FAILED 或 CONFIG_READBACK_FAILED 和实际变化，保留恢复记录，先执行下方协调配置与文件恢复，不报成功、不先单独回退文件。content.entries 仅管理已识别/主动登记内容，不把缺记录当文件不存在。

### Step 4：异常恢复

部分提交失败立即停止；用操作日志和备份生成逐文件回退计划。目标在提交后又被外部修改则不得覆盖，返回 BLOCKED: RECOVERY_CONFLICT。未变更的已提交文件可在原授权的失败回退范围恢复并重新验 hash；完整跨版本恢复另走 [恢复备份](backups.md#恢复备份)。保留失败记录和备份，不能清 staging 掩盖损坏。

### Step 5：验证与收尾

读回配置与实际文件，验证计划的全部目标和原数据保留范围；凡启动材料、Mod/整合包、路径、复制/恢复等写入，即使未修改全局配置，也须按 [配置变更后刷新缓存](launch-cache.md#配置变更后刷新缓存) 更新每个受影响实例的 Java 指令缓存（复用本 operation 的锁，不递归保存配置），材料未就绪记 stale 原因，不沿用旧指令。释放本操作拥有的锁，不清其他任务文件。成功才 DONE，报告文件级结果，不把无启动验证的 Mod/loader/Shader 变更称为游戏兼容。失败如实报告阶段、已应用部分、备份与恢复入口。

## 协调配置与文件恢复

### Step 1：冻结并分类

继续持有父级稳定锁，停止清理隔离目录/staging/备份。配置保存返回 not_committed/committed/unknown；已 replace 后读回异常不代表没有提交。读取全局 journal 的负责字段 before/after image、旧/候选指纹与文件阶段，不输出未知/秘密原配置。

### Step 2：读回实际配置

重新以 config.lock 互斥读取正式文件，对比本操作负责字段及 ID 引用。仍无法读取/解析、或目标字段既非 before 也非 after，则记 recovery_required，保留两份目录和备份并阻断，不猜测、不覆盖其他任务配置。全文件 hash 变化可由无关字段引起，补偿只比较负责字段，不能用旧全文件覆盖新全文件。

### Step 3：决定完成还是补偿

负责字段已是 after 且全部目标文件与计划验证一致时，完成本次提交的读回验证，不回退目录。负责字段仍是 before 时，视为未提交，仅对本次已改且未再被外部修改的文件按 journal 安全复位。已是 after 但文件未完整或确需失败回退时，先确认负责字段仍等于 after，再用配置保存流程仅补偿到 before（恢复实例记录/content/selectedId/retiredIds/managedPack 等本次实际改动），保留其他字段和任务结果，补偿读回成功才进入文件复位。

### Step 4：复位文件并再核对

仅对可验证仍属本次写集的文件/隔离目录复位；冲突或无法复位就保留材料，标 recovery_required，不伪称全体回滚。补偿配置和文件分两步，不是全局原子事务；中断后再次读取 journal 和实际配置从本流程继续。任何恢复中的写入仍受停机/稳定锁保护。

### Step 5：终结

再次核对负责字段、目录身份、全部目标 hash 与墓碑是否配套。无法确认就保留稳定锁所属记录与恢复入口，不清理原/新材料；明确停止后如何由玩家选择继续恢复。确认一致才解除本 operation 锁并报告完成提交或已回退的真实范围，不能把 CONFIG_READBACK_FAILED 直接当作“可以撤回文件”。

## 网络与来源

### Step 1：检查策略

新增管理操作读取 management.network.offline（缺省 false）；true 时任何搜索、版本、哈希识别、状态 ping、下载、上传均停止网络子步骤并返回 BLOCKED: OFFLINE_POLICY。允许本地列举/复制，不能静默联网。已有登录、Java、启动流程在调用网络子流程前也必须执行本检查；“离线账号”与此全流程禁联网策略不是同一概念。

### Step 2：选择 provider

Modrinth 默认 `https://api.modrinth.com/v2`；CurseForge 官方认证 API 使用本项目 credentialRef 或 [已授权 Prism 应用 Key](curseforge-import.md#已授权-prism-应用-key) 的部署来源，不强制玩家申请；显式 offline/禁用选择仍须遵守，不把未配置的旧默认值误判为必须申请个人 Key；默认采用 Prism 官方 API/官方下载 URL/权威 hash 链路；缺应用凭据先说明授权缺口，不默认切换来源。玩家另行选择第三方适配时，无凭据的本地 CF 清单可经明确确认后使用 [CFWidget + Curse Maven](curseforge-public-import.md)，不受官方 API enabled 开关误阻断，也不自动启用该开关。不能读取其他启动器凭据或把 API key 写普通配置。其他整包 provider 按 [整合包边界](modpacks.md#其他平台与受限文件)；来源切换须玩家批准，但 [CF 精确同字节兜底](curseforge-import.md#解析与恢复) 默认允许：仅官方下载 URL 缺失/下载受限项可用 Modrinth 官方 SHA-1 查询，要求 CF 权威 SHA-1+size 完全一致、合法公开 URL、CF 身份与实际来源分开记录，不重复询问同范围授权；不适用于任意镜像、换版本、撤回或已知禁止分发限制。显式 offline/禁用/拒绝仍优先；CFWidget/Curse Maven 的来源和验证等级确认不变。HTTP User-Agent 标明 MinecraftStartup 与可用的项目联系地址，不伪装 Prism 身份。

### Step 3：执行请求

API 请求保持 TLS，校验重定向及解析目标，拒绝未授权本机/私网地址和凭据跨域转发；自建内网服务只在明确授权目标时使用。请求 60 秒超时、最多 3 次，429 遵循 Retry-After，5xx/连接失败退避，其他 4xx 停止并分类。分页按实际服务游标/offset 继续，不用一页结果说全部。下载复用单文件流程；网络返回不是落盘成功。

### Step 4：返回证据

返回真实 provider、请求筛选、状态、结果数/分页信息、来源 hash 类型和可操作候选。API schema 不匹配或服务权限缺失就 BLOCKED，保留已读内容，不猜字段、不绕过受限分发。

## 新增持久字段

下列是配置 schema，不是额外操作流程：旧配置只补缺省，不覆盖玩家值；显式类型错误必须阻断。

- `management.network.offline: false`：所有网络步骤共用；`management.providers.modrinth.apiBaseUrl`、`management.providers.curseforge.{enabled,apiBaseUrl,credentialRef}`；后者缺省禁用。
- `management.backup.relativeDirectory: "downloads/backups"`：相对 dataRoot 的受控目录；绝不自动清理。
- `management.historyLimit: 50`：仅限制已结束的 operations 记录；未结束/失败待恢复的记录不能自动裁掉，不删除真实备份。
- `content.entries: []`：每项 `id/instanceId/kind/relativePath/provider/projectId/versionId/hashes/enabled/updateLocked`。kind 为 mod/resourcepack/texturepack/shader/datapack；provider 为 local/modrinth/curseforge，未知来源不能伪造平台身份。路径相对 gameRoot，世界数据包含 saves/<世界目录>/datapacks 前缀。本地可算 sha256，用于快照而非上游验证。enabled 为布尔磁盘可扫描/隔离状态，不代表游戏内激活；游戏观察未知用实例 content-state sidecar 标 unknown，不添加 nullable enabled。来源/依赖证据 sidecar 放 instanceRoot/content-state（不是被备份排除的 .management），整实例备份和复制应保留。
- `backups.entries: []`：每项 `id/instanceId/kind/archivePath/manifestPath/archiveSha256/manifestSha256/createdAt/verified/sourceRelativePaths`；kind 为 instance/files/world，archivePath/manifestPath 是 dataRoot 相对路径，verified 只表示恢复材料校验通过，不表示恢复演练已通过。
- `instances.retiredIds: []`：删除身份的永久墓碑；新建 ID 还排除历史备份/operation 的 instanceId，不把旧备份自动绑定到后来新建的实例。
- `operations.entries: []`：每项 `id/instanceId/action/status/createdAt/backupId/journalPath`；status 为 planned/staged/applying/verified/failed/recovery_required，journalPath 为 dataRoot 相对路径，日志只含路径/hash/阶段，不含秘密。op/content/backup ID 用真实 UUID 工具生成 `op-<hex>`、`content-<hex>`、`backup-<hex>`。
- `instances.entries[].managedPack`：可选 `provider/projectId/versionId/manifestPath/manifestSha256`，不能将平台项目 ID 当本地实例 ID；manifestPath 相对 instanceRoot。导入的外部命令只作不可信数据，默认不启用。
