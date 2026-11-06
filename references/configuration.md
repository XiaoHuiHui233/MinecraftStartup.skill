# 配置读写

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

执行前遵循 [玩家运行能力契约](../SKILL.md#玩家运行能力契约)：只用已有专用工具或目标 OS 原生命令；缺能力 BLOCKED，不安装额外编程运行时/库（游戏 Java/材料下载例外）、不写临时脚本、不用 Computer Use 或 GUI 自动截图。游戏内操作和视觉验收由玩家完成确认。

配置路径：`<当前用户主目录>/.minecraftstartup/config.json`。以下流程是所有模块的共同子流程。

## 读取配置

### Step 1：定位文件

用宿主取得当前用户主目录，拼接固定路径。不存在返回 `MISSING`，由调用方进入首次初始化；不要扫描其他用户或项目寻找配置。

### Step 2：读取并解析

读取完整文件和原始字节/指纹，用 JSON 工具解析。语法错误时保留原文件，报告 `BLOCKED: CONFIG_INVALID`，询问修复或从明确备份恢复；不默默清空。

### Step 3：检查字段

检查 `dataRoot` 是绝对路径、`folders` 的五个值是安全单层目录名、Java/账号/实例列表为数组、非空当前 ID 能匹配记录且 ID 无重复。显式无效值不得当作缺省值。旧配置只缺新增字段时，在内存视图中补模板缺省，不覆盖已有值；只读操作只报告缺省，不为迁移自动写盘。下一次授权写操作才通过保存流程补字段。

instances.retiredIds 缺省 []，须为唯一合法 ID 数组且不能与当前实例记录重叠；仅恢复原身份可在同一配置提交中从墓碑移回活动列表。新增 management/content/backups/operations 字段只补模板缺省；验证 management.network.offline 是布尔值、historyLimit 是正整数，API base 为合法 HTTPS URL、备份目录安全地落在 dataRoot 内且不在实例范围内。各 entries 必须数组，ID 唯一，instanceId 引用既有实例（已删除实例的备份与历史 operation 可保留并标 orphan），content.relativePath 为 gameRoot 内安全相对路径，backup/archive/journal 引用为 dataRoot 内安全相对路径；provider/状态取 [公共 schema](operations.md#新增持久字段) 的枚举。外部 credentialRef 仅是非秘密句柄。缺失非必需新字段可迁移，显式类型错误不能静默重置。

实例可选 `memory.maxMiB` 为正整数，缺省不设置 Xmx；`recommendedRamMiB` 只存在导入计划建议中，采用后才保存 memory。实例完整记录已在缓存固定投影内，无需将内存放到投影外。新增 pack-state/loader 收据是实例 sidecar，不把 API key/实际 argv/测试结果加进全局配置。

`javaId` 失效返回需重新选 Java，不删除实例。缺少 `ignoreJavaCompatibility` 按 `false`。目录不存在返回 `ROOT_UNAVAILABLE`，交本地文件模块让玩家确认，不建立同名空目录伪装原数据。

### Step 4：返回结果

返回配置、磁盘指纹、可用路径和缺失项。不得输出可能含秘密的未知字段全文。读取本身不代表历史账号/进程/Java 状态仍有效。

## 保存配置

### Step 1：重读并合并

先独占创建配置同目录的 `config.lock`，序列化所有遵守本 Skill 的写者；锁存在不得覆盖或仅因超时删除，确认原任务退出后才询问清理。持锁直到 Step 5 结束，失败也释放自己拥有的锁；无可靠互斥工具返回 BLOCKED: CONFIG_LOCK_UNAVAILABLE。然后重读当前正式文件；调用方已提供 operation/journal 时复用；单独配置操作无上下文则用真实 UUID 创建配置同目录 `.transactions/<opId>.json`，仅记录非秘密负责字段的 before/after 与提交指纹，不能因没有实例而省略提交阶段记录。日志不可创建就返回 BLOCKED: CONFIG_JOURNAL_UNAVAILABLE，尚未替换正式文件。只合并本操作负责的字段，保留其他/未知字段。首次创建仅在玩家确认根目录后使用模板。若本操作依据的目标记录已被其他任务修改，返回 `BLOCKED: CONFIG_CONFLICT`，重新核对，不用旧列表覆盖新列表。

### Step 2：验证候选配置

用读取流程的字段规则检查。核对当前 ID 引用、账号/实例/Java ID 唯一性、版本检查豁免范围。禁止写入密码、令牌、设备码、实际含秘密 argv 或虚构检测结果。

### Step 3：写入临时文件

在配置文件同目录创建任务唯一的临时文件，不覆盖其他任务临时文件。序列化完整合法 JSON，刷新写入；系统支持时设置仅当前用户读写。写失败返回 `BLOCKED: CONFIG_WRITE_FAILED`，旧配置保留。

### Step 4：提交

先向本操作全局 journal 记录 configBeforeFingerprint、configCandidateFingerprint 和仅本操作负责的非秘密字段 before/after image（含记录删除/新增标记），状态 configCommit=before_replace；正式文件与 Step 1 指纹一致才原子替换，返回后记 configCommit=committed；首次创建使用防覆盖方式。崩溃或替换结果未知时为 unknown，不能仅靠 journal 推断替换未发生。并发冲突不继续替换。清理本任务临时文件。无等效安全写入工具时停止，不能宣称保存成功。

### Step 5：重新读取验证

重新读取正式文件，核对改动字段与候选相同、其他字段未丢失。验证成功得到 commitState=committed；释放本层 config.lock 后，必须执行 [配置变更后刷新缓存](launch-cache.md#配置变更后刷新缓存)，对每个受影响实例保存/更新 Java argv 缓存，不限 selectedId。启动输入未变（如 lastLaunch/退出状态保存）复用已有缓存，不递归保存配置；缺前提则保留具体 stale 原因。缓存刷新另记结果，不能将其失败误报成配置未提交，完成判定后返回调用方。替换前失败返回 commitState=not_committed。替换后读回失败/不能确定则返回 BLOCKED: CONFIG_READBACK_FAILED 和 commitState=committed/unknown，保留指纹与 before/after image，释放自己的 config.lock；调用方必须执行 [协调配置与文件恢复](operations.md#协调配置与文件恢复)，禁止直接仅回退目录。不一致不能盲写。

## 分配稳定 ID

### Step 1：收集

收集目标列表已有 ID；实例还必须收集 instances.retiredIds、backups.entries 和 operations.entries 中所有 instanceId，以及 instances/ 下已有目录名。已删除身份不能因历史记录裁剪而复用，墓碑永久保留；仅明确恢复同一历史实例时可复原其身份，不能让新实例绑定旧备份。

### Step 2：选择

op/content/backup 使用真实 UUID 工具生成 `op-<hex>`、`content-<hex>`、`backup-<hex>`，提交前同样查重。Java/离线账号/实例分别使用 `java-001`、`offline-001`、`instance-001` 形式，从 001 起选第一个未占用编号；使用工具计算。微软账号使用 `microsoft-<角色UUID无连字符>`。

### Step 3：复核

提交前再查冲突。ID 只能含字母、数字、连字符或下划线，不能含路径分隔符；不能用显示名作路径。变更显示名不变 ID。
