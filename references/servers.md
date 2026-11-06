# 客户端服务器列表

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

执行前遵循 [玩家运行能力契约](../SKILL.md#玩家运行能力契约)：只用已有专用工具或目标 OS 原生命令；缺能力 BLOCKED，不安装额外编程运行时/库（游戏 Java/材料下载例外）、不写临时脚本、不用 Computer Use 或 GUI 自动截图。游戏内操作和视觉验收由玩家完成确认。

锁操作一律执行 [公共门禁](operations.md#写入前门禁) 的父级稳定锁→实例锁协议，整个操作持有，持锁后再次核对所有权；下文“检查发布锁/持锁”不是只检查不存在。子流程复用同一 operation 锁，不重复取得。

只管理指定 Java 实例的 `<gameRoot>/servers.dat`，不是服务端部署、控制台、白名单或凭据管理。共七个玩家操作：清单、新增、编辑、删除、排序、状态、启动并加入。只读不登录、不写配置；列表不是 config.json 中第二份服务器库。行标识使用本次文件 SHA-256 + 零基 index，重名/重地址允许存在，不用名称静默去重。

## NBT 读写子流程

遵循公共发布锁/实例锁与持久日志合同：`journalPath=.management/journals/<opId>.json` 相对 dataRoot，绝不存放在待删除/恢复的实例内。实例 ID 必须来自当前注册记录；不复用 instances.retiredIds 或历史备份/operation 引用的 ID，不根据旧备份猜当前目标。恢复 servers.dat 通过 backups 单文件范围 `.minecraft/servers.dat`；若用户选择整个目录范围，则先列出快照外新增文件的删除确认，再在 staging 清空该批准范围并完整重建，不用目录覆盖合并充当恢复。

### Step 1：检查具体依赖

按 [世界 NBT 工具](worlds.md#nbt-工具与格式) Step 1 探测已有 typed NBT 工具，本模块额外要求未压缩/大端/空根名及完整保留列表元素类型。缺能力 `BLOCKED: CAPABILITY_MISSING (servers NBT)`；不安装库、不运行临时脚本。子流程返回调用步骤，不要求微软登录。未解析的数据不当空列表覆盖。

### Step 2：读取精确数据结构

输入 validated servers_path。文件不存在时查询返回空且 exists=false，不创建；只有新增计划可构造空根。支持未压缩、大端、空名根 Compound，根 `servers: List[Compound]`；每项 `name: String`、`ip: String`，可选 `icon: String`（原样保留）、`acceptTextures: Byte`。该字段**缺失=询问、1=始终接受、0=从不**；显式其他类型/值阻断该条编辑，不转成默认。其他根 tag/条目 tag 原样保留，包括未知的隐藏/资源包字段，不重建整个条目丢字段。向专用工具传入已验证真实路径、只读、未压缩/大端/空根名、16 MiB 文件限额；拒绝 gzip、尾随字节及根/字段类型不符。列表元素类型必须为 Compound；仅空列表允许原 TAG_End subtype（追加时的显式类型变更见下文）。不能提供这些格式校验则 BLOCKED。



受宿主 15 秒/内存限额约束；损坏/不支持返回 `BLOCKED: NBT_FORMAT_UNSUPPORTED` 并保留源。不得尝试文本替换、手拼二进制或从未经确认的 servers.dat_old 恢复。可以列只读部分错误，但必须明确 incomplete。

### Step 3：构造授权差异

在取得停机/稳定锁后，必须确认当前文件 SHA-256 与最初解析 doc 的 SHA-256 完全一致（含首次不存在的状态）；变化就重新执行 Step 2、重新确认所选行与备份，不能拿旧 doc 配新门禁指纹。对这个锁后可证明有效的 doc 通过专用 NBT 工具克隆完整 typed tree 保留 root_name 和全部 typed tag；只改玩家指定字段/行/顺序。首次新增才构造未压缩、大端、空名根 Compound，含空的 Compound 类型 servers 列表。未知 tag 不经过 JSON 中转；icon 不自动下载/刷新。行选择必须匹配原 SHA-256 与 index；用户资源包偏好未指定时完全保留既有 acceptTextures，不给 absent 补 0。

### Step 4：暂存并保留类型验证

写入必须已经取得公共门禁许可和覆盖前备份。输入独占且同文件系统 `staged_path`，由专用 NBT 工具写未压缩/大端/原根名暂存文件，fsync 暂存；重新执行 Step 2 解析暂存并比较根名、tag 类型、值及列表顺序。对比基线树，除明确授权新增/删除/修改/排列路径外，其他 tag 必须完全一致；数组按 dtype/shape/逐元素比较，无法完成则 `BLOCKED: NBT_ROUNDTRIP_UNVERIFIED`。压缩/大小端不做自动转换。

### Step 5：单文件原子提交

执行 [提交与恢复](operations.md#提交与恢复) Step 2～5：再次检查实例停止、源 hash/存在性与锁；用已有安全文件工具同文件系统原子替换 staged_path → servers_path，随后尽可能 fsync 父目录（宿主不支持时说明耐久性限制），重新读最终路径验证本次完整目标。只报单文件原子替换，不称文件+config+backup 为整体事务。备份用 [创建备份](backups.md#创建备份) kind=files、sourceRelativePaths=['.minecraft/servers.dat']（相对 instanceRoot）；缺失文件首次新增不伪造旧备份。servers.dat_old 留原样，绝不是唯一备份。登记公共 operations 日志与 backupId，不新增持久字段；原文件后验异常按公共恢复流程停止，保留材料。

## 查看服务器

### Step 1：定位实例

输入可选 instanceId，执行 [公共上下文](operations.md#输入与结果约定) Step 1～3，action=server.list；范围仅该 gameRoot，不扫描其他启动器文件。

### Step 2：解析与展示

执行 NBT 子流程 Step 1～2，返回 exists、文件 hash、实际条目总数与每行 index/name/ip/acceptTextures 三态、icon 是否存在；不展示完整图标大字符串。地址仅本地显示，不自动 DNS/联网、不更新 icon，也不注册账号。

### Step 3：只读验收

复查文件指纹；变化则标 changed 并要求后续编辑重选。文件缺失返回空不保存；依赖缺失准确报告 BLOCKED，不报告零服务器。运行中只读标可能不一致，不把临时在线人数写回 NBT/config。

## 新增服务器

### Step 1：取得输入

输入 instanceId、name、address，可选 packPolicy=ask/always/never。名称非空，不含 NUL；地址是玩家输入的主机名/IPv4/`[IPv6]`，可带合法端口 1～65535，无端口使用游戏默认 25565；不接受 URL、路径、用户信息或任意控制字符。不为保存列表发 DNS；原地址保留，展示规范化 host/port 用于确认而不默默改 token。缺名称/地址返回 WAIT_INPUT。

### Step 2：预读与写入门禁

执行 NBT 子流程 Step 1～2 和 [写入门禁](operations.md#写入前门禁) Step 1～5；目标运行/状态未知停止。现有同地址条目展示给玩家，但不要去重删除。默认新增到末尾，插入其他位置需明确 index。packPolicy 未指定采用 ask（字段缺失），在计划中显示。

### Step 3：追加具体 Compound

深拷贝原树或明确首次构造空根，通过专用 NBT 工具创建仅含 name:String、ip:String 的新 Compound 行；always 写 acceptTextures:Byte=1，never 写 Byte=0，ask 不写此 key。兼容 vanilla 空列表的 TAG_End subtype：只在列表确实为空且本次要追加时，将 servers 明确换为 Compound 元素类型的空列表，此类型变化列入授权差异；非空错误 subtype 不修复。然后将新行追加到 servers，其余字段/行不变。

### Step 4：提交与后验

执行 NBT 子流程 Step 4～5，最终读回条目数应恰为原数+1、新条目与确认输入相同、旧条目 typed tree 和顺序不变；返回实际新增 index/hash。保存列表不等于服务器可达或已加入。

## 编辑服务器

### Step 1：选择与变更集

输入 instanceId、原文件 hash/index，以及显式 name/address/packPolicy 中至少一项。先清单定位，重名不猜。只改给出的字段；packPolicy 不给保留原值；ask 才明确删除 acceptTextures key。

### Step 2：门禁与备份

读取完整原树，展示旧/新字段和值；地址校验同新增，执行写入门禁 Step 1～5 并备份完整 servers.dat。记录本次行身份；不因 index 仍存在就接受已变更文件。

### Step 3：精确修改

深拷贝后对 servers 的精确 index 行设置实际 String 字段；packPolicy=always/never 设置 Byte(1/0)，ask 仅删除该行 acceptTextures 字段（本来缺失则不变）。保留 icon、未知 tag 及未授权字段，不直接用新 Compound 替代旧行。

### Step 4：验证提交

执行 NBT 子流程 Step 4～5；行数/顺序不变，其余行和本行未改字段完全保留。后验显示新的三态而不是把 missing 说 never；失败返回阶段、已应用状态、backupId。

## 删除服务器

### Step 1：冻结精确行

输入 instanceId、hash 和一个/多个 index；列出每项 name/ip，批量必须明确确认，单项仍显示实际删除目标；未选定返回 WAIT_INPUT。索引不能重复或越界，不把同地址其他条目一起删。

### Step 2：门禁与备份

执行完整写入门禁 Step 1～5，必须成功备份完整 servers.dat；源 hash 变化则 BLOCKED: TARGET_CHANGED，重新展示而非沿用旧 index。

### Step 3：删除并原子提交

深拷贝原树，对已确认 index 以降序删除对应行；空列表仍保持 `List[Compound]` 类型。执行 NBT 子流程 Step 4～5，不删除整个 servers.dat 或 icon 缓存目录。

### Step 4：验收

确认最终数=原数-去重后的选中数、剩余项按原相对顺序且 typed tree 不变；返回真实数量与备份入口，不报告服务端被删除。

## 调整排序

### Step 1：明确排列

输入 instanceId、hash，及 fromIndex/toIndex 或包含全部旧 index 的 newOrder。展示前后顺序；newOrder 必须恰好包含从 0 到原行数减一的每个整数各一次，不遗漏、不复制，输入错误 WAIT_INPUT。

### Step 2：停止、备份、执行

执行写入门禁 Step 1～5。通过专用 NBT 工具从原列表取出 fromIndex 行并插入 toIndex（移动后的零基最终位置）；全排列按 newOrder 的旧 index 顺序引用原 typed 行，写回 Compound 元素类型列表，不重建行字段。

### Step 3：提交与验收

执行 NBT 子流程 Step 4～5；验证行数、每条 typed tree 与出现次数不变，精确顺序等于确认排列；同值重复条目也按原 index 跟踪，不能用 name/ip 的 set 代替排列检查。

## 查询服务器状态

### Step 1：解析目标与网络策略

输入列表 hash/index 或明确 address；显式地址查询可不读 NBT。先执行 [网络与来源](operations.md#网络与来源) Step 1；offline=true 直接 `BLOCKED: OFFLINE_POLICY`，不做 DNS。查询不启动游戏、不登录、不写 servers.dat。只查询明确选中的目标，不扫描网段；内网/回环目标必须玩家明确授权该地址。

### Step 2：检查协议依赖与边界

需要宿主已有专用 Java 1.7+ TCP Server List Ping（status/ping）工具，核实真实接口支持协议版本、握手域名/端口、固定已审 IP、响应大小/解析与超时限额及不自动跟随 SRV。普通 HTTP GET、端口连通测试不等于 SLP；不把 Prism C++ 类名当可调用命令，不安装协议库、不用通用 socket/临时脚本或 Java helper 手拼 VarInt。缺能力返回 `BLOCKED: CAPABILITY_MISSING (Java SLP)`；Bedrock/UDP Query/旧式 1.6 ping 不支持。

### Step 3：解析并授权实际终点

解析 address 为 host/port，默认 25565；用已有 DNS 工具只解析玩家明确主机，显示所有实际 IP/端口并检查地址策略。SRV 必须另获授权，工具可在连接前显示验证 `_minecraft._tcp` 的实际目标，否则 `BLOCKED: SRV_RESOLUTION_UNAVAILABLE`。缺 DNS 工具则域名查询 BLOCKED，可询问玩家提供明确数字 IP；不得默默更换地址或修改 hosts。

### Step 4：执行真实协议查询

向已核实 SLP 工具传入已授权目标、握手 host、固定审核 IP/port、连接超时 3 秒、总任务 15 秒和响应上限 1 MiB；不自动重试，玩家明确要求时最多两次。DNS 所有结果均须符合范围，混有未授权私网即阻断。无法固定连接地址或工具会重新解析/跟随未授权 SRV，则 `BLOCKED: DNS_TARGET_UNPINNED`。若工具仅支持数字 IP，可在玩家明确同意后使用受限方式并说明不保留域名虚拟主机语义；不能声称 SSRF 安全或用裸 socket 猜协议。

### Step 5：报告实际响应

仅输出实际 `result.version.name/protocol`、`result.players.online/max`、`result.latency`（毫秒）、清理控制字符后的 MOTD和查询时间/目标；响应不可信，不执行 MOTD 的链接/命令。超时/拒绝/协议不支持记 unknown/error，绝不写 online=0 或保证离线。状态结果不证明账号能登录/白名单通过/版本兼容；不缓存玩家姓名或自动写图标。能力缺失时只报告可执行条件与 BLOCKED，不能造响应。

## 启动并加入

### Step 1：明确连接授权

输入 instanceId、列表 hash/index 或显式 address、明确加入意图；从实际清单保留准确地址，校验 host/port，显示目标与网络外发风险。执行公共网络策略；禁联网则停止，不因账号为 offline 而放行。仅编辑列表不等于授权连接，未知/内网目标须确认。

### Step 2：准备正常启动

进入 [启动游戏](launch.md#启动游戏) Step 1～10，临时保留一次性目标，沿用正常账号/Java/下载门禁，不在本模块强制新登录。已运行则报告现有客户端，由玩家在多人游戏菜单加入；不为同一实例再起进程、不向窗口注入未知命令。不为单次连接自动改 servers.dat。

### Step 3：接入 canonical gameArgv

在 [启动命令](launch-command.md#拼接-command) Step 5 后、Step 7 前，若最终 profile.traits 有 `feature:is_quick_play_multiplayer`，追加 `['--quickPlayMultiplayer', endpoint]`；endpoint 为已核实的 host:port，IPv6 用 `[IPv6]:port`，若该版本对 IPv6 格式未证实则 BLOCKED，不猜。无 trait 时仅对已经核实支持传统连接参数的 profile 追加 `['--server', host, '--port', 端口的十进制字符串]`；无法核实则 WAIT_INPUT，选择只进主菜单手动连接。已有 server/quickPlaySingleplayer/quickPlayMultiplayer/quickPlayRealms 参数等价去重、冲突 BLOCKED，不修改共享 argv 合同或添加第二套启动器。

### Step 4：执行与验证

本模块注入目标后立即返回 gameArgv，不执行命令 Step 6～11；由启动游戏 Step 11 完成命令 Step 6～8。随后仅由启动游戏 Step 12～14 创建一次进程并观察/持久化，记录 spawn/running/exited 与真实日志。需要玩家明确确认进入目标服务器才报告 joined；有 PID/主菜单/SLP 响应仅能报告 join_requested/entry_unverified。认证、资源包提示、封禁、白名单、版本不符均如实返回，不自动接受服务器资源包、绕过登录或将失败换成离线身份。

## 来源与验证范围

- PrismLauncher `899d2f3957215a09fe398a8a8cbb8a8a9c619968`：`launcher/ui/pages/instance/ServersPage.cpp:60–155`（字段/三态/大端未压缩空名根）、`ServersModel::load/saveInternal`（列表顺序）、`ServerPingTask.cpp:21–48`（DNS+协议状态）、`launcher/minecraft/MinecraftInstance.cpp:768–786`（连接参数）。保留未知 tag、严格备份和原子替换为本 Skill 的安全合同，不声称上游保证所有字段无损。
- NBT/SLP 的专用工具接口和版本需执行时核实；本文 typed 数据与查询输出字段是能力契约，不是虚构可调用 API。
- 执行时重新检查已有专用 NBT/SLP 工具的实际能力；缺能力按上述门禁阻断。本流程不构成玩家 servers.dat 读写、实际 ping、启动或认证的完成证据。当前协议边界不支持 Bedrock、UDP Query、旧式 ping、无受控 DNS 的域名安全探测或自动服务端管理。
