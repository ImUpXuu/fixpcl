# PCL 2.99.0.0 "关闭 MC 下载"分析记录

本文记录对官方 2.99.0.0 正式版（2026-10-20 构建）的静态分析过程与证据。分析工具：ilspycmd 9.1（ILSpy 命令行版）+ dnlib。

## 1. 版本背景

- GitHub Releases 上 2.14.0.0 发布于 2026-10-16，四天后（10-20）版本号直接跳到 **2.99.0.0**；
- Release 说明原文：*"为遵守相关法律法规，PCL 不再提供 MC 下载功能，非常抱歉 qwq……注：与网易微软麻将无关，中国式家长很神奇吧"*；
- 公开仓库的 `git diff 5ef81e4..6b9a50d`（2.14 → 2.99）显示代码差异极小（79 增 / 302 删），全部围绕下载限制。

## 2. 限制点全景（共 7 处，核心 5 处）

对公开仓库 diff 与官方二进制反编译结果交叉比对：

| 位置 | 实现方式 |
|---|---|
| `ModSecret.vb` | 新增 `Public Const RestrictMcDownload As Boolean = True`（2.14 无此符号） |
| `PageDownloadInstall.LoaderInit` | `If RMD Then` 用名为"关闭下载"的假加载器（worker 直接 throw）`Else` 用真加载器 |
| `ModModpack.ModpackInstall()`（无参） | 守卫后 `Hint(红条) : Return`，文件对话框流程成死代码 |
| `ModModpack.ModpackInstall(File,…)` | 解析压缩包后 `If RMD Then Throw`，7 条安装链路调度成死代码 |
| `FormMain` 更新日志 | 追加"删除：…不再提供 MC 下载功能…"文案 |
| `FormMain` 文件关联 | 捕获该异常消息并转红条提示 |
| `PageDownloadModify` | `CardMinecraft.Visibility = If(RMD, Collapsed, Visible)`（隐藏 MC 版本卡片） |
| `PageDownloadResourceDetail` | ModPack 类型资源不再显示"点击安装"按钮 |

由于是 `Const True`，VBC 编译期折叠后，**官方二进制里以上守卫全部消失，被挡代码被物理裁剪**。

## 3. 官方版与开源版的差异（顺带验证"PCL 非完全开源"）

官方 exe 中 `ModSecret` 的真实内容与公开仓库的空桩对照：

| 公开源码（空桩） | 官方二进制（真实值） |
|---|---|
| `VersionBranchMain = "OpenSource"` | `"Official"`（注册表 `PCL` 而非 `PCLDebug`） |
| `OAuthClientId = ""` | 内嵌真实微软 OAuth ClientId |
| `ClsBaseUrl = ""` | 腾讯云 CLS 埋点地址（受设置项 `SystemSystemTelemetry` 控制） |
| 更新 / 联网配置全空壳 | 完整实现（腾讯云 COS：公告 / 更新检查） |
| 赞助码空实现 | `CheckPotatoCode` + 内嵌 ECDSA 验签公钥 |
| CurseForge key 走环境变量 | `x-api-key` 以 Base64 内嵌于 `SecretHeadersSign` |

官方二进制不含开源版标记与桩实现消息，但含全部真实实现符号——证明官方版由"公开仓库 + 未公开补丁"构建。仓库根目录每个正式版还对应一个加密二进制 patch 文件，佐证这一构建模式。

另外：`MeloongCore` / `PCLCS` / `Newtonsoft.Json` 等 13 个托管依赖以字节数组内嵌于 `PCL.Resources.resources`，运行时经 `AssemblyResolve` → `Assembly.Load(byte[])` 加载，因此官方分发包是单 exe。

## 4. 关闭范围（哪些还能用）

**被封锁**：原版安装 / 下载页（版本列表都不加载）、整合包安装（本地文件与资源页一键安装）。

**未受影响**（相关模块无任何限制字符串）：Mod / 资源包 / 光影 / 数据包浏览与下载；已安装实例的启动与缺失依赖库补全。

## 5. 远程开关排查

官方版 `ServerSub` 拉取的 ServerConfig 仅使用 `Link`（下载镜像覆盖）、`Update`、`Version` 三类键，**不存在**控制下载功能的远程开关——关闭是纯编译期行为，也不存在"服务器悄悄恢复"的暗门。

## 6. 对补丁设计的影响

1. 因为是编译期裁剪 → 必须 IL 手术，不能翻开关；
2. 因为下游"器官"幸存 → 手术可行，只需重建 5 个入口的接线；
3. 因为单 exe 内嵌依赖 → P1 需要的 `Dialogs.SelectFile` 引用必须先从内嵌资源中提取 MeloongCore.Wpf.dll 才能导入；
4. 因为方法签名 / 类型布局与 2.14 兼容 → 可以直接以 2.14 源码为语义蓝本逐句翻译回 IL。
