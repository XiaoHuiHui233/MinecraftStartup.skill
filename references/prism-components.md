# Prism 组件与下载链路

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

执行前遵循 [玩家运行能力契约](../SKILL.md#玩家运行能力契约)：只用已有专用工具或目标 OS 原生命令；缺能力 BLOCKED，不安装额外编程运行时/库（游戏 Java/材料下载例外）、不写临时脚本、不用 Computer Use 或 GUI 自动截图。游戏内操作和视觉验收由玩家完成确认。

所有 HTTP/授权轮询/元数据/文件请求前先执行 [公共网络策略](operations.md#网络与来源) Step 1；management.network.offline=true 时停止网络子步骤，不把离线账号误当全流程禁联网。策略检查返回后继续本文件当前步骤，不重复执行无关模块。

默认下载链路是 Prism 的组件元数据 → 依赖解析 → profile 合成 → 文件夹/库/legacy FML/assets 任务；不是默认直接套 Mojang 原始 version JSON，也不要求安装 Prism 程序。游戏文件从元数据提供的真实 URL 下载，不把所有文件地址替换成 Prism 域名。

默认元数据根地址 `https://meta.prismlauncher.org/v1/`，从 `downloads.metadataBaseUrl` 读取；未配置时由模板补默认值。元数据只作数据，不执行其中未知脚本。

## 读取组件版本列表

### Step 1：读取总索引

GET `<base>/index.json`，要求 formatVersion=1、packages 数组；精确找到所需 uid，不猜组件存在。缓存放数据根目录 downloads/meta/index.json，TLS 和来源校验保持开启。

### Step 2：读取组件索引

GET `<base>/<uid>/index.json`，对原始字节算 SHA-256，必须等于总索引该 uid 的 sha256。要求 JSON.uid 匹配、formatVersion=1，versions 为数组。校验失败重取总索引后最多重试一次，仍不一致阻断。

### Step 3：筛选候选

MC 默认显示 type=release；玩家明确切换才加 snapshot/alpha/beta/experiment。按 releaseTime 降序提供真实候选，不一次输出数千条。已有指定版本就精确定位，不要求重选。

loader UID：Forge=net.minecraftforge、NeoForge=net.neoforged、Fabric=net.fabricmc.fabric-loader、Quilt=org.quiltmc.quilt-loader、LiteLoader=com.mumfrey.liteloader。候选 requires 中对 net.minecraft 的 equals 必须满足所选 MC；候选还需通过依赖可解性检查。没有匹配项不显示成可安装。不要用“版本号大于某值”的模型判断代替依赖检查。

### Step 4：建议而不代选

存在 recommended=true 的匹配版本时标推荐，没有就按真实时间列出。版本/loader 已给完整就复用；不明确时返回候选交实例创建流程提问。无 loader=原版。

### Step 5：返回

返回 uid、版本列表、各项 sha256/requires/recommended、已校验索引来源和文件位置。索引成功不代表组件或游戏已安装。

## 解析并合成实例组件

### Step 1：锁定玩家选择

根组件为 net.minecraft:<MC版本>，可选再加一个已确认 loader。实例根保存 `mmc-pack.json`，格式为 `{"formatVersion":1,"components":[{"uid":"net.minecraft","version":"<所选版本>"}]}`，追加已解析的依赖/loader。该文件只记录组件选择，不代表完整 Prism 实例导入格式。

### Step 2：获取具体组件

先通过“读取组件版本列表”获得指定版本 hash，再 GET `<base>/<uid>/<version>.json`，按索引 sha256 验证原始字节；uid/version/formatVersion 必须匹配。实例根 metadata/<uid>/<version>.json 保存原始已校验数据，不能改字段后仍声称符合原 hash。uid/version 参与路径前检查安全单层标识。

### Step 3：解析 requires

依次收集依赖，显式玩家版本和 equals 为硬约束；无硬约束时优先已锁版本，再用 suggests，仍无版本时从匹配已锁 MC/其他硬约束的真实推荐候选选取，多个无唯一推荐则询问。新依赖重复 Step 2；记录依赖边，检测循环和相冲突的 equals。与玩家选择冲突/未知约束格式时阻断，不擅自换 MC 或 loader。

示例：当前 1.16.5 的 requires 指向 org.lwjgl3，suggests=3.2.2；必须读取该组件，不可只用 net.minecraft 自身的 libraries。组件列表和具体文件中的依赖都应核对。

### Step 4：检查组件冲突

读取 conflicts，互斥 loader 不混装。发现已选冲突项时列明并等玩家改选，不自动禁用另一项。依赖和选择锁定后保存 mmc-pack.json；失败状态保存 created/download_failed，不伪装 installed。

### Step 5：按顺序合成 profile

按组件 order 升序合成，缺失 order 或相同 order 的关键覆盖无明确顺序时进入异常核对，不随意决定。仅 net.minecraft 提供 MC版本/type/assetIndex；mainJar、mainClass、minecraftArguments 采用后应用的非空值；+jvmArgs 追加，+traits 合并去重；compatibleJavaMajors 按组件收集，compatibleJavaName 取后者非空值。logging 使用已确认 MC 组件配置，不把 Prism 的 logging.file 当 Mojang logging.client.file。

普通库、native 库、mavenFiles 三组分开处理。OneSixVersionFormat 同时接受 libraries/+libraries；mavenFiles 按相同 library 格式读取，执行 rules 后排除 native，按组件顺序追加，**不套用普通库的按名称去重/版本提升**。后续下载三组，classpath 只使用普通库；无 processors 字段不代表无需安装。主类为 io.github.zekerzhayard.forgewrapper.installer.Main 时只建立 [现代 loader 子流](forge-install.md) 的 loaderInstall 计划，实际执行延迟到更新任务 Step 3，不从解析器递归调用安装。普通/native 库各自先按“运行环境规则”过滤，再按 Gradle 坐标的 group/artifact/classifier/extension（不含 version）匹配；同身份按 Prism 版本比较规则保留较高版本，不按字符串排序。遇到无法确定的版本排序/覆写或未知 processors、agents、jarMods、未知 trait 等未覆盖分支，明确阻断并查实现，不宣称通用 loader 安装成功。

### Step 6：保存派生结果

实例根保存 launch-profile.json，至少记录 provider=prism、runtimeClassifier、源组件及 hashes、minecraftVersion/type、mainClass、mainJar、minecraftArguments、jvmArgs、traits、compatibleJavaMajors/compatibleJavaName、assetIndex、logging、普通/native 库、mavenFiles、loaderInstall（无安装链时 null；否则包含 kind=forgewrapper、installer/wrapper 坐标及收据引用）。此派生文件是本 Skill 工作结果，不冒充 Prism 的上游原文件。重读源 hash 与派生字段，返回解析结果。

## 运行环境规则

### Step 1：确定 classifier

system 为 windows/osx/linux，架构以实际 Java 进程为准，映射 amd64→x86_64，i386/i686→x86，aarch64→arm64，arm/armhf→arm32，其余保留真实值。classifier=`<system>-<映射架构>`。

### Step 2：匹配 Prism OS 名

先精确匹配完整 classifier；仅 x86/x86_64 允许退回 system 名。ARM 上 osx-arm64 与 osx 不是同一个规则，不能沿用 Mojang 格式的简单 OS 比较。

### Step 3：执行 allow/disallow

无 rules 允许；有 rules 初始禁止，按顺序应用最后匹配 action。已审 Prism 此路径只按其 OS classifier 执行；若遇到其他非空条件，进入异常核对，不假装这里完整实现 Mojang features/OS正则规则。

### Step 4：选择 native

natives 先用完整 classifier；仅 x86/x86_64 才用 system 回退。无兼容项的 native 库按 Prism 路径不激活，同时核对同功能是否存在适用替代；关键 native 全无可用替代才报架构缺口。不要在读取 org.lwjgl3 等依赖前就断言老 MC 没有 ARM 支持。

### Step 5：构造文件路径

下载项有 path 时验证后使用；Prism 元数据常省略 path，此时按坐标 `group:artifact:version[:classifier][@extension]` 构造 `group替换点为斜杠/artifact/version/artifact-version[-classifier].extension`，extension 缺省 jar。native 使用选定 classifier 替换，不能将常规 artifact 和 native 的文件名混用。没有 downloads 的标准 Maven 库按 Prism Library::getDownloads 处理：已知绝对 URL 优先，否则用库的 url 仓库地址（补唯一斜杠）拼接上述坐标路径，仓库地址缺省才用已核实的默认 library base。Fabric intermediary/loader 的 url=https://maven.fabricmc.net 属于这条正常分支，不应判为未知安装器。元数据无 hash 时标明未提供上游 checksum；可从同一可信仓库读取有效 checksum sidecar 作额外核对，不能将本地自算 hash 冒充上游预期 hash。特殊本地工件/hint/自定义文件名无已验证规则时阻断，不猜 URL。

## 执行更新任务

### Step 1：FoldersTask

创建目标实例内部所需目录；所有缓存和游戏数据仍使用本项目实例隔离路径，不复制 Prism 的全局库布局。

### Step 2：LibrariesTask

收集合成 profile 中的普通/native 库、mavenFiles 和 mainJar，调用单文件下载，使用各自 metadata URL/大小/hash。classpath 库与 native 下载分别记录。local 工件缺失要求用户提供，不从不明源补齐。

### Step 3：LegacyFMLLibrariesTask

现代原版没有 legacy FML 需求就明确跳过。现代 Forge/NeoForge 不是 LegacyFML：LibrariesTask 完成后调用 [安装与验收](forge-install.md#安装与验收)，成功回本节 Step 4；输入/授权不足停在其具体步骤。组件若要求真正 legacy FML 或未知安装分支，阻断并指出具体阶段；不能忽略后继续称安装成功。

### Step 4：AssetUpdateTask

按 MC assetIndex 下载并校验索引，再按对象 hash 下载/验证 assets；处理实际 logging.file 和已知旧资源布局。复用下载模块的原子落盘、重试和校验规则。

### Step 5：验证并返回

复核全部必需文件，核对 loader 安装收据与必需 patch 产物后才允许 installed；导入/迁移调用只返回 staged_verified，由父流程发布后保存 installed，不注册 staging 或修改 selectedId。失败保留可验证缓存、状态标 download_failed，重试只补缺失/损坏项；不重新询问同一创建选择的下载许可。创建阶段若 loader 安装需要 Java 则先选定并探测；只有玩家请求启动才进入账号/实际游戏启动，不因只创建实例就强制登录。

## 实现依据

以下路径已在固定源码 PrismLauncher `899d2f3957215a09fe398a8a8cbb8a8a9c619968` 重新核对：

- [创建选择及 loader 列表](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/ui/pages/modplatform/CustomPage.cpp#L71-L165)
- [创建时保存组件并下载](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/VanillaInstanceCreationTask.cpp#L14-L37)
- [DownloadGameFilesDuringInstanceCreation](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/InstanceTask.cpp#L136-L171)：Prism 有开关，本 Skill 按玩家要求默认开启创建后下载，不宣称上游总是无条件下载。
- [组件应用](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/VersionFile.cpp#L54-L87)、[库选择](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/LaunchProfile.cpp#L174-L199)
- [classifier 与架构回退](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/RuntimeContext.h)、[更新任务顺序](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/MinecraftInstance.cpp#L1132-L1143)

- [mavenFiles 解析](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/OneSixVersionFormat.cpp#L188-L214)、[独立追加且不去重](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/LaunchProfile.cpp#L202-L214)。
- [材料池](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/update/LibrariesTask.cpp#L42-L51)、[classpath 排除 mavenFiles](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/LaunchProfile.cpp#L348-L379)、[下载 URL 与 SHA-1](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/Library.cpp#L108-L227)。
