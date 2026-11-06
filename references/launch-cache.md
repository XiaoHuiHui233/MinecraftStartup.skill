# 启动指令缓存

<!-- SPDX-License-Identifier: GPL-3.0-only; Upstream notices and adaptation details: ../NOTICE.md; license: ../LICENSE. -->

缓存是每实例独立的已分词 Java 指令数据，不是启动脚本。直接启动优先执行 [主入口顶部快速路径](../SKILL.md#最高优先级直接启动走缓存)，命中无需阅读本文件或其他管理手册。本文件只在配置变更、安装完成或缓存失效时加载。玩家运行能力与授权边界不变，不生成 helper、不安装额外运行时、不做 GUI 自动化。

## 缓存数据契约

固定位置：`<instanceRoot>/launch-cache.json`；模板见 [launch-cache.example.json](../templates/launch-cache.example.json)。模板为 uninitialized，不能当真实检测结果。缓存必须只允许当前用户修改，拒绝根外链接；不从下载包直接信任导入的缓存，复制/恢复/迁移后在目标机器重新生成。

- `schemaVersion: 1`、`state: ready | stale | uninitialized`、`instanceId`、`generatedAt`、`os`、`architecture`。
- `executable` 为已验证 Java 绝对路径；`cwd` 为目标实例 gameRoot；`argvTemplate` 不含 executable，每项是已经分词的字符串，包括完整单项 classpath。静态值全部解析，不缓存可执行 shell 字符串。不把 lastLaunch 当缓存。
- `dynamicBindings` 为数组，每项 `{index, valueTemplate}`，index 为 argvTemplate 的零基下标且唯一；只允许 `${auth_access_token}` 和 `${auth_session}`，对应 argvTemplate 项必须与 valueTemplate 相同。session 值按 `token:<当前MC token>:<当前角色UUID>` 生成，不能缓存已注入版本；普通账户名称、UUID、user_type 在配置绑定中固定，切账号重建。离线身份按既有规则填 0/-，不可把离线值替换为在线授权。
- `configBinding` 的固定投影为 `{dataRoot, folders, instance, java, account, clientId}`：instance 为该完整实例记录移除 status/installationError，java 为该实例 javaId 对应的完整记录移除 status，account 为当前实际账号完整非秘密记录移除 status，clientId 为 accounts.microsoft.clientId（无则 null）。比较结构化 JSON 值，不比较整个 config.json hash；selectedId 的变化只定位目标，不让其他实例缓存失效。配置中的 lastLaunch、operations、backups、content 记录与管理网络策略不进入投影，网络策略每次实时读取。若未来 argv 使用投影之外的设置，须先升级协议并扩展固定投影；旧 schema 不准静默忽略新输入。
- `sources` 保存实际 launch-profile、mmc-pack（若存在）、installation-plan、installation-result 及源组件文件的绝对 path/sha256；现代 loader 同时绑定 loader-install-receipt、install_profile/version JSON、包状态指针。只散列这些小文件，不散列全配置或全部 assets。
- `javaIdentity` 与 `requiredFiles[]` 保存实际 path/size/mtime（宿主最高可用精度）/fileId（可用时），后者覆盖普通库、主 JAR、日志配置、asset index、已提取 natives、现代 loader 的 installer/wrapper/mavenFiles 和实际 patch 产物；有上游校验值时登记 expectedHash/algorithm，native 等本地派生文件登记 localSha256，不能冒充上游校验。Java 另记录实际 version/architecture 与实例豁免组合。路径必须归属于当前实例或已注册 Java；classpath 路径顺序不变。
- `nativeDir` 是已安全提取、架构验证、由本实例独占使用的真实目录，可复用之前的任务目录；任务退出不自动删除缓存引用的目录。移除该目录前先使缓存失效。native 内容不变无需每次复制或解压。
- `installationReceipt` 保存真实已验证安装收据路径/hash；快速启动只检查收据与关键文件，不全量遍历/散列 assets。外部 assets 删除可能到运行时才被发现，此时转安装补缺；不要宣称快速检查等于完整安装验证。
- `authValidation` 只存 accountId/clientId/uuid/playerName/canPlay/profileCheckedAt/validUntil/sessionRevision；sessionRevision 使用本项目安全存储中的非秘密 savedAt 或单独生成的会话修订号，用于绑定实际取得验证的当前会话（不得使用 token 本身）。来自真实服务检查，validUntil 不得超过对应 Minecraft token 到期时间；没有这份记录或身份/会话变更时只补认证检查。token 刷新、服务拒绝或账号改变使此验证失效；成功刷新重新核对授权与角色。access/refresh token、device_code、完整授权响应、已注入真实 token 的 argv 均禁止进入缓存。联网检查仍服从当前 offline 策略。

## 配置变更后刷新缓存

所有模块每次完成配置保存和读回后调用本子流程；游戏 options、版本、loader、Java、Mod/整合包部署、实例路径、复制/恢复等本地变更即使没有改变全局配置，也在收尾调用。只读请求不调用。不得只给 selectedId 对应实例写缓存。

### Step 1：确定受影响实例

比较本次提交 before/after 固定配置投影和实际文件写集。实例/Java 修改影响引用该记录的所有实例；账号选择/角色/clientId 变化影响使用该账号选择的所有已安装实例；根路径改变影响全部实例。仅切换 selectedId 不覆盖其他实例缓存，但目标实例缺缓存时补建。新增/恢复/复制实例必须使用目标稳定 ID/路径重建；删除实例不保留可启动条目，不允许给未来新实例复用旧身份。仅 lastLaunch/就绪/退出/操作历史等无启动输入变化时保留现有指令，不重复重建或扫描其他实例。每次保存都完成这项判定，不能把无关变更当成清空所有缓存的理由。

### Step 2：稳定锁下读取真实已提交状态

在配置 Step 5 读回后，先释放本层 config.lock，再进入本流程取得父级稳定操作锁→实例锁；已有相同 operation 的锁标记 borrowed，本层新取得的标记 acquiredHere。不可持 config.lock 等待实例锁造成反向死锁。先确定完整受影响集合，再按稳定 ID 排序取锁；禁止持某实例锁后反序扩展集合。账号刷新等全局变更优先在取得实例锁前完成。若调用方已经持有较后实例的锁且需扩展集合，不等待/获取较前锁：将完整待刷新集合和原因写入外层 journal，配置仍为 committed；旧绑定或会话修订号不同的缓存不得执行。先返回外层完成当前受保护写集并释放其锁，再按完整集合排序刷新，刷新结束才报告缓存结果；失败保留 pending/stale 原因，不谎称全部缓存更新。调用方必须消费该 deferredCacheRefresh 结果，不得丢弃，也不能通过子流程释放借用锁来抢先扩锁。重读配置和必要本地 profile/安装收据；并发改动则重新判定，不用提交前旧数据写 ready 缓存。检查实际实例进程，运行中不修改其 natives/内容；纯缓存刷新可保存供下次使用的参数。

### Step 3：生成或复用指令

输入及文件身份未变且缓存 ready 时保留原 argv，只更新确有变化的非秘密验证信息，无需重建。其他情况仅加载 [command 构造](launch-command.md#拼接-command) 的必要步骤，用已安装材料生成规范 argv；先分词后替换，始终保留会话动态槽，不能从含 token 的历史日志/进程 command 字符串反推缓存。按真实平台生成，不硬编码 MC/Fabric/Java 版本。一次性世界/服务器/临时参数不写基础缓存，防下次普通启动自动加入旧目标。

已安装且本地材料齐全时立即生成并保存，不推迟到下次启动。nativeDir 缺失仅在停机及安全提取能力齐全时重建；运行中或未安装完、账号/Java未选定、材料缺失时写 state=stale 与具体 reason/missing（如 JAVA_NOT_SELECTED、JAVA_ARCHITECTURE_UNKNOWN、LOADER_INSTALL_INCOMPLETE），即使配置中有 javaId 也不把未实测架构当 ready；此时不伪造 ready、不发起未授权下载/认证/启动；报告配置已提交但缓存尚未就绪。不得把重建失败说成原配置保存失败，也不回退已经成功的配置。下一次直接启动按该原因补齐。

### Step 4：原子发布并读回

缓存放同目录唯一暂存，权限限制当前用户；写入来源 hash、配置绑定、文件身份与脱敏 argv，校验动态槽和 schema。提交前重读负责投影和源文件指纹，变化则丢弃本任务候选并停止发布；记录非秘密 cache before/after hash 与阶段到当前 operation journal，或 dataRoot/.management/journals 下独立 UUID journal。原子替换 launch-cache.json 后读回，验证 argv 元素、目标 ID/cwd、来源和动态槽一致，无凭据泄漏。缓存写入不调用配置保存，禁止保存→刷新→保存递归；仅释放本层 acquiredHere 锁；borrowed 锁由最外层操作收尾释放，子流程即使 operation 相同也不得提前释放。返回调用方收尾。只把实际就绪实例报告为缓存已更新。

## 缓存缺失或失效时的定点回退

### Step 1：分类，不先全量重装

缺缓存且实例已安装：从本地 profile/安装收据重建，先执行上方刷新 Step 1～4，不重新联网查询所有版本。仅凭缓存损坏不能执行其指令；有未知 schema 或非法动态槽则拒绝并重建。

### Step 2：只修必要前提

配置投影改变只重算受影响 argv；Java 身份改变才探测该 Java 并重核架构/兼容性；库/native 身份改变只校验变化文件，hashless 文件无法确认时失效。profile/源组件改变、缺安装收据或材料缺失才进入 [完整启动](launch.md#启动游戏) 对应准备步骤（元数据 Step 6、Java Step 7、材料 Step 8、natives Step 10、命令 Step 11），不得一律从初始化/全 PATH 扫描开始。只有认证问题读取 [微软刷新](microsoft-login.md#刷新与复用)，有效会话已有有效验证则不请求网络。无法安全完成则 WAIT_INPUT/BLOCKED，准确说明缺项。

### Step 3：返回快速路径，不重复创建进程

必要准备完成后执行缓存刷新，成功返回主入口快速 Step 2，并由快速 Step 4 唯一创建进程；若完整启动已进入 Step 12 则不得返回再启动一次。一个请求最多一次失效重建，第二次仍不一致返回 BLOCKED: LAUNCH_CACHE_UNSTABLE。保留具体原因，不自动换源、改账号或跳过安全检查。
