# 独立备份与恢复

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

输入：实例 ID、kind（instance/files/world）、明确的实例根相对路径列表；无范围时默认 instance。备份是独立字节副本，不用软/硬链接。实例备份包括实例设置/组件和 .minecraft（含存档），不复制全局账号凭据、系统 Java 或整个 dataRoot；因此恢复后仍需选择可用账号和 Java。

## 查看备份

### Step 1：读取清单

只读配置 backups.entries，按显式实例/备份 ID 筛选；无配置/无记录返回空。显示 ID、时间、类型、相对范围、verified 及归档实际是否存在，不创建目录、锁、备份或登录。

### Step 2：检查引用

路径按 dataRoot 解析，拒绝根外路径。列表中的 verified 是上次结果；未重新验文件时明确显示未复验，缺失标 missing，不删除记录。返回 DONE 的备份清单，不把记录存在当可恢复。

## 创建备份

### Step 1：解析范围和运行状态

用 [公共输入](operations.md#输入与结果约定) 确定实例；sourceRelativePaths 均相对 instanceRoot。world 范围必须是 `.minecraft/saves/<唯一世界目录>`；files 为已批准文件/目录列表；instance 是整个 instanceRoot。执行路径/停止检查，但不要调用包含本流程的完整写入门禁造成递归。上游已持同一 operation 父级稳定锁/实例锁则复用；独立备份必须先独占 `<instancesRoot>/.management-publish-<instanceId>.lock`，再取 `<instanceRoot>/.management/lock` 并复核父锁所有权，整个备份持有两锁，结束只释放自己拥有者。不能只检查父锁不存在或仅取实例内锁。游戏运行/状态不明时阻断，不把运行中 ZIP 称一致备份。

### Step 2：建立真实快照清单

用宿主递归列举 regular files 和目录，不跟随链接；拒绝 symlink/reparse point、socket/设备、逃逸、不可读文件。排除 `.management`（包含锁、staging、日志）；不自动排除 saves、config、mods。逐个记录 `path/size/sha256` 及所需空目录，path 使用 `/`，相对 instanceRoot，无绝对路径或 ..。每次读取前后复核 stat/内容变化；变化返回 BLOCKED: BACKUP_SOURCE_CHANGED。空间检查用真实磁盘工具，不能预估成已满足。

### Step 3：写入独立归档

以真实 UUID 生成 backup-<hex>，目录取 management.backup.relativeDirectory（缺省 downloads/backups，位于 dataRoot 内且不在被备份实例内）。先以独占创建写 `.partial`，每源文件复制真实内容到 `payload/<path>`，空目录写相应 ZIP directory。内置 `backup-manifest.json`：`formatVersion:1,instanceId,kind,createdAt,sourceRelativePaths,minecraftVersion,loader,files:[{path,size,sha256}],directories:[]`；可选 Java 引用只是元数据，不含凭据。禁止外部程序/网络上传。无可靠 ZIP 工具返回 BLOCKED，不生成空包冒充备份。

### Step 4：独立验证归档

关闭后重开 ZIP，执行 CRC/testzip，拒绝重复/穿越/链接成员，比较 payload 名单与 manifest 完全一致；逐成员流式重算 size/sha256 与清单比较，再复核源内容尚未变化。把无歧义 manifest 字节写同名 `.manifest.json` sidecar，记录其 sha256 和最终归档 sha256。验证失败保留隔离 partial，不设 verified。备份文件总量超过解压默认限额时，必须先明确授权并保存可复验的限额，不在恢复时悄悄跳过限制。

### Step 5：提交和登记

将本任务完整归档和 sidecar 提交至唯一最终文件名，不覆盖已有文件；按 [保存配置](configuration.md#保存配置) 追加真实 `backups.entries`（id/instanceId/kind/archivePath/manifestPath/archiveSha256/manifestSha256/createdAt/verified=true/sourceRelativePaths，路径均相对 dataRoot）。重读归档、sidecar 与记录一致才 DONE。作为父操作子流程返回 backupId 及指纹，不释放父锁、不另备份、不重新确认同一备份范围；单独请求报告位置。只表示材料校验通过，未完成恢复演练。

## 校验备份

### Step 1：定位与检查边界

输入 backup ID 或玩家明确指定的本地备份；不以文件名猜实例。已登记包验证 archiveSha256/manifestSha256；外部备份无可信登记时标“外部来源未认证”，先查看元数据，未知 formatVersion 返回 BLOCKED: BACKUP_FORMAT_UNSUPPORTED。读取 ZIP 中 manifest 和 sidecar（若有）一致。

### Step 2：逐项校验

按 [创建备份](#创建备份) Step 4 重算成员，不只比 ZIP 文件大小。预检成员数/单文件/总展开量（默认 20,000/2 GiB/8 GiB），超限须明确批准。path 必须合法且同名规范化后唯一；拒绝符号链接、设备与 payload 外未知成员。清单不得包含 .management、根外路径或凭据配置。失败记录 verified=false；普通只读检查不改其他状态。

### Step 3：返回可恢复范围

列可恢复文件范围、实例/MC/loader、缺失项和验证结果。与实际目标不兼容时仍可报告归档完整，但必须阻止直接恢复到错误实例。不执行归档内文件。重新验证成功可用配置保存更新 verified，不宣称已恢复。

## 恢复备份

### Step 1：确定目标和破坏范围

先执行校验备份。确认目标实例 ID；默认恢复原实例，也可经明确选择恢复到新稳定 ID，不复用未知目录。输出将恢复/覆盖/移除的路径、游戏版本和存档；world/files 只影响清单范围，instance 全量恢复会替换目标实例内容。对每个目录范围，比较当前完整路径集合与快照，明确列出备份后新增、将被移除的文件；这不是覆盖部分文件而保留未知新增文件。原身份已删除时只能经明确选择恢复该身份，并在配置提交时移除其墓碑；若该 ID 已占用且不能证明为同一历史实例，必须阻断，不能默认覆盖。未经明确恢复确认返回 WAIT_INPUT。

### Step 2：停止并保留现状

执行公共路径与运行门禁，取得/复用唯一锁；存在目标时先创建当前范围的独立恢复前备份（kind 与范围对应，独立调用本文件创建流程，复用锁）。读取原指纹与配置。即使旧备份已完整，也不能省略本次恢复前备份；空新目标无需备份。无法确认停机、空间或备份就 BLOCKED。

### Step 3：隔离展开

在实例父目录的独占 `.restore-<opId>` 展开 payload，用 validated paths 写 regular files；全过程限额并逐文件验 hash，不能只靠 ZIP 声明大小。instance 恢复 staging 构成完整实例；files/world 恢复先以目标真实复制构成 staging；对已确认的目录范围，在 staging 内清空该整个范围，再严格按 manifest 重建文件和空目录，不能遗留快照外新增 region/data 等文件。单文件范围只替换该文件；范围外数据保持不变。验收需比较范围内完整路径集合及所有 hash。拒绝 links，禁止运行归档中的 scripts/hooks。不直接展开到正在使用的目录。

### Step 4：提交并处理失败

重新确认停止和指纹，journal 记录每次移动。把现有实例移到同文件系统的独占 `.restore-old-<opId>`，再把 staging 移到目标；两次移动不是一个原子事务，中途失败用 journal 复位原目录。旧目录无法安全保留则停止，不先删旧实例腾位置。实例 `.management/lock` 随旧目录移动时，继续持有开始操作时已独占取得的 `<instancesRoot>/.management-publish-<instanceId>.lock` 父级稳定操作锁，覆盖整个交换窗口及新目录锁建立，不重复申请、不在移动间隙释放；其他管理流程发现父级发布锁必须等待。不能支持这种互斥的宿主进入 BLOCKED: RESTORE_LOCK_UNAVAILABLE。

### Step 5：保存和验收

保留本地稳定 ID，按归档中真实 MC/loader 与目标文件更新记录；恢复到新 ID 时重写目录关联，不照搬归档的 selectedId/账号秘密。清除旧 Java major 豁免，重新解析/验证 Java 与材料，缺失 Java 只标待选择不假报可启动。content.entries 只重建可验证的目标范围，失效来源标 local/unknown，不篡改其他实例。配置保存/读回失败或 hash 不符，停止并保留旧目录/恢复前备份，先执行 [协调配置与文件恢复](operations.md#协调配置与文件恢复) 核实磁盘配置；不能在配置已改 MC/loader 后仅恢复旧目录而不补偿配置；成功逐文件验收后释放自己的锁，旧目录先保留并告知，需玩家批准才清理。不自动启动世界验证，DONE 只表示文件恢复。

## 删除备份

### Step 1：明确备份

定位唯一 backup ID，显示归档、sidecar、范围与是否是未完成操作的恢复依据。被 applying/recovery_required 操作引用时阻断；不自动按 historyLimit 清理备份。

### Step 2：确认并移除

玩家明确确认丢失该恢复点后，仅删除/回收对应归档和 sidecar；复核二者是允许范围内原文件，不跟随链接。不递归删整个 backups 目录；部分失败保留记录并说明。

### Step 3：保存与复核

文件均按授权移除后才移除单条 backups.entries，读回验证其他备份仍在。DONE 报实际移除；无法回收不擅自扩大为永久删除。

## 归档路径检查表达式

宿主可在一次性 Python 任务中复用下列相对路径校验；它不替代目标根的 resolve/链接检查、成员类型/数量/展开量限制或 hash 验证。归档清单必须在写任何成员前完整预检；seen 检查同时用于目录与文件，另检查文件/目录前缀冲突，例如 `a` 是文件就不能再有 `a/b`。

```python
import re
import unicodedata

def member_key(name):
    if not isinstance(name, str) or not name or "\\" in name:
        raise ValueError("unsafe archive path")
    path = name[:-1] if name.endswith("/") else name
    parts = path.split("/")
    reserved = {"CON", "PRN", "AUX", "NUL"} | {
        prefix + str(i) for prefix in ("COM", "LPT") for i in range(1, 10)
    }
    for part in parts:
        if (part in ("", ".", "..") or part.endswith((".", " "))
                or re.search(r'[\x00-\x1f<>:"|?*]', part)
                or part.split(".")[0].upper() in reserved
                or part.casefold() == ".management"):
            raise ValueError("unsafe archive path")
    return unicodedata.normalize("NFC", path).casefold()
```

同名键冲突一律拒绝（采用跨平台保守规则，即使当前文件系统区分大小写）；归档仅 `payload/…` 和受支持 manifest 可进入本备份恢复器，外部 mrpack/实例 ZIP 走各自导入流程，不混用格式。

## 边界

备份是本地独立快照，不是云同步、发布或自动灾备。实例配置、内容、存档可能含隐私，默认不外发。基于 Prism 的复制/导出思路但明确拒绝将链接副本当独立备份：上游 [InstanceCopyTask](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/InstanceCopyTask.cpp)。本项目备份 manifest 是自身约定，不冒称 Prism 格式。
