# 启动命令的构造与执行

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

本流程提炼自已实际进入主菜单的 `Minecraft 1.16.5 + Fabric Loader 0.20.1 + Java 21 / macOS ARM64` 路径。其他版本/平台须按真实 profile 计算，不把这个组合的主类、库、Java 或参数硬编码给所有实例。

**command 是 argv 数组，不是 shell 字符串。** 路径、名字和 token 都是某个数组元素的值，不额外包引号，不再次按空格 split，不经 shell 展开。

## 需要的输入

- `instanceRoot`：配置的数据根目录 / folders.instances / 实例稳定 ID。
- `gameRoot`：instanceRoot / `.minecraft`。
- `profile`：实例根 launch-profile.json，provider=prism，源组件/hash 与当前 MC/loader 一致。
- `javaExecutable`：实例 javaId 对应且已实际探测的可执行文件绝对路径。
- `account`：已选择账号的非秘密信息；微软秘密只由本地执行器通过 credentialRef 读取。
- `nativeDir`：本次任务在 gameRoot/natives/ 下的独占目录，已安全解压并确认架构。
- `loggingFile`：profile.logging.resolvedFile，存在且已验证；没有 logging 时不加日志配置参数。

任何必要输入缺失都返回调用它的准备流程，不用空字符串或别的实例路径补齐。

## 拼接 command

### Step 1：确定主类

直接取合成 profile.mainClass，必须非空。原版和不同 loader 的主类不同；本次实际验证的是 `net.fabricmc.loader.impl.launch.knot.KnotClient`。Fabric 实例不改成原版 `net.minecraft.client.main.Main`，不使用 `java -jar client.jar`。

### Step 2：构造 classpath

严格按 profile.libraries 的顺序收集每项 resolvedFile 的绝对路径，最后追加 profile.mainJar.resolvedFile。每项必须存在且属于该实例的实际下载结果。native-only JAR 不放入 classpath，不按文件系统目录顺序重排，不使用 `libraries/*` 代替递归库清单。

- Windows：以 `;` 连接路径。
- macOS/Linux：以 `:` 连接路径。
- 整个连接结果作为 `-cp` 后的**一个 argv 元素**，其中的单个路径不再额外加引号。

### Step 3：建立占位符映射

由已核对的数据生成下表，不从聊天历史拼路径：

| 占位符 | 来源与处理 |
|---|---|
| version_name | profile.minecraftVersion |
| version_type | profile.type |
| game_directory | gameRoot 绝对路径 |
| assets_root | gameRoot/assets 绝对路径 |
| assets_index_name | profile.assetIndex.id |
| library_directory | gameRoot/libraries 绝对路径 |
| natives_directory | 本次 nativeDir 绝对路径 |
| classpath / classpath_separator | Step 2 的完整字符串 / 对应系统分隔符 |
| auth_player_name | 当前账号真实游戏角色名；离线为已确认名称 |
| auth_uuid | 当前账号游戏 UUID，用工具转换为无连字符的标准十六进制形式 |
| user_properties | 没有已知旧用户属性时为 JSON 空对象字符串 `{}` |
| launcher_name / launcher_version | MinecraftStartup / Skill frontmatter.version |

未知的必需占位符停止并查对应版本格式，不能保留 `${...}` 原样传入、随便清空，或把 OAuth client ID 当游戏的 clientid。

### Step 4：在本地补充会话字段

微软账号的安全执行器读取凭据、核对到期时间/应用 ID，必要时走微软刷新链，不将秘密返回模型。填充：

- `auth_access_token`：有效 Minecraft access token，**不是** MSA access token 或 refresh token。
- `auth_session`：仅在该版本需要时取 `token:<Minecraft access token>:<游戏UUID>`。
- `user_type`：微软账号按所审 Prism 映射为 `msa`。

离线账号填充 `auth_access_token=0`、`auth_session=-`、`user_type=legacy`；这些不是在线授权，不用于伪装正版会话。若实际目标格式需要其他 user_type 规则，须单独核实，不声称跨全部历史版本一致。

所有在线占位符替换与进程创建在同一本地秘密上下文完成。普通配置、调试输出和示例中不写 token；玩家设备授权用的 user_code 不作为游戏参数。

### Step 5：构造 gameArgv

对 Prism profile.minecraftArguments **先切分模板、后替换变量**。本次已验证的 1.16.5 模板为：

```text
--username ${auth_player_name} --version ${version_name} --gameDir ${game_directory} --assetsDir ${assets_root} --assetIndex ${assets_index_name} --uuid ${auth_uuid} --accessToken ${auth_access_token} --userType ${user_type} --versionType ${version_type}
```

用宿主工具按该模板的引用规则解析；例如这个已知模板可使用 `shlex.split(template)` 得到列表，再逐个元素替换。**替换后的 gameRoot 即使有空格也仍是一项**。不把已替换的整条命令再次 split，不执行模板中的 shell 语法。未知转义规则进入异常处理。

### Step 6：构造 jvmArgv

按以下顺序建立列表，每行标注的参数是一项，`-cp` 和其值是两项：

1. 有 profile.logging 时，将 logging.argument 中的 `${path}` 替换为实际 loggingFile，作为一项。例如 `-Dlog4j.configurationFile=<loggingFile>`。Prism logging 直接是 argument/file，不是 Mojang logging.client。
2. macOS 且 profile.traits 包含 FirstThreadOnMacOS 时加入一项 `-XstartOnFirstThread`；Windows/Linux 不添加。
3. 依原顺序追加 profile.jvmArgs 的每个元素，用同一映射替换其中已定义占位符。
4. 加入一项 `-Djava.library.path=<nativeDir>`。
5. 加入两项 `-cp`、`<Step 2 构造的完整 classpath>`。

不要擅自追加“通用优化参数”或修改系统环境。profile.jvmArgs 若已含 classpath/native/日志等同功能关键参数，先解析它们及参数值：完全等价时只保留一份，冲突时停止核对，不能盲目重复。缺失内存配置时不发明一个已经被玩家选择的内存值；本次测试使用 JVM 默认内存策略。

### Step 7：生成最终数组

完整构造规则：

```python
# 示意变量全部来自前述步骤；不是向模型回传在线秘密的代码。
command = [
    javaExecutable,
    *loggingArgv,       # 没有日志配置时为 []
    *platformArgv,      # 本次 macOS 为 ["-XstartOnFirstThread"]
    *profileJvmArgv,    # profile.jvmArgs，逐项展开后保留顺序
    "-Djava.library.path=" + nativeDir,
    "-cp",
    classpath,
    mainClass,
    *gameArgv,
]
```

若 Step 6 已去掉等价重复项，此处采用其去重结果，不能为了照抄示意数组又加一次。确保每个元素均为字符串，没有未解决占位符。

**本次成功启动的脱敏形状**（尖括号是说明，不能直接执行）：

```text
<Java 21绝对路径>
-Dlog4j.configurationFile=<本实例实际日志配置文件>
-XstartOnFirstThread
-Djava.library.path=<本实例本次native目录>
-cp
<普通库绝对路径按顺序连接，最后是Minecraft主JAR；整体一项>
net.fabricmc.loader.impl.launch.knot.KnotClient
--username <实际角色名>
--version 1.16.5
--gameDir <本实例.minecraft绝对路径>
--assetsDir <本实例.minecraft/assets绝对路径>
--assetIndex 1.16
--uuid <游戏UUID无连字符>
--accessToken <仅在本地安全上下文中注入>
--userType msa
--versionType release
```

上述多行只是便于阅读的 argv 展示；每个 `--flag` 和其值实际分别占一个数组元素。不输出替换了真实 token 的展示字符串。

### Step 8：执行前检查

核对所选 Java 实际可执行；版本不符只有玩家已为该实例确认 ignoreJavaCompatibility 才放行。检查全部必需文件、classpath、assets、native 架构和令牌期限。跳过 major 检查不跳过这些检查。

检测目标实例是否已有启动任务/游戏进程，用 PID+开始时间+程序身份复核，不能打印其他进程完整 argv。已运行则报告并复用，不重复启动。

### Step 9：实际创建进程

使用宿主后台任务/进程工具，执行参数数组而非 shell 字符串，工作目录显式设为 gameRoot。等效的 Python 形状为：

```python
process = subprocess.Popen(
    command,
    cwd=gameRoot,
    stdin=subprocess.DEVNULL,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    shell=False,
)
```

本次 macOS 用 start_new_session=True 交由可观察的本地后台任务托管；Windows 不直接复制 POSIX 专用设置。宿主必须维持进程、持续排空日志管道、取得 PID/退出码；不能让 Agent 返回后因管道无人读取而阻塞，或因工具前台超时杀游戏。用户要求持续运行时选择相应受支持托管方式。

### Step 10：先脱敏再落日志

本地执行器持有本会话所有敏感值；读取完整日志行后逐值替换为 `[REDACTED]`，再写新的独立 launcher 日志。分块处理必须保留边界，避免 token 跨块漏出。禁止先保存原始输出再脱敏，也不把完整在线 command 打印到终端/任务输出。

本次使用后台日志读取线程持续处理合并后的 stdout/stderr，主线程监控存活/退出。记录 PID、开始时间、程序路径、taskId、脱敏日志位置，不记录秘密命令。

### Step 11：验证并持久化

按主流程观察运行：spawned 不等于 ready。记录真实日志中的版本/loader 证据，使用目标游戏窗口画面或玩家确认验证主菜单；不操作无关窗口，不强制切换焦点。

将 reachedReady=true 和验证证据作为历史事实保存。若游戏随后正常退出，保留当前 status=exited 和真实 exitCode，不为了补主菜单证据把已退出进程改回 running/ready。不要仅因测试进入主菜单就声称世界加载、联机和所有模组都已验证。

## 检查清单

- 主类来自合成 profile；classpath 全部实际路径且有主 JAR。
- JVM 参数在主类之前，游戏参数在之后；工作目录固定为目标 gameRoot。
- 占位符先分词后替换，最终 argv 不再 split，不添加多余引号。
- macOS 首线程参数只按平台和 trait 添加。
- native 解压排除 META-INF 元数据及声明的 exclude；不同实际 native 二进制同名冲突仍阻断。
- accessToken 为 MC token；未输出秘密；日志消费者持续运行。
- 只有真实主菜单证据才标 reachedReady，退出状态不被历史证据覆盖。
