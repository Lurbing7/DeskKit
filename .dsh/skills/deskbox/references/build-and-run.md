# 构建 / 测试 / 运行 / 排障（命令级）

> 全部在本机实测过（2026-09-23），除非标注"未验证"。
> shell 是 **Windows PowerShell 5.1**：脚本文本保持 ASCII，不要用内联 `if` 表达式。

## 1. 停进程（任何构建前）

输出 exe 被运行中的实例锁住会导致构建失败。

```powershell
Get-Process -Name DeskBox -ErrorAction SilentlyContinue |
  Where-Object { $_.Path -like 'D:\projects\DeskKit\*' } |
  ForEach-Object { "  PID $($_.Id)  $($_.Path)" }
# 确认是本仓库的实例后再停：
Stop-Process -Name DeskBox -Force
```

**别停仓库外的实例**（如果用户装了官方版，进程路径会是安装目录）。`WidgetManager` 退出时会 flush 设置，强停有丢 1 秒防抖窗口的风险（`SettingsService` 的待保存队列只在内存里）。

## 2. 构建 x64 Debug

```powershell
cd 'D:\projects\DeskKit'
dotnet restore .\DeskBox.sln -p:Platform=x64
dotnet build .\src\DeskBox\DeskBox.csproj --configuration Debug --no-restore -p:Platform=x64 -v:minimal
```

- 实测：**0 错误 / 24 警告 / 约 1 分 50 秒**；离线可 restore（NuGet 缓存 37/37 齐全）。
- 警告是既有的（`CS8602` 可空解引用、`CS0169` 未用字段 `NativeFileDragOut.s_dragThread`、`CS0414`、`CS0108`、以及 Rust 代理的一条 `AlphaThreshold` 命名警告）——**不要为了让警告归零顺手改代码**。
- 默认还会编 Rust 缩略图代理（`DeskBoxShellThumbnailProxy` 默认 `true`），所以要 `cargo` 在 PATH 上。
- `DeskBoxRustNative` 默认 **false** → 原生快捷方式/音量等走 **C# 回退实现**。这是正常状态，不是缺依赖。

**产物路径有两种，取决于是否传 `Platform`**（`DeskBox.csproj` 的 `Platforms` 只有 `x64;ARM64`）：

| 命令 | 产物 |
|---|---|
| `-p:Platform=x64` | `src\DeskBox\bin\x64\Debug\net10.0-windows10.0.22621.0\DeskBox.exe` |
| 不带（上游 `AGENTS.md` 的 canonical） | `src\DeskBox\bin\Debug\net10.0-windows10.0.22621.0\DeskBox.exe` |

**先 `Test-Path` 再启动，别照抄路径。**

## 3. 跑测试

上游 `AGENTS.md` 规定的命令（**未在本机验证过**，这次没跑）：

```powershell
dotnet test .\tests\DeskBox.Tests\DeskBox.Tests.csproj --no-restore --verbosity:minimal -p:Platform=x64
# 使用架构专属还原资产时追加： -p:RuntimeIdentifier=win-x64
```

- **必须带 `-p:Platform=x64`**：MSIX 打包拒绝 processor-neutral 的 app-host，默认 `AnyCPU` 会失败。
- 规模：`[Fact]/[Theory]` 标记 **2,959** 个；发布说明自述 1.5.5 实测 4,059 全绿。
- `AssemblyInfo.cs` 里全局 `DisableTestParallelization = true` → 跑得慢是正常的。
- CI 给测试 10 分钟超时并带 `--blame-hang --blame-hang-timeout 5m`。

## 4. 启动并验证"跑的确实是这次构建"

```powershell
$exe = 'D:\projects\DeskKit\src\DeskBox\bin\x64\Debug\net10.0-windows10.0.22621.0\DeskBox.exe'
Start-Process -FilePath $exe
Start-Sleep -Seconds 12
Get-Process -Name DeskBox -ErrorAction SilentlyContinue |
  ForEach-Object { "PID $($_.Id)  $($_.Path)" }
```

**必须核对 `Path`** —— 官方安装版与源码版可能同时存在，只报"起来了一个 DeskBox"是不够的。
启动成功后看日志确认管线健康：

```powershell
$log = "$env:LOCALAPPDATA\DeskBox\DeskBox.log"
Get-Content $log -Tail 8
# 期望看到： [Startup] Pipeline: 36 steps (5 critical), 0 degraded, 0 failed; ...
```

## 5. 数据与日志位置

| 路径 | 内容 |
|---|---|
| `%LOCALAPPDATA%\DeskBox\DeskBox.log` | 运行日志（带滚动，5 MB 上限） |
| `%LOCALAPPDATA%\DeskBox\data\settings.json` | 应用偏好（14 个 slice，schema 9） |
| `%LOCALAPPDATA%\DeskBox\data\widget-layout.json` | **格子的配置与布局**（设备域；`mappedFolderPath` / `managedFolderName` 在这里，不在 settings.json） |
| `%LOCALAPPDATA%\DeskBox\data\widgets\<widgetId>\` | 每格数据（如 `todo.json`） |
| `%LOCALAPPDATA%\DeskBox-Recovery\automatic\` | 自动快照 zip（`data/settings.json` + `widget-layout.json` + manifest） |
| `%USERPROFILE%\DeskBox\`（默认收纳根） | 管理文件夹格子的真实目录 |

**改配置文件必须先退出 App**，否则退出时会用内存状态覆盖你的修改。

## 6. 排障三招（本机验证有效）

### 6.1 看被其他窗口遮挡的桌面 / 应用窗口

`CopyFromScreen` 只能拍到最上层窗口。用 UI Automation 取句柄 + `PrintWindow` 离屏渲染：

```powershell
Add-Type -AssemblyName UIAutomationClient, UIAutomationTypes, System.Drawing
Add-Type -Namespace P -Name W -MemberDefinition @'
[DllImport("user32.dll")] public static extern bool PrintWindow(IntPtr h, IntPtr hdc, uint flags);
[DllImport("user32.dll")] public static extern bool GetClientRect(IntPtr h, out RECT r);
[StructLayout(LayoutKind.Sequential)] public struct RECT { public int L, T, R, B; }
'@
$root = [System.Windows.Automation.AutomationElement]::RootElement
$pc = New-Object System.Windows.Automation.PropertyCondition([System.Windows.Automation.AutomationElement]::ClassNameProperty, 'Progman')
$pm = $root.FindFirst([System.Windows.Automation.TreeScope]::Children, $pc)
$dc = New-Object System.Windows.Automation.PropertyCondition([System.Windows.Automation.AutomationElement]::ClassNameProperty, 'SHELLDLL_DefView')
$dv = $pm.FindFirst([System.Windows.Automation.TreeScope]::Descendants, $dc)
# 再往下找 SysListView32 拿图标列表；或直接 PrintWindow($dv 句柄, 2) 渲染桌面
```

对 **WinUI 3 的格子窗口 / 引导窗口同样有效**。枚举某进程的可见窗口：`EnumWindows` + `GetWindowThreadProcessId` 过滤 + `GetWindowRect` 取尺寸 + `GetWindowText` 取标题。

### 6.2 精确读桌面图标数

```powershell
# 需要先拿到桌面 SysListView32 的句柄（见 6.1）
[P.W]::SendMessage($lvHandle, 0x1004, [IntPtr]::Zero, [IntPtr]::Zero).ToInt64()   # LVM_GETITEMCOUNT
```

⚠️ **UIA 枚举桌面子项返回 0，不可靠**——必须走 `LVM_GETITEMCOUNT`。
⚠️ 取图标坐标要跨进程读内存（`LVM_GETITEMPOSITION`）——**不要做**，那是注入级操作。

### 6.3 解析桌面快捷方式的真实目标

```powershell
$sh = New-Object -ComObject WScript.Shell
$sh.CreateShortcut($lnkPath).TargetPath
```

**归类必须按目标判定，不能只看图标/名字**（这次就因为凭截图 OCR 猜，把几个 AI 工具误报成网盘）。

## 7. 回滚与清理

- 桌面整理/搬家类操作的保险：**先整目录快照 + 写 `原路径 → 新路径` 的 CSV 映射**（`D:\backups\desktop-snapshot-20260923\` 就是这么做的，60 项）。
- 仓库内清理：`bin/` `obj/` `.artifacts/` `artifacts/` `Output/` `TestResults/` `native/target/` 都在 `.gitignore` 里 → 改源码不会污染 `git status`。
- `scripts/cleanup-deskbox-install.ps1` 可清理安装残留；`Enable-/Disable-DeskBoxLocalDumps.ps1` 控制崩溃转储。
