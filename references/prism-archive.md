# Prism 式整合包解包

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices: ../NOTICE.md; license: ../LICENSE. -->

本子流由 [整合包导入 Step 2](modpacks.md#导入整合包) 调用；输入 archivePath、mode=inspect/stage、operationId、stagingRoot、rootPrefix、caller/returnStep，完成后回原步骤。不启动游戏、不注册 installed、不改变账号或实例选择。遵守 [玩家能力边界](../SKILL.md#玩家运行能力契约)，使用已有归档工具或原生命令，不编写 helper、不安装启动器。

## 与 Prism 对齐的实现依据

固定上游 commit `899d2f3957215a09fe398a8a8cbb8a8a9c619968`：

- `launcher/InstanceImportTask.cpp:129–211`：ArchiveReader 扫描格式签名，选择 staging 目标，再调用异步 ExtractZipTask；成功才进入平台导入。CF 根 manifest 的识别不要求先解析每个 Mod。
- `launcher/archive/ExtractZipTask.cpp:33–107`：collectFiles 后重新逐条遍历，按 subdirectory 过滤、转换相对路径、检查 target_top_dir 包含关系，再建目录/writeFile；取消或失败时删除 extracted 列表里的文件。这不是完整目录事务回滚。
- `launcher/archive/ArchiveWriter.cpp:231–247`：libarchive disk writer 使用 TIME/PERM/ACL/FFLAGS 和 SECURE_NODOTDOT/SECURE_SYMLINKS。
- `launcher/archive/ArchiveReader.cpp:122–224`：archive_read_data_block → archive_write_data_block 复制，检查读取/写入/完成状态；willEscapeRoot 检查软/硬链接目标是否逃出根。Prism 不一概拒绝根内链接。
- `launcher/archive/ArchiveReader.cpp:65–80,227–265`：readAll 聚合成员字节，parse/skip 逐成员处理；这里没有本 Skill 旧版本要求的流式总量计数器、NFC+casefold 冲突索引、固定压缩比门禁或有界 JSON 内存接口。不要把这些额外要求说成 Prism 的前提。
- `launcher/InstanceImportTask.cpp:214–255`、`launcher/modplatform/flame/FlameInstanceCreationTask.cpp:375–409`：解包后修正用户读写权限，CF 读取/保存 manifest、检查 overrides 没有逃逸并移动到游戏目录，再继续文件解析下载。

Prism 使用 C++/Qt/libarchive；Skill 用可核实的宿主接口重现文件流程，不要求运行 Prism 或编译其代码。下文 Windows、Linux、macOS 适配均优先选择经实测识别为 libarchive 的 tar/bsdtar，但不是声称两个程序完全等价：它拒绝 `..`、默认剥去绝对路径前导 `/` 并防止通过中间软链接写出；不能代替 Prism 对链接本身目标的 willEscapeRoot。因此本适配在展开前保守拒绝所有链接/特殊条目，后续也不复制链接；这是明确的宿主适配差异。默认不恢复 owner/ACL/xattr/flags，仅保留正常文件和游戏所需访问权限。备份/世界等模块的专属契约不因本子流而自动放宽。

## 执行

### Step 1：核对输入、工具与资源

先识别**实际执行工具所在的 OS**，不是玩家所在地或聊天客户端。Windows、Linux、macOS 都是正式目标，按下表选择适配；三端共用后续 Step 2～5，不能将任一平台命令作为跨平台默认值。源必须为玩家指定的普通非链接文件；记录 size/mtime/SHA-256（本地 hash 只作快照）。

| 目标 OS | 只读发现与选择 | 文件/JSON/hash 的已有原生能力 |
|---|---|---|
| Windows | 在系统原生 PowerShell 中 `Get-Command tar.exe -CommandType Application`，优先核对系统目录内候选，再执行该绝对路径 `--version`、`--help`；只有确认输出标识 bsdtar/libarchive 且支持 ZIP 才设 archiveExecutable。不要假定任何叫 tar 的程序都相同。 | `$HOME`/`Join-Path` 定位路径；`Get-Item -LiteralPath`、`Get-ChildItem -LiteralPath` 核对普通文件及 ReparsePoint；`Get-FileHash -LiteralPath ... -Algorithm SHA256`；`ConvertFrom-Json -ErrorAction Stop`。不要求 jq、Bash、WSL、Homebrew 或 GNU 命令。 |
| Linux | `uname -s` 确认 Linux，`command -v bsdtar` 后核对版本/帮助；找不到再核对已有 `tar --version`，仅 libarchive 实现可复用此适配。**GNU tar 不作为 ZIP/mrpack/JAR 解包后端**，不能照抄 bsdtar 参数假称可用。 | 使用宿主已有结构化文件/JSON工具；原生命令用 `stat`、`df`、`sha256sum`，参数按本机手册；已有 jq 可解析 JSON，缺 jq 不先要求安装编程运行时。 |
| macOS | `uname -s` 确认 Darwin，核对 `/usr/bin/bsdtar --version` 和 `--help`，确认 libarchive 与 ZIP 支持后选择其实际路径。 | 使用宿主已有结构化文件/JSON工具；原生命令用 `stat -f`、`df`、`shasum -a 256`；已有 jq 可解析 JSON。不把 Darwin 的 `stat -f` 复制给 Linux。 |

三个平台均先核实 `-t/-v/-x/-O/-C/-f` 与路径防护；也允许宿主已有专用归档工具按同一输入/输出与安全契约实现，不强制某个可执行文件名。选定后携带 `targetOS/archiveExecutable/archiveImplementation/archiveVersion` 到每一步；不要在后续偷偷退回 `/usr/bin/bsdtar`。某目标仅有不支持 ZIP 的 GNU tar、缺 tar.exe、或缺清单/类型/写时路径防护时，返回具体缺口及现有候选，不说“Linux/Windows 不支持”，不擅自安装软件或改用其他 OS。`Expand-Archive` 不能仅凭能解 ZIP 就代替类型预检和链接防护。

调用均为 argv 数据，路径作为单独实参传递；专用工具优先 shell=false。只有原生 shell 时：Windows 用 PowerShell 调用运算符 `&` 和字符串参数数组，POSIX shell 逐实参转义；不 eval、不拼接执行包内文本。Windows 的外部程序调用后立即读取 `$LASTEXITCODE`，不能只看 `$?`；Linux/macOS 读进程退出码或 shell `$?`。

检查剩余空间和工具能提供的声明数量/展开大小。旧 import-plan 的 limits 作为预检告警阈值，不是证明宿主有流式配额强制器；超阈值先 WAIT_INPUT。正常包不因缺少流式配额、Unicode 归一化索引、JSON 深度/内存预算接口而阻断；不要伪称这些能力已经验证。异常体积、可疑条目或用户明确要求的硬限额仍需停止处理；工具不能满足用户硬限额时报告该具体缺口。无需递归展开嵌套 JAR/ZIP。

### Step 2：枚举成员与检查目标范围

使用真实完整清单，三端均以 Step 1 选定的 archiveExecutable 调用 argv `[archiveExecutable, "-tf", archivePath]` 和 `[archiveExecutable, "-tvf", archivePath]`；Windows 同样调用已验证的 tar.exe，不切换为 POSIX shell。工具输出过长保存到任务审计，不以截断聊天输出判断全部通过。Linux/macOS 先核对可用 UTF-8 locale，必要时仅给本次子进程设置已存在的 UTF-8 locale（例如确认可用后设置 LC_ALL=C.UTF-8）；Windows 核对外部程序输出的实际 UTF-8 解码，不依赖系统默认代码页。不能把正常中文路径在非 UTF-8 locale 下的转义显示误判为恶意路径。普通换行分隔清单只适用于能确定不含换行/控制字符的名称；转义/不可判读名称停止该包，不能用字符串反解猜名字。核对名称及类型，拒绝绝对路径、驱动器/UNC、`..` 路径组件、控制字符、软/硬链接及设备/FIFO/socket 等特殊条目；Windows 另拒绝 ADS 冒号、保留设备名、尾随点/空格及 ReparsePoint/junction，路径比较按目标文件系统的大小写规则，不能只比字符串前缀；根 `./` 可剥一次并记录。不擅自修复恶意路径。对清单中实际出现的重复文件、文件/目录冲突及宿主可识别的大小写等价冲突报告并停止，不要求为普通包额外提供全 Unicode 跨平台归一化引擎。

确认目标根及祖先不是根外链接，inspect 不展开整包；stage 目标必须为本 operation 私有、空、非链接目录，不能直接对现有实例解包。解压工具自身必须保持路径和 symlink 防护，不能仅用预检替代写时防护。只有在已确认格式包含单层包装目录时，才能使用经核实的 strip-components 或先解到私有隔离根再移动已验证子目录；不从归档自行接受输出目录参数。

### Step 3：读取格式清单并返回只读计划

先确认所需清单恰好一个、类型为普通文件及声明大小正常，再通过成员输出接口读取完整字节，例如 argv `[archiveExecutable, "-xOf", archivePath, "manifest.json"]`。检查工具真实退出状态与 UTF-8/JSON 解析结果；格式/必需字段错误停止，不把 stderr 混入 JSON。默认 1 MiB 是异常大小告警阈值，不要求专用流式 JSON 解析器；超过先询问。Windows 使用已有原生 PowerShell 的 ConvertFrom-Json，Linux/macOS 使用宿主 JSON 工具或已有 jq，不额外运行 Python/Node。Windows 外部命令 stdout 的文本捕获可能改变编码/换行，原始 manifest hash 必须取工具无损文件输出或 staging 内原文件；未取得原始字节就标 hash pending，不能把重新编码的字符串 hash 冒称源成员 hash。具有重复键诊断能力时使用；缺该诊断接口不独自阻断普通清单，不宣称已完成重复键检测。已观察到的重复关键字段/不明确结构停止，绝不将多个同名清单拼接后按最后一个接受。

返回格式、原清单字节/hash、名称、MC/loader、实际工具/检查和未提供的增强保证，回导入 Step 3；mode=inspect 到此结束。CF 精确文件解析继续调用 [CF 子流](curseforge-import.md)，不要把 overrides 解包完成当成整个包下载完成。

### Step 4：持锁展开到私有 staging

仅在父导入 Step 6 已确认计划、取得父稳定锁→实例锁并给定 stagingRoot 后进入。复核源 size/mtime/hash 与 Step 1 一致，目录仍为空且无链接/reparse point。三端执行相同的基础 argv：

```text
[archiveExecutable, "-xf", archivePath, "-C", stagingRoot]
```

按 Step 1 的**本机帮助/版本**补充已支持的权限选项，禁止把 macOS 专属参数无条件复制到其他 OS：

| 平台 | 选项与调用要求 |
|---|---|
| Windows | 使用已核实的 tar.exe 完整路径；支持时添加 `--no-same-owner`、`--no-same-permissions`，不用 POSIX chmod 或 macOS metadata 开关；依靠私有 staging 的正常用户 ACL，不恢复归档 owner/特权权限。PowerShell 调度示意：`& $archiveExecutable @archiveArgv`，archiveArgv 是从可信基础数组构造的字符串数组，不含 executable；立即检查 `$LASTEXITCODE`。 |
| Linux | 使用选定的 libarchive 后端；支持时添加 `--no-same-owner --no-same-permissions --no-acls --no-xattrs --no-fflags`。不添加 `--no-mac-metadata`，不执行仅支持 tar 格式的 GNU tar 来处理 ZIP。 |
| macOS | 使用选定的 libarchive 后端；支持时添加 `--no-same-owner --no-same-permissions --no-acls --no-xattrs --no-fflags --no-mac-metadata`。 |

不支持的增强选项可省略，但必须核对正常用户下的实际默认行为仍不恢复特权权限；若无法保证，使用已有专用工具或具体阻断。源路径和 staging 路径都取调用帧的绝对路径，中文/空格保持单独参数；Windows 不把反斜杠当 POSIX shell 转义。

不使用 `-P/--absolute-paths`、`-U/--unlink-first`、sudo、外部解压程序或包内命令。不加 `-k` 掩盖归档冲突；冲突应在 Step 2 报告。只有本机支持的选项才传递，其他平台须先核对接口。stdout/stderr、实际退出状态写任务审计；长任务由已有后台进程工具监督。成员 CRC/数据错误或任何非零退出均停止；不能因部分文件出现就继续发布。默认保持工具正常校验，不添加 ignore-CRC/忽略错误选项。

### Step 5：检查输出并返回

按 Step 1 的本机文件接口重新枚举 staging（Windows 包括 Hidden 项，Linux/macOS 包括点文件），不跟随链接/reparse point；核对选中成员文件集、实际大小、路径范围、链接/特殊文件、清单字节及 overrides 存在性；异常则隔离，不复制到游戏实例。对所需内容计算最终 hash，独立的上游 hash 检查仍由下载/CF 模块负责。按包格式规定层序复制/移动 overrides 和下载文件，不把所有源档文件盲目当游戏根；不执行 shell/bat/PS1 或未知 JAR。失败只处理本 operation 私有目录中的已知产物，不清理其他路径，不称已完整回滚。

成功返回 archive_staged（及真实检查/日志路径）到父导入 Step 6，继续游戏组件、全部必需 Mod 和官方 loader 安装；只有父流程最终验收通过才 installed。不自动启动。
