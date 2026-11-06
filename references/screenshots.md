# 截图管理

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

锁操作一律执行 [公共门禁](operations.md#写入前门禁) 的父级稳定锁→实例锁协议，整个操作持有，持锁后再次核对所有权；下文“检查发布锁/持锁”不是只检查不存在。子流程复用同一 operation 锁，不重复取得。

六个玩家操作：PNG 扫描、预览、复制导出、重命名、删除、明确确认后的分享。作用域为选定实例 `<gameRoot>/screenshots` 的普通 PNG，不扫描其他实例/桌面/照片库。只读不写配置、不登录；截图操作日志使用公共 operations，不建立第二份截图索引或强制安装图像库。链接/共享硬链接和路径逃逸按公共门禁拒绝。

## 扫描 PNG

写操作复用公共发布锁/实例锁，操作日志统一 `dataRoot/.management/journals/<opId>.json`，不置于 screenshots 或实例内，不随实例删除清理。只接受当前已注册实例身份，不复用 retiredIds 或历史备份/operation 引用 ID。实例内截图备份 sourceRelativePaths 使用 `.minecraft/screenshots/<实际文件>`（相对 instanceRoot），不是 gameRoot 相对路径。单文件恢复只替换批准文件；若选择恢复截图目录，则按 backups 先确认将删除的快照外新增文件、在 staging 清空批准范围再按 manifest 重建，不能覆盖合并后声称精确恢复。外部导出旧目标不冒充实例内路径。

### Step 1：取得只读范围

输入可选 instanceId，执行 [公共上下文](operations.md#输入与结果约定) Step 1～3，action=screenshot.list。缺配置返回未初始化，screenshots 缺失返回空，不 mkdir、不初始化实例。

### Step 2：识别普通文件

宿主 Python `Path(screenshots).iterdir()` 仅扫描直接文件；使用 `.suffix.lower()=='.png'` 筛选，拒绝 symlink/特殊文件，记录真实路径、size、mtime_ns、本地 SHA-256；默认单张上限 64 MiB，超过则列作 too_large 不加载。读取文件前 24 字节验证 `b'\x89PNG\r\n\x1a\n'`、首 chunk 长度 13、类型 IHDR，通过 `struct.unpack('>II', header[16:24])` 取 width/height；宽高必须正且像素不超过 100000000。仅这些检查只能称 PNG 候选，不是完整解码/CRC 验证。不因扩展名就信任文件内容。

### Step 3：可选解码与稳定性

若已有可信本地图像工具，可验证实际格式/解码；Python Pillow 是可选依赖，先用 `python3 -c "import importlib.util; print(importlib.util.find_spec('PIL'))"` 检查。存在则在宿主限时/内存进程中使用 `Image.open(path).verify()`，重新打开 `load()`，将 DecompressionBombWarning 设为 error；APNG/多帧默认只列元信息，不将首帧当全部内容。缺 Pillow 不自动装包：返回候选清单并标未解码；真正需要解码且无宿主替代能力才询问安装授权，未授权 `BLOCKED: DEPENDENCY_MISSING`。重查 size/mtime/hash，写入中的截图标 changed，不能操作旧指纹。

### Step 4：返回清单

返回相对路径、像素尺寸、时间、大小、候选/decoded/invalid/changed 状态与实际各状态数量，不外发图像、不自动生成分享 URL。游戏运行允许扫描，但可能捕获正在写入文件，失败项保留说明而不删。

## 预览

### Step 1：精确选择

输入 instanceId、相对 PNG 路径或清单 index+指纹；多个匹配询问，不按最新文件猜。重新执行扫描的路径/尺寸/稳定性校验，输入不是有效可解码图片就 BLOCKED，不“修复”源图。

### Step 2：调用明确图像工具

用宿主 `vision_analyze(image_url=<已验证本地绝对路径>, question=<玩家要求>)` 加载目标图像，或宿主受支持的本地图像查看接口打开该精确路径；不得用 browser 导航陌生外站来预览。工具缺失返回 `BLOCKED: CAPABILITY_MISSING (local image preview)`。若宿主 vision 会将图片送外部服务且玩家仅授权本地预览，先说明目的地并确认或只用本地查看器，不能把“预览”当匿名图床上传许可。调用工具时检查实际接受的是本地文件而非含秘密的 URL。

### Step 3：核对可见结果

只有工具成功展示/返回图像且指纹未变才报告已预览；权限/解码/窗口工具失败就如实报告。裁剪/缩略图如需要只写宿主操作暂存，不覆盖原图；没有画面不得编造截图内容。预览不写 selectedId 或 management 配置、不自动分享。

## 复制导出

### Step 1：选择文件和目的地

输入 instanceId、准确文件集合、exportDirectory 或单文件 exportPath。列出相对路径、大小/hash 和目标绝对路径；目标必须玩家明确授权且不指向源文件/其他实例。只要用户明确要求本地复制到该路径即可，不追加网络步骤；剪贴板复制只有宿主有真实文件/图像剪贴板接口且用户要求时才执行，无能力不假报复制成功。

### Step 2：授权与备份

执行 [写入门禁](operations.md#写入前门禁) Step 1～5，复用公共锁；为与公共提交的停机复核一致，本流程保守要求停止实例后才复制，运行中可继续扫描/预览但写入导出返回 WAIT_INPUT，不擅自停止。必须两次源 hash 相同才导出该文件。覆盖冲突需明确确认，旧输出先执行 [创建备份](backups.md#创建备份)，或改用未占用目的路径；备份模块不能安全备份外部旧目标则 BLOCKED，不跳过。截图文件副本不称一致实例备份。

### Step 3：真实复制与单文件提交

每项先在目标同文件系统 staging 用 `shutil.copy2(source, staged_target)`，拒绝链接，比较复制前源 hash、复制后源 hash、暂存 hash 和大小完全相同；每项 `os.replace` 前重新检查目标指纹/存在性。多文件逐项 journal，不宣称批次整体原子；目标新出现则 BLOCKED，不能静默覆盖。

### Step 4：后验与结果

执行 [提交与恢复](operations.md#提交与恢复) Step 3～5，读最终目标 hash/大小验证全部已确认项，检查源未改变；报告各目标及失败/成功数量。只写 operations，不写 content.entries。未调用上传绝不能给出虚构公开链接。

## 重命名

### Step 1：确定新名称

输入 instanceId、旧相对文件名及 SHA-256、新 basename；新名须为单个合法文件组件，保留 `.png`，拒绝空、路径分隔、NUL、`.`/`..`、平台保留名和大小写/Unicode 归一化碰撞。不自动批量改其他图；同名冲突 WAIT_INPUT，不能自动覆盖。

### Step 2：门禁与备份

执行写入门禁 Step 1～5，并保守要求实例已停机（这是修改 screenshots 路径，不让运行中游戏与本操作竞争）。先备份原 PNG，记录其他截图路径/hash；无法确定运行状态 BLOCKED。锁与源 hash 保持不变才继续。

### Step 3：执行精确 rename

普通改名用宿主 `os.rename(old_path,new_path)`，提交前验证新名不存在；宿主需要无覆盖 rename 能力或受控私有目录锁，否则报告不能保证不覆盖外部并发文件。仅大小写改名在大小写不敏感卷上使用 operation 唯一中间名再 rename，每段记录 journal，不称整体原子。异常立即停止，不能用复制后删除悄悄改变语义。

### Step 4：后验

确认旧路径已不存在（仅大小写时按实际目录项拼写核对）、新文件 hash/大小等于原图、其他截图未变，执行提交与恢复 Step 3～5。失败保留备份和已改到的实际路径，提供恢复计划。

## 删除

### Step 1：列出精确删除集

输入 instanceId、路径集合与各自 hash；重新扫描并展示文件名/尺寸/大小，批量必须明确确认，未选定/存在歧义 WAIT_INPUT。PNG 候选无效不自动清理；“清理截图”不是全目录删除授权。

### Step 2：停止、授权与备份

执行完整写入门禁 Step 1～5；同重命名保守要求停止实例。备份全部目标，verified 后才允许删除；不能把系统回收站支持当作已备份。缺备份能力/空间就 BLOCKED。

### Step 3：逐文件删除

重核每个文件普通类型/路径/hash，使用 `Path(target).unlink()` 仅删除明确选中 PNG；不执行 rmtree(screenshots)，不跟随目录链接，不扩大 glob。逐项 journal planned/applied/verified，出错停止后续项。

### Step 4：验证与恢复入口

确认每项路径不存在，未选中文件路径/hash 不变，执行提交与恢复 Step 3～5。报告实际删除数、未删除数、backupId 与 [恢复备份](backups.md#恢复备份)。未取得真实回收站结果不能声称“在回收站随时恢复”。

## 分享

### Step 1：取得服务与范围

输入 instanceId、准确 PNG/指纹、玩家已授权的 service、明确 endpoint、credentialRef（如需）和可见性；无已配置/明确选定服务则 `WAIT_INPUT: SHARE_SERVICE_REQUIRED`，提供先本地导出选项，不自动选 Imgur、读取 Prism 内置 client ID、从浏览器提取凭据或使用公共匿名上传站。只读扫描/预览/复制不会进入此流程。凭据由宿主安全存储/授权连接持有，模型和普通 JSON/日志不接收密码、Bearer token；无可用秘密注入能力则 `BLOCKED: SECURE_HTTP_CAPABILITY_MISSING`，不向用户索要聊天粘贴 token。

### Step 2：单次外发明确确认

先执行 [网络与来源](operations.md#网络与来源) Step 1～3 的策略与目标审查（本操作写请求另遵循不盲重试）；offline=true 直接 BLOCKED。检查 PNG 可解码及稳定，展示文件数、名字/大小、目标 HTTPS 域名/路径、授权主体、可见性、保留/撤回限制和可能含聊天、坐标、服务器地址、UUID/玩家名/元数据。邀请本地预览/人工遮挡，不能宣称自动脱敏。明确问“是否把这些精确文件上传到此服务”，未确认 WAIT_INPUT；确认仅绑定本次文件 hash+endpoint+已证明服务可满足的实际可见性，文件或目标变更必须重新确认。POST 前必须把请求可见性映射到服务真实访问控制：本模块的 Imgur 单图上传不提供成员限定、密码、登录校验或“仅自己可见”保证，不能用 unlisted/不进 gallery 冒充私有；仅在玩家明确接受“持链接者可直接访问、可能继续传播”的暴露范围时继续。玩家要求受限可见性则在上传前 BLOCKED: VISIBILITY_UNSUPPORTED，选择具有已验证 ACL 的其他服务或重新确认公开链接范围，绝不先上传后解释。默认仅一张；多张需逐项清单并确认，不隐式创建相册。

### Step 3：执行已配置协议

已授权 Imgur 只采用以下具体单图请求，服务策略/凭据必须事先可用：宿主安全 HTTP 工具向 `https://api.imgur.com/3/image` 发 `POST`，`Content-Type: multipart/form-data; boundary=<工具生成>`，文件 part 名 `image`，filename 使用玩家确认的无敏感名称，媒体类型 `image/png`，字节为该 PNG；可选 type=file，不自动附 title/description/地理信息。Authorization 由安全工具注入服务批准的 `Client-ID <应用ID>` 或 `Bearer <授权token>`，不能复用 Microsoft/游戏 token。禁止转成模型可见的 curl 明文命令。用工具的 multipart/标准库编码能力，不手工拼图像或 NBT 二进制。60 秒超时，TLS 必须验证，禁止凭据跨域重定向。

其他已授权服务仅当玩家/其连接提供可审阅合同：准确 HTTPS endpoint、HTTP 方法、文件字段/媒体类型、授权方式、成功响应 JSON 路径、返回 URL 验证、访问控制/可见性的具体请求参数与读回验证（含未授权访问测试）、撤回方式时才执行真实请求；缺合同则 WAIT_INPUT，不猜 `/upload`。本模块不向普通配置新增 service 字段，使用当前已授权连接/本次输入；若要持久记住服务，先提出 schema 扩展而非私写配置。

### Step 4：检查真实上传返回

Imgur 必须取得实际 2xx 响应并解析 `success == true`、`data.id: String`、`data.link: HTTPS URL`；`data.deletehash` 若返回属于撤回秘密，只交宿主安全存储，不返回模型/日志/普通 config。响应字段缺失/错误返回 upload_failed，不凑 URL。超时可能已经上传：标 `upload_uncertain`，保留非秘密 request ID/阶段，不能盲重试 POST 造成重复；只有服务有已核实幂等机制才能按其合同重试。授权错误停止，不换客户端 ID/匿名方式。

### Step 5：读回精确目标并验证 URL

检查 link 必须属于已授权服务的 HTTPS 图像域（Imgur 为 i.imgur.com，不从不可信响应跟到任意地址），无 userinfo/未授权端口，重定向每跳验证、不转发 Authorization。对 `https://api.imgur.com/3/image/<实际 data.id>` 发 GET（安全工具注入原服务授权），要求 2xx、success=true、data.id/link 与上传返回相符。再对实际 data.link 做不带秘密的 GET，限制最大 64 MiB、60 秒，检查状态、Content-Type 与 PNG signature，并用实际本地图像解码器验证是图片；若服务承诺原字节，再比较下载 SHA-256 与源。Imgur 可能转换图片，字节不同不能伪称原文件保真；要宣称同一画面须本地解码比较尺寸/逐像素或人工查看一致，缺能力报告 `uploaded_but_content_unverified`。域名/元信息/图像读取任一验证失败，不能标完整 DONE 或说链接可用；分别记录“POST 已返回/远端读回未通过”，不重复上传。

### Step 6：收尾与真实声明

最终验证通过才返回实际服务返回且已 GET 验证的 URL、对应文件与实际证实的可见性，以及是否验证原字节/画面。Imgur 无授权图片 GET 成功只证明持链接可访问，不证明访问者受限；不得把玩家请求值复制成远端 ACL。自定义受限服务必须读回 ACL 并验证未授权访问被拒绝，无法证明就标 visibility_unverified；发现比已批准范围更开放时立即停止后续分享，报告潜在已暴露并提供需授权的撤回选项，不伪称私有成功。操作日志只含非秘密状态/路径/本地 hash/请求阶段，URL 有访问秘密时也不写普通日志。没有执行 POST 就报告 not_uploaded；POST 成功但 GET 未完成报告未验证并说明潜在已外发。只在玩家要求时把已验证 URL 复制剪贴板/发到指定会话，每个新目的地独立外发确认；不静默群发、不把本地文件路径说成分享链接。保留原 PNG 不删除，撤回另需明确授权及安全存储句柄。

## 来源与验证范围

- PrismLauncher `899d2f3957215a09fe398a8a8cbb8a8a9c619968`：`launcher/ui/pages/instance/ScreenshotsPage.cpp:262–294,395–469,495–593`（PNG 模型、隐私确认、上传/相册与本地文件操作）。上游 Imgur 能力不等于本宿主拥有可用授权或本 Skill 自动具备服务；本流程未执行外发。
- 格式：[PNG specification](https://www.w3.org/TR/png/)（signature/IHDR），[Imgur API Image](https://apidocs.imgur.com/#eff532c1-61ee-4a0e-b3d1-13e4bd618d2c)（POST /3/image、GET /3/image/{id}）。执行时必须核对当前服务合同/权限，不凭静态源码声称匿名上传仍可用。
- 执行时重新探测实际宿主的本地图像解码能力，不把 IHDR/签名检查当完整解码；缺能力先询问可用工具/依赖授权，不自动安装。本流程说明不是玩家图片解码、真实上传或认证的完成记录。不支持自动录屏、图像编辑/脱敏、隐式相册、多服务猜测兼容或绕过凭据。
