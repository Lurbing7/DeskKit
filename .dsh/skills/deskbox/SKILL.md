---
name: deskbox
version: 1.0.0
description: "维护/开发 D:\\projects\\DeskKit（DeskBox 的 GPL fork：C# + WinUI 3 + .NET 10 Native AOT + Rust）时使用。覆盖构建、跑测试、启动与重启、改动雷区（契约测试/棘轮/本地化/AOT）、发版与分发、以及这台机器上的环境事实。当任务涉及该仓库的代码改动、编译、看日志、改格子/桌面整理/原生层、或打包发版时触发。不负责：知识库整理（那是 C:\\workspace 的规则）、其他项目。"
---

# DeskBox / DeskKit 维护契约

> 这份 skill 是**操作手册**，不是项目文档的副本。
> **事实源永远在仓库内**：`D:\projects\DeskKit`（`README.zh-CN.md` / `CHANGELOG.md` / `docs/`）。
> 冲突时以仓库内为准，并回来修这份 skill。

## 每次开工前，按序读这三份

| 顺序 | 读什么 | 为什么 |
|---|---|---|
| 1 | 仓库根的 `AGENTS.md` | **上游作者的工作流**（构建/重启/测试平台/提交身份）。它是权威，本 skill 不复制它的内容 |
| 2 | `C:\workspace\notes\projects\DeskKit桌面工具箱\功能点清单.md` | 功能域 → 代码位置的索引 + 二次开发硬约束 + 已知技术债。**改动前先查这里，很可能已经有实现或半成品** |
| 3 | 本 skill 的 `references/` | 需要时读：`build-and-run.md`（命令级）· `change-guardrails.md`（改动雷区）· `release-and-distribution.md`（发版） |

## 0. 铁律（这个项目 + 这位用户）

1. **先方案后改码**：用户明确要求过"先不着急改代码"。给出改动面评估（要碰哪些文件、风险、验收标准）并等确认，再动手。
2. **批量改动先报清单**：>5 个文件、或涉及删除/移动，先给清单与影响面。
3. **不要碰上游的 `AGENTS.md` 语义**：它是上游作者的贡献者工作流。你要遵守它，但别为了让"我们的约定"生效去改它——那会造成与 `upstream` 的合并摩擦。我们自己的约定写在本 skill 里。
4. **提交信息不得含 AI 署名**：`.githooks/commit-msg` + CI 双重执法（`Co-Authored-By`、`Generated with`、任何助手署名一律拒绝）。**本机该钩子默认未启用**：`git config core.hooksPath .githooks` 只在该仓库执行一次即可。
5. **不要 push**，未经用户明确要求。改完只报 `git status`。
6. **不要顺手"清理"死代码**：这仓库里的死代码往往被测试钉住（见 `references/change-guardrails.md`），删之前先算连带面。

## 1. 身份与许可边界

| 项 | 值 |
|---|---|
| 产品名 / 仓库目录 | **DeskBox** / `D:\projects\DeskKit` |
| `origin` | `Lurbing7/DeskKit` ← 用户的 fork |
| `upstream` | `Tianyu199509/DeskBox` ← 原作者（README 署名"朱天雨"，个人独立开发，**不接受外部 PR**） |
| 许可 | **GPL-3.0-only**。自用随意；**对外分发时衍生作品整体必须 GPL-3.0**，不能合进闭源或换许可的项目。早期 MIT 版本不追溯（`LICENSE_CHANGE.md` 已于 `fa11a88` 删除） |

## 2. 标准工作流（命令级细节见 `references/build-and-run.md`）

```
① 停进程  → 任何仓库内路径下的 DeskBox.exe 都要先退出（否则锁住输出 exe）
② 构建    → dotnet build 必须带 -p:Platform=x64（MSIX 打包拒绝 AnyCPU 的 app-host）
③ 重启    → 从当前 Debug 产物启动，并**验证运行进程的路径确实是本次构建**
④ 报告    → 把 exe 路径与启动日志里的 pipeline 行报给用户
```

**关键**：`DeskBox.csproj` 的 `Platforms` 只有 `x64;ARM64`，且**带不带 `Platform` 会产出不同目录**：
- 带 `-p:Platform=x64` → `src\DeskBox\bin\x64\Debug\net10.0-windows10.0.22621.0\DeskBox.exe`
- 不带（上游 `AGENTS.md` 说的 canonical 路径）→ `src\DeskBox\bin\Debug\net10.0-windows10.0.22621.0\DeskBox.exe`

两者都存在过；**从实际产物去确认，别照抄路径**。

## 3. 改动雷区（改任何代码前必读 `references/change-guardrails.md`）

一句话版：**这仓库用"契约测试"当边界执法，而其中很大一部分是读源码文本做字面量断言。** 所以：

- 新增一处 `DllImport` / `File.Delete|Move|Copy|Replace` → 可能直接让 `ModuleBoundaryContractTests` 失败（它逐文件冻结调用点计数，**只许减不许增**）
- 新增一处 `JsonSerializer` 调用 → 要同步 `JsonSerializationBaselineContractTests` 的基线（AOT 下必须走源生成 context）
- 加任何用户可见文案 → **12 个语言 JSON 都要加键**（键数被契约测试对齐，当前 2,848 键）
- **新增一个格子 kind 的历史成本：25 个文件、单文件最多 34 处分支**（实测含 `WidgetKind.` 引用的文件有 72 个）。收敛方案见 `docs/architecture/widget_contribution_seam.md`
- 碰 AOT 相关 → 跑 `scripts/publish-aot-x64.cmd` 验证并**还原 `packages.lock.json`**

## 4. 架构速查（放东西之前先看这 8 条）

1. **中心链路**：`WidgetKind → WidgetRegistry → WidgetContentDescriptor → WidgetContentFactory / IWidgetContentProvider → IWidgetContent → ContentWidgetWindow → WidgetManager`
2. **`File` 是唯一"目录支撑"的格子**；Todo / QuickCapture / Glance / Weather / Music / **Search（索引聚合）** 都是自己的 store/索引支撑 → **框架层不要求格子绑定目录**，加"非目录支撑"的格子有先例。
3. **格子 ↔ 目录**：`MappedFolderPath` + `FollowsDefaultStoragePath` + `ManagedFolderName`（管理目录 vs 映射已有目录）。
4. **内容契约**：`Contracts/IWidgetContent.cs`（1 基础接口 + 6 个可选能力接口）。
5. **容器层次**：桌面 → 格子 → 格子组（1 HWND + 标签页）→ 目录（物理）→ 叠放（虚拟分组）→ 胶囊。
6. **叠放（虚拟分组）有硬限制**：自定义规则只有 `Extensions` 字段（`FileStackCustomRule`），判定是 `rule.Extensions.Contains(extension)` → **所有 `.lnk` 扩展名相同，永远分不出"AI 工具/IM"**。
7. **数据三层**：`settings.json`（偏好，14 slice，迁移 schema 9）· `widget-layout.json`（**设备**布局，含 widget 配置，不进云备份）· 各域 store。写盘统一走 `ResilientJsonStore`（原子写 + `.bak` + 损坏隔离）。
8. **原生边界**：Rust 只做窄 C ABI（10 个导出，ABI 2）；缩略图与原生右键菜单**在独立进程** `DeskBox.ThumbnailProxy.exe`。`DeskBoxRustNative` 默认 **false** → 普通 Debug 构建走 C# 回退实现，这是正常的。

**完整功能域索引**在知识库那份 `功能点清单.md`（本 skill 不复制它）。

## 5. 本机环境事实 —— 在 `AGENTS.local.md` 里

**本机专属**（工具链版本、绝对路径、自启项、shell 特性、网络状态、本机故障、已实测命令）统一写在**仓库根的 `AGENTS.local.md`**：它是 DSH 的原生本地覆盖层，与 `AGENTS.md` 同作用域、**优先级更高**、每轮自动注入，且**不提交进 git**。

> 为什么分家：这份 skill 要能随 fork 走（换机器/分享），所以只放**项目通用**的规则与事实；跟这台机器绑定的东西放 `.local` 层。
> 通用的数据/日志布局（`%LOCALAPPDATA%\...` 形式）与排障手段仍在 `references/build-and-run.md`。

## 6. 高频坑（踩过的，别重踩）

- **首启会另建收纳目录**：默认名已存在就自动加 ` (2)`，而自动创建的默认格子指向的是**新目录**。想让它显示已有内容 → **把内容移进格子指向的目录**，不用改配置。
- **`stable.json` 落后**：`https://deskbox.fun/update/stable.json` 里还是 `1.5.0`，而 GitHub 最新是 `1.5.5` → **不要依赖应用内"检查更新"**，直接下 GitHub Releases 的包。
- **死代码删不掉**：`Views/QuickCaptureWidgetWindow.*`（13 文件 / 6,635 行，全仓 `new` **0 处**）被 **26 个测试文件**读源码钉住。
- **文档漂移已实锤**：`[重要勿删]widget_zorder_lifecycle.md` 声称的 `RunInteractionLeakWatchdog` / `ForceResetInteractions` 代码 **0 命中**；`current_architecture.md` 引用**不存在**的 `MusicBarViewModel.cs`、测试基线还写"187/187"（实际 4,000 级）；`search-core-native-abi-v3.md` 描述 2026-08-25 已被删除的模块。
- **AOT smoke runner 的版本门禁已过期**：审计脚本是 profile 59 / schema 55，10 个 `run-aot-*-smoke.ps1` 仍门禁 46–56 的旧值（它们不在发布主链路上，属潜伏债）。
- **`Services/WeatherService.cs:25` 有硬编码的天气 API key** —— 值不要写进任何产出。

## 7. 这个 fork 的方向（用户的增补意图）

- **标签（Tags）**：上游已把 `WidgetKind.Tags` 登记为 `Placeholder` / `Planned`，文案原文是 **"Coming soon" / "File tagging"**——作者规划的就是文件打标签，而且做成**独立 kind**，不是文件格子的开关。
  → 方向约定：**优先做独立 `Tags` 格子，不要往 `FileSurfaceContent`（5,325 行 + 14 partial，全项目最难）里硬塞标签栏。**
- **日历卡片**：插槽已存在——`Contracts/ICalendarPresentationSource.cs`，原文注释写明"账号提供方 / CalDAV 客户端以后可以喂这个契约，而不成为格子视图的依赖"，已有本机实现 `LocalCalendarPresentationSource` 接在 Glance 格子上。
  → 先做**数据源调研**（授权、多源去重、失败降级），再做 UI；并先决定是**扩 Glance** 还是**新 `Calendar` kind**。

## 8. 交付与报告格式

收尾时固定报四件事：

1. **动了哪些文件**（含新增/删除计数）
2. **验证证据**（构建 exit code、测试通过数、进程路径与启动日志行、或截图路径）
3. **还剩什么没做**
4. **下一步建议**

涉及知识库的沉淀（要写进 `C:\workspace\notes\`）走**工作区**的 `AGENTS.md` 规则，不在这份 skill 里定义。
