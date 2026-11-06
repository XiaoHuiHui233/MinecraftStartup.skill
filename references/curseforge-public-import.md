# 无个人 API Key 的 CurseForge 清单导入

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices: ../NOTICE.md. -->

本子流由 [CF 精确解析 Step 2](curseforge-import.md#解析与恢复) 调用。它是独立的第三方公开服务适配，不是假称 Prism 匿名访问官方 API。Prism 可使用构建时应用 Key，让玩家不必填写个人 Key；本项目已有维护者确认的授权部署按 [应用 Key 接入](curseforge-import.md#已授权-prism-应用-key) 使用；本第三方分支不读取或发送该 Key。遵守根 Skill 玩家能力边界、归档安全、网络、锁和下载契约，不安装启动器、不运行自写程序。

## 解析、下载与返回

### Step 1：明确来源与验证选择

输入原始 manifest 的全部 projectID/fileID、mode=plan|stage、operationId/caller/returnStep 和已批准 stagingRoot。保持所有必需项，先检查 offline 策略；离线禁止任何网络。没有本子流的来源许可时，向玩家一次说明并确认：使用 `https://api.cfwidget.com` 获取公开项目/精确文件记录，使用 `https://cursemaven.com` 下载按 ID 映射的文件；两者是独立第三方，不是 CurseForge 官方 API。提供选择：接受第三方来源验证，或坚持官方/作者强 hash 并提供相应来源。没有选择返回 WAIT_INPUT: PUBLIC_SOURCE_CONFIRMATION_REQUIRED，不自动降低验证等级。

许可限本次包的精确 ID 集合和上述两个主机；不默扩展到其他包、镜像或 beta 域名。CFWidget 响应中官网链接只能是 `www.curseforge.com` 的有效项目/文件页；其他域名另确认。持久化许可和验证选择到任务 import-plan，不修改全局官方 API enabled/credentialRef。若公开文件页或已有证据标记禁止第三方分发、已撤回或受限，该项仍进入人工队列，不借镜像绕过；服务可下载不等于已证明分发授权。

### Step 2：按项目读取公开文件记录

按 projectID 去重后 GET `https://api.cfwidget.com/{projectID}`，并发最多 4，每响应最多 8 MiB，超限停止请求并 WAIT_INPUT 新预算；使用已有有界 HTTP/JSON 工具。要求响应 id 严格等于 projectID、game=minecraft；仅从 files[] 查找精确 fileID，禁止使用 download、latest 或同名替代。files 中匹配项必须唯一，name 是安全单层文件名，filesize 是非负整数，url 是 HTTPS 官方文件页且末尾 ID 严格匹配。保留 urls.curseforge/project、type、versions、name、filesize 及完整记录证据 hash。

type=Mods 映射 mods，Resource Packs 映射 resourcepacks；其他实际类型先按资源模块/官网证据确认，不猜测。versions 若与包 MC/loader 明确冲突则停止该项；缺少版本标签记 unknown，不伪造兼容证明。404、缓存未含历史 fileID、重复 ID、错误项目或缺字段记录 unresolved，不能因为 CFWidget 没列出就宣称作者删除了文件。按精确 ID 集合计算覆盖率；503/429 按公共重试，不轮换 IP/代理。

### Step 3：构造已文档化的 Maven 文件计划

Curse Maven 的公开契约是 `curse.maven:<descriptor>-<projectID>:<fileID>`，descriptor 可自定。固定使用安全常量 `mcstartup`，避免把远端 slug 拼进路径。映射 HTTPS URL：

`https://cursemaven.com/curse/maven/mcstartup-{projectID}/{fileID}/mcstartup-{projectID}-{fileID}.jar`

这不是从 fileID 猜 forgecdn 路径，而是 Curse Maven 已公开的 ID 寻址接口；保留实际服务来源。最终游戏文件名取 Step 2 的 name，不采用 Maven 别名。文件名非 .jar 的项目不使用该适配器，转人工准确映射，不能把资源 ZIP 假装 JAR。目标路径走父流程全部安全检查和冲突确认。

若已有官方 API/作者强 hash，保留并强制验证；否则 expectedHashes={}、verification.level=third-party-id-size、upstreamHashVerified=false。`.jar.sha1` 不保证存在，404 不能当空 hash，ETag 不是文件 hash。玩家选择严格强 hash 时，无强 hash 项返回 FILE_IDENTITY_UNVERIFIABLE；玩家在 Step 1 明确接受第三方来源验证时，才允许继续。仅 plan 返回待下载计划，不提前下载全部文件。

### Step 4：在 staging 下载与逐项验证

mode=stage 由父 CF Step 4 调用，复用父稳定锁/实例锁、选择和计划，不再次请求整包所有项目；先重验源包/许可/已有暂存文件身份。通过共享单文件下载契约下载到任务私有 .partial，沿用文件数/单文件/总量预算，验证 TLS；每跳重定向重新核对主机，未批准主机先 WAIT_INPUT，不添加 API Key 或 Cookie、不探测替代 CDN 绕过 401/403。失败保留该项 unresolved，禁止省略必需项。

下载字节数必须等于 CFWidget filesize；已有强 hash 全部必须匹配。用已有归档检查工具验证实际为正常 JAR/ZIP、成员 CRC 通过（不展开嵌套包、不执行 JAR），HTML 错误页、损坏文件和大小不符必须失败。计算本地 SHA-256 作为首次来源快照，复制/恢复后必须匹配，但不能称其作者/CF 官方验证。CFWidget 和 Curse Maven 的 ID 映射与大小验证不是密码学作者认证。

每项在 import-plan.files 和最终 pack-state.files 保留 sourceResolution=public-service、projectID/fileID、真实来源 URL、metadataEvidence、size、expectedHashes、localSha256、verification={level,upstreamHashVerified}；任务级 publicSourceConsent 记录本次 ID 集合、主机、验证选择与确认时间。模板使用声明式空默认值，不填写真实结果。未知许可/受限项不因玩家接受较低 hash 等级就解除。

### Step 5：验收覆盖并返回

按 (projectID,fileID) 集合而非请求/尝试次数去重核对：每个 required/已选 optional 必须有唯一计划项；stage 时还必须有通过所选验证等级的实际文件。缺项携带精确 ID、原因和恢复点，返回 WAIT_INPUT: MANUAL_FILE_REQUIRED；不发布 installed。plan 返回父 CF Step 4 合并，stage 返回父 CF Step 5 做完整性验收；父流程仍须完成全部游戏材料、loader、overrides、收据和发布，不自动启动。

最终报告明确“第三方 ID/大小/归档验证；无官方 hash”的文件集合，强 hash 与 hashless 分开计数。全部通过已明确选择的验收后，才允许父流程登记 installed；installed 不等于已启动或全部文件经作者认证。来源/验证选择变更重新确认；无个人 Key 不代表无网络、无信任或受限文件必然可得。

## 已核对的接口边界

- [CFWidget 公开接口样本](https://api.cfwidget.com/638111)：id=638111，files 中 fileID=6372979，name=Prism-1.21.1-neoforge-1.0.11.jar，filesize=62031；不含官方强 hash。
- [对应官方文件页](https://www.curseforge.com/minecraft/mc-mods/prism-lib/files/6372979) 提供同一 project/file Maven 示例。
- [Curse Maven 接口说明](https://www.cursemaven.com/) 定义 descriptor/projectID/fileID，不保证覆盖全部包或所有资源类型。
- 开发探针实际下载该文件为 HTTP 200；`.jar.sha1` 返回 404。一个样本不是 ATM10 全部 498 项已解析、安装或兼容的证明。
- [Prism 应用 Key 选择](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/Application.cpp#L1950-L1958) 与 [请求头](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/launcher/net/ApiHeaderProxy.h#L65-L73) 解释“不需要玩家填 Key”而非“官方服务无需认证”。
