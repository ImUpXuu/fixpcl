# 使用指南

## 前置条件

| 项 | 要求 |
|----|------|
| 操作系统 | Windows 10 / 11 |
| PowerShell | 系统自带 5.1 即可（无需 PowerShell 7，无需安装任何软件） |
| 目标文件 | PCL 官方 2.99.0.0 正式版 `Plain Craft Launcher 2.exe` |

版本号可以在 exe 属性的"详细信息"里确认，也可以不看——工具会自己校验，版本不符会拒绝执行并提示。

## 方式一：双击（推荐）

1. 把 `fixpcl.cmd` 与官方 `Plain Craft Launcher 2.exe` 放同一文件夹；
2. 双击 `fixpcl.cmd`；
3. 看到 `===== REPAIR FINISHED =====` 即成功，同目录会多出 `Plain Craft Launcher 2.patched.exe`；
4. 备份你 PCL 安装目录里的旧 exe，用补丁版替换。

文件夹里没有 exe 时，工具会依次尝试 GitHub raw 和 jsdelivr 两个镜像自动下载官方 2.99.0.0，下载后自动继续。

## 方式二：命令行 / 环境变量

工具本质是一个 cmd 引导 + 内嵌 PowerShell 的单文件，高级用法通过环境变量控制：

```bat
:: 指定官方 exe 的位置（可以是任意路径）
set PCL_EXE=D:\downloads\Plain Craft Launcher 2.exe
fixpcl.cmd

:: 生成后自动启动 12 秒做冒烟测试（默认关闭）
set PCL_SMOKE=1
fixpcl.cmd
```

也可以直接调用内嵌的 PowerShell 主体：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\pcl-repair.ps1   # 如已从 cmd 中提取
```

## 成功时的输出

```
[+] 目标: ...\Plain Craft Launcher 2.exe (v2.99.0.0)
[+] dnlib 已加载
[+] Dialogs.SelectFile 引用就绪
[*] 开始打补丁 ...
[+] P5  版本修改页 MC 卡片 -> Visible
[+] P3  下载页加载器 -> DlClientListLoader (移除 13 条指令)
[+] P2  整合包安装调度已重建 (76 条指令)
[+] P1  整合包安装入口已重建
[+] P6  资源详情页安装分支已恢复 (29 条指令)
[+] 写盘前校验通过（无悬空引用）
[+] 已生成: ...\Plain Craft Launcher 2.patched.exe
[+] 写盘后复检通过：P1/P2/P3/P5/P6 全部生效
```

任何一步失败都会显示具体原因并以非零退出码结束，**不会**生成残缺文件。

## 修复后建议做的事

1. **关闭自动更新**：PCL 设置中关闭更新相关选项。否则补丁版联网检测到新版本后可能被官方受限版覆盖；
2. （可选）关闭匿名统计：补丁版仍保留官方遥测，介意可在设置中关闭；
3. 想回到官方原版：用你备份的 exe 换回去即可（补丁从不修改原文件）。

## 常见问题

**Q: 杀毒软件报警？**
工具的形态（cmd 引导 + 大段 Base64 + 修改 PE 文件）确实容易触发启发式。本机 Windows Defender 实测不拦截。如被拦截可添加信任，或自行审查源码后重建（见 [how-it-works.md](how-it-works.md) 末尾的构建说明）。

**Q: 提示"版本不匹配"？**
工具仅支持官方 2.99.0.0。你的 exe 是其他版本（或已被修改过）。请获取官方 2.99.0.0 正式版。

**Q: 自动下载失败？**
GitHub 与 jsdelivr 都不可达时会提示手动放置 exe。也可以自己从 PCL 官网下载正式版后放入同目录。

**Q: 双击后窗口一闪而过？**
用命令行方式运行看完整输出；或先 `cd` 到该目录再执行 `fixpcl.cmd`。窗口末尾有 `pause`，正常不会闪退——闪退通常是文件被杀软拦截删除。

**Q: 修复后下载页还是空的？**
确认你运行的是 `*.patched.exe` 而不是原文件；确认网络能访问 Mojang 版本清单（`piston-meta.mojang.com`）。

**Q: 以后 PCL 出新版了能用吗？**
不能。补丁与 2.99.0.0 的二进制结构强绑定，新版会被版本守卫拒绝。需要等本工具适配新版本。
