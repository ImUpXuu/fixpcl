# 补丁原理

一句话：**限制是编译期裁剪掉的，不是运行时开关，所以补丁不是"翻开关"，而是把被编译器删掉的接线按 2.14 无限制版的源码语义用 IL 重新缝回去。**

## 1. 限制是怎么实现的（决定了补丁思路）

2.99.0.0 的公开源码中，所有下载入口都被同一个常量守卫：

```vb
' ModSecret.vb
Public Const RestrictMcDownload As Boolean = True

If RestrictMcDownload Then Throw New Exception("为遵守相关法律法规……")
```

关键在 `Const`。VB 编译器（VBC）对常量做**编译期折叠**：

- `If True Then Throw …` 的判断被消除，变成无条件 `throw`；
- `If RMD Then A Else B` 只保留 A 分支，**B 分支根本不会编进二进制**；
- 被 `Return` / `Throw` 挡住的后续语句作为不可达代码被删除。

因此官方 exe 里既没有开关字段可翻，也没有大量"被注释掉的功能"——它们是被物理裁剪的。

## 2. 关键发现：编译器删的是"入口接线"，不是"器官"

反编译对比 2.14.0.0（git 历史中的最后一个无限制正式版）后发现，大量下游实现完整幸存：

| 幸存物 | 为什么没被删 |
|---|---|
| 7 个 `InstallPack*` 安装方法（CurseForge / HMCL / MMC / MCBBS / Modrinth / 带启动器 / 压缩包） | VBC 的死代码消除只删**方法内不可达语句**，不删方法定义 |
| `ModDownload.DlClientListLoader`（Mojang 版本清单加载器） | 启动流程 `FormMain` 仍在使用 |
| 整合包安装闭包类 `_Closure$__0-0`（含 `File` 字段与完整安装调用体） | VBC 把闭包**分配**提升到方法开头（守卫之前），守卫后的死代码删了，但闭包类型与分配留下 |
| `Install_Click` 资源安装点击处理器 | 独立方法，不在被消除的语句流里 |

也就是说作者只把 5 个"总闸"换成了抛异常，下游链路全在——补丁要做的只是重新合闸。

## 3. 手术工具：dnlib 读写 .NET 元数据与 IL

.NET 程序集 = 元数据表（类型 / 方法 / 签名 / 字符串堆）+ 每个方法的 IL 指令体。dnlib 将其解析为对象模型，可以按名定位方法、增删改指令、导入跨程序集引用、重新序列化写盘（token / RVA 自动重排）。五处补丁用了四种手法，按复杂度递进：

### P5 — 单指令替换

`CardMinecraft.Visibility = Collapsed` 编译为 `ldc.i4.2`（枚举值编译期内联），改成 `ldc.i4.0`（Visible）即可。

### P3 — 整段换一条

下载页初始化中构造"关闭下载"假加载器的 13 条指令（`ldstr` + `newobj` + 委托链）整段移除，替换为一条 `ldsfld DlClientListLoader`。类型严丝合缝——该字段本就是 `LoaderTask<string, DlClientListResult>`。

### P2 — 控制流重建

`ModpackInstall(File,…)` 末尾的 throw 三连（`ldstr` + `newobj Exception` + `throw`）原地改写为 `ldloc PackType` + `switch(10)`，再插入 7 个 case 块，每个块按 2.14 源码逐句翻译：`Logger.Info("整合包种类：…")` → 压参数 → `call InstallPackXxx` → `stloc 结果` → `leave` 跳出 try 区。两个关键坑：

- **不能删除 throw，只能改写**：它是外层 `Try…Finally` 的末条指令，异常处理表边界引用着它，删除即悬空；
- **switch 目标必须指向真实入列的指令对象**：拼接完成后按各 case 开头唯一的日志字符串回体内定位再绑定。

### P1 — 方法体整体重写

无参 `ModpackInstall()` 的方法体清空重建：`Dialogs.SelectFile(弹窗)` → 取文件 → 存进幸存闭包 → `RunInThread(new Action(closure.闭包方法))`。引用导入是最麻烦的一步：主模块没有任何 `Dialogs.SelectFile` 的引用（调用点全被消除），而 MeloongCore.Wpf.dll 以字节数组内嵌在 `PCL.Resources.resources` 资源中运行时经 `AssemblyResolve` 加载——补丁先把它从资源里提取出来加载，再用 dnlib Importer 生成新的跨程序集方法引用。

### P6 — 分支重建

资源列表循环里插入 `TargetResourceType == ModPack(=2)` 判断：THEN 分支用两组 `ldarg.0; ldftn; newobj` 构造 Install / Save 双委托调用三参 `ToListItem`，ELSE 原样保留。

## 4. 单文件形态

`fixpcl.cmd` 的结构：

```
┌ cmd 引导段（纯 ASCII，十几行）────────────────┐
│ @echo off / cd / powershell -Command "…"     │ ← 以显式 UTF-8 读取自身
│ exit /b                                      │    #PS_BODY 标记之后的内容
├ #PS_BODY ────────────────────────────────────┤   并 Invoke-Expression
│ PowerShell 修复主体（约 27 KB）               │
│   - 版本守卫 / exe 定位与自动下载              │
│   - 5 处 IL 补丁                              │
│   - 写盘前校验 + 写盘后复检                    │
│ dnlib.dll（1.15 MB，Base64 内嵌）             │
└──────────────────────────────────────────────┘
```

cmd.exe 用 OEM 代码页只解析头部 ASCII 行（绝不接触中文与大段 Base64），执行完即退出；PowerShell 侧用显式 UTF-8 编码读取正文，中文路径与字符串全程无损。两个运行时各读各的部分，互不踩编码坑——这也是实测中文 + 空格路径通过的原理。

## 5. 为什么不走其他路

| 备选 | 不选的原因 |
|---|---|
| 改配置 / 远程开关 | 不存在——编译期固化，ServerConfig 远程配置中也无下载开关 |
| 修改公开源码后重编译 | 需 .NET Framework 4.8 SDK；且得到的是"开源版"，缺官方更新 / 正版登录 / 内嵌密钥，等于换了个软件；二进制补丁保留官方版其余全部功能 |
| 直接改十六进制 | 对 P5 这类单指令替换勉强可行，对 76 条指令的 switch 重建 + 新字符串 + 新跨程序集引用完全不现实（元数据堆需整体重排——正是 dnlib 存在的意义） |
