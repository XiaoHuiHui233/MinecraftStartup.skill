# 启动配置解析

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

执行前遵循 [玩家运行能力契约](../SKILL.md#玩家运行能力契约)：只用已有专用工具或目标 OS 原生命令；缺能力 BLOCKED，不安装额外编程运行时/库（游戏 Java/材料下载例外）、不写临时脚本、不用 Computer Use 或 GUI 自动截图。游戏内操作和视觉验收由玩家完成确认。

默认 provider=prism，使用实例根 launch-profile.json；官方原始 JSON 是玩家明确选择时的兼容入口，不能混合解释字段。下载规划只生成清单，实际启动才检查全部文件及展开账号会话。

本文件负责识别 profile、rules 与待用字段；最终 argv 顺序、classpath连接、占位符替换及执行方式统一按 [启动命令](launch-command.md#拼接-command)，不得临时设计另一套 command。

## 解析启动配置

### Step 1：核对 provider 与来源

读取实例根 launch-profile.json；缺失则执行 [Prism 组件解析](prism-components.md#解析并合成实例组件)。核对 MC/loader、源组件哈希、运行 classifier 与当前选择；Java 架构变更须重新合成及检查 native，不用旧环境的库清单。provider 非 prism 且玩家明确选择官方兼容路径时才进入下方原始 Mojang 兼容解析。

### Step 2：执行组件库规则

调用 [Prism 运行环境规则](prism-components.md#运行环境规则)，取得适用普通/native 库和坐标路径，保留合成顺序。规划时返回下载清单，文件未到位不冒充验证成功；启动时逐项检查主 JAR/库/native 真实存在及已通过 hash。

### Step 3：构造 classpath 与 JVM 参数

普通库路径按合成顺序排列，最后加 mainJar 的实际 Maven 存储路径；Windows 用分号、其他用冒号。不要要求主 JAR 必须放 Mojang versions/<id>/<id>.jar，也不把 native-only JAR加进去。

以 profile.jvmArgs 为基础，补唯一 `-cp <classpath>`、实例任务 native 路径的 `-Djava.library.path=<路径>`；重复关键参数有冲突时阻断而非盲加。FirstThreadOnMacOS trait 在 macOS 添加 -XstartOnFirstThread；XR:Initial 在本 Skill 不额外注入任意命令。其他未知且影响启动的 trait/tweaker/agent/processor 进入异常核对，不能丢弃后声称完全兼容。

### Step 4：展开游戏参数

将 profile.minecraftArguments 用工具按元数据已知引用规则切成 argv，再逐项替换，不把它交给 shell。明确映射：version_name=minecraftVersion、version_type=type、game_directory=本实例 gameRoot、assets_root=本实例 assets、assets_index_name=assetIndex.id、library_directory=本实例 libraries、natives_directory=本任务目录；旧 game_assets 按实际索引布局。未解决 ${...} 返回 PLACEHOLDER_UNSUPPORTED，不随意塞空值。

### Step 5：安全展开账号与日志

auth_player_name/auth_uuid 使用选中角色，UUID 转所需无连字符格式；无已知旧属性时 user_properties 为 JSON 空对象。微软 token/session/user_type 由当前工具链按实际协议填充，不向聊天展示令牌；离线 token=0、session=-、user_type=legacy。不将公开 OAuth client ID 冒充游戏 clientid。

Prism profile.logging 直接取 argument/file，不是 logging.client。将实际日志配置文件路径替换 argument 的 ${path} 后作为 JVM 参数，不遗漏该项。

### Step 6：检查可执行结果

实际启动前检查 mainClass、全部文件、native 架构、参数占位符及实例范围。返回绝对 Java、JVM argv模板、mainClass、game argv模板、cwd、日志/native 路径；在线会话按需读取并在实际启动时展开，不要求专用安全执行器。计划模式仅返回清单，不调用游戏进程。

### Step 7：回到调用点

下载调用继续获取未齐全文件；主启动调用使用已经验证的完整参数数组执行。不要混用 Prism profile 与 Mojang 原始 arguments/rules，也不要求安装 Prism 的 NewLaunch.jar 才能执行标准原版 mainClass。

## 原始 Mojang 兼容解析

### Step 1：确认格式和范围

JSON.id 必须符合实例记录。当前标准路径支持官方原版 JSON；非原版必须有真实已安装且已确认格式的 profile，不使用原版 mainClass 冒充 loader。若含 inheritsFrom，先检查父文件及循环；没有该格式的已验证合并规则时返回 PROFILE_INHERITANCE_UNSUPPORTED，不让 Agent临时发明数组去重/覆盖规则。

原版须有 mainClass、客户端下载信息、libraries、assets/assetIndex 和游戏参数；缺项先核实是否属于已知旧格式，否则返回 PROFILE_UNSUPPORTED。此分支属于异常，需要查文档，不是通用猜测兜底。

### Step 2：建立运行条件

OS 名固定映射：Windows=windows、macOS=osx、Linux=linux。使用实测 OS 版本；arch 使用真实 Java 属性并保留原值，对需要常见别名匹配的格式仅用已证实映射。32/64 位与 ARM/x86 不能混为一个变量。

默认 features：is_demo_user=false、has_custom_resolution=false、quick-play 各项=false；其他 feature 默认未启用。玩家要求这些功能但无对应流程时停止确认，不把缺参数的 feature 置真。

### Step 3：执行 rules

对每个库/参数元素：没有 rules 就允许；有 rules 初始禁止，按数组顺序检查，每条所有条件均匹配才应用 action，最后匹配者生效。os.name 精确匹配，os.version/os.arch 按元数据正则在真实值上匹配；features 中每一布尔值都要与实际条件相等。action 仅 allow/disallow；未知条件、非布尔 feature、非法正则返回 RULE_UNSUPPORTED，不忽略。

### Step 4：收集客户端与库

下载清单使用已允许库的 downloads.artifact 和适用 classifier。classpath 依元数据顺序放普通 artifact，最后放实际 client JAR；不把 native-only classifier 加入 classpath，不按文件系统遍历顺序重排。路径必须在实例内。

native 依据 natives 中 OS 映射取 classifier，必要时按所用格式展开 ${arch} 位数；计划阶段先记录待核验架构，实际启动阶段必须检查其真实 native 二进制架构与 Java 一致。元数据仅有 x86 native 而 Java 为 ARM 时返回 NATIVE_ARCH_MISMATCH，不改成同名 ARM 文件、不借跳过 Java major 门禁放行。新格式已通过独立 artifact 提供 natives 时不要再虚构旧 classifier。

### Step 5：收集参数模板

现代 arguments.jvm/game：字符串原样作为一个 argv 元素；带 rules 的对象先跑 Step 3，允许时其 value 字符串加入一项，字符串数组逐项加入，其他类型停止。

旧式 minecraftArguments：用工具按该已知格式的引号规则解析为 argv，再替换占位符，禁止交给 shell 执行。原始参数含无法确认的引用/转义规则时进入异常处理。旧格式没有 JVM 参数模板时仅构造该已知版本必需的 native 路径和 classpath，不凭记忆加不明兼容 flags。

### Step 6：替换非秘密占位符

建立以下明确映射，逐 argv 元素替换；路径有空格仍是一项，不能再次 split：

| 字段 | 值 |
|---|---|
| version_name / version_type | 实际 JSON.id / type |
| game_directory | 目标实例绝对 gameRoot |
| assets_root / assets_index_name | 本实例 assets 绝对路径 / 实际 assetIndex.id |
| game_assets | 按实际索引建立的旧 virtual/resources 路径，不是猜的目录 |
| library_directory | 本实例 libraries 绝对路径 |
| natives_directory | 本次实例任务独占 native 解压目录 |
| classpath / classpath_separator | Step 4 构造的路径串 / Windows 为分号、其他为冒号 |
| launcher_name / launcher_version | MinecraftStartup / Skill frontmatter.metadata.version |
| auth_player_name / auth_uuid | 选中账号真实游戏名称 / 规范化为游戏所需无连字符 UUID |
| user_properties | 没有已知旧用户属性时使用 JSON 空对象字符串；不从私有账号文件读取 |

### Step 7：在本地展开会话

微软会话由当前工具链填 auth_access_token/auth_session 等实际所需字段，user_type 取已确认的 Minecraft 会话类型；离线使用 auth_access_token=0、auth_session=-、user_type=legacy。这不是在线凭据。

clientid、auth_xuid 或其他未在上表定义的字段，只能由已验证的对应协议/版本返回值与格式规则填充；没有就返回 PLACEHOLDER_UNSUPPORTED，不将公开 OAuth client ID 随便填成游戏 clientid。任何 ${...} 未替换均停止，不能原样传进游戏或用空串凑齐。

### Step 8：添加日志与必需平台条件

logging.client.argument 存在时，用实际下载日志文件替换其 ${path}，作为一个 JVM argv 元素。按元数据已允许的 macOS 首线程等条件执行，不对全平台一律添加。不得添加 metadata/包内要求的任意 shell hooks。

### Step 9：返回并核对

检查 mainClass 非空、classpath 文件齐全、native 路径归本实例、参数无未知占位符。返回参数模板、文件清单和不含秘密的摘要；完整在线 argv 不回显。安装阶段只生成清单，文件未齐全时必须调用下载，再重新运行相关检查。

Prism 使用转换后的组件 profile，本文件对官方原版参数进行明确解释，不声称它等同 Prism 的元数据格式。尚未验证的 loader/历史格式归入异常，不进行猜测兼容。
