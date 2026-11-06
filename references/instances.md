# 实例管理

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

执行前遵循 [玩家运行能力契约](../SKILL.md#玩家运行能力契约)：只用已有专用工具或目标 OS 原生命令；缺能力 BLOCKED，不安装额外编程运行时/库（游戏 Java/材料下载例外）、不写临时脚本、不用 Computer Use 或 GUI 自动截图。游戏内操作和视觉验收由玩家完成确认。

每个实例使用 `instances/<稳定ID>/.minecraft/`。其中 versions、libraries、assets、mods、config、saves、resourcepacks、shaderpacks、logs 和游戏设置各自独立；不建跨实例的可写软/硬链接。Java 可通过 ID 复用，根目录 mods 只作文件库。

配置 `instances.entries` 每项保存 `id/name/minecraftVersion/loader/javaId/ignoreJavaCompatibility/status`；原版 loader 为 null，Java 未指定为 null，ignoreJavaCompatibility 默认 false；creationConfirmed 记录创建选择确认，downloadOnCreate 默认 true。当前选择存 `instances.selectedId`。

## 创建

### Step 1：加载真实版本选项

执行 [Prism 组件版本列表](prism-components.md#读取组件版本列表)，读取 net.minecraft。默认只显示正式版，玩家可选择快照/历史版本筛选。指定版本已给就定位并标选中，不强迫重选；未给则列候选供选择，返回 WAIT_INPUT。

### Step 2：展示 Prism 式创建表单

一次展示：实例名称、Minecraft 版本/类型筛选、加载器、加载器版本；分组/图标可选，默认无分组/默认图标，不强制逐项问。名字未给时建议“MC版本”，带 loader 时建议“MC版本 loader名称”，玩家可覆盖。

加载器选项为无（原版）、Forge、NeoForge、Fabric、Quilt、LiteLoader，但仅将真实元数据可满足所选 MC 的选项显示为可选；不兼容的注明不可用。玩家已明确“原版”则加载器已选无，不再问。没选 loader 时必须让玩家选择，不能静默决定原版。选 loader 后再列真实兼容版本并标推荐，不把“推荐”当已同意该版本。

### Step 3：确认最终选择并确定范围

表单说明“确认创建后自动下载本实例所需游戏文件、依赖库和资源”。玩家补齐最后一个选择或明确提交完整创建请求，就作为创建及这些下载的确认，不再追加“是否允许下载”问题；取消则不创建。更换来源、额外 Java、受限文件/第三方代码执行不在这项默认授权内；[CF 精确同字节兜底](curseforge-import.md#解析与恢复) 有独立的默认允许策略，仅适用于官方 URL 缺失/下载受限且满足 CF 权威 SHA-1+size、合法公开 URL 和分发限制检查的项，不重复询问同范围授权。

### Step 4：建立隔离结构与记录

分配未占用稳定 ID，检查目录位于配置 instances 内、不覆盖未知目录，建立 `<ID>/.minecraft/`。状态 created，保存名称、MC/loader、javaId、ignoreJavaCompatibility=false、creationConfirmed=true、downloadOnCreate=true；分组/图标有选择则存 group/icon。首个选中，已有选择保留除非玩家要求切换。执行配置保存。

本次已有 created 空实例且玩家是在补齐其选择时，复用已确认的该实例 ID 和目录，不额外建第二个；保存新的明确选择。旧记录 created 且缺 creationConfirmed 时先回到本表单，不能把历史默认 loader 当玩家已选。

### Step 5：解析 Prism 组件

执行 [组件解析与合成](prism-components.md#解析并合成实例组件)，写 mmc-pack.json、已校验元数据及 launch-profile.json。不能默认换成 Mojang 原始 JSON 链路。需要下载规划架构时，优先实例 Java 的实测架构，否则唯一已检测可执行 Java；多候选/无可执行 Java 时询问运行架构或选择 Java。这里只用于材料筛选，不表示豁免 Java major 门禁。

### Step 6：自动下载

立即执行 [Prism 更新任务](prism-components.md#执行更新任务)，按 Folders → Libraries → 必需时 LegacyFML → Assets 的顺序下载并校验。没有明确异常不要再向玩家请求一次同范围下载许可。游戏材料下载不要求先登录账号，单独创建实例不强制启动游戏。

### Step 7：保存状态并验证

材料齐全才设 installed；失败记 download_failed，列失败项/保留缓存，重试不重复创建。重新读取实例配置、组件记录与文件清单，不把“空文件夹存在”当安装完成。

### Step 8：返回调用点

单独创建请求：报告实例已安装/具体失败，不自动启动。来自“启动MC/创建并启动”的请求：返回主启动流程继续账号、Java major 检查和真实进程启动。

## 复制

### Step 1：确定源与目标

用查看与选择定位源，询问新名称（未给默认“原名 副本”）；分配新稳定 ID，目标不能占用已有或未注册目录。默认完整独立复制，不建立软/硬链接，不自动切换当前选择或启动。

### Step 2：冻结源并建清单

执行 [公共路径/运行检查](operations.md#写入前门禁) Step 1～2；源必须停止或明确返回 BLOCKED。按公共契约取得源和目标父级稳定操作锁（稳定 ID 字典序），再取源实例锁，记录所有文件 hash/目录清单，排除 .management，拒绝链接/特殊文件；新目标持发布锁并以独占 staging 创建。不复制全局账号秘密；真实 Java 引用可复用。

### Step 3：复制并验证

以 regular-file 字节复制构建新实例，保留空目录；每目标重算 hash 与源清单比较，再确认源未变。失败只清本任务 staging，保留源；未知冲突不覆盖。路径必须按新实例解析，禁止把源 gameRoot 绝对路径继续当目标。

### Step 4：登记新实例

全部材料验证后才提交独占最终目录。登记新 ID/名称和实际 MC/loader/javaId，清 ignoreJavaCompatibility；内容记录复制时分配新 content ID 并替换 instanceId。源账号/配置及当前选择不变，managedPack 清单仅在可验证且无需重绑绝对路径时保留，否则标非受管来源。失败不能仅创建列表记录冒充复制成功。

### Step 5：保存与返回

执行配置保存、读回与目录 hash 核验，再释放源/目标锁。DONE 报源/目标及独立副本范围，不保证模组/世界在新版本可用。需要仅复制部分数据时先列范围确认，不擅自丢存档。

## 查看与选择

### Step 1：读取列表

读取实例记录，列 ID、名称、MC/loader、Java、状态、实际路径及当前选择。目录缺失显示 missing，不自动建空目录。仅查看/列表时到此返回 DONE（空列表也直接返回），不自动选择唯一实例、不创建、不改 selectedId；启动或显式切换才继续 Step 2。

### Step 2：确定目标

本次明确 ID 优先，唯一名称其次，否则当前 selectedId。无当前且只有一个可选实例时选它；有多个问玩家，没有实例转创建。

### Step 3：核对文件位置

确认目标目录存在且在数据范围内。缺失返回 `BLOCKED: INSTANCE_MISSING`，请玩家恢复/迁移，不冒充新实例。

### Step 4：保存选择

只有需要切换时更新 selectedId，执行配置保存。查看列表不强制切换。

### Step 5：返回

返回稳定 ID 和 gameRoot；切换不影响已经运行的游戏。

## 修改

### Step 1：定位

使用“查看与选择”确定唯一目标，不对同名多实例批量修改。

### Step 2：修改显示名称

只改 name，不改 ID、不搬目录；玩家未要求改名则跳过。

### Step 3：修改 Java 或版本

Java 选择必须是现有记录，变更时清除 ignoreJavaCompatibility。MC/loader 变更转 [版本与加载器管理](loaders.md)，默认创建新隔离实例并保留旧数据；子流程验证真实组件与材料后返回，不只改字段冒充升级。

### Step 4：保存

执行配置保存，保留其他实例。

### Step 5：验证

重读并展示实际改动；无版本文件变更不能称完成版本升级。

## 删除

### Step 1：明确目标

列出唯一实例 ID、名称和完整路径。

### Step 2：询问删除范围

问“仅移除列表记录，还是同时删除这个实例目录及其存档/配置？”明确确认前不删除。不得扩大到根目录库或其他实例。

### Step 3：检查运行状态并执行

目录删除前执行 [公共写入门禁](operations.md#写入前门禁) 的路径、实际停机、明确范围和独立备份检查；取得 `<instancesRoot>/.management-publish-<instanceId>.lock` 父级锁，journal 必须位于 dataRoot/.management/journals，不能放在将删除的实例内。只移除记录则保留目录；同时删文件时，先把实例移到同文件系统的独占 `.delete-<opId>` 隔离目录，在全局 journal 写明原/新位置，暂不删除真实文件。移走后重新核实锁与目标身份；移动失败原记录不变。

### Step 4：保存

在一次配置提交中将目标 ID 加入 instances.retiredIds 墓碑，移除目标实例记录及其 content.entries；对应备份和历史 operation 保留并标 orphan，不删除恢复材料。若为当前实例则 selectedId=null，不静默选另一个。执行配置保存并读回；保存/读回异常必须先走 [协调配置与文件恢复](operations.md#协调配置与文件恢复)，确认正式配置实际 before/after。若配置已提交且目录隔离计划已完成，可重验后完成提交；确需回退且配置已经删除身份时，先安全补偿实例/content/selectedId/retiredIds 再按 journal 复位目录。原位置被占则保留隔离目录和备份，不覆盖，不能只恢复目录却留下已删除记录。读取规则允许 orphan，墓碑不参与 historyLimit 裁剪。

### Step 5：验证

配置提交及读回成功后，才按已确认范围回收/清理本操作的隔离目录；不删除 dataRoot 级 journal/备份。清理失败报告“记录已移除，隔离文件仍保留”和精确恢复入口，不称文件已删除；不可因回收失败擅自扩大到未批准的永久删除。核对墓碑、记录和实际目录，再释放父级锁及本操作锁。仅移除记录的原目录原样保留；未知遗留目录与退休身份以后均不能被新实例占用。

## 字段与状态约定

created=目录建立；download_failed=创建后的下载未完成；installed=全部必需材料验证通过。历史状态不代表当前文件仍完整，更不代表正在运行。数据根目录变更不代表实例已迁移。
