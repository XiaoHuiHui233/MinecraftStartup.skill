# 世界管理

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

锁操作一律执行 [公共门禁](operations.md#写入前门禁) 的父级稳定锁→实例锁协议，整个操作持有，持锁后再次核对所有权；下文“检查发布锁/持锁”不是只检查不存在。子流程复用同一 operation 锁，不重复取得。

作用域仅 Java Edition 的 `<gameRoot>/saves/<目录>`；不支持 Bedrock、任意第三方编辑器或在线世界商店。这里给出可执行流程，不代表已在玩家世界实测。只读不登录、不写配置。世界的临时标识是实例 ID + 相对目录 + 本次 level.dat SHA-256，不新增 worlds 配置列表。文件级操作登记公共 operations/backups；数据包才登记 content.entries。

## NBT 工具与格式

所有写操作先遵循公共发布锁/实例锁检查，journal 固定为 `dataRoot/.management/journals/<opId>.json`，不放进世界/实例，不随删除或恢复移除。仅使用真实已注册实例 ID，找不到已删除 ID 时不得重建同名目录或分配墓碑 ID；恢复历史身份只能走 backups 的明确恢复选择。备份 sourceRelativePaths 相对 instanceRoot（世界为 `.minecraft/saves/<目录>`），content.relativePath 仍相对 gameRoot。恢复 world 或目录型 files 必须先列出快照后新增、将删除的文件并获确认，再在 staging 清空批准目录范围、按 manifest 全量重建并验证路径集合/hash；禁止把旧 region 覆盖混合进当前世界。本模块不另建覆盖式恢复捷径。

### Step 1：检查宿主能力

通过宿主 terminal 执行 `python3 -c "import importlib.util; print(importlib.util.find_spec('nbtlib'))"`，然后在一次性宿主 Python 进程核对实际 Python/nbtlib 版本。仅使用已审阅的一次性 Python + nbtlib，不在 Skill 中放运行程序。没有包：先询问是否批准在宿主隔离环境安装 nbtlib，说明环境/版本/命令与来源；未授权返回 `BLOCKED: DEPENDENCY_MISSING (nbtlib)`，不得自动 pip install。获准后由宿主执行已批准命令、验证 import，再返回调用方原步骤。若宿主连本地 Python 都没有，返回 `BLOCKED: CAPABILITY_MISSING`。只列目录仍可进行，但不得把未经解析者说成有效世界。以下片段由宿主工具执行，变量必须来自已验证上下文，不是直接粘贴占位符运行。

### Step 2：限量读取 Java level.dat

安装仅可在 Step 1 获得明确批准之后执行。一个可审阅的 macOS/Linux 宿主隔离环境方案是先确认新目录未被占用，再运行 `python3 -m venv "$HOME/.minecraftstartup/host-tools/nbtlib-2.0.4"`，随后 `"$HOME/.minecraftstartup/host-tools/nbtlib-2.0.4/bin/python" -m pip install --index-url https://pypi.org/simple 'nbtlib==2.0.4'`。批准范围须含依赖解析/下载（例如 numpy）；不执行 sudo，不装进游戏 Java/实例或 Skill 仓库。Windows 用宿主已探测 Python 的 argv 创建相同获批绝对目录，venv 解释器为 `Scripts/python.exe`；不要把 POSIX shell 命令直接交给 Windows。将实际解释器记作本次 `hostPython`，后续所有 nbtlib 片段均由它执行并用 `importlib.metadata.version('nbtlib')` 验证；安装失败/平台不兼容保持 BLOCKED，不自动换版本/源。此命令本次开发未执行。

输入 `level_path` 必须是世界根的普通 `level.dat`；先执行 [路径门禁](operations.md#写入前门禁) Step 1 的只读检查。限制文件 16 MiB，gzip 展开 64 MiB、解析进程 15 秒/宿主内存限制；超限返回 `BLOCKED: FORMAT_LIMIT`，不自动提高上限。执行：

```python
from pathlib import Path
import gzip, io, nbtlib
p = Path(level_path)
if p.stat().st_size > 16 * 1024**2:
    raise ValueError('FORMAT_LIMIT')
raw = p.read_bytes()
if raw[:2] != b'\x1f\x8b':
    raise ValueError('UNSUPPORTED_LEVEL_ENCODING')
with gzip.GzipFile(fileobj=io.BytesIO(raw)) as stream:
    payload = stream.read(64 * 1024**2 + 1)
if len(payload) > 64 * 1024**2:
    raise ValueError('FORMAT_LIMIT')
stream = io.BytesIO(payload)
doc = nbtlib.File.parse(stream, byteorder='big')
if stream.read(1) or not isinstance(doc.get('Data'), nbtlib.Compound):
    raise ValueError('INVALID_LEVEL_STRUCTURE')
data = doc['Data']
```

预期根为命名 Compound（保留原 root_name），`Data` 为 Compound；常见字段 `LevelName: String`、`LastPlayed: Long`（毫秒）、`GameType: Int`、`hardcore: Byte`、`DataVersion: Int`、`Version: Compound`。种子可能为旧 `Data.RandomSeed: Long` 或 `Data.WorldGenSettings.seed: Long`；缺失按 unknown，不猜、不修改种子。只展示类型正确且实际存在的字段。损坏时报告具体文件/异常，`level.dat_old` 仅作为另一个只读候选明确标注，不自动替换。NBT 字符串编码、未来格式、模组自定义 tag 未经过本环境往返验证时不宣称通用支持。

### Step 3：返回解析结果

返回解析状态、原文件 SHA-256、字段类型/值与文档对象到调用方；禁止使用 `with nbtlib.load(...)`，其退出可能保存源文件。写入必须保留整棵原 NBT、root_name、未知 tag 和原 tag 类型；不转 JSON 重建，不手拼二进制。改名后保存到独占 staging，显式 `doc.save(str(staged_level), gzipped=True, byteorder='big')`；再次限量解析并递归比较 typed tree，除授权字段外全部相等（数组用 dtype/形状/逐元素值比较，不用数组直接作 bool）；不能证明保留范围则 `BLOCKED: NBT_ROUNDTRIP_UNVERIFIED`，不覆盖原文件。

## 世界清单

### Step 1：定位只读范围

执行 [公共上下文](operations.md#输入与结果约定) Step 1～3，action=world.list，输入 instanceId 可选；只读选择不保存 selectedId。`saves` 不存在返回空清单，不创建目录；拒绝越界链接。

### Step 2：扫描与识别

用宿主 Python `Path(saves).iterdir()` 仅列直接子目录；对普通 `level.dat` 执行 NBT 工具 Step 1～3。逐项返回相对目录、LevelName、DataVersion/Version.Name、LastPlayed、大小及解析状态；没有 level.dat 的目录列作 unrecognized，不删除。缺 nbtlib 时可返回目录候选并明确解析 BLOCKED，不能把部分结果称完整有效世界清单。保留目录名与显示名的区别。

### Step 3：验证结果

重新核对已扫描 level.dat 指纹；扫描中变化标记 changed，不声称一致快照。返回实际目录数、有效数、失败/未知数以及是否游戏运行；无配置写入、登录、下载。运行中只读可能不一致。

## 复制与独立备份

### Step 1：选择目标

输入 instanceId、sourceWorld、mode=copy/backup；copy 还需新目录名，backup 还需明确备份范围。sourceWorld 必须从真实清单定位，不按 LevelName 猜重复项；新目录是单个合法路径组件，不允许空白、`.`、`..`、分隔符、平台保留名或大小写归一化冲突。copy 默认仅同实例，不跨实例复制。

### Step 2：停止与授权

执行 [写入门禁](operations.md#写入前门禁) Step 1～5，目标不存在才继续；目标冲突返回 WAIT_INPUT，不覆盖。源世界必须停止写入；缺 process/cwd 核验能力即 BLOCKED。锁定源文件清单、大小、SHA-256，拒绝 symlink、特殊文件、共享硬链接，包含隐藏文件、region/entities/poi、各维度、playerdata、datapacks、level.dat_old 等实际存在内容，不能只复制 level.dat。

### Step 3：执行真实复制

mode=backup：执行 [创建备份](backups.md#创建备份)，kind=world、sourceRelativePaths=[`.minecraft/saves/<源目录>`]（备份范围相对 instanceRoot，不同于 content 的 gameRoot 相对路径），复用当前锁；备份 verified 后回到 Step 4。mode=copy：在同文件系统 operation staging 下，执行 `shutil.copytree(source_world, staged_world, copy_function=shutil.copy2, symlinks=False)`，但仅在预扫描和提交前均已拒绝链接后使用；不用 os.link、symlink、clone 或引用列表冒充独立复制。执行前审阅这次 Python 脚本和真实路径，源树若变化则停止。

### Step 4：验证并提交

copy 对源/副本逐文件重算 SHA-256、大小、相对路径集合，确认文件不同 inode（适用平台）、没有链接；限量解析副本 level.dat。执行 [提交与恢复](operations.md#提交与恢复) Step 1～5；无冲突时把完整 staged_world rename 到新目录，新目标出现则停止。复制保持原 LevelName，若需改名另走改名流程，不暗改源。backup 读回归档及 manifest 实际校验结果，报告 backupId；普通 copy 不冒称已登记备份或经过恢复演练。

## ZIP 导入

### Step 1：取得输入

输入 instanceId、本地 zipPath、targetDirectory；URL 输入先执行网络门禁及 [单文件下载](downloads.md#单文件下载)，只对用户选定来源下载，不把 URL 当本地路径。读取 ZIP 中央目录，不执行内容；支持根 `level.dat` 或唯一顶层目录下的 `level.dat`，多个世界返回 WAIT_INPUT 选择单一根，未知布局 BLOCKED。

### Step 2：预检与门禁

先只读列出全部成员、数量与总展开大小；默认上限 100000 个文件、20 GiB 展开、单文件 4 GiB、压缩比 200:1，超限返回 WAIT_INPUT 讨论新限额，不能静默放宽。逐成员拒绝绝对路径、`..`、反斜杠/盘符/UNC、重复规范化名、大小写/Unicode 归一化冲突、加密项、symlink/特殊 Unix mode 和逃逸；禁止直接 extractall。候选世界根之外若有文件，展示排除清单并确认，不静默导入其他实例配置。执行写入门禁 Step 1～5；目标已存在则要求新目录，不合并/覆盖旧世界。

### Step 3：限量解压与验证

用 `zipfile.ZipFile`、`infolist()`、`open(info)` 与 `open(target,'xb')`，按已审核映射逐项 `shutil.copyfileobj` 或限量块循环，实时累计实际字节数并 enforce Step 2 限额，路径再次 resolve；目录逐层 mkdir，staging 私有，不跟随外部创建的链接。`ZipFile.testzip()` 和实际读取 CRC 通过后，对 staging 根 level.dat 执行 NBT 工具；无依赖授权或解析不支持则 BLOCKED，只保留暂存，不发布为世界。对展开目录生成路径/大小/SHA-256 manifest，源 ZIP 指纹变化则停止。

### Step 4：提交与验收

执行提交与恢复 Step 1～5；复核目标仍不存在，整目录 rename。重新扫描目标清单、逐文件核对 manifest，解析目标 level.dat，返回真实 saves 相对路径和状态。导入完成不代表版本兼容；DataVersion 高于目标或 loader 缺失时先警告，启动另需用户选择，不自动降级、认证或进入世界。

## ZIP 导出

### Step 1：确定范围与位置

输入 instanceId、world、exportPath；默认完整世界且不外发。目标必须明确授权、位于世界之外，不覆盖源，不包含备份/staging 自身。输出布局为 `<世界目录>/level.dat` 与该目录的全部内容，适配本文件导入。若游戏运行，返回 WAIT_INPUT 停止后导出；本流程不提供运行中尽力备份。只读列举可继续，但不能把列举/不一致快照称可恢复备份。

### Step 2：门禁与收集

执行写入门禁 Step 1～5；导出只写输出，停止要求用于一致性，不改源。覆盖旧导出须确认并备份旧文件。扫描源普通文件含隐藏项，拒绝链接/特殊文件，记录每个 SHA-256；提示世界含玩家 UUID/位置/服务器或模组信息，导出不是脱敏。

### Step 3：写 ZIP 并验证

用 `zipfile.ZipFile(staged_zip, 'x', compression=zipfile.ZIP_DEFLATED)`，对排序后的已确认文件逐个 `write(source, arcname=world_name + '/' + relative_posix)`；空目录可显式写目录项。关闭后用 `testzip()` 校验 CRC，逐项流读重算 SHA-256，与源 manifest/路径集合完全核对；重查源树无新增/删除/变化。CRC 不是内容 hash，不能代替比对。

### Step 4：提交结果

执行提交与恢复 Step 1～5，原子替换单个 ZIP，重开最终 ZIP 验 CRC/manifest 与实际输出大小/hash。只报告本地 exportPath，不声称上传/备份恢复测试；需要正式备份用独立备份流程登记 manifest 与 backupId。

## 改名

### Step 1：明确改哪种名称

输入 instanceId、world、新 displayName，可选 newDirectoryName。默认只改 `Data.LevelName`，不改目录；要改目录须明确新路径并确认，名称冲突返回 WAIT_INPUT，不加随机后缀。显示名必须非空 String，拒绝 NUL，目录名遵循复制流程的组件规则。

### Step 2：解析、停止与备份

先检查 NBT 工具能力，不复用之前列表/预读取得的 doc。执行写入门禁 Step 1～5，停机并取得稳定锁后，重新限量解析当前 level.dat；其 SHA-256 必须与锁定指纹及本次备份对应源文件 hash 一致，否则 BLOCKED: TARGET_CHANGED，重新解析/备份和规划，不用旧树覆盖最后一次保存。仅改显示名备份 level.dat，改目录则备份整个世界；使用这次锁后解析的 typed tree 作为唯一修改基线，提交前再次核对该原 hash。目标运行或状态未知则 BLOCKED。`level.dat_old` 保持原样，不当作本次独立备份。

### Step 3：暂存改动

保留整棵 typed tree，执行 `doc['Data']['LevelName'] = nbtlib.String(display_name)`，按 NBT 工具 Step 3 保存/读回验证；其余 tag、root_name 全保留。可选目录改名为单独 journal 项，涉及 content.entries 中已登记 datapack 的 `saves/<旧目录>/datapacks/...` 相对路径只更新本世界相关前缀，保留其他字段。

### Step 4：提交与后验

执行提交与恢复 Step 1～5；先原子替换 level.dat，再可选目录 rename，逐项记录，不能称两者整体原子。复扫新目录和 LevelName；只授权显示名时目录必须完全不变。后半失败保留备份，按逐项恢复规则处理，不以“改名成功”掩盖部分状态。

## 删除世界

### Step 1：精确选择

输入 instanceId、唯一世界相对目录；重扫列出 LevelName、完整路径、文件数/大小。世界删除无论原请求措辞都再次明确确认此世界及其玩家/区块数据，多个世界必须逐项列出并确认；未确认 WAIT_INPUT。

### Step 2：停机并备份

执行写入门禁 Step 1～5，必须成功创建 kind=world 的独立备份并验证 manifest。没有备份空间/能力就 BLOCKED，不以 level.dat_old 或 session.lock 充当备份。确认最新目录树指纹和实例已停机。

### Step 3：删除与登记

用 `shutil.rmtree(validated_world_path)` 只移除已确认普通目录树，提交前再查链接/逃逸与树指纹；不 glob 整个 saves。逐文件记录结果；清除本世界已登记 content 项而非其他世界，保留 backup 与操作记录；部分失败标 recovery_required，不继续批量删。

### Step 4：验收

确认目标目录不存在、其他世界清单/hash 未变化，执行提交与恢复 Step 3～5，返回 backupId 和 [恢复入口](backups.md#恢复备份)。恢复操作另行确认；不宣称删除天然可撤销。

## 世界 datapacks

### Step 1：定位与列举

输入 instanceId、world、action=list/add/remove、本地 sourcePath 或准确目标名。先执行公共上下文和世界识别，再只读扫描 `saves/<world>/datapacks` 的直接 ZIP/目录。目录缺失只读返回空。ZIP 用 zipfile 检查安全成员、CRC、根 pack.mcmeta；目录验证普通文件及根 pack.mcmeta。JSON 必须实际解析，展示 pack.description 与 pack.pack_format/版本所需 supported_formats，未知版本 schema 返回 unsupported，不自动改格式。

### Step 2：区分文件与游戏启用状态

只读展示 `Data.DataPacks.Enabled/Disabled` 中实际 String 列表（若存在）；内置/模组包、`file/<名称>` 不能当普通本地文件删除。list 此处返回，不写配置。add/remove 不等于 `/datapack enable/disable`；本模块不编辑这些 NBT 列表，不自动 `/reload`，不使用 `.disabled` 后缀宣称 vanilla 数据包已禁用。需要游戏内切换时说明需玩家在世界中执行有权限的 `/datapack list`、`/datapack enable "file/<实际名称>"` 或 `/datapack disable "file/<实际名称>"` 并观察真实返回；未执行不报生效。

### Step 3：文件增删

执行写入门禁 Step 1～5，变更范围仅该世界 datapacks；添加冲突 WAIT_INPUT，删除或覆盖前备份相应文件树。添加用 copy2/copytree 到 staging，校验文件路径/hash/pack.mcmeta，提交逐项 rename；删除用 unlink/rmtree 精确目标。内容来源默认 local，不假装平台下载；在线搜索/更新不在本流程内，不能把全局 datapacks 写进世界或反向操作。

### Step 4：验证文件状态

执行提交与恢复 Step 1～5，重新扫描并合并 `content.entries`，kind=datapack、relativePath=`saves/<world>/datapacks/<文件或目录>`、provider=local、hashes 为本地快照。enabled 与公共 schema/资源模块一致，只表示经过结构校验且未隔离的磁盘可扫描状态，不表示游戏是否启用；合法落盘填 true，明确隔离填 false，未经格式校验的未知文件不强行登记。游戏侧启用状态保存在 content-state/<content-id>.json 的激活证据中，未观察为 unknown；不能因游戏激活未知就阻断已明确授权的文件导入。报告文件已管理但游戏加载未验证。

## 启动世界入口

### Step 1：只读选择目标

输入 instanceId、world、明确启动意图；从实际目录与 NBT 定位，显示名不能代替目录参数。读取最终 profile.traits，需 `feature:is_quick_play_singleplayer`；缺 trait/参数格式未核实则 WAIT_INPUT，提供启动主菜单后手动选择世界，不偷偷尝试旧参数。

### Step 2：形成一次性启动目标

玩家同意启动世界后进入 [启动游戏](launch.md#启动游戏) Step 1～10 的准备，保留本次目标，不在此写永久启动偏好。任何版本高于/低于世界 DataVersion 的转换、尤其降级风险先说明并确认，启动前执行 world 备份；实例运行中则让玩家在现有客户端选择世界，不向未知进程注入命令。网络步骤均先执行公共网络策略。

### Step 3：接入唯一 argv 流程

在 [启动命令](launch-command.md#拼接-command) Step 5 完成模板 gameArgv 后、Step 7 最终组装前，审核本次最终 profile 允许此 trait，追加 `['--quickPlaySingleplayer', world_directory_name]` 两项；名称是 saves 下单个目录而非绝对路径。若已有任意 quickPlaySingleplayer/quickPlayMultiplayer/quickPlayRealms 或 server 目标，等价去重、冲突 BLOCKED，不能并列注入。在启动游戏 Step 11 的构造回调中完成本次注入后立即返回 gameArgv；不在本模块执行启动命令 Step 6～11。由启动游戏 Step 11 完成命令 Step 6～8，再仅由启动游戏 Step 12～14 创建一次进程、观察并持久化，不重复调用进程创建。账号、Java、进程与日志合同保持不变。

### Step 4：验证进入世界

返回真实 PID/状态、画面或玩家确认世界已加载的独立证据；主菜单、存活进程和启动参数仅证明发起了目标，不证明进入世界。没有世界画面则报告 launched/entry_unverified，失败读脱敏日志，不反复强开。只读列表不会进入本流程。

## 来源与支持界限

- PrismLauncher `899d2f3957215a09fe398a8a8cbb8a8a9c619968`：`launcher/ui/pages/instance/WorldListPage.cpp:226–304,438–535`（删除/世界数据包/ZIP/复制/改名/trait）；`launcher/minecraft/World.cpp:335–377,449–568`（NBT 改名与字段）；`launcher/minecraft/MinecraftInstance.cpp:768–786`（quick-play 参数）。本 Skill 的严格停机、独立备份与 ZIP 导出为自身安全流程，不伪称均为上游按钮。
- 格式/API：Java level.dat gzip、大端命名 NBT Compound；[nbtlib File/load/save 实现与示例](https://github.com/vberlier/nbtlib/blob/main/nbtlib/nbt.py)。只支持已识别的 Java 格式，不支持 Bedrock little-endian/LevelDB、通用区块修复、跨版本转换或自动运行外部工具。
- 验证边界：执行时重新检查宿主 nbtlib 与实际版本；缺依赖按前述门禁阻断。本流程与片段不构成真实 NBT 往返或世界加载的完成证据。
