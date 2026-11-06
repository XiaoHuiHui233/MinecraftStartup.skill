# 微软登录执行流程

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

执行前遵循 [玩家运行能力契约](../SKILL.md#玩家运行能力契约)：只用已有专用工具或目标 OS 原生命令；缺能力 BLOCKED，不安装额外编程运行时/库（游戏 Java/材料下载例外）、不写临时脚本、不用 Computer Use 或 GUI 自动截图。游戏内操作和视觉验收由玩家完成确认。

所有 HTTP/授权轮询/元数据/文件请求前先执行 [公共网络策略](operations.md#网络与来源) Step 1；management.network.offline=true 时停止网络子步骤，不把离线账号误当全流程禁联网。策略检查返回后继续本文件当前步骤，不重复执行无关模块。

按以下步骤用现有 HTTP 工具或原生命令依次执行请求和轮询，按需读取本项目凭据并传递响应字段；不要求专用秘密任务或模型不可见通道。对玩家只展示官方 verification_uri、user_code、有效期、阶段、错误摘要及角色资料，不展示完整授权响应或 token。不能仅说“调用登录工具”而不执行下列步骤。

## 首次登录

### Step 1：读取应用配置

读 accounts.microsoft.clientId，字段缺失从模板补默认公开 ID `c36a9fb6-4f2a-41ff-90bd-ae7cc92031eb` 并保存。显式为空才询问如何配置。说明授权页可能显示 PrismLauncher，不声称本项目拥有该应用或获其背书。public client ID 不是秘密。

### Step 2：准备 HTTP 与凭据存储

核实已有 HTTP 工具或原生命令可发请求和按 interval 轮询；定位系统凭据存储及本项目 credentialRef，需要刷新时正常读取。允许普通 HTTP 响应解析及普通进程参数传递，不要求专用 OAuth、安全编排、秘密中转或私密注入工具，不因缺少这些接口在发码前阻断。只有实际 HTTP、凭据读写或服务请求失败才报告具体错误；不索要聊天粘贴 token。额外运行时和临时脚本禁令不变。

### Step 3：请求设备授权

仅在玩家已选择微软登录后，向以下地址 POST 表单：

```text
https://login.microsoftonline.com/consumers/oauth2/v2.0/devicecode
client_id=<配置值>
scope=XboxLive.SignIn XboxLive.offline_access
```

Content-Type 为 application/x-www-form-urlencoded，Accept 为 application/json。验证返回包含非空 device_code、user_code、HTTPS verification_uri，以及正数 expires_in/interval。缺字段或 HTTP 错误报告具体阶段，不能造登录链接或复用过期代码。

### Step 4：提示玩家登录并轮询

直接在当前玩家发起的会话中显示微软返回的 verification_uri、user_code 和剩余有效期，不再要求只能通过本机弹窗提供。固定提示为：“请打开微软官方地址 <verification_uri>，输入设备授权码 <user_code>，核对授权应用和账号后确认；此码将在 <剩余有效期> 后失效。完成后无需把任何验证码或令牌发回聊天。”可同时打开官方页面；玩家自行输入该码。

只显示本次响应中的 user_code，不显示 device_code、完整响应、token；不自行生成或复用过期代码。user_code 不应公开发布到仓库/网页或交给无关者代授权，不能宣称无任何风险。密码和短信/MFA 验证码仍由玩家本人或宿主专用凭据界面处理，不在聊天收集，也不通过普通浏览器输入工具代填。后台按 interval 轮询同一 `/consumers/oauth2/v2.0/token`，表单为 client_id、grant_type=urn:ietf:params:oauth:grant-type:device_code、device_code。

authorization_pending 等待；slow_down 后续间隔加 5 秒；连接超时间隔退避。拒绝/取消/期限到达停止，重发要玩家同意。成功在内存保留 access_token、refresh_token、expires_in 及实际取得时间，继续下一步。等候玩家时报告 WAIT_INPUT，但已获同意的本地授权轮询可持续，不要反复创建授权请求。

### Step 5：Xbox 用户认证

POST https://user.auth.xboxlive.com/user/authenticate，JSON/Accept 头，x-xbl-contract-version: 1。本地填充：

```json
{
  "Properties": {
    "AuthMethod": "RPS",
    "SiteName": "user.auth.xboxlive.com",
    "RpsTicket": "d=<MSA access token>"
  },
  "RelyingParty": "http://auth.xboxlive.com",
  "TokenType": "JWT"
}
```

读取 Token、DisplayClaims.xui 中的 uhs；缺失则停止。RelyingParty 字符串不是请求地址，不发送 token 到 HTTP 站点。

### Step 6：XSTS 授权

POST https://xsts.auth.xboxlive.com/xsts/authorize，使用同一组 JSON 头：

```json
{
  "Properties": {"SandboxId": "RETAIL", "UserTokens": ["<Xbox Token>"]},
  "RelyingParty": "rp://api.minecraftservices.com/",
  "TokenType": "JWT"
}
```

读取 Token 与 uhs，检查 uhs 与 Step 5 一致。失败只回报 HTTP 状态/脱敏 XErr；家庭、年龄、地区、封禁限制要求玩家到官方处理，不无限重试或自动离线。

### Step 7：取得 Minecraft 会话

POST https://api.minecraftservices.com/launcher/login，JSON 为：

```json
{"xtoken": "XBL3.0 x=<uhs>;<XSTS Token>", "platform": "PC_LAUNCHER"}
```

检查 access_token 与 expires_in；本地保存，不输出。服务拒绝或 schema 变化时 BLOCKED，核对实际协议，不擅自混用旧教程端点。这是固定 Prism 提交的实现，不能当永不变化的公共契约。

### Step 8：核对游戏授权与角色

使用实际 Minecraft token 通过 HTTP Authorization 头的 Bearer 方案授权，由当前 HTTP 请求工具填值，不输出头内容。GET https://api.minecraftservices.com/entitlements/license?requestId=<工具生成UUID>，解析 items[].name；game_minecraft 记 canPlay、product_minecraft 记 owns，分开记录，不用 owns 替代可玩检查。未知格式停止。

再 GET https://api.minecraftservices.com/minecraft/profile，读取 id 和 name，验证 UUID；404 记 no_profile，无可玩授权记 no_entitlement，限制记 restricted。只有 canPlay 和有效角色都成立才 ready。不得将 OAuth 成功当游戏授权成功；本 Skill 不假称实现了 entitlement 签名校验。

### Step 9：保存并核对

安全存储 MSA refresh token、MC 会话、应用 ID、到期时间，得到稳定非秘密 credentialRef。使用 `microsoft-<UUID无连字符>` 去重注册，保存 playerName、uuid、status 和句柄；首个账号选中，其他情况保留原选择。执行配置保存，核对凭据记录可读取且到期时间、应用 ID 正确。成功返回 DONE 和脱敏账号信息，销毁过期设备会话/临时秘密；失败不覆盖旧有效账号。

## 刷新与复用

### Step 1：定位句柄

读账号 credentialRef，用系统凭据命令或现有工具读取本项目凭据；缺失记 needs_login，转首次登录，不伪造 token。

### Step 2：核对应用和期限

应用 ID 与原授权不同须重新登录。直接启动缓存命中时，MC 会话未到期且 launch-cache.authValidation 匹配当前角色/clientId/会话修订号并仍有效，直接复用已验证可玩性/角色，不重复发 entitlement/profile 请求；没有有效记录才重新核对。MC 会话有效时不因 MSA access token 单独过期而阻断游戏；MC 会话需要续期时才通过 MSA refresh token 刷新。网络策略仍实时检查。

### Step 3：刷新微软会话

POST 同一 `/consumers/oauth2/v2.0/token` 表单 client_id、grant_type=refresh_token、refresh_token=<安全读取值>。成功保留轮换后的 token，服务要求交互时转首次登录；网络失败不删除原秘密。

### Step 4：更新游戏会话

对成功的新 MSA 会话执行首次 Step 5～8，再由 Step 9 安全存储并合并原账号。不能只刷新微软就沿用过期 MC token。

### Step 5：返回

验证句柄/配置，返回真实 ready 或 needs_login/no_entitlement/no_profile/restricted；成功的 entitlement/profile 检查把非秘密角色、会话修订号和不晚于 MC 会话期限的 validUntil 返回缓存维护流程，刷新/账号改变必须替换旧 authValidation，无验证不造记录。将有效会话用于后续启动，不向聊天展示令牌。

## 失败处理

用户取消/过期不留假账号；429 按服务等待，5xx/网络有限退避；400/401/403 记录所处步骤并核对原因。不能笼统归咎玩家未登录、关闭 TLS 或擅自换客户端身份。日志永不打印完整授权响应。

## 实现依据

固定源码基线：PrismLauncher `181408813f46367ee8f709194424292bb355bbaf`。流程与默认公开 client ID 来自其源码，不复制账号秘密或调试响应日志：

- [公开 Client ID 与上游使用说明](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/CMakeLists.txt#L265-L274)。上游要求重命名项目更换其 API 标识；默认值的纳入不构成上游授权声明，发布方仍需处理适用条款。

- [设备授权与轮询](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/auth/steps/MSADeviceCodeStep.cpp#L58-L271)
- [认证步骤顺序](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/auth/AuthFlow.cpp#L20-L39)
- [Xbox](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/auth/steps/XboxUserStep.cpp#L17-L75)、[XSTS](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/auth/steps/XboxAuthorizationStep.cpp#L24-L94)
- [MC 登录](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/auth/steps/LauncherLoginStep.cpp#L20-L74)、[可玩性/所有权解析](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/auth/Parsers.cpp#L351-L380)
- [刷新前检查](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/auth/steps/MSAStep.cpp#L185-L205)
