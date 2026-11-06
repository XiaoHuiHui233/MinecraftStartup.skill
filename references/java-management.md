# Java 管理

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

配置：`java.installations` 保存 `id/name/executable/version/architecture/sources/status`。来源为 `path/manual/downloaded`；状态为 `available/unavailable`，失败可另存脱敏 `error`。不改系统 PATH 或系统默认 Java。

## 搜索与刷新

### Step 1：读取范围

读配置和当前 PATH，按系统分隔符拆成目录；忽略空项。不扫描全盘，不能声称发现所有系统安装。

### Step 2：找出候选

逐目录检查 `java`（Windows 为 `java.exe`），不只取第一个。解析实际文件路径去重；同版本但不同实际安装保留。加上列表已有的手动/下载路径作复检。

### Step 3：执行检测

用参数数组执行 `[绝对路径, -XshowSettings:properties, -version]`，超时 15 秒，合并读取 stdout/stderr。退出码 0 且能识别真实 `java.version` 才有效；读取 `os.arch` 与 `java.home`。无法确认字段写未知，不猜版本。launcher stub 返回可确认 java.home 时，另查其 `bin/java` 并重新检测，成功才将其注册为实际可执行文件。

### Step 4：合并列表

以实际可执行文件去重；新记录分配稳定 ID，默认名为 `Java <版本>`。已有记录更新检测字段、合并来源，不改玩家名称/ID。失效已有记录标 unavailable，不自动删除；失败新候选不注册成可用。

### Step 5：保存并展示

执行配置保存，列出 ID、名称、位置、版本、架构、状态。无可执行 Java 时问“自动下载 Java，还是指定已有 Java 目录？”，返回 `WAIT_INPUT`；选择下载转下载模块，选路径转“添加”。

## 查看与查询

### Step 1：读配置

仅从磁盘读列表；玩家说“刷新”才调用搜索流程，不把缓存说成新检测。

### Step 2：筛选

有 ID 用精确 ID；否则名称/路径精确匹配、版本按玩家给定 major 或完整版本匹配。多个命中全部列出，不自动挑一个。

### Step 3：返回

展示匹配记录及“上次检测状态”；无匹配明确说明，不下载，不改配置。返回 `DONE`。

## 添加

### Step 1：获取路径

接受可执行文件，或在玩家指定的安装目录下查 `bin/java`、`bin/java.exe`；macOS JDK bundle 另查 `Contents/Home/bin/java`。缺少路径则提问并等待。

### Step 2：检测

执行搜索流程 Step 3。失败返回具体原因，不注册可用项。

### Step 3：去重

同实际路径复用已有 ID 并添加 manual 来源；否则分配 ID，按玩家给定名或 `Java <版本>` 命名。

### Step 4：保存

合并记录，执行配置保存；不绑定任何未指定实例。

### Step 5：验证

重读并展示 ID、真实路径/版本；返回 `DONE`。

## 修改

### Step 1：确定记录

按 ID 或唯一匹配选中；歧义时列候选并等待。

### Step 2：应用允许改动

显示名称可直接改；路径必须按添加 Step 2 检测成功才替换。版本/架构不可手填。失败保留原记录。

### Step 3：清除相关豁免

路径变化时，对引用该 Java ID 的实例将 `ignoreJavaCompatibility` 置 false；说明需要重新验证。只改名称不清除。

### Step 4：保存

执行配置保存，保留其他 Java 和模块。

### Step 5：验证

重读核对新名称/路径与检测结果，返回 `DONE`。

## 删除记录

### Step 1：定位

唯一确定记录；多项时等待选择。

### Step 2：说明范围

明确这是“取消注册”，不卸载、不删文件。若有实例引用，列出实例并确认是否清空它们的 Java 选择；未确认不删。

### Step 3：更新引用

移除该记录；引用实例 `javaId=null`、`ignoreJavaCompatibility=false`，其他实例不改。

### Step 4：保存

执行配置保存。

### Step 5：验证

确认记录已移除，文件仍在；提醒 PATH 中仍有的 Java 主动刷新时可再次加入。返回 `DONE`。

## 版本不符时让玩家选择

### Step 1：分别记录要求与实测

从实际版本元数据/loader 取得 major 要求，从可执行探测取得实测 major。没有已知兼容性依据时说明未知，不假定所有高版本兼容。

### Step 2：提出选择

问“该实例要求 Java X，目前为 Java Y。是否跳过版本检查尝试 Java Y？也可以下载要求版本或指定其他路径。”返回 `WAIT_INPUT`。不同 major 不等于该 Java 必然无法运行。

### Step 3：按回答处理

同意跳过：绑定实际 javaId，设实例 `ignoreJavaCompatibility=true`。拒绝：选其他路径或经确认下载。没有回答不能跳过。

### Step 4：保存范围

执行配置保存。仅对当前实例的 MC/loader/Java 组合有效；变更任一项重置 false。组合未变且已同意不反复询问，但每次报告保留风险警告。

### Step 5：继续门禁

跳过仅免 major 门禁，继续可执行性、native 架构、完整性与账号检查。它不是启动成功保证。

依据：[Prism IgnoreJavaCompatibility](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/launch/VerifyJavaInstall.cpp#L47-L90)。
