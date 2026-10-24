# fixpcl

> PCL (Plain Craft Launcher 2) v2.99.0.0 MC 下载功能一键修复工具 —— 单文件、双击即用、无需安装任何依赖。

官方 2.99.0.0 为遵守相关法律法规关闭了 MC 下载功能（原版安装、整合包安装等）。
本工具直接对**官方发布的 exe** 做二进制 IL 手术，把被编译期裁剪掉的下载入口按上一无限制版本（2.14.0.0）的源码语义原样接回。

## 快速开始

1. 把官方 2.99.0.0 的 `Plain Craft Launcher 2.exe` 和本工具放在**同一文件夹**；
2. 双击 `fixpcl.cmd`，等待约 10 秒；
3. 同目录生成 `Plain Craft Launcher 2.patched.exe`（原文件不会被修改）；
4. 用它替换你 PCL 目录里的 exe（替换前请自行备份）。

文件夹里没有官方 exe？工具会自动从 GitHub / jsdelivr 下载官方 2.99.0.0 再修补。

## 恢复了什么

| # | 功能 | 位置 |
|---|------|------|
| P3 | 原版下载页：Mojang 官方版本列表加载与安装 | `PageDownloadInstall.LoaderInit` |
| P2 | 整合包文件安装：CurseForge / HMCL / MMC / MCBBS / Modrinth / 带启动器压缩包 / 未识别压缩包 共 7 条安装链路 | `ModModpack.ModpackInstall(File,…)` |
| P1 | "安装整合包"入口：文件选择对话框 + 后台安装 | `ModModpack.ModpackInstall()` |
| P5 | 版本修改页的 MC 版本卡片显示 | `PageDownloadModify` |
| P6 | 资源详情页整合包"点击安装"按钮 | `PageDownloadResourceDetail` |
| P7 | **禁用自更新**：服务器推送的静默更新与手动触发的更新均不再下载替换，防止补丁版被官方版覆盖 | `ModSecret.UpdateStart` |

下载的游戏文件来自 Mojang 官方 CDN，与 PCL 原有行为一致。

## 工作原理（一句话版）

限制的实现方式是 `Public Const RestrictMcDownload As Boolean = True` 触发的**编译期常量折叠**——守卫分支被消除、下游代码被裁剪，二进制里没有开关可翻。而 7 个安装链路方法、版本清单加载器、闭包类等"器官"仍完整幸存，本工具用 dnlib 把 5 个"总闸"的接线按 2.14 源码语义重新缝回去。详见 [docs/how-it-works.md](docs/how-it-works.md)。

## 环境要求

- Windows 10 / 11（自带 Windows PowerShell 5.1，无需安装任何东西）
- 官方 2.99.0.0 正式版 exe（其他版本会被版本守卫拒绝）

## 内置安全机制

- **版本锁**：仅接受 FileVersion = 2.99.0.0，否则拒绝执行；
- **只读原文件**：补丁输出为独立的 `*.patched.exe`；
- **写盘前校验**：全模块扫描悬空引用（分支目标 / 异常处理边界），任何异常立即中止；
- **写盘后复检**：重新加载产物，逐项验证 P1/P2/P3/P5/P6 全部生效，失败即删；
- **确定性输出**：同一官方 exe 的多次修复产物 SHA-256 完全一致。

## 文档

- [docs/usage.md](docs/usage.md) —— 详细用法、环境变量、常见问题与故障排查
- [docs/how-it-works.md](docs/how-it-works.md) —— 补丁原理：常量折叠、幸存代码、五处 IL 手术细节
- [docs/analysis-2.99.0.0.md](docs/analysis-2.99.0.0.md) —— 2.99.0.0 关闭下载的分析过程与证据
- [docs/verification.md](docs/verification.md) —— 验证方法：反编译回读、哈希确定性、边界测试

## 局限

- 与 2.99.0.0 二进制结构强绑定。官方发布新版本后本工具会**拒绝执行**（版本守卫与结构定位双重保险），需要重新分析适配；
- 自更新已被禁用（P7）：设置中的"检查更新"按钮仍可检测新版本，但点击更新后不会实际下载替换；
- 遥测与公告等联网行为与官方原版一致，未被改动。

## 法律与使用声明

使用前请阅读完整的 [DISCLAIMER.md](DISCLAIMER.md)。要点：

- 本项目与 PCL 作者（龙腾猫跃）及 Mojang / 微软 / 网易均无关，仅为技术研究与个人互操作用途；
- 本仓库**不分发** PCL 本体或修补后的二进制，仅提供在用户本机上运行的工具；
- **修补产物仅限个人使用**，请勿分发——分发既不符合 PCL 的许可条款，也处于合规灰色地带；
- 本工具不绕过任何正版验证或 DRM，恢复的下载来自 Mojang 官方 CDN；请**购买正版 Minecraft**；
- 工具按"现状"提供，无任何担保，使用风险自担；
- 补丁已**自动禁用自更新**（P7），补丁版不会被官方受限版静默覆盖。

## 致谢

- [PCL / Plain Craft Launcher 2](https://github.com/Meloong-Git/PCL) 及其作者龙腾猫跃；
- [dnlib](https://github.com/0xd4d/dnlib)（MIT License，已以 Base64 形式内嵌于单文件工具中）；
- [ILSpy / ilspycmd](https://github.com/icsharpcode/ILSpy)（分析验证阶段使用，不随工具分发）。

## 许可

本仓库自身的原创代码与文档以 **[MIT License](LICENSE)** 发布。

注意：MIT 仅覆盖本仓库的原创部分；PCL 的权利归其原作者所有，基于 PCL 的衍生作品仍需遵循 PCL 原作者的许可条款。第三方组件声明见 [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md)。
