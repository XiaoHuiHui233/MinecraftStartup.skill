# 致谢、上游来源与第三方声明

## 致敬 Prism Launcher

感谢 [Prism Launcher](https://prismlauncher.org/) 及其贡献者，也感谢其历史中的 MultiMC、PolyMC 贡献者。Prism 对实例、组件依赖、Java、认证、游戏资源及启动生命周期的开放实现，是 MinecraftStartup.skill 的重要学习与设计来源。本项目希望将这些经验转化为工具型 AI Agent 可以逐步执行的操作手册，而不是抹去上游工作的来源。

MinecraftStartup.skill 是独立项目，**不是 Prism Launcher，不是其官方版本，也未获其赞助、认可或关联背书**。致谢不是授权证明，不以 GPL 获得上游商标或服务身份。本项目不将 Prism 的 logo 作为自己的标识。

## 来源与改编说明

本项目把所参考的实现流程重新组织为中文 Skill 操作步骤、持久化约定和示例，并加入路径/授权门禁、停机检查、独立备份和失败恢复等策略。许可与署名整理日期：**2026-11-06**；该日期并非所有功能的首次创作日期。既有与后续修改应由版本历史和适用文件的修改声明追踪。

源码来源：[PrismLauncher/PrismLauncher](https://github.com/PrismLauncher/PrismLauncher)。已有流程引用基线 `181408813f46367ee8f709194424292bb355bbaf`，后续模块分析基线 `899d2f3957215a09fe398a8a8cbb8a8a9c619968`；各手册保留其实际固定提交与文件链接，不用一个新提交替换所有旧证据。

| 本项目内容 | 参考的 Prism 实现位置 |
|---|---|
| 实例创建、复制、版本及隔离流程 | launcher/InstanceList.cpp、InstanceCopyTask.cpp、minecraft/PackProfile.cpp、ui/dialogs/InstallLoaderDialog.cpp |
| 元数据、组件、库、assets 和 Java | launcher/meta/、minecraft/ComponentUpdateTask.cpp、minecraft/update/、java/、minecraft/launch/AutoInstallJava.cpp |
| 微软设备流、Xbox/XSTS、会话与角色 | launcher/minecraft/auth/ 及 steps/ |
| 参数、进程、native、日志与启动观察 | launcher/minecraft/MinecraftInstance.cpp、minecraft/launch/、launch/、LoggedProcess.cpp |
| Mod、资源及整合包 | launcher/modplatform/、minecraft/mod/、ResourceDownloadTask.cpp、ui/pages/instance/ |
| 世界、客户端服务器和截图 | launcher/ui/pages/instance/WorldListPage.cpp、ServersPage.cpp、ScreenshotsPage.cpp |

这是设计/流程来源说明，不是逐行代码复制清单，也不证明每个参考材料在法律上必然构成衍生作品。当前仓库发布 Skill 文本和 JSON 模板，不捆绑 Prism 的 C++/Java 程序、NewLaunch.jar、Qt、第三方库或品牌图片。若以后复制/改编特定源码文件或图形，必须对该材料另做逐文件许可和署名核查，不能凭本表覆盖所有上游依赖。

特别是：本 Skill 直接构造游戏主类 argv 的路径不是 Prism 的 EntryPoint/stdin Java bridge 的逐字实现；不能以致谢暗示行为等价或兼容性已获上游验证。

## 保留的上游版权与许可声明

以下原文取自上述后续基线的 COPYING.md 中 Prism Launcher 与 PolyMC 两节；保留其中 MultiMC / Apache-2.0 的继承声明。附录是适用来源的权利说明，不表示将每个上游依赖重新许可为 GPL，也不声称我们打包了上游全部程序。

```text
## Prism Launcher

     Prism Launcher - Minecraft Launcher
     Copyright (C) 2022-2026 Prism Launcher Contributors

     This program is free software: you can redistribute it and/or modify
     it under the terms of the GNU General Public License as published by
     the Free Software Foundation, version 3.

     This program is distributed in the hope that it will be useful,
     but WITHOUT ANY WARRANTY; without even the implied warranty of
     MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
     GNU General Public License for more details.

     You should have received a copy of the GNU General Public License
     along with this program.  If not, see <https://www.gnu.org/licenses/>.

    This file incorporates work covered by the following copyright and
    permission notice:

         Copyright 2013-2021 MultiMC Contributors

         Licensed under the Apache License, Version 2.0 (the "License");
         you may not use this file except in compliance with the License.
         You may obtain a copy of the License at

             http://www.apache.org/licenses/LICENSE-2.0

         Unless required by applicable law or agreed to in writing, software
         distributed under the License is distributed on an "AS IS" BASIS,
         WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
         See the License for the specific language governing permissions and
         limitations under the License.

## PolyMC

     PolyMC - Minecraft Launcher
     Copyright (C) 2021-2022 PolyMC Contributors

     This program is free software: you can redistribute it and/or modify
     it under the terms of the GNU General Public License as published by
     the Free Software Foundation, version 3.

     This program is distributed in the hope that it will be useful,
     but WITHOUT ANY WARRANTY; without even the implied warranty of
     MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
     GNU General Public License for more details.

     You should have received a copy of the GNU General Public License
     along with this program.  If not, see <https://www.gnu.org/licenses/>.

    This file incorporates work covered by the following copyright and
    permission notice:

         Copyright 2013-2021 MultiMC Contributors

         Licensed under the Apache License, Version 2.0 (the "License");
         you may not use this file except in compliance with the License.
         You may obtain a copy of the License at

             http://www.apache.org/licenses/LICENSE-2.0

         Unless required by applicable law or agreed to in writing, software
         distributed under the License is distributed on an "AS IS" BASIS,
         WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
         See the License for the specific language governing permissions and
         limitations under the License.
```

上游 GPL v3 正文完整保存在根 [LICENSE](LICENSE)；MultiMC 继承声明涉及的 Apache License 2.0 正文保存在 [LICENSES/Apache-2.0.txt](LICENSES/Apache-2.0.txt)。Apache 部分保留原有许可与适用通知，不将项目整体改成 Apache-only 或允许任选其一。新增复制材料还须检查其自己的版权、NOTICE 和例外条款。

## 品牌、API 身份与在线服务边界

- [Prism README](https://github.com/PrismLauncher/PrismLauncher/blob/899d2f3957215a09fe398a8a8cbb8a8a9c619968/README.md#forkingredistributingcustom-builds-policy) 明确区分 GPL 的法律责任与定制/改名项目的礼貌要求：清楚声明独立身份，将上游 API 标识替换成自己的或留空。不能把该礼貌要求改写成 GPL 正文的额外限制。
- **当前发布模板仍保留此前选定的 Prism 微软公开 client ID。**本次整理按维护者明确选择未更改模板或玩家配置；这不是上游对本项目的授权证明。公开、非秘密的应用标识不等于无条件获得该应用身份的使用许可；GPL 也不能代替 Microsoft/Minecraft API 准入或服务条款审查。
- 后续发布者应优先使用自己获准的 Microsoft 应用标识，或取得可核实的适用授权；不能满足时应禁用相关在线登录入口。保留当前值意味着该非代码许可风险尚未消除，不能宣称“添加 GPL 后所有服务使用已合法/已获批准”。不要仅凭请求成功或上游未追究的讨论推断授权。
- 不使用 Prism 的名字、logo 或 OAuth 页面显示名暗示本项目是官方客户端；本项目没有取得其 logo 的商标权或图片授权。上游 logo/相关素材另有 CC BY-SA 4.0 声明，本仓库没有因采用 GPL 获得这些素材的任意再许可权。
- Microsoft、Mojang、Minecraft、CurseForge、Modrinth 等名称仅用于识别协议或来源，各自权利和条款不因本项目 GPL 改变。通过元数据下载的游戏、库、Java、Mod 和图片不是一概按本项目 GPL 再许可；使用与再分发分别核查其权利。

## 下游分发提示

保留适用的版权、许可、无担保与修改说明；一并分发 LICENSE、COPYING.md、本 NOTICE 及适用许可文件。GPL 义务以 [LICENSE](LICENSE) 为准，本 NOTICE 的解释与致谢不创设额外非商用/禁止修改限制，也不授权本项目无权授权的第三方材料。

本文件是基于当前可核对材料的署名与合规说明，不是完整的第三方权利清算或特定司法辖区法律意见。尤其不能用它代替尚未解决的 API 身份/服务授权。
