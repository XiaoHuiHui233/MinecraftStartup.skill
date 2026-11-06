# 从首次运行到进入 Minecraft

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

本文件是完整流程，不是需求说明。`Step 1 → Step 14` 顺序执行；子流程成功后回到下一步。已有配置按各步骤的复用分支跳过重复提问。完整 command 构造见 [启动命令](launch-command.md#拼接-command)。

## 启动游戏

### Step 1：读取请求和配置

提取玩家已明确的实例名、MC 版本、loader/版本、账号类型、Java 选择；不自动沿用已明确要求重置的旧选择。执行配置读取，配置不存在进入 Step 2；存在则核对目录后进入 Step 3。损坏或冲突按配置异常处理，不清空重置。

### Step 2：首次询问数据根目录

询问玩家完整路径，等待明确回答。执行本地文件首次初始化：建立 instances/mods/java/skins/downloads，按模板安全保存配置并重读。不得使用项目路径或对话记忆替代磁盘配置；用户提供完整路径不再反复确认。

### Step 3：首次绑定账号

当前没有已选账号就调用账号查看/选择；没有账号时询问微软登录或离线。已有账号只读出非秘密资料，再检查会话状态。

微软：读取默认/自定义 public client ID → 发起设备授权 → 在当前会话展示官方 verification_uri、user_code 和有效期 → 等待玩家在微软页面确认 → 按 interval 轮询 → Xbox → XSTS → MC token → entitlement/profile → 系统安全存储 → 保存角色和 credentialRef。执行 [微软详细步骤](microsoft-login.md#首次登录)，不能只保存 Client ID 就称绑定完成。已有等待授权任务直接复用，不重复发代码。

离线：取得合法玩家名称 → 工具生成稳定离线 UUID → 保存账号。微软失败不自动转离线。没有完成账号准备就返回 WAIT_INPUT/BLOCKED，不先把下载完成当登录成功。

### Step 4：扫描和注册 Java

执行 Java 搜索：遍历所有 PATH 项，实测可执行文件的版本和架构，去重并保存。已有实例 Java 仍要实际探测。没有可执行 Java时询问自动下载或手动路径；尚未选择 MC 时先保留待处理选择，目标版本确定后再下载，不猜版本。

### Step 5：选择或创建实例

玩家指定实例优先，其次 selectedId；无当前且仅一个实例可选则选它，多项请玩家选择。没有实例或正在补齐 created 草稿时，展示 Prism 式选择：实例名称、MC 版本/筛选、loader、loader版本。已给字段直接标为选中。

读取真实 Prism 候选；未知名称/版本不猜。未指定 loader 时必须询问，无（原版）也是一个明确选择。玩家选定最后缺失项就视为提交创建及所需材料下载，不再追加下载许可问题。执行实例创建，复用本次空草稿 ID，不多建实例。

### Step 6：走 Prism 元数据链

从配置 metadataBaseUrl 读取总索引 → 所选 uid 索引 → 具体版本 JSON，每层 SHA-256 核对；锁定 net.minecraft、loader 及 requires 依赖，不能漏掉 LWJGL/intermediary。检查 equals/conflicts，按 order 合成 profile，保存 mmc-pack.json、metadata、launch-profile.json。

依赖版本无唯一约束/推荐、未知 processor 或 schema 时才进入异常分析。不把 Prism 组件 JSON 当 Mojang 原始 JSON，不在失败时静默切换元数据源。

### Step 7：选定实际运行 Java

优先明确指定的 Java/实例 javaId；否则按 profile 的兼容 major 筛选可用安装，一个兼容者可选中，多项询问。需要自动下载时走 Prism net.minecraft.java 的 runtime manifest/archive链，检测成功再注册。

有可执行 Java 但 major 不符时，询问是否跳过该实例版本检查。明确同意后保存 javaId 和 ignoreJavaCompatibility=true，组合未变不重复询问。玩家已在本次请求同意则直接执行保存。不能把版本不符说成必然不能运行，也不能跳过可执行性/native 架构检查。

### Step 8：按更新任务顺序下载

执行 Folders → Libraries（含主 JAR）→ 需要时 LegacyFML → Assets/index/objects/logging。完整创建选择已授权这些材料，自动执行，单独创建不强制启动。

下载规划用选定/唯一已探测 Java 的真实架构；创建子流程已完成下载时只复核并补缺项，不重复重下。Prism patched LWJGL 可提供旧 MC 的 ARM 替代，必须按完整 classifier rules 取真实组件，不能未解析依赖就判不支持。

每项校验元数据提供的大小/hash；Maven-only 库按仓库 URL+坐标路径下载。上游无 hash 的文件标明边界，不把本地 hash 说成预期 hash。正常材料齐全后设 installed 并重读，不能只凭文件数报安装完成。

### Step 9：处理需要玩家决定的下载异常

正常网络重试按固定次数，不换源、不关闭 TLS。403/不可达时保留已验证缓存；同源可重试已确认的传输方式。换镜像必须说明文件、原来源、替代来源及校验方法，取得明确授权再下载。

授权只覆盖玩家答应的文件/来源，不改全局默认；保留 originalUrl 与实际 url、原 hash。长任务用可观察后台任务和逐文件 audit，超时先查真实进程/落盘文件再恢复，不盲目重新下载。

### Step 10：解压并验证 natives

建立 gameRoot/natives/<任务ID>，解压适用 native JAR。排除 META-INF 和声明 extract.exclude，防路径/链接逃逸，限制展开大小。META-INF manifest/module-info/INDEX 重名不是二进制冲突；实际 native 同路径不同内容仍阻断。

使用工具确认提取的二进制与 Java 架构一致。本次 macOS ARM64 验证了实际 arm64 dylib；其他平台必须使用各自真实结果，不复制该结论。成功才进入 command 构造。

### Step 11：构造 command 数组

逐步执行 [启动命令 Step 1～8](launch-command.md#拼接-command)：profile.mainClass → 普通库+主 JAR classpath → 模板分词后逐项替换 → 本地注入 MC token → 日志/平台/profile JVM 参数 → native 路径 → -cp → 主类 → 游戏参数。

若调用方带一次性世界/服务器目标，在命令 Step 5 完成后、最终组装前调用该目标模块的纯参数注入步骤；子模块只返回修改后的 gameArgv，再由本 Step 完成命令 Step 6～8。只有下方 Step 12 执行进程创建，子模块不能另外执行命令 Step 6～11。canonical 形状为 `[javaExecutable, *jvmArgv, mainClass, *gameArgv]`。不拼可执行 shell 字符串，不在带空格路径外加多余引号，不把 token 放进普通配置或日志。JVM 参数在主类前，游戏参数在主类后，cwd=本实例 gameRoot。

### Step 12：实际启动并接管日志

复核本实例没有同一存活任务，按启动命令 Step 9～10 通过宿主后台进程工具执行数组，显式 shell=false/cwd。必须维持日志消费者，先脱敏再写独立 launcher 日志，不能让管道无人读取或让前台工具超时杀游戏。

记录真实 PID、开始时间、taskId、程序路径及脱敏日志位置。无需启动 Prism 程序或其 NewLaunch.jar；标准原版/Fabric 从真实 mainClass 进入。未知启动格式进入异常处理。

### Step 13：观察和验证主菜单

观察最多 15 秒，约每 2 秒查同一进程，提前退出读取真实 exitCode 和脱敏错误。仅拿 PID 为 spawned，持续存活为 running；日志能证明版本/loader，但通用日志行不证明主菜单。

默认目标为主菜单：捕获目标游戏窗口或让玩家确认主菜单，才记录 reachedReady=true。显式一次性世界/服务器目标则按对应模块取得真实进入该目标的画面/玩家确认，记录目标类型和证据；仅主菜单或 PID 不能冒称已经进入世界/服务器。没有画面证据就准确停在 running，不伪造百分比/就绪。

### Step 14：保存结果与报告

合并保存最终实例/账号/Java选择、lastLaunch，以及独立的 reachedReady 和就绪证据，重新读取核对。游戏随后退出则 status=exited、exitCode=实际值，保留已达主菜单证据；不能把已退出进程改回 ready/running。

只报告本次实例/版本/loader、角色、Java、已验证状态、日志位置和未完成项。进入主菜单不等于世界、联机或全部模组都测试通过。

## 再次启动

### Step 1：重读当前选择

读取磁盘配置，不重复询问已明确且有效的目录、账号、实例、loader版本或版本检查豁免。

### Step 2：检查实时状态

检查旧 PID+开始时间+程序身份，游戏仍运行就报告并复用，不重复启动。检查账号句柄与期限，必要时刷新，失效才重新绑定。

### Step 3：复核当前环境

Java路径/版本/架构实际探测；MC/loader/Java任一变化清旧豁免，并重新解析需要变更的材料。

### Step 4：补齐并启动

从完整流程 Step 6 开始复核组件及缓存，只补缺失/损坏文件，再走 natives/command/启动。

### Step 5：验证并保存

执行完整流程 Step 13～14，不以历史 ready 证明本次启动成功。

## 停止游戏

### Step 1：定位进程

按本实例 taskId、PID、开始时间和程序路径核实身份；不对 PID 复用后的无关进程发信号。

### Step 2：正常退出

由玩家游戏内退出或宿主明确支持的正常退出方式处理。不要把任意 SIGTERM 当世界已保证保存。

### Step 3：必要时询问强制终止

仍未退出时说明存档风险，取得明确确认才强杀；未确认只报告仍在运行。

### Step 4：确认退出

取得真实退出码；无法取得记 null。确认没有关联存活进程后才清理本次独占 native 暂存，不删游戏数据。

### Step 5：持久化

更新当前 status=exited，保留历史 reachedReady，执行配置保存并报告实际结果。
