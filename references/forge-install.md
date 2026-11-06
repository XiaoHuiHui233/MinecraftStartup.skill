# 现代 Forge 与 NeoForge 安装及启动

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices: ../NOTICE.md; license: ../LICENSE. -->

只在合成 profile.mainClass=`io.github.zekerzhayard.forgewrapper.installer.Main` 且存在可验证官方 installer 与 metadata-pinned ForgeWrapper 时调用。输入为 caller/returnStep/operationId、stagingInstanceRoot、MC/loader 精确版本、profile 三组材料、Java 身份；side 默认 client。不是通用 processor 解释器，不执行包内 shell/钩子，不写自定义 Java helper。遵守 [能力边界](../SKILL.md#玩家运行能力契约)、[组件 hash 链](prism-components.md) 和 [锁/恢复](operations.md)。官方 installer/Wrapper 是游戏依赖，授权必须明确覆盖其 Java 代码与声明 processors，不能把“允许下载”当执行许可。

Prism 的更新任务只下载材料；Wrapper 在启动 JVM 中读取 installer 的 install_profile/version、执行客户端 PostProcessors、补模块设置、再调用真正游戏主类。**Wrapper Main 没有本手册可用的 install-only 开关**。因此 Skill 导入阶段走官方 installer 的 CLI 客户端安装，随后启动仍按 Prism Wrapper 主类。此两阶段安排是 Skill 为“不自动启动”增加的流程，不冒称 Prism 原样实现。

## 安装与验收

### Step 1：锁定元数据与材料身份

接收调用方已解析的总 index → uid/index → uid/version 的 SHA-256 原始字节链；缺少链则返回调用方组件准备点，不递归调用解析器；检查 net.minecraft equals、Java major 与实际 Java 架构，不按宿主 CPU 猜 Java 架构。runtimeClassifier 未确定可先返回计划，执行为 WAIT_INPUT: JAVA_REQUIRED 或 BLOCKED: JAVA_ARCHITECTURE_UNKNOWN。1.21.1 的 major 以已校验 MC 组件为准，不靠 loader 名猜测。规划模式只返回材料与待确认事项，执行模式必须复用父 operation 锁。

分成 libraries（普通 classpath）、nativeLibraries（仅对应架构提取）、mavenFiles（installer/processor/neoform 等安装材料，保留所有版本、不合并至 classpath）。同一实际坐标路径跨组字节完全相同可以共用本实例普通文件，hash 冲突阻断；不能把 mavenFiles 的 ASM/Gson 旧版提升到运行库版本。按元数据 URL、size/hash 下载；路径省略时用 Maven 坐标规则，@zip、classifier 都保留。Wrapper 必须来自已校验元数据的 library 项，installer 必须是对应 loader 的官方 Maven 工件，不使用整合包携带同名文件替代。

成功检查：明确 wrapper/installer/mainJar 唯一身份及路径、三组材料、元数据链来源。无对应分支 BLOCKED: LOADER_INSTALLER_UNRESOLVED，返回调用方当前准备步骤，不退回 Fabric 或原版。

### Step 2：有界读取官方安装描述

用已有安全归档工具检查 installer JAR，按 [归档安全预算](modpacks.md#导入整合包) 读取 install_profile.json 以及其 json 引用的 version JSON、MANIFEST.MF；UTF-8/路径/CRC 按该归档子流检查，不额外要求专用流式配额或重复键检测接口。安装器中的 `/version.json`、`/data/client.lzma` 是 JAR 资源名，仅在归档内去掉一个前导 `/` 定位，绝不能作为宿主绝对输出路径。manifest 默认 1 MiB，不够需明确调整。

要求 install_profile spec=1、minecraft 与 MC 相同、version 与 loader 对应；读取 data 的 client 分支、processors 的 sides（缺省两端，只有 server 的跳过）、jar/classpath/args/outputs 和版本 libraries。预先列每个 processor 的真实 Java 工件及 Main-Class（从其 JAR manifest 读取，不能猜）。`${...}` 是版本 JVM 模板，`{KEY}` 是 installer data token，`[g:a:v:c@ext]` 是 Maven 文件引用，`'literal'` 是字面量；不混成 shell 插值。

生成计划表：输入路径、预期来源/hash、输出路径、客户端顺序。{ROOT}=stagingGameRoot、{LIBRARY_DIR}=其 libraries、{MINECRAFT_JAR}=下面 Step 4 的 vanilla 副本、{INSTALLER}=已验证 installer、{SIDE}=client、{MINECRAFT_VERSION}=锁定 MC；data 的 Maven 引用解析到本实例 libraries，嵌入资源在受控临时目录由官方 installer 处理。每个引用与实际输出必须在本 operation 范围，符号链接/绝对外部输出/未知 token、未覆盖任务或额外程序入口 BLOCKED: PROCESSOR_CONTRACT_UNSUPPORTED，列 processor 索引与字段。

NeoForge 21.1.265 的已核对实例是 MCP_DATA → DOWNLOAD_MOJMAPS → MERGE_MAPPING → 客户端 jarsplitter → AutoRenamingTool → binarypatcher；server 的脚本提取与 bundler 步骤跳过。实际版本若改变任务/参数，重核官方实现而非沿用此列表。检查版本 JSON 的空 URL 工件、data 与 processor 输出共同确定必需生成材料，不能只看 version.libraries；本例后者无空 URL 项，仍需要 client patched/extra 等文件。

### Step 3：确认执行链与网络范围

展示 loader/MC/Java 版本及架构、installer/wrapper/processor 的来源及 hash、会运行官方 Java 代码、写入 staging 范围、patch 输入输出、可能请求的官方 Mojang/Maven 服务和原始本地日志位置。针对该精确链等待 WAIT_INPUT: LOADER_EXECUTION_CONFIRMATION；已在本任务明确确认过且材料/范围不变则复用，不反复询问。Java 下载单独沿用 Java 模块授权。

这不授权执行整合包中的 shell/bat/PS1/未知 JAR；kubejs/scripts 内容只由以后玩家启动的游戏加载。下载和执行身份均不依赖 CF key。网络禁用时不能让 installer/processor 隐式联网：`--offline` 只控制安装器下载层，不证明其任意 processor 都不联网；没有已核实完整本地输入和实际阻止外联的宿主能力，返回 BLOCKED: LOADER_OFFLINE_UNPROVEN。正常联网模式也须核对官方链的实际下载实现/目标；未知镜像/重定向不能由 installer 自行换源，无法约束或核实则停在 BLOCKED: INSTALLER_NETWORK_SCOPE_UNVERIFIED，不关闭 TLS 或加入 skip-hash-check 来继续。

### Step 4：准备独立的客户端安装目录

写前必须持父级稳定锁→实例锁，journal 记录计划、授权和所有原文件指纹；无完整门禁回父调用点。定义 stagingGameRoot=`<stagingInstanceRoot>/.minecraft`，库根固定为其 libraries，tmp=`<stagingInstanceRoot>/.management/loader-tmp`，只在本 operation 私有非链接目录创建。使用已有文件/JSON 工具写 [launcher-profiles 模板](../templates/loader-launcher-profiles.example.json) 到 stagingGameRoot/launcher_profiles.json；此文件仅为官方 ClientInstall 的输入输出，不读取玩家官方启动器 profile，也不是账号配置。

将已通过 Prism hash 链验证的 vanilla mainJar 普通复制到 `versions/<MC>/<MC>.jar`，再验 size/hash；保留原 mainJar 路径供 Wrapper 使用。ClientInstall 若已有此 JAR 不再取 vanilla 版本描述，不能借此放宽校验。补齐 installer 的 libraries、version libraries 和三组材料，下载条目遵守源/hash/预算；嵌入 `/maven/` 资源由已校验安装器提供，预先检查其归档路径与允许写集。不伪造 Mojang version JSON、不创建全局 launcher 安装。

先记录安装目录基线，派生输出位置不得来自导入包、旧实例或未经验证缓存。全新构建从空的派生输出开始；恢复失败构建仅能复用 hash 与本操作收据一致者。空间/路径防护/原子复制确实不足时 BLOCKED: CAPABILITY_MISSING；已有 bsdtar 等工具按 [Prism 式解包](prism-archive.md) 核实即可，不要求额外归档运行时。

### Step 5：调用官方客户端 CLI 并等真实退出

用已有进程工具执行以下 **argv 数据**（每行一项，尖括号由工具替换，不能当 shell 脚本运行）：

```text
<已确认 Java 绝对路径>
-Djava.awt.headless=true
-Djava.io.tmpdir=<本 operation 的 loader-tmp 绝对路径>
-jar
<metadata 指定且已验证的官方 installer 绝对路径>
--installClient
<stagingGameRoot 绝对路径>
```

cwd=stagingGameRoot，stdin 关闭，stdout/stderr 写本任务私有日志，等待真实退出码。必须显式传动作及目标，不能依赖默认 mcdir/user.home，不能无参打开 GUI，也不能因 headless 默认执行 server。仅 Step 3 的离线证明成立才添加单项 `--offline`；绝不加 `--skip-hash-check`。不能把 `forgewrapper.installer` 三属性当 installer CLI 参数，不能执行 Wrapper Main 来“仅安装”。工具必须能监督退出及失败停止；超时/异常停止后核对实际子进程已退出再恢复，不能留下安装器继续写 staging。

由官方 PostProcessors 选择 client、展开 data、按序加载其 JAR/依赖并调用 main；Skill 不重写 processor Java 实现、不自行发明 processor 协议。本手册覆盖已审官方 spec=1 / PostProcessors 契约；Forge 的 CLI 需确认为对应官方版本支持同一 installClient 合约（见下方固定 Forge 来源），不同实现不能仅凭同名参数放行。失败保留日志和逐文件结果，BLOCKED: LOADER_INSTALL_FAILED；退出 0 仅进入下一步，不直接 installed。

### Step 6：验证真实产物并生成收据

读取 installer 写入的 `versions/<install_profile.version>/<version>.json`，与嵌入 version JSON 对比；复核所有下载/嵌入材料的预期 hash、实际 size，核对 vanilla 输入未变、client processor 计划全部完成。outputs 若提供预期 SHA-1 必须验证；缺 outputs 不是“无需产物”。本例必须按 data 的 client Maven 坐标找到 MAPPINGS、MOJMAPS、MERGED_MAPPINGS、MC_SLIM、MC_EXTRA、MC_SRG、PATCHED，检查非空、格式/JAR CRC，以及 patch 输入/输出与日志对应；MC_UNPACKED 只被 server 使用，不能误要求客户端生成它。版本 JSON 的空 URL 运行工件（如有）也必须存在。

对于无上游预期 hash 的派生产物，记录 fresh_build、完整已验证输入链、执行版本/退出码、size 与 localSha256，明确这是本地派生收据，**不是作者发布的产物校验值**。没有可信构建收据的旧同名产物不采纳，需重新运行确认过的安装链。安装器新下载的每个文件都纳入实际清单并与批准来源/预期 hash 核对；出现额外未批准写集、损坏/遗漏或未知结果 BLOCKED: LOADER_OUTPUT_UNVERIFIED。文件存在或一条“success”日志不能代替验收。

按 [loader 收据模板](../templates/loader-install-receipt.example.json) 写 instanceRoot/loader-install-receipt.json，保存原始安装描述副本及其 hash、全部 inputs/outputs、Java version/architecture、relativePath、size、localSha256、expectedHashes、source、confirmation、exitCode。模板 uninitialized 不能冒充 verified。原始文件和收据由父 journal 保护，不含账号/token/CF key。只返回 staged_verified 到组件更新 Step 4 或父指定准备步骤；正式 installed 由父导入收尾决定。

### Step 7：发布后的路径重绑定与失败恢复

父流程移动 staging 后，收据内部使用实例根相对路径保持不变，重新核对最终路径和关键产物；启动参数必须用正式 gameRoot/libraries/installer/mainJar/tmp。组件/profile 的 resolvedFile 若有 staging 绝对路径须重新解析后原子写入派生文件，不能字符串盲替换整个清单。只有强验收后父流程注册 installed，不修改旧 selectedId、不自动启动。

配置保存按 [配置契约](configuration.md#保存配置) 完成，先释放 config.lock 再借用父锁刷新本实例缓存。保存结果 committed/unknown 时按 [协调恢复](operations.md#协调配置与文件恢复) 判断补偿，禁止先删已发布目录；恢复只改本操作写集且当前 hash 仍是 after 的文件。任何 WAIT/BLOCKED 记录具体阶段和恢复点，保留失败材料，释放本层拥有的锁，借用锁交父收尾。没有选 Java/账号或安装未齐时缓存 stale，不能将配置已提交误报未提交。

## 启动参数与缓存

### Step 1：检查已验证安装及执行授权

command 首次构造/重建调用本节；核对 profile 源链、loader 收据、Java/架构和所有关键输入/产物身份。执行授权必须覆盖该精确 Wrapper/installer/processor 链在游戏启动中复核或补丁的行为；参数/代码 hash 改变重新确认。未完成安装先返回上节，不带游戏会话尝试安装。不重复认证/下载已有效材料；快速缓存命中不加载本手册。

### Step 2：返回 Wrapper JVM 参数与游戏参数

向 command JVM 列表返回三个各自独立元素：`-Dforgewrapper.librariesDir=<gameRoot/libraries>`、`-Dforgewrapper.installer=<安装器绝对路径>`、`-Dforgewrapper.minecraft=<profile.mainJar.resolvedFile>`，另加受控 `-Djava.io.tmpdir=<实例私有 loader-tmp>`。使用这些上游正式属性适配本 Skill 隔离布局，不依赖 Prism 全局路径探测。普通 libraries（含 Wrapper）+mainJar 构造 classpath，mavenFiles 不进 classpath。

mainClass 保持 Wrapper Main；gameArgv 取 Prism minecraftArguments 并按既有规则先分词后替换。必须保留 --fml.mcVersion、对应 --fml.forgeVersion 或 --fml.neoForgeVersion、--fml.forgeGroup（Forge 分支若需要）、fml/neoForm 版本与 launchTarget 的真实值，不能硬编码 forgeclient 适用于所有 Forge。Wrapper 内部读取 installer version JSON 的 JVM 模块/exports/opens、安装产物与真实 mainClass；不再把该 JSON 全套 module-path 参数重复塞到外层或把主类换成 BootstrapLauncher 绕过 Wrapper。

### Step 3：缓存与返回

回 command Step 2～8 完成最终数组并按 [缓存维护](launch-cache.md#配置变更后刷新缓存) 保存。sources 绑定组件/profile/安装描述/收据/pack-state，requiredFiles 包含 installer、Wrapper、mavenFiles、实际 patch 产物与普通/native 材料。只把 auth_access_token/auth_session 留动态槽；一次性世界/服务器目标不进基础缓存，未选 Java 或架构 unknown 精确 stale。快速入口依旧最高优先级；收据/文件身份不变复用缓存，不每次联网或重跑 installer。变化项无法验证时定点失效到安装验收，不能开启 forgewrapper.skipHashCheck。此模块只构造 argv，实际游戏进程只由快速 Step 4 或完整启动 Step 12 创建一次。

## 固定实现依据与覆盖边界

- Prism `899d2f3957215a09fe398a8a8cbb8a8a9c619968` 的 [更新和 launch 顺序](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/MinecraftInstance.cpp#L1132-L1259)、[JVM 附加参数](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/minecraft/MinecraftInstance.cpp#L526-L549)；三类材料见 [组件实现依据](prism-components.md#实现依据)。
- ForgeWrapper 固定 `6a13388c72a473db32e8040d97c7f4e0b1e4d3d9`：[Main.java L19–67](https://github.com/PrismLauncher/ForgeWrapper/blob/6a13388c72a473db32e8040d97c7f4e0b1e4d3d9/src/main/java/io/github/zekerzhayard/forgewrapper/installer/Main.java#L19-L67)、[Installer.java L27–139](https://github.com/PrismLauncher/ForgeWrapper/blob/6a13388c72a473db32e8040d97c7f4e0b1e4d3d9/src/main/java/io/github/zekerzhayard/forgewrapper/installer/Installer.java#L27-L139)、[Bootstrap.java L10–75](https://github.com/PrismLauncher/ForgeWrapper/blob/6a13388c72a473db32e8040d97c7f4e0b1e4d3d9/src/main/java/io/github/zekerzhayard/forgewrapper/installer/Bootstrap.java#L10-L75)、[IFileDetector 属性](https://github.com/PrismLauncher/ForgeWrapper/blob/6a13388c72a473db32e8040d97c7f4e0b1e4d3d9/src/main/java/io/github/zekerzhayard/forgewrapper/installer/detector/IFileDetector.java#L24-L98)。它在无 outputs 时会推导现有文件 hash，因此 Skill 必须先验证自己的构建收据，不能把 Wrapper 缓存命中当独立作者校验。
- NeoForged LegacyInstaller 固定 `b8e9393440981747bc4f067ac0ceb2c9c93fa360`：[CLI 动作和目标](https://github.com/neoforged/LegacyInstaller/blob/b8e9393440981747bc4f067ac0ceb2c9c93fa360/src/main/java/net/minecraftforge/installer/SimpleInstaller.java#L86-L186)、[ClientInstall profile/vanilla/processor](https://github.com/neoforged/LegacyInstaller/blob/b8e9393440981747bc4f067ac0ceb2c9c93fa360/src/main/java/net/minecraftforge/installer/actions/ClientInstall.java#L42-L158)、[processor data/args/outputs](https://github.com/neoforged/LegacyInstaller/blob/b8e9393440981747bc4f067ac0ceb2c9c93fa360/src/main/java/net/minecraftforge/installer/actions/PostProcessors.java#L75-L292)。此源码 commit 不冒称 installer 3.0.48 的可重现构建证明；执行前仍须核对目标二进制的接口及校验值。

- MinecraftForge Installer 固定 `87803a77943a1b2d02d69e7c3108042b21f5382d`：[installClient/动作参数](https://github.com/MinecraftForge/Installer/blob/87803a77943a1b2d02d69e7c3108042b21f5382d/src/main/java/net/minecraftforge/installer/SimpleInstaller.java#L67-L134)、[客户端输入与 patch](https://github.com/MinecraftForge/Installer/blob/87803a77943a1b2d02d69e7c3108042b21f5382d/src/main/java/net/minecraftforge/installer/actions/ClientInstall.java#L24-L89)。Forge PostProcessors 返回值与 NeoForge 有差异，由官方 installer/Wrapper 适配，不自行将非布尔结果当失败。

本模块不覆盖 legacy FML/jar-mod、修改版 installer、自定义 agents/未知 processor、服务端安装；实际目标 CLI/处理器语义不符时在对应字段 BLOCKED。默认新实例与独立 CLI 预安装是 Skill 差异。源码核对和合成参数测试不等于真实安装、全部平台或游戏运行已通过。
