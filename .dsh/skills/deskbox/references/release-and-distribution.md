# 发版与分发

> ⚠️ **先问用户**：发版/打包/推送属于破坏性与对外动作。工作区规则要求"未经明确要求不 push"，打包发布同理。
> 本节内容来自仓库文档与脚本（`README.zh-CN.md`、`docs/architecture/distribution_channel_workflow.md`、`store_msix_build_notes.md`、`scripts/*`）。**本机没有实际跑过完整发版链**——标"未验证"的均为文档口径。

## 1. 两条分发通道

| 通道 | 产物 | 特点 |
|---|---|---|
| **Direct**（直装） | Inno Setup：`DeskBox_Setup_<ver>_x64.exe` / `_arm64.exe` | Full Native AOT，**内置匹配架构的私有 Windows App Runtime**，可离线安装；**未做 Authenticode 签名** |
| **Store**（Microsoft Store） | `DeskBox_<ver>_x64_arm64.msixupload` | 提交 Partner Center；`Package.appxmanifest` 身份为 `D1FC332A.DeskBoxWidgets` |

**代码分叉极窄**：`DESKBOX_STORE` 在整个仓库**只出现 1 次**（`Services/AppDistributionService.cs:13`），其余靠服务接口分流（`IAppUpdateService` / `IStartupService` / `AppDistributionService`）。资源分叉在 `DeskBox.csproj`（Store 时移除微信二维码等资源）。

## 2. 权威入口：不要用裸 `dotnet publish`

```powershell
# 零售载荷（权威入口）
.\scripts\publish-aot-retail.ps1 -Platform x64        # 或 -Platform ARM64
```

它负责：生成 Full Native AOT 载荷 → 内置 Windows App Runtime → 编译对应架构的 Rust DLL → 生成升级清单 → 校验产物。

**为什么不能用裸 `dotnet publish` 替代**：安装器需要该脚本生成的 `DeskBox.InstallManifest.txt` 才能安全清理旧载荷。

它固定的发布属性（脚本内为真源）：

```
DeskBoxDistribution=Direct   DeskBoxAotAudit=true      DeskBoxAotSmokeHarness=false
PublishAot=true              DeskBoxRustNative=true    DeskBoxRustCrtLinkage=Static
SelfContained=true           WindowsAppSDKSelfContained=true   IlcUseEnvironmentalTools=true
```

⚠️ **两条自我包含策略不要混**：上游 `AGENTS.md` 说"Keep `SelfContained=false` and `WindowsAppSDKSelfContained=false` for the **runtime-download installer workflow**"——那是**另一条**（运行时下载型）路线。Full AOT 零售路线是自包含的。改这两个值之前先确认你在哪条路线上。

## 3. 一键全链（x64 + ARM64 双架构）

```powershell
.\scripts\build-stage-7c1-distribution.ps1 -Platform x64      # 或 ARM64
```

它串起：`publish-aot-audit.ps1`（smoke harness 版审计）→ `publish-aot-retail.ps1`（零售载荷）→ ISCC 编 Inno → `build-store-msix.ps1 -NativeAot -PackageBuildMode StoreUpload` → `audit-store-native-aot-package.ps1`。

**耗时预警**：单次 `publish-aot-audit.ps1` 实测约 **150–370 秒**（roadmap 记录的逐阶段数据），而全链每平台都要跑它，所以 CI 给 `distribution-audit` 配了 **120 分钟**超时。

手工编安装包（需要 ISCC，即 Inno Setup 编译器）：

```powershell
ISCC.exe /DDeskBoxNativeAot=1 /DDeskBoxBundledRuntime=1 /DMyAppReleaseDir=..\.artifacts\aot-retail\win-x64\publish .\installer\DeskBox.iss
ISCC.exe /DDeskBoxNativeAot=1 /DDeskBoxBundledRuntime=1 /DMyAppReleaseDir=..\.artifacts\aot-retail\win-arm64\publish .\installer\DeskBox.arm64.iss
```

`/DDeskBoxBundledRuntime=1` 会让 `PrepareDeskBoxDependencies` 直接返回、**跳过** .NET 10 与 Windows App Runtime 的检测/下载（`installer/DeskBox.Dependencies.iss`）。

## 4. 已知的坑与未完成项

### 4.1 `stable.json` 落后于最新 release（**会影响用户**）
`https://deskbox.fun/update/stable.json` 当前仍是 `1.5.0`（2026-09-06），而 GitHub Releases 已有 **v1.5.5**（2026-09-22）。
原因：发版流程里"更新并部署 `deskbox-site/public/update/stable.json` + 公网 curl 复验"是**人工步骤**，1.5.5 没做。

→ **不要依赖应用内"检查更新"**；引导用户直接到 GitHub Releases 下包。
→ 如果用户要发版，这一步必须补，否则老用户永远收不到更新提示。

### 4.2 自动化边界之外的动作（summary 里明写 `false`）
发布证据摘要里有三个诚实的布尔位：`signingExecuted=false`、`wackExecuted=false`、`inPlaceUpgradeExecuted=false`。
即：**代码签名、Windows App Certification Kit、实机安装/覆盖升级验证，全部不在自动化里**，需要人工。别把"结构校验通过"说成"发布验证通过"。

### 4.3 Store 的 MSIX 聚合是手工的
`store_msix_build_notes.md` 记录：要把两个架构聚合成 `DeskBox_<ver>_x64_arm64.msixupload`，需手工用 MakeAppx，且**必须显式带 `/bv`**，否则 Bundle 版本会变成当前日期时间；之后还要手工 unbundle 复核。单架构的 `.msixupload` 是中间证据，**不是**最终提交物。

### 4.4 WASDK 2.4.0 自包含载荷缺一个签名 DLL
`publish-aot-retail.ps1` 会额外从 restore 锁定的框架 MSIX 里抽 `Microsoft.WindowsAppRuntime.Insights.Resource.dll`——因为 WASDK 2.4.0 的 self-contained 载荷缺它，Win10 上会 `ERROR_MOD_NOT_FOUND`。**别把这段"多余"的抽取逻辑当成可清理的代码。**

### 4.5 CI 实际覆盖很窄
- `ci.yml`（唯一跑在 push/PR 上的）：只做**非 AOT** 的 restore/build/test，测试 10 分钟超时。
- `arm64-runtime.yml` / `distribution-audit.yml`：只在 `workflow_dispatch` 或推送到一个**已不存在的分支**（`codex/stage7b-arm64-actions`）时触发 → **事实上等于手动**。

## 5. 发版前 checklist

1. `CHANGELOG.md` 补齐该版本小节（中英双语，作者的习惯是两段都在）。
2. `docs/releases/v<ver>.md` 写发布说明，**含安装包与 MSIX 的 SHA-256**（作者每版都写，作为校验依据）。
3. `src/DeskBox/DeskBox.csproj` 的版本号四处同改：`Version` / `AssemblyVersion` / `FileVersion` / `InformationalVersion`。
4. x64 全量测试绿（`-p:Platform=x64`）。
5. **发布前后工作树指纹必须一致**（脚本会校验）→ 发布完检查 `packages.lock.json` 是否被改动并还原。
6. 直装包：`publish-aot-retail.ps1` 两个架构都跑 + ISCC 出双架构安装包 + 生成 `.sha256` 旁文件。
7. Store：聚合双架构 `.msixupload`（带 `/bv`）+ unbundle 复核。
8. **人工**：签名、WACK、实机安装与覆盖升级验证。
9. **部署 `stable.json`** 并公网复验（见 4.1）。
10. tag + GitHub Release（**推送前先问用户**）。

## 6. 已入库的证据产物（别当垃圾清掉）

`docs/architecture/stage-reports/` 有约 53 个一次性阶段报告；`docs/architecture/evidence/<runId>/` 有约 23 个文件（**含真实 DLL 二进制，约 2.6 MiB**），都是 AOT 迁移的证据链，已入 git。
`deskbox-aot-curve-samples.csv`（仓库根）与 `docs/baselines/memory-scenarios.json` 被 `scripts/measure-*.ps1` 与 `summarize-aot-memory-curve.ps1` 引用——**不是散落文件**。
