# 改动雷区（改任何代码前必读）

> 这个仓库的核心工程特征是：**用契约测试当边界执法**。其中很大一部分是**读仓库源码文本做字面量断言**——所以你的改动可能因为"某个字符串不再出现"而让测试变红，与功能对错无关。
> 下面每条都有证据；标"未验证"的是我没亲手跑过的。

## 1. 棘轮测试：只许减，不许增

`tests/DeskBox.Tests/ModuleBoundaryContractTests.cs` 逐文件冻结两类违规的**调用点计数**：

| 冻结项 | 例子 |
|---|---|
| `DllImport` / `LibraryImport`（平台互操作） | `App.xaml.cs = 10`、`NativeDropTarget.cs = 12`、`ShellClipboardHelper.cs = 12` … |
| `File.Delete/Move/Copy/Replace`（破坏性文件操作） | `App.xaml.cs = 3`、`FileService.cs = 9`、`QuickCaptureService.cs = 8` … |

源码注释写得很明确：**清单条目只允许变小或消失**，而且故意精确到文件级（避免"A 文件清理掉 5 处、B 文件新增 5 处"互相抵消）。

**对你意味着什么**：
- 新增一处 `[DllImport]`，或在新文件里用 `File.Delete` → **测试直接红**，与功能无关。
- 正当解法：把该调用收敛进已有的**拥有者**（如 `FileService` 是文件安全内核、`DeskBoxDataBackupService` 拥有数据目录），或在**同一个提交里**把清单条目按实际数字更新并写清理由（作者自己在 `DeskBoxDataBackupService.cs = 12` 那条上就是这么加注释的）。

## 2. 其他会咬人的冻结基线

| 基线 | 位置 | 约束 |
|---|---|---|
| JSON 源生成 | `JsonSerializationBaselineContractTests.cs` | 冻结 **35 个文件 / 83 处** `JsonSerializer` 调用。AOT 下反射被禁（`JsonSerializerIsReflectionEnabledByDefault=false`），**新增序列化必须走源生成 context** 并同步基线 |
| 设置切片 | `SettingsSliceContractBaselineTests.cs` / `SettingsSliceOwnershipContractTests.cs` | `AppSettings` 是扁平 facade + 14 个 slice；新增设置项要按既有归属放，别平铺新属性 |
| 本地化 | `LocalizationResourceContractTests.cs` + `src/DeskBox/Strings/*.json` | **12 个语言 × 2,848 键**，键数被对齐钉住。加一条用户可见文案 = 12 个文件各加一键 |
| AOT 阶段字面量 | `AotStage4D1A…7C1 ContractTests.cs`（几十个文件） | 把 `scripts/publish-aot-audit.ps1` 的 profile/schema 版本常量与脚本里的 throw 文本**逐字**钉住。升级审计版本要改几十个测试文件 |
| 模块边界清单 | 见下 §3 | 迁移进度与违规清单都被测试描述 |

## 3. 模块边界迁移的现状（别以为它已经拆好了）

目标是"单程序集内的模块化单体"，进度（实测文件数）：

| 目标命名空间 | 已迁入 |
|---|---|
| `DeskBox.Platform` | 13 |
| `DeskBox.Contracts` | 6 |
| `DeskBox.Core` | 4 |
| `DeskBox.FileSafety` | 1 |
| `DeskBox.Features` / `DeskBox.App` | **0** |

还有 **6 个文件的命名空间与所在目录不符**（例：`Services/SyncOutboxStore.cs` 声明的是 `DeskBox.Core.Persistence`）。这是迁移中途的正常状态。

**做新功能时的判断**：不要为了"更整洁"顺手搬家——搬家会同时改动棘轮清单与命名空间断言。**新代码放在既有位置**，搬家单独成 PR。

## 4. 新增一个格子 kind 的真实成本

| 指标 | 数值 |
|---|---|
| 历史上必须接触的文件 | **25 个** |
| 单文件最多 kind 分支 | **34 处**（`WidgetManager.FeatureWidgets.cs`） |
| 实测含 `WidgetKind.` 引用的文件 | **72 个** |

上游的官方接入顺序在 `docs/architecture/current_architecture.md` → "Adding A New Content Widget"（14 步），收敛方案在 `docs/architecture/widget_contribution_seam.md`（目标：一处声明 + 三处实现）。

**做新 kind 时必须同步**：`WidgetKind` 枚举 → `WidgetContentDescriptor` → `WidgetRegistry`（先保持不可创建）→ Content/ViewModel/Store/Adapter/Provider → 注册 → **12 语言键** → 测试（描述符/提供者/store/viewmodel/registry 覆盖）→ 最后才放开可创建。

## 5. 提交约定（机器执法）

- `.githooks/commit-msg` + CI 双重检查：**提交信息不得含任何 AI 署名**（`Co-Authored-By`、`Generated with` 等一律拒绝）。
- **本机该钩子默认未启用**（`git config core.hooksPath` 为空）。在该仓库执行一次 `git config core.hooksPath .githooks` 即可生效。
- 上游作者的规则还要求：提交与 PR 只带用户自己的身份。
- **不要 push**，未经用户明确要求。

## 6. 死代码清单（**别顺手删**）

| 位置 | 状态 |
|---|---|
| `Views/QuickCaptureWidgetWindow.*` | **13 个文件 / 6,635 行**，全仓 `new QuickCaptureWidgetWindow` **0 处**（QuickCapture 已迁到 `QuickCaptureSurfaceContent` + `ContentWidgetWindow`）。但 **26 个测试文件**读它的源码做字符串断言 → **删它等于删 26 个测试文件的内容** |
| `Services/WidgetManager.ZOrder.cs` 的 `QueueRequestedLayerRestoreCheck` | 0 个调用者（真正的回落靠 200ms 定时器 `TrayLayerRestoreTimer_Tick`；同文件的 `RequestRestoreRaisedWidgetsToDesktopLayer` 只记日志返回 true，**不是 bug 是命名误导**） |
| `Helpers/NativeFileDragOut.cs` | 写入侧（`BeginFileDragOut` / `BuildSourceTagPayload` / `MapOleEffect`）**0 调用者**；只有 `TryReadSourceTag` 被 `NativeDropTarget.cs` 调用一次 |
| `RecycleBinNativeBackend.Invoke` | 只被 `App.AotRecycleBinSmoke.cs` 调用；生产删除仍走 C# `SHFileOperation` |
| `Models/WidgetKind.cs` 的 `Productivity` | 有枚举成员，但无描述符、无 registry 条目 → **创建不出来** |
| `Services/Sync{Outbox,State,Revisions}Store.cs` + `src/DeskBox/Sync/` | 零件齐、有测试，但**无任何生产调用点**（云同步未接线）。契约文档要求的"outbox 按实体 LWW 合并"**未实现**（现在按 `operation_id` 落文件） |

## 7. 文档漂移清单（**别无条件相信 docs/**）

| 文档 | 问题 |
|---|---|
| `docs/architecture/[重要勿删]widget_zorder_lifecycle.md` | 声称的 `RunInteractionLeakWatchdog` / `ForceResetInteractions` 在代码里 **0 命中**（是否另有替代机制**未确认**） |
| `docs/architecture/current_architecture.md` | 引用**不存在**的 `ViewModels/MusicBarViewModel.cs`；测试基线还写"187/187"（实际 4,000 级）；Widget Types 一节没有 Glance 与 Search；仍说 QuickCapture 用旧专属窗口"intentional" |
| `docs/architecture/search-core-native-abi-v3.md` | 描述 2026-08-25 `82cd480` 已被删除的 `deskbox_search_core` 模块；`docs/architecture/evidence/` 里还留着 6 个该模块的 `.dll` |
| `docs/architecture/json-source-generation-baseline.md` | 写"29 文件 / 65 调用"，契约已冻结 **35 / 83** |
| `scripts/run-aot-*-smoke.ps1`（10 个） | 版本门禁停在 46–56，而审计脚本已是 profile 59 / schema 55 → 这些 runner 现在跑会必然抛错（**不在发布主链路上**，属潜伏债） |

**结论：以代码和测试为准；文档当"设计意图"读，不当"当前事实"读。**

## 8. 改动前置 checklist（照做能省一次回归）

1. **先查有没有现成实现**：grep 关键词 + 读知识库的 `功能点清单.md`。云同步就是典型"零件齐、没接线"。
2. **判断会不会碰冻结清单**：新增 `DllImport`？新文件里删文件？新增 `JsonSerializer` 调用？→ 会，就先想好计数怎么同步。
3. **用户可见文案** → 预算 12 个语言文件。
4. **新 kind** → 预算 25 个接触点 + 12 语言 + 测试矩阵。
5. **碰 `DESKBOX_NATIVE_AOT` 相关文件** → `scripts/publish-aot-x64.cmd` 编译验证，完成后**还原 `packages.lock.json`**（这是踩过的流程纪律）。
6. **改完必做**：停进程 → 构建（`-p:Platform=x64`）→ 跑测试 → 从 Debug 产物重启 → **报进程路径**。
7. **碰序列化/边界时**：棘轮是**双层**的（基线计数 + `AotStage*` 里的字面量），两处都要同步。
