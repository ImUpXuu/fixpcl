# 验证方法与结果

补丁是对二进制的修改，验证链必须闭环。本文记录 fixpcl 采用的全部验证手段。

## 1. 语义蓝本：不凭空发明代码

每一处补丁都对照公开仓库 git 历史（commit `5ef81e4`，2.14.0.0 正式版）中的原始源码翻译成 IL，不自行设计逻辑。补丁后的反编译结果与 2.14 源码语义逐句等价。

## 2. 写盘前：全模块引用校验

用 dnlib 遍历所有方法（含嵌套类型），检查：

- 每条分支指令（`br*` / `bne*` …）的目标指令在**同一方法体**内；
- 每条 `switch` 的全部目标在方法体内；
- 每个异常处理区域（`Try` / `Catch` / `Finally` / `Filter`）的四个边界引用均在方法体内。

任何悬空引用都会立即中止且不产出文件。该校验在开发期真实抓到过两个 bug：删除了作为 EH 边界的 `throw` 指令、switch 目标指向未入列对象。

## 3. 写盘后：重新加载复检

对产物用全新的 `ModuleDefMD.Load` 做五项标记检查：

| 检查 | 断言 |
|---|---|
| P3 | `LoaderInit` 中存在 `ldsfld DlClientListLoader` |
| P2 | `ModpackInstall(File,…)` 含 `switch` 且 `InstallPack*` 调用 ≥ 7 |
| P1 | `ModpackInstall()` 含 `Dialogs::SelectFile` 与 `RunInThread` 调用 |
| P5 | `PageDownloadModify` 中不再存在 `get_CardMinecraft → ldc.i4.2 → set_Visibility` 序列 |
| P6 | `InitializeStackResourceVersions` 中存在 `ldftn Install_Click` |

## 4. 反编译回读（开发期最强验证）

用 ILSpy 对补丁产物整体反编译，人工比对五处代码与 2.14 等价性。反编译器本质上是最严格的 IL 语义阅读器——它读不顺的 IL 大概率有问题。实测五处全部还原出等价高级代码（例如 P6 还原出 `if ((int)TargetResourceType == 2) { … ToListItem(Install_Click, Save_Click, …) }`，与 2.14 逐字一致）。

## 5. 冒烟测试

实际启动补丁版 15 秒：WPF 主窗口正常出现、无崩溃、无 `InvalidProgramException`。

## 6. 确定性与一致性

同一官方 exe 的多次修复（含单文件工具与开发期脚本两种路径）产物 **SHA-256 完全一致**：

```
f02b495e45c75ce2127cb2d121986a4dcc8d457550f0863528ba1bc645b10c3c
```

这意味着工具的输出就是那份被反编译验证 + 冒烟测试过的二进制，逐字节相同。

## 7. 边界测试

| 场景 | 结果 |
|---|---|
| 干净目录、仅有官方 exe + 工具单文件，模拟双击 | 全流程通过，哈希与基准一致 |
| 中文 + 空格路径（`test-路径 测试\新建 文件夹\`） | 正常识别路径、正常生成、哈希一致 |
| 自动下载回退（GitHub raw） | 可达，2.66 MB 官方包下载后正常续跑 |
| 失败注入（开发期一次语法错误构建） | 正确报错、退出码非零、FAILED 横幅、不产出文件 |
| 版本守卫 | 非 2.99.0.0 拒绝执行 |

## 8. 失败即安全（fail-closed）设计

工具任意环节失败（版本不符 / 结构定位失败 / 校验不过 / 复检不过）都会：

1. 打印具体失败原因；
2. 以非零退出码结束（cmd 横幅显示 FAILED）;
3. **不生成或删除**输出文件；
4. 从不触碰原始 exe。
