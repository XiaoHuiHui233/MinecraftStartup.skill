# 下载准备

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

所有 HTTP/授权轮询/元数据/文件请求前先执行 [公共网络策略](operations.md#网络与来源) Step 1；management.network.offline=true 时停止网络子步骤，不把离线账号误当全流程禁联网。策略检查返回后继续本文件当前步骤，不重复执行无关模块。

只在玩家要求下载/修复或已确认后执行。“下载并启动指定版本”已经授权该版本官方游戏材料，不重复要求同一下载授权。额外下载 Java、换来源、执行第三方 installer 另行确认。

## 获取原版版本元数据

### Step 1：读取选择

读取实例 MC/loader 及 creationConfirmed。创建选择已确认就直接准备下载，不追加同范围许可问题；尚未确认回到实例选择表单。

### Step 2：读取 Prism 索引

执行 [Prism 版本列表](prism-components.md#读取组件版本列表)，以默认 metadataBaseUrl 加载总索引、net.minecraft 和所选 loader 索引，按 SHA-256 链验证。

### Step 3：解析全部依赖

执行 [Prism 组件解析](prism-components.md#解析并合成实例组件)，递归锁定 requires，不能漏掉 LWJGL 等独立组件。明确需要的运行环境架构，不因 Java major 未豁免而拒绝下载游戏材料。

### Step 4：保存 profile

写实例根 mmc-pack.json、metadata 原文件和 launch-profile.json，保留 hash/URL 来源，重读核对。不把 Prism 格式假装保存为 Mojang 原始 version JSON。

### Step 5：返回

返回 provider=prism 的合成 profile、Java major 列表/运行时名称及文件清单。官方原始 JSON 仅作玩家明确选择的兼容链路，不在失败时静默切换。

## 单文件下载

### Step 1：验证目标和来源

输入必须有 URL、目标、预期大小/hash（若提供）和所属任务。检查 HTTPS、允许的官方/已批准来源及每次重定向，不可导向本机/内网地址；路径必须在目标数据范围内，拒绝穿越/链接逃逸。

### Step 2：验证已有文件

存在时按可得 hash 重算并核对大小；匹配就返回文件，不下载。不仅比大小。元数据无 hash 时标 unverifiable，不报完整性通过；执行程序来源无可核验依据时停止确认。

### Step 3：下载到暂存

根目录 downloads 下创建任务唯一临时文件；流式下载并累计字节/hash，不覆盖其他临时文件。TLS 验证保持开启，每请求超时 60 秒；单文件总尝试最多 3 次；网络/5xx 退避，429 遵循 Retry-After 且计入次数，其他 4xx 停止。默认最多 4 个文件并发。

### Step 4：校验与提交

核对预期大小/hash，失败隔离临时文件、不提升为缓存。成功提交目标；跨磁盘先复制到目标同目录临时文件再替换，不假称跨盘 rename 原子。已存在不同内容属于本次基础缓存才替换，不碰用户内容。

### Step 5：重新检查

检查最终位置及可得 hash，返回路径、验证状态。失败给出 URL/相对目标、HTTP/校验错误及已保留部分，不能把暂存完成说成安装成功。

## 安装原版材料

### Step 1：取合成 profile

执行“获取原版版本元数据”，确认 provider=prism 和源 hash；已有已验证 profile 可复用但仍检查与实例选择一致。

### Step 2：确定运行环境

使用实例所选 Java 实测架构，未选且只有一个可执行 Java 时以其作材料规划；多项或没有时请求选择。执行 Prism 的 classifier/rules，不按 Mojang 简单 OS 规则遗漏 ARM 替代组件。规划不是 major 豁免。

### Step 3：执行任务链

执行 [Prism 更新任务](prism-components.md#执行更新任务)：FoldersTask → LibrariesTask（包括 mainJar）→ 必需时 LegacyFMLLibrariesTask → AssetUpdateTask。材料 URL 来自已校验组件，文件最终进入本实例独立目录。

### Step 4：核对结果

检查全部普通/native 库、主 JAR、asset index/对象及日志文件。失败标 download_failed 并保留可验证缓存，不报告安装完成。未知 processors/格式明确报具体阶段，不编造成功。

### Step 5：保存返回

所有必需材料通过才设 installed，执行配置保存和重读；仅创建请求到此结束，启动请求返回主启动流程。创建确认已包含这些下载，不再追问一次。

## 自动下载 Java

### Step 1：取得玩家授权和 profile 要求

只有玩家选择自动下载才继续。读取 compatibleJavaMajors 与 compatibleJavaName，按其顺序找运行时；不能因为创建时自动下载游戏就擅自装 Java。显示目标 Java、平台、来源和数据根目录 java 安装位置。

### Step 2：走 Prism Java 元数据

通过同一索引/hash 链加载 net.minecraft.java/index.json，再对需要的 major 加载版本 java<major> 的组件 JSON。检查 runtimes 数组，按 runtimeOS 和 profile 的 compatibleJavaName 精确匹配。runtimeOS 显式映射如 Windows x64=windows-x64、ARM64=windows-arm64，macOS x64=mac-os-x64、ARM64=mac-os-arm64，Linux x64=linux-x64、ARM64=linux-arm64；其他架构先核对实际源码，不按字符串猜。

匹配项读取 url、downloadType、checksum.type/hash、version.major、name。无匹配尝试 profile 的下一兼容 major；均无匹配说明缺口。不得自动改架构、安装 Rosetta 或静默换成别的厂商 API。玩家另指定厂商时才进入其已核实来源。

### Step 3：按 downloadType 安装

manifest：先校验顶层 manifest，再处理 files 的 directory/file/link，逐文件取 downloads.raw.url/sha1，设置 executable 标记；所有路径与 link target 限定在该安装目录内。archive：按 checksum（只接受已知 sha1/sha256）验证包，再安全解压，按实际布局移除已证实单层外壳。无 hash 或未知类型停止确认，不假报验证成功。

暂存及安全限制沿用单文件下载：20,000 个成员、单文件 2 GiB、累计展开 8 GiB，预先检查磁盘余量；超限询问。最终安装在 java/<稳定安装ID>，不覆盖旧安装。

### Step 4：适配路径并探测

优先定位 bin/java（Windows java.exe）；macOS 按实际 Contents/Home/bin/java 定位，必要时仅在此 runtime 内建立安全链接，不改系统 Java。执行版本/架构探测，必须与选择一致。失败不注册 available，不清 pendingDownload。

### Step 5：注册保存

注册 downloaded 来源与真实可执行路径，按玩家指定绑定实例并重置旧 ignoreJavaCompatibility，清 pendingDownload，执行配置保存及重读。返回实际 Java，不把下载成功当游戏启动成功。

## 实现依据

参考 Prism 的“库任务 → asset index → 对象下载”“运行时选取 → manifest/archive 安装 → 执行验证”顺序。默认与 Prism 使用同一组件元数据入口和依赖链；真实文件仍按组件指定 URL 获取，不将它误称所有文件都由 Prism 托管。完整校验比 Prism 此提交仅凭存在/大小复用部分缓存更严格。

固定源码基线 `181408813f46367ee8f709194424292bb355bbaf`：

- [库下载任务](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/update/LibrariesTask.cpp#L13-L71)
- [asset index 与任务顺序](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/update/AssetUpdateTask.cpp#L19-L91)
- [assets 对象布局与缓存判断](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/AssetsUtils.cpp#L269-L311)
- [Java 自动选择](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/minecraft/launch/AutoInstallJava.cpp#L63-L145)
- [Java manifest 下载](https://github.com/PrismLauncher/PrismLauncher/blob/181408813f46367ee8f709194424292bb355bbaf/launcher/java/download/ManifestDownloadTask.cpp#L38-L124)
