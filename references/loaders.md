# 加载器管理

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

执行前遵循 [玩家运行能力契约](../SKILL.md#玩家运行能力契约)：只用已有专用工具或目标 OS 原生命令；缺能力 BLOCKED，不安装额外编程运行时/库（游戏 Java/材料下载例外）、不写临时脚本、不用 Computer Use 或 GUI 自动截图。游戏内操作和视觉验收由玩家完成确认。

锁操作一律执行 [公共门禁](operations.md#写入前门禁) 的父级稳定锁→实例锁协议，整个操作持有，持锁后再次核对所有权；下文“检查发布锁/持锁”不是只检查不存在。子流程复用同一 operation 锁，不重复取得。

管理列表、安装、切换（含同 loader 改版/MC 改版）和移除。默认将玩家数据普通复制到**新隔离实例**，保留源实例、旧世界和选择；不靠修改 `instances.entries[].loader` 一个字段冒充安装完成。数据根内各实例的 versions/libraries/assets 分开；Java 可引用现有 ID，不能搬用 Prism 的全局游戏库或真实启动器配置。

## 共同契约与限制

输入为 sourceInstanceId、目标 MC（默认源版本）、loader 名称和精确版本（移除时 null）、新名称/数据复制范围；loader 未指定不能静默选 vanilla。需要配置/文件/JSON/hash/安全复制/HTTP/进程工具。仅调用已有合格工具动作，不通过通用编程执行器补齐能力，不创建 runtime 脚本或要求安装 Prism。

每项先执行 [操作上下文](operations.md#输入与结果约定)，返回本项定位步骤；元数据/下载前完整执行 [网络与来源](operations.md#网络与来源)，返回当前 HTTP处。列表纯本地分支不联网。执行 [写入门禁](operations.md#写入前门禁) 返回调用方暂存步骤；其中备份 [创建备份](backups.md#创建备份) 复用当前 operation/锁，禁止递归拿锁或备份。新实例由本 operation 独占预留未占用 ID/目录，源实例锁保护读取一致性；子流程不重新加源锁、不同操作不写此预留目标。最终提交执行 [提交与恢复](operations.md#提交与恢复)，返回调用方验收步骤。调用下载/组件模块时仅在本次暂存实例写派生材料，把其“保存 installed”延迟到本模块提交，不能让子流程提前注册半成品或改变 selectedId。

UID映射沿用 [组件版本列表](prism-components.md#读取组件版本列表)：Forge=net.minecraftforge，NeoForge=net.neoforged，Fabric=net.fabricmc.fabric-loader，Quilt=org.quiltmc.quilt-loader，LiteLoader=com.mumfrey.liteloader。列表存在不等于所有 MC都能安装。此 Skill 一次只选一个普通 loader，不能继承 Prism“保留冲突 loader”的风险选项。

现代 Forge/NeoForge 的 metadata-pinned ForgeWrapper + 官方 installer 分支执行 [安装与验收](forge-install.md#安装与验收)，按三组材料、客户端 processors 和产物收据处理。不支持 jar-mod/替换 Minecraft.jar、未知 agents/processors、未知 traits、自定义本地组件或 legacy FML 安装阶段。遇到它们返回 BLOCKED: COMPONENT_BRANCH_UNSUPPORTED，列具体 UID/版本/字段/阶段；可保留已验证缓存，不运行 installer/脚本绕过，不偷偷换原版/原始 Mojang JSON。原地 loader/MC 变更暂不提供；用户明确要原地仍返回 BLOCKED: IN_PLACE_CHANGE_UNSUPPORTED 并提供新隔离实例方案，不能据此改源实例。移除只表示新实例不含该 loader，不能保证已使用 Mod 的世界可在原版安全打开。

## 列表

### Step 1：选择本地或在线视图

输入 instanceId、mode=installed/available（默认 installed）、可选 loader/目标 MC。执行只读上下文；无配置返回未初始化，源实例缺失停止。available 需要明确的实例 MC/玩家目标 MC，否则 WAIT_INPUT，不生成“所有版本都可用”的列表。

### Step 2：检查本地组件

读配置、instanceRoot/mmc-pack.json、metadata/ 原文件及 launch-profile.json；展示实际 net.minecraft 和各 loader 组件 UID/version、启用状态、profile 源 hashes及必需文件状态。三者不符显示 inconsistent，不自动修复。只有配置 loader 无材料显示 declared_only，不算 installed；没有组件记录不从 mods 文件名猜 loader。mode=installed到 Step 4，绝不发 HTTP。

### Step 3：获取真实可选项

available 执行网络流程，再调用 [读取组件版本列表](prism-components.md#读取组件版本列表) 全五步，返回本 Step：GET 默认 Prism meta 总 index.json → 按总索引 sha256 校验 `<uid>/index.json`，对选定 MC检查 requires.equals/依赖可解性，只显示真实候选和 recommended；需要具体候选验证时读取 `<uid>/<version>.json` 并按版本索引 sha256核验。明确 source URL/UID/version，不因 Fabric名气自动映射 Quilt。快照/历史版按玩家筛选；离线可显示已有 hash链校验缓存并注明缓存时间/未核实当前可用，没有缓存返回 OFFLINE_POLICY。

### Step 4：核对返回

用工具计数并输出本地组件与可选版本，分页仅展示截取的真实列表并标剩余数，不捏造服务端 offset。记录是否仅索引已取/具体组件已验证/材料已装；不写配置、不改变选择。选择版本后进入安装或切换 Step 1，搜索列表不代表授权下载。

## 安装

### Step 1：检查输入与源状态

输入源实例、目标 loader/精确版本、新名称；上下文和本地列表确认源是 vanilla。源已有 loader 时转“切换” Step 1，不把第二 loader追加进去。版本未定运行 available 列表后 WAIT_INPUT；玩家完整指定兼容 loader/version 时复用选择。明确默认创建新隔离实例、复制用户数据、源世界保留，不自动启动任何世界。

### Step 2：解析到可执行组件计划

目标 MC 默认源 MC；先网络门禁，再调用 [解析并合成实例组件](prism-components.md#解析并合成实例组件) Step 1–5 的**规划模式**：在 operation 的内存/暂存位置解析，不改源 mmc-pack。根为 net.minecraft:<目标MC> 和一个 loader；获取版本 SHA256链、递归 requires、检测 equals/conflicts、按 order合成，选择所选 Java的实际架构、普通/native 库及 mainJar/assetIndex/logging。源组件自定义/未知 traits/agents/jarMods/legacy 分支必须先阻断；已识别的现代 installer processors 转 loader 子流，未知 processor 实现只阻断该具体项。Java未定则按 [安装材料运行环境](downloads.md#安装原版材料) 取得实际架构；不得把下载所需架构选择当 Java major豁免。

### Step 3：冻结复制与下载计划

分配新实例稳定 ID：[分配稳定ID](configuration.md#分配稳定-id)，分配前查配置和真实目录。列新实例路径、MC/loader/version、全部依赖/素材来源及下载量；数据普通复制范围默认 saves、config、mods、resourcepacks、texturepacks、shaderpacks、options.txt 等实际存在用户数据，但逐项枚举，不复制旧 versions/libraries/assets/natives、旧 mmc-pack/launch-profile/metadata、.management 锁日志、旧进程记录或含秘密启动参数。源世界只复制，绝不移动或链接；备份/空间不足停止。读取源 runtime/游戏/JVM参数，分类如下：普通内存/窗口/显示设置可保留经核实值；`-cp`、mainClass、-javaagent、旧 loader库路径、旧版本/native参数、未知 wrapper/钩子不继承，列出移除项并确认，不执行。不能解释的关键参数阻断，不能粗暴复制全部。

### Step 4：确认和备份

展示新增隔离实例、源只读、用户数据复制清单、隔离的不兼容 Mod/配置、源 metadata/参数差异、将把 ignoreJavaCompatibility 重置 false。玩家确认完整选择即授权本计划游戏/依赖/assets下载，不再问同范围下载许可；额外下载Java/执行installer/换源不包含。执行写入门禁，返回 Step 5；源停运、源文件指纹、目标未占用及空间必须核实，源备份校验失败则停止。持源 operation锁，独占预留目标；不调用 instances.md#创建 重走授权和注册。

### Step 5：复制与重新构建材料

在目标同文件系统的专属暂存实例普通复制用户数据，拒绝链接/特殊文件/越界路径；逐文件核对hash与源快照，原源树不改动。所复制 mods 必须按目标 MC/loader逐项检查，匹配证据不足/明确不兼容者不投入运行 mods；经计划批准普通移至新实例 `migration-quarantine/mods/` 并列原路径/hash，不自动删或称已禁用。config 保留不代表兼容，列风险；改变MC可能升级世界且无法降级，世界复制完成也不自动打开。

以暂存实例为目标执行 [组件解析](prism-components.md#解析并合成实例组件) 全流程，写全新 mmc-pack.json/metadata/launch-profile.json，全部源 hash来自目标链，不复用旧派生文件；再调用 [执行更新任务](prism-components.md#执行更新任务) Folders→Libraries→必要时LegacyFML→Assets，默认每实例独立材料。下载经 [单文件下载](downloads.md#单文件下载)，返回当前材料校验处。LegacyFML/未知 processors 仍按限制阻断，不因下载一些库就改成 installed。

### Step 6：提交新记录与验收

全部必需文件校验通过后执行提交与恢复：复核源文件/进程未变，目标预留仍属本 operation，提升新实例目录；新实例记录使用实际 MC/loader精确版本、javaId（可引用源已登记且真实可执行者）、ignoreJavaCompatibility=false、creationConfirmed=true/downloadOnCreate=true/status=installed。content.entries 普通复制到新实例的受管文件必须生成新 content ID/instanceId、重算hash与相对路径，来源锁只有字节一致才保留；迁移隔离区不冒充 runtime content，不注册已丢失路径。新 sidecar也用新ID，不能抄源绝对路径。managedPack不自动继承“仍受该整包管理”的身份；此处新实例是自定义副本，提示整包后续更新另需审查。源记录、ignoreJavaCompatibility、选择和来源锁全保留，selectedId 只有玩家明确要求切换才改。

返回本 Step 后读新配置/组件/profile/所有材料及用户数据hash，确认源世界完整未写、没有共享可写链接、loader字段与实际组件一致。仅报告 materials_installed；Java major按新profile重新检查且旧豁免失效，架构/完整性不豁免；需要另行Java选择则 installed 但 launch_blocked，不说 ready。真实启动只由玩家独立请求交 [启动流程](launch.md)，不自动运行。

## 切换

### Step 1：确定差异

输入源实例、目标 loader/version，可选目标 MC；上下文和列表读真实源组件。目标与源完全一致时只校验材料，完整返回 no_change；缺材料交下载修复，不能无谓复制新实例。无 loader 的移除请求转“移除” Step 1。MC/loader/loader版本/Java任一变化，新实例 ignoreJavaCompatibility=false，不能复制旧豁免。

### Step 2：重建依赖与迁移计划

复用“安装” Step 2–3 的规划，返回本操作 Step 3：目标只从 net.minecraft 与新 loader开始计算requires，不把旧组件列表仅改version字符串。对比旧新组件/metadata hash/主类/Java要求/native分类/参数；不将旧 loader依赖当共享永久根留下，也不删除源实例库。明确不兼容 Mod 和源世界副本的风险；无法判定兼容放入新实例隔离区，先让玩家确认。loader兼容与 Mod依赖兼容分开，不把 Forge/NeoForge/Fabric/Quilt当可互换。

### Step 3：复制安装而非原地篡改

执行“安装” Step 4–6，返回本操作 Step 4；这是同一 operation 的一次门禁/备份/共享锁，安装子流程不重复拿锁。持锁前只有计划；持锁后仅按已确认目标复制/下载/提交。目标材料来源改变、关键参数未覆盖、LegacyFML/processor分支不支持即停止；保留源实例供继续使用，不改成 vanilla作为“修复”。

### Step 4：验收和失败恢复

成功展示源→新ID、实际 MC/loader/version、Java豁免重置、隔离 Mod列表和未做的游戏验证；读回选择只有明确授权才变化。失败按公共提交与恢复逐文件记录：源应该完全未动；未发布目标保留 staging和失败材料，已发布但配置失败标 CONFIG_WRITE_FAILED，不再复制第二个实例。恢复须检查日志与当前目标指纹，其他任务改动返回 RECOVERY_CONFLICT；仅清本 operation独占且未修改的失败目标需确认，不删源。需要恢复源完整世界另交 [恢复备份](backups.md#恢复备份)，不以降回loader版本代替世界恢复。

## 移除

### Step 1：定位与说明范围

输入源实例和明确“移除当前loader”；上下文+本地列表确定实际组件，不以显示名猜。有多个未知/自定义组件先BLOCKED，原本无 loader返回 no_change。说明默认新建同MC vanilla副本，保留旧loader实例和全部旧世界；不是卸载所有Java或清整个库，不删除根mods文件库。用户要求仅取消一个中间依赖而非根loader时返回 WAIT_INPUT，不能让requires失效。

### Step 2：规划 vanilla副本

复用“安装” Step 2–3，目标根仅 net.minecraft:<当前MC>，loader=null；重解析完整 vanilla依赖与profile，不从旧profile删loader名就完事。将依赖旧loader的运行mods移入新实例迁移隔离区，并列出config保留风险；未经证实的Mod不自动在vanilla启用。旧世界是独立副本但缺Mod可能丢方块/实体，明确禁止自动打开；未知legacy/自定义注入状态阻断，不用移除操作绕过不支持分支。

### Step 3：确认、构建、提交

执行“安装” Step 4–6，在同一operation中只锁一次，返回本操作 Step 4。对新实例按vanilla下载库/assets/主JAR，写新metadata/profile，ignoreJavaCompatibility=false；源所有材料不删除。确认新profile主类/库不再包含旧loader，新的component选择确为vanilla；不能只验证config.loader=null。

### Step 4：验收边界

读回新实例的实际无loader组件、全部必需材料、源实例/旧世界hash、迁移隔离区清单及选择。报告 vanilla副本材料已验证，未启动/未证明世界兼容。失败同“切换” Step 4，恢复原世界不能用删除datapack/Mod或切回loader冒充。玩家若还想删除旧实例，必须独立进入 [实例删除](instances.md#删除) 的明确范围确认，不能捆绑执行。

## 协议与固定源码依据

元数据协议及hash链由 [Prism组件](prism-components.md#读取组件版本列表)、[依赖合成](prism-components.md#解析并合成实例组件)、[材料任务链](prism-components.md#执行更新任务) 给出；默认端点 `https://meta.prismlauncher.org/v1/index.json`，其 packages[].uid/sha256 → `<uid>/index.json` 的 versions[].version/sha256/requires → `<uid>/<version>.json`。本次是阅读协议/固定源码，不是在线安装验证；共享模块原有依据提交不变，本模块新增依据固定在下列提交，不擅自更新共享文件。

固定源码基线 `899d2f3957215a09fe398a8a8cbb8a8a9c619968`：
- [真实loader列表、冲突选择和resolve](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/ui/dialogs/InstallLoaderDialog.cpp#L136-L223)。Prism允许保留冲突，本Skill主动限制单loader。
- [jar-mod/主JAR替换不同入口](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/ui/pages/instance/VersionPage.cpp#L280-L296) 不在本模块支持范围。
- [创建组件后下载](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/VanillaInstanceCreationTask.cpp#L14-L37)、[更新任务顺序](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/MinecraftInstance.cpp)。
- [复制/链接模式实现](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/InstanceCopyTask.cpp#L38-L116)。本Skill仅独立普通复制，不把链接副本称为备份；默认新实例/不自动启动世界是本Skill安全策略，不声称与Prism就地操作等价。
