# CurseForge 清单与精确文件导入

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices: ../NOTICE.md; license: ../LICENSE. -->

遵守 [玩家能力契约](../SKILL.md#玩家运行能力契约)、[归档安全](modpacks.md#导入整合包) 和公共网络/锁/恢复契约。这里只用已有结构化 JSON/HTTP/hash/安全复制工具；名称不是虚构的工具命令。无工具返回具体 CAPABILITY_MISSING。调用帧固定携带 caller/returnStep/operationId/stagingRoot；规划不写实例，暂停不发布 installed。

## 解析与恢复

### Step 1：仅解析本地清单

输入为已通过安全预检的 rootPrefix、UTF-8 manifest 原始字节/hash，默认 side=client、optionalSelection=[]、mode=plan。要求 manifestType=minecraftModpack、manifestVersion=1，minecraft.version 非空、files 数组、projectID/fileID 为正整数；required 缺省 true，显式值必须布尔。相同 fileID 重复或映射多个 projectID 拒绝，不能像上游 map 插入那样覆盖。name 缺省 Unnamed，version 缺省空，overrides 缺省 overrides；overrides 必须安全相对目录且存在，空值表示无覆盖。recommendedRam 缺省 0（不设置内存），正值仅作为 MiB 建议。

从 modLoaders 的 id 精确分离 forge-/neoforge-/fabric-/quilt-，映射组件 UID；NeoForge 仅特殊前缀 neoforge-1.20.1- 去掉其中 1.20.1-，其余保留完整版本。只有一个 loader 时直接复用；多个时列 primary 与冲突，WAIT_INPUT，不照搬 Prism 循环中后一个覆盖前一个的行为。recommended 不是精确版本：用 Prism 索引 requires.equals 过滤后给出真实推荐候选，获得选择再锁定。未知 libraries 字段/loader 或缺版本在对应字段 BLOCKED，不能忽略变原版。CF manifest 没有 mrpack env，不从 required 或文件名猜端侧；服务端包请求返回 BLOCKED: SERVER_IMPORT_UNSUPPORTED。

输出 name/packVersion/MC/loader/version、建议内存、required/optional 数量、overrides 和所有 ID 对。返回导入 Step 4 形成实例计划，**不访问 CF API、不要求 key、不初始化目录**。例如 ATM10 8.5 的实际清单应得 MC 1.21.1、net.neoforged:21.1.265、8196 MiB 建议和 498 个 required；这不是 498 个文件已下载。

### Step 2：选择解析链路，不强制玩家申请 Key

导入 Step 5 调用此步，输入精确 ID 对与已有来源证据，先检查 offline 策略。可复用可追溯的 API/作者校验清单缓存。官方 API 分支接受本项目 credentialRef，或下述“已授权 Prism 应用 Key”部署来源；不要求每位玩家自行申请。显式 offline=true 仍禁止请求；显式由玩家关闭官方 provider 时不得自动启用，需在实际安装调用时确认启用（配置编辑任务不改玩家配置）。旧默认 enabled=false 且 credentialRef=null 不等于授权无效，应先解析已授权部署来源，不重复要求申请个人 Key。默认遵循 Prism 的官方链路：官方 API 精确 ID 解析 → 优先官方下载 URL → URL 缺失/下载受限时按 Step 4 默认尝试 Modrinth 官方 SHA-1 同字节兜底 → 权威 size/hash 校验 → 未解决项人工回填；不要因没有个人 Key 自动切到第三方来源。没有已获授权的本项目凭据时返回 WAIT_INPUT: CF_APPLICATION_AUTHORIZATION_REQUIRED，解释缺的是应用授权而不是要求每位玩家申请 Key。只有玩家另行明确选择第三方适配时才展示 [无个人 Key 导入](curseforge-public-import.md)，确认其来源和验证差异后调用 Step 1～5，返回本流程 Step 4。等待期间保留精确 unresolved ID 对和 resumeStep=2，不创建实例。不要在聊天索取 key，不读取其他启动器账号、令牌或钥匙串无关项目；已获明确授权的 Prism 构建应用 Key 按下述来源读取，不受无关凭据禁令误阻断。

真实 Prism `ApiRequest` 添加 API headers；`ApiHeaderProxy` 对配置的 Flame API/download host 加 x-api-key，`Application::getFlameAPIKey` 先取用户 override，再用 BuildConfig key。这不是匿名公共 API。Prism 源码 CMakeLists.txt 的 API Keys 注释明确写明该 Key 专门签发给 Prism，CurseForge 要求衍生项目更换；因此公开可读不等于本项目获准复用。本项目维护者已明确表示获得 Prism 官方对该 Key 使用的授权；在该授权部署范围内按此声明允许复用，不再因“来自 Prism”一律阻断，也不重复索要个人 Key。此记录是维护者提供的授权声明，不是 Agent 独立法律审核或对其他部署的通用许可。使用下述部署流程读取获准 Key；不在 Skill、模板、日志或聊天嵌入 Key，不提取任何用户登录凭据。已授权应用 Key 的使用范围包括 Prism 同一官方链路的 api.curseforge.com 和 edge.forgecdn.net，不重复询问同一范围；其他自定义 API/认证主机另确认，任何跨域重定向都重新决定请求头，不无条件转发。公开微软 OAuth client ID 仅用于微软登录，与 CF key 无关。真正无需凭据的 Prism meta/Modrinth 公共查询可独立使用，不能假装能凭它们解析任意 CF fileID。

### Step 3：批量解析并严格配对

使用 HTTP POST `<apiBaseUrl>/mods/files`，JSON `{"fileIds":[<整数ID>,...]}`，Accept/Content-Type=application/json，在安全 HTTP 请求中注入 x-api-key。默认每批最多 50 项（本 Skill 资源限额，不冒称上游硬限制），并发最多 4；498 项为 10 批。对 data[] 按 id 连接原清单，复核 modId=projectID，要求 fileName 安全单层名称、fileLength 非负整数、hashes 格式合法。响应缺项记录 missing，不以响应条数当齐全；重复/额外/错配 ID 为 BLOCKED: CF_RESPONSE_MISMATCH。缺项可 GET `/mods/{projectID}/files/{fileID}` 精确重核一次，404/已撤回保留 missing，不改用 latest。

再 POST `/mods`，JSON `{"modIds":[<去重整数ID>,...]}`，核对 data[].id/gameId=432/classId/links.websiteUrl；不能把世界/资源包一律放 mods。classId=6 放 mods，12 放 resourcepacks；其他类别先用已核实类别及 [资源模块](resources.md) 确定路径，世界须另行模板世界选择，类型不明 BLOCKED: CF_RESOURCE_TYPE_UNRESOLVED。保留文件 gameVersions 等兼容证据，与包 MC/loader 明确冲突则停下；不自动添加 API 依赖列表中的最新文件，清单才是锁定集合。

hashes algo=1 为 SHA-1、2 为 MD5；SHA-1+size 可作为 CF 权威文件验证，额外计算 SHA-256 保存本地快照。只有 MD5 不满足本 Skill 强验收，需额外可追溯作者强 hash；缺少则 BLOCKED: FILE_IDENTITY_UNVERIFIABLE。成功输出每项 identity/size/expectedHashes/metadataEvidence/resourceType/目标路径，不保存含秘密请求头或签名 URL 查询值到普通报告。

### Step 4：分流下载、可选与受限项

required=false 列出可选列表，默认不选不下载；required=true 全部保留。此策略不同于 Prism 将未选可选项也下载成 .disabled。CF 客户端清单无端侧字段；明确作者服务端文件不自动投入客户端。先铺 overrides 后写已选下载项（Prism 创建顺序）；覆盖目录按父流程归档规则应用，下载项与覆盖同路径时列明两层 hash 和最终来源，未知冲突 WAIT_INPUT。

sourceResolution=public-service 的项不使用下段官方 downloadUrl/hash 门禁：mode=plan 复用无个人 Key 子流 Step 1～3 的结果，mode=stage 调该子流 Step 4～5，返回本流程 Step 5；只消费其已确认验证等级，不能再降级。混合计划逐项分流，最终按完整 ID 集合验收。

mode=plan 时只生成下载/本地候选计划，不下载或复制；mode=stage 仅由导入 Step 6 持父锁与实例锁调用本 Step～5，复用计划及授权。非空合法 HTTPS downloadUrl 通过 TLS/重定向/私网检查后交 [单文件下载](downloads.md#单文件下载)，返回逐文件校验点；不能由 fileID/fileName 拼 forgecdn 地址。仅 downloadUrl 为 null/空或官方下载受限、且已取得该精确 CF 文件权威 SHA-1 与 size 时，默认允许下述 Modrinth 同字节兜底，不再询问同范围来源授权。空 URL 或单次 HTTP 拒绝本身不证明禁止分发，但已知撤回、禁止分发或明确不允许该替代渠道的限制必须保留，不能借兜底绕过；CF API 缺失/撤回、无权威 SHA-1、缺 size 或玩家已显式禁用此兜底的项直接保留 unresolved。

本地已有候选（含 overrides 中嵌入 JAR）也必须按权威 size+所有可用强 hash 验证，字节相同可普通复制去重，不能只比同名/fileID。Prism FileResolvingTask 有按 SHA-1 查询 Modrinth 的 fallback；本 Skill 默认启用这一精确同字节来源策略，范围固定如下：

1. 复用 Step 3 的 CF projectID/fileID、权威 SHA-1（40 位十六进制）和 fileLength；不得用本地计算值、Modrinth 自报值或同名搜索反推 CF 预期 hash。先检查 offline、玩家显式拒绝和上述分发限制；策略默认允许不等于已经授权本次安装/下载，也不覆盖玩家更严格的选择。
2. 用已有 HTTP 工具请求 `GET https://api.modrinth.com/v2/version_file/{sha1}?algorithm=sha1`，不发送 CF x-api-key。只接受 files[] 中 hashes.sha1 与 CF SHA-1 完全一致且 size 与 CF fileLength 完全一致的文件；记录真实 Modrinth project_id/version id。404、无匹配、响应不合法或校验不一致均保留 unresolved，不搜索近似版本、不用 latest、不替换其他字节。
3. 仅接受该官方 API 返回、可合法公开下载的 HTTPS 文件 URL；默认文件主机精确限定为 `cdn.modrinth.com`，其他主机及跨域重定向不在本默认授权内，须另行确认。逐跳执行 TLS、重定向、私网与凭据隔离检查，不使用自定义代理/任意镜像，不绕过撤回或已知禁止分发限制。mode=plan 只记录候选和证据；mode=stage 才复用 [单文件下载](downloads.md#单文件下载)，实际落盘后强制复核 CF 权威 size、SHA-1 及所有可用强 hash，失败拒绝并保留原因。
4. 在 import-plan 的 files[] 逐项分别保存 identity={provider:curseforge,projectID,fileID}、sourceResolution=modrinth-sha1、actualSource={provider:modrinth,projectId,versionId,url}、expectedHashes、size 和 metadataEvidence；不将实际来源冒称 CF CDN，不覆盖 CF 身份，URL 不保留秘密查询值。sourcePolicy.curseforgeMissingDownloadUrl=modrinth-official-sha1-exact 表示默认策略，不是假造玩家确认；该字段缺失时仅在内存补此默认，显式 disabled 表示拒绝，不覆盖已有拒绝值，不为应用文档策略而改真实玩家配置。恢复同范围任务复用此策略与逐项证据，不重复询问。CFWidget/Curse Maven、其他来源、降低校验等级或更换版本仍需另行明确授权。

兜底失败或不适用的项进入人工清单，字段为 projectID/fileID/name/size/expectedHashes/官方文件页/原因/state。官方页使用已核实 links.websiteUrl 加 `/files/{fileID}`；没有项目 slug/网站证据就只列 ID 并请提供官方页面，不能编造。

### Step 5：人工回填、恢复并返回

有必需项未齐时返回 WAIT_INPUT: MANUAL_FILE_REQUIRED，给精确清单和原 caller/returnStep；不可将 missing 当 optional。玩家只需提供合法下载的文件路径和缺失的权威校验依据，不需要把秘密写入聊天。仅在指定范围内检查普通文件、拒绝链接，流式计算 size/hash；plan 模式匹配后只记录本地文件身份/hash；stage 模式重验该身份后复制至已授权 staging（复用父流程锁），复制后重验。大小相同/同名但 hash 不同拒绝；无权威 hash 不因手工来源放宽；只有玩家明确选择 public-service 分支时，才按该分支已确认的第三方来源验证等级验收，不能冒充官方强 hash。

暂停可保存 [import-plan](../templates/import-plan.example.json) 中 unresolved/manualFiles 和逐项状态，恢复重验源包 SHA-256、清单、计划及已完成文件，再从本步处理未完成项，复用既有选择/下载授权。身份来源变更、超出 Step 4 默认精确同字节范围的来源变更或新冲突重新确认；默认兜底内 CF 身份不变的来源恢复不重复确认。plan 模式每个 required/已选 optional 必须有合法待下载 URL 或按所属分支验证等级验收的本地候选，返回导入 Step 5 合并计划（resolved 不等于已安装）；stage 模式全部实际字节校验后返回导入 Step 6 当前验收点；此子流自己不写 installed/managedPack/selectedId。不从本地包名/version 推造 CF 整包 projectId/fileId。

## 已授权 Prism 应用 Key

此子流在解析与恢复 Step 2 或在线搜索 Step 1 调用，完成后返回原调用点。仅用于维护者确认获准的本项目部署；不扩大为其他项目通用许可。

### Step 1：选择应用凭据来源

优先读取本项目已配置 credentialRef；无 override 时使用已授权的 Prism 构建来源，和 Prism 的 override → BuildConfig 顺序一致。来源固定为 PrismLauncher 官方仓库 commit `899d2f3957215a09fe398a8a8cbb8a8a9c619968` 的 `CMakeLists.txt` 中 `Launcher_CURSEFORGE_API_KEY` 单一字符串值。可信本地该 commit 的只读 checkout 可复用；没有本地来源时，经公共网络门禁获取 `https://raw.githubusercontent.com/PrismLauncher/PrismLauncher/899d2f3957215a09fe398a8a8cbb8a8a9c619968/CMakeLists.txt`，响应限制 256 KiB，验证真实来源/版本。不要读取另一启动器的用户配置、账户库或二进制提取秘密。来源不存在、格式不符或重复定义就明确 BLOCKED，不猜值、不复制旧聊天中的 Key、不自动换其他项目 Key。

### Step 2：安全读取，不执行构建文件

用已有文本工具/原生命令仅取该命名定义的双引号字面值，要求唯一、非空、无换行；不能执行 CMake、source/eval 文件或解析其他 shell 表达式。工具输出只返回提取成功/失败，不回显值。用普通请求工具在内存构造 x-api-key；已有系统安全存储可持久化为本项目专属条目并读回验证，普通 config 只保存 credentialRef。编辑 Skill 时不实际创建安全存储或改玩家配置。不得为了安全注入另装运行时或编写 helper。运行能力缺失就报告具体能力，不把它说成玩家缺 Key。

### Step 3：限定主机并核对服务响应

只给 `api.curseforge.com` 和 Prism BuildConfig 的官方认证下载主机 `edge.forgecdn.net` 注入 x-api-key；其他主机即使来自官方 downloadUrl，也不继承秘密请求头。对重定向逐跳校验并重建头，绝不使用跨域无条件保留认证的选项；只发实际获准主机，不伪装 UA 为 Prism。TLS/私网门禁仍生效。官方批量解析按主流程返回并逐项核对 ID、modId、大小和权威 hash。401/403 明确报告 API_KEY_REJECTED 或 HTTP_ACCESS_DENIED（状态本身不能证明具体拒绝原因），不重试其他身份、不切镜像绕过。

### Step 4：返回并保留最小非秘密证据

返回 applicationCredentialReady、来源仓库/commit/变量名、授权依据=maintainer_statement、实际 HTTP 状态和原 caller/returnStep，不保存 Key、请求头或带秘密查询值。来源已就绪就继续官方流程，不反复询问同一授权。API 成功不代表 CDN 成功，必须独立验证官方实际下载及 size/hash；一个样本成功也不代表整包已安装。Key 失效或授权撤回时停用该来源，保留任务可恢复状态。

## 在线搜索

### Step 1：检查输入和 API 前提

输入 query 或 projectID，默认 gameId=432/classId=4471、pageSize=20/index=0；依上方 Step 2 检查网络和玩家凭据。查询未授权安装，不写配置。缺 query WAIT_INPUT；无官方凭据时在线搜索返回 CF_SEARCH_CREDENTIAL_REQUIRED，但给定本地清单的精确导入可走无个人 Key 子流。CFWidget 不作为全站搜索 API。

### Step 2：搜索并选精确版本

GET `/mods/search`，编码 gameId/classId/searchFilter/index/pageSize；读取 data/pagination，按 resultCount 续页，不声称一页全部。选项目后 GET `/mods/{id}/files`，按真实分页列 id/displayName/fileName/gameVersions/releaseType；玩家选包文件 ID 后 GET `/mods/{id}/files/{fileID}` 复核身份、URL、size/hash，空 URL 转人工清单。未明确版本 WAIT_INPUT，不默认 latest。

### Step 3：返回

只查询返回候选；授权导入则下载选定 ZIP 并传可靠整包 sourceIdentity 至导入 Step 2，归档内 MC/loader 再复核。不支持 CF 受管在线更新/导出，返回具体 CF_OPERATION_UNSUPPORTED。

## 实现依据

Prism 固定 `899d2f3957215a09fe398a8a8cbb8a8a9c619968`：

- [清单默认值与结构](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/modplatform/flame/PackManifest.cpp#L5-L70)。
- [覆盖目录、loader 映射及推荐内存](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/modplatform/flame/FlameInstanceCreationTask.cpp#L375-L538)、[可选/受限下载/回填](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/modplatform/flame/FlameInstanceCreationTask.cpp#L541-L699)。
- [批量请求路径](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/modplatform/flame/FlameAPI.cpp#L68-L124)、[解析与 Modrinth fallback](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/modplatform/flame/FileResolvingTask.cpp#L48-L237)。
- [API header 身份行为](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/net/ApiHeaderProxy.h#L65-L83)、[key 选择逻辑，无 key 值](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/Application.cpp#L1950-L1958)、[官方 REST 字段定义](https://docs.curseforge.com/rest-api/)。严格一一配对、预算、强校验和凭据隔离是 Skill 安全差异。
