<!-- Sanitized public case study. No raw logs, screenshots, API keys, or personal project paths. -->

[English README](../README.md) · [中文首页](../README.zh-CN.md)

# Codex Windows 更新后启动失败、组织设置循环弹窗、切窗未响应：完整排障复盘

> **案例日期**：2026-10-24  
> **环境**：Windows 11；通过 WindowsApps / Microsoft Store 包运行的 Codex 桌面客户端，故障日志显示版本 `26.1021.3705.0`（x64）  
> **最终结果**：应用能够启动、进入已有项目并加载历史对话；手动恢复另一套 Node 运行组件后，用户实际测试切窗流畅，未再观察到「未响应」。  
> **适用对象**：以后协助排查同类问题的 AI Agent，或具备基本 PowerShell 操作能力的 Windows 用户。  
> **重要说明**：本文是一次**已成功的个案复盘**，不是对所有 Codex 版本通用的修复配方。文中版本号、哈希目录名都来自当日真实日志；在其他版本上**严禁直接照搬目标目录名**。

## 0. 给下一个 Agent 的 30 秒结论

**触发事件**：用户点击 Codex 应用内「更新」，此前正常的客户端随即报错「无法加载组织设置」，点重试或退出登录均反复弹出。更新后的程序无法把自带运行文件从临时 `.staging-*` 目录重命名到正式版本目录：`rename_staging` 返回 Windows `EPERM`。因此，最初缺失 `codex.exe`，后来又缺失 `cua_node` 内的 `node.exe`、`node_repl.exe`，并出现卡顿。

**本次有效解决办法**：

1. 清理用户级 `CODEX_CLI_PATH` 外部 CLI 路径覆盖，重启 Windows，避免它干扰新版内置组件的选择；**这一步本身未解决报错**。
2. 从**当前安装的官方 Codex 包** `app\resources` 中手动复制并校验 4 个可执行文件，放入日志指定的 `bin\<version-hash>` 目录；**组织设置循环弹窗消失，客户端能够进入主界面**。
3. 在进一步检查切窗卡顿日志后，从同一安装包 `app\resources\cua_node` 复制**完整目录**（2,384 个文件，约 241.36 MB）到日志指定的 `runtimes\cua_node\<version-hash>`；**切窗未响应症状在用户实际测试中消失**。

**仍未知的更深层原因**：为什么 Windows 会对这批临时目录的 `rename` 操作返回 `EPERM`？现有证据不足以区分：目录占用、实时防护、文件系统权限、Codex 更新/并发部署 Bug 等。不能声称「一定是 Defender」或「一定是 Clash」。

---

## 1. 症状与故障演变

### 阶段 A：点击更新后无法启动

- 更新前 Codex 可以正常使用。
- 用户主动点击「更新」后出现「无法加载组织设置」。弹窗提示网络、代理或防火墙可能造成问题。
- 「重试」「退出登录」都无法消除，形成弹窗循环。
- 更改 `.codex\config.toml` 中的 `notify = ...` 为注释后无效。
- 删除外部 CLI 环境变量并重启后仍无效。
- 重命名旧 `bin` 缓存后仍无效。
- 最终通过手动部署内置 `codex.exe` 等组件恢复进入主界面。

### 阶段 B：能够启动，但一切回窗口就卡住

- 能看到项目、历史对话、已有 Agent 回复等内容。
- 从浏览器 `Alt+Tab` 返回 Codex，窗口短暂整体发白、Windows 标题出现「ChatGPT（未响应）」；有时等待明显过长。
- 客户端后来能自行恢复，说明不是每次都直接崩溃。
- 新日志显示另一套内置 Node / CUA 运行组件反复部署失败，且记录了 Electron 主线程卡顿。
- 复制完整 `cua_node` 后，用户反馈切换窗口「很丝滑」，目前没有观察到未响应。

> **名称说明**：截图中 Windows 窗口标题出现 ChatGPT，但这里排查的是包含项目、Agent 和任务的 **Codex Windows 桌面客户端**，不要与普通网页版 ChatGPT 的加载问题混为一谈。

---

## 2. 最关键的日志证据（已从用户上传的真实日志核对）

### 2.1 初期：外部 CLI 覆盖

早期启动日志：

```text
windows_core_runtime_component_resolved
component=codex
consumer=app-server
source=override
executablePath=C:\Users\<USERNAME>\AppData\Local\Programs\OpenAI\Codex\bin\codex.exe
```

使用 PowerShell 检查 `CODEX_CLI_PATH`，`Process` 与 `User` 有相同外部路径，`Machine` 为空。

**解释**：存在自定义外部 CLI 路径覆盖。移除后，后续日志不再显示这一覆盖路径；但是组织设置错误仍然存在。因此它是需要排除的配置干扰，**不能单凭这一点认定它就是首次报错的根因**。

### 2.2 真正阻止主程序启动的错误：内置 EXE 部署失败

手动复制前，新版 `26.1021.3705.0` 的日志反复出现：

```text
bundled_executable_relocation_failed
errorCode=EPERM
operation=rename_staging
executableName=codex.exe
```

并有：

```text
windows_core_runtime_component_resolved
component=codex
executablePath=null
source=missing

Desktop startup policy read failed
phase=preparation
reason=unavailable
```

本次目标版本目录：

```text
C:\Users\<USERNAME>\AppData\Local\OpenAI\Codex\bin\2993767bd1dfb009
```

**解释**：新版内置程序未成功部署到预期目录，主程序初始化/组织策略检查继而失败。弹窗是**后续症状**，不是证明账号没有组织权限。

### 2.3 主程序手动恢复成功的证据

复制后，日志明确变化为：

```text
windows_core_runtime_component_resolved
component=codex
copiedRuntimeLocation=true
source=copied
executablePath=C:\Users\<USERNAME>\AppData\Local\OpenAI\Codex\bin\2993767bd1dfb009\codex.exe
```

其后还有：

```text
[StdioConnection] stdio_transport_spawned
[AppServerConnection] Transport start success
```

用户实际可进入主界面、查看历史会话。**这证明 4 个主组件的修复有效。**

### 2.4 第二阶段：CUA Node 仍然出现同样的部署错误

新日志中反复出现：

```text
bundled_executable_relocation_failed
errorCode=EPERM
operation=rename_staging
executableName=node.exe
```

同样报错的还有 `node_repl.exe`，之后出现：

```text
component=node
source=missing

component=node-repl
source=missing
```

当前版本需要的目标目录：

```text
C:\Users\<USERNAME>\AppData\Local\OpenAI\Codex\runtimes\cua_node\1fb7137c46617d50
```

此前检查 `cua_node` 只看到两个旧目录 `1b23c930bdf84ed6` 和 `2b841c3d44b123ec`；**只看到旧文件夹存在，不代表当前版本的组件已经部署成功**。

### 2.5 客户端卡顿确实有日志证据

在上传的主日志 `codex-desktop-<REDACTED>.log` 中：

| 现象 | 观察记录 |
|---|---:|
| `bundled_executable_relocation_failed` | 58 条 |
| `main_thread_jank_snapshot` | 11 条 |
| 一次 `thread_hydration` | 54,542.7 ms |
| 一次 `thread_navigation` | 57,131.3 ms |

相关快照包括：

```text
main_thread_jank_snapshot
processRole=browser_main
windowAttention=focused
```

**解释**：可以确认 Electron 主进程卡顿，与重复组件部署失败在同一日志时段出现。手动恢复组件后用户主观体验恢复，这构成较强的故障关联证据；但**没有用户上传的修复后性能日志**，不能把「58 次全部归零」写成已证实事实，也不能声称已严格证明唯一因果链。

### 2.6 其他真实但未证实是本次主因的日志

- `ERR_TLS_CERT_ALTNAME_INVALID`：连接 `codex-cloud-backend.chatgpt.com` 时收到 `*.azureedge.net` 证书，说明那次 TLS 路径有异常，可能与网络代理、DNS、路由或服务端有关。
- `ETIMEDOUT` / WebSocket 连接错误：确实存在网络超时，但不能单独解释所有本地文件重命名失败。
- `bundled_plugins_reconcile_failed`：`openai-bundled` Marketplace 提示已经从不同来源添加；窗口 focus 时可能发生重复插件同步。
- `rg.exe`（ripgrep）另有一次 `rename_staging / EPERM`。
- `wsl.exe --status` 报错：可能表示当前没有完整 WSL 环境，并不直接等于本次未响应主因。

**优先级**：本次通过修复明确缺失的本地运行组件恢复功能，因此先处理 `source=missing` / `EPERM`，只有相关问题解除后仍不正常，才逐项排查网络、插件冲突和 WSL。

---

## 3. 当天实际尝试过什么，哪些有效？

| 操作 | 实际结果 | 后续建议 |
|---|---|---|
| 点击「重试」「退出登录」 | 循环报错 | 不要无限重复 |
| 给 `config.toml` 中 `notify` 加 `#` | 无效 | 仅在日志/配置支持时尝试，原配置应备份 |
| 删除用户级 `CODEX_CLI_PATH` 并重启 Windows | 清除了 `source=override`，但仍无法启动 | 如果确实存在覆盖，先备份再处理 |
| 把旧 `bin` 重命名备份 | 执行成功，但仍无法启动 | 可排除缓存问题，不代表解决权限错误 |
| 手动复制并校验 `bin` 的 4 个内置 EXE | **有效：恢复进入主界面** | 当 `source=missing` 且安装源完好时优先考虑 |
| 手动复制完整 `cua_node` 运行目录 | **有效：用户测试切窗不再卡顿** | 当新日志重复显示 CUA Node `EPERM` 时考虑 |

这次案例里的**最短成功路径**不是卸载重装，而是「读取最新日志 → 确定具体缺失组件 → 从当前官方包恢复 → 校验 → 测试」。

---

## 4. 已验证的修复命令（仅针对本案例版本）

### 通用前置原则

1. 完全退出 Codex，并确认没有正在运行的 Codex 任务；否则可能存在文件占用或工作丢失风险。
2. 不要删除整个 `%USERPROFILE%\.codex`，其中可能有配置、会话及工具状态；不要清空用户项目目录。
3. 不要修改 `C:\Program Files\WindowsApps` 的所有权或权限；源目录只读即可。
4. 不要关闭 Windows Defender、禁用 TLS 校验或全局放开防火墙来尝试掩盖报错。
5. **先读取 `Get-AppxPackage` 的当前版本、源文件是否存在、日志实际指定的目标目录。不要直接按旧哈希目录写入新版本。**

### 4.1 查询包信息和日志

```powershell
$pkg = Get-AppxPackage -Name OpenAI.Codex
$pkg | Select-Object Name, Version, PackageFamilyName, InstallLocation
```

查询用户级 CLI 环境变量：

```powershell
foreach ($scope in 'Process','User','Machine') {
    [pscustomobject]@{
        Scope = $scope
        Value = [Environment]::GetEnvironmentVariable('CODEX_CLI_PATH', $scope)
    }
}
```

如果**确认** `User` 中是不再需要的旧 CLI 覆盖，先备份再移除（本次做过）：

```powershell
$old = [Environment]::GetEnvironmentVariable('CODEX_CLI_PATH', 'User')
if ($old) {
    $old | Set-Content "$env:USERPROFILE\codex_cli_path_backup.txt"
    [Environment]::SetEnvironmentVariable('CODEX_CLI_PATH', $null, 'User')
}
Remove-Item Env:CODEX_CLI_PATH -ErrorAction SilentlyContinue
```

这只是排除覆盖项，不是单独的修复保证。删除用户环境变量后需重启 Windows，让新启动进程加载新环境。此步骤会影响依赖该自定义 CLI 路径的其他程序，操作前应核对用途。

### 4.2 曾尝试的缓存重命名（单独执行无效）

当时使用以下命令对旧 `bin` 做了**非破坏性备份**：

```powershell
$bin = Join-Path $env:LOCALAPPDATA 'OpenAI\Codex\bin'
if (Test-Path -LiteralPath $bin) {
    $backup = "bin.backup-$(Get-Date -Format 'yyyyMMdd-HHmmss')"
    Rename-Item -LiteralPath $bin -NewName $backup -ErrorAction Stop
    Write-Host "已备份为：$backup"
}
```

实际备份名：`bin.backup-<timestamp>`。**执行成功，但重开 Codex 仍弹窗**，这一步不应被误记为最终解决方案。

### 4.3 手动恢复主程序的 4 个 EXE（成功）

这是本次恢复「无法加载组织设置」最关键的步骤。**脚本特意锁定当日版本**；未来版本必须由 Agent 核查并修订，不能删掉校验后盲目运行。

```powershell
$ErrorActionPreference = 'Stop'
$pkg = Get-AppxPackage -Name OpenAI.Codex

if (-not $pkg -or [string]$pkg.Version -ne '26.1021.3705.0') {
    throw '版本不匹配：不要用旧版脚本操作新版 Codex'
}

$src = Join-Path $pkg.InstallLocation 'app\resources'
$dst = Join-Path $env:LOCALAPPDATA 'OpenAI\Codex\bin\2993767bd1dfb009'
$names = @(
    'codex.exe',
    'codex-code-mode-host.exe',
    'codex-windows-sandbox-setup.exe',
    'codex-command-runner.exe'
)

foreach ($name in $names) {
    if (-not (Test-Path -LiteralPath (Join-Path $src $name))) {
        throw "当前官方安装包缺少：$name"
    }
}

New-Item -ItemType Directory -Path $dst -Force | Out-Null

foreach ($name in $names) {
    $source = Join-Path $src $name
    $target = Join-Path $dst $name
    Copy-Item -LiteralPath $source -Destination $target -Force -ErrorAction Stop
    $hash1 = (Get-FileHash -LiteralPath $source -Algorithm SHA256).Hash
    $hash2 = (Get-FileHash -LiteralPath $target -Algorithm SHA256).Hash
    if ($hash1 -ne $hash2) { throw "校验失败：$name" }
    Write-Host "OK: $name" -ForegroundColor Green
}
```

**实测结果**：四行 `OK`，SHA-256 一一匹配；重开 Codex 后能正常进入主界面。

> 安全提醒：本脚本使用 `Copy-Item -Force`。它是**案例复现命令**，仅适用于已经确认目标目录是该版本的、对应文件应由当前官方包提供的情况。遇到目标文件存在且用户不确定来源时，应先备份而不是直接覆盖。

### 4.4 手动恢复完整 CUA Node 运行时（成功）

**不能只复制 `node.exe` 或 `node_repl.exe`**。本次安装包的 CUA Node 包含 manifest、可执行文件和许多依赖文件，最终采用完整复制。

```powershell
$ErrorActionPreference = 'Stop'
$pkg = Get-AppxPackage -Name OpenAI.Codex

if (-not $pkg -or [string]$pkg.Version -ne '26.1021.3705.0') {
    throw '版本不匹配：停止操作，重新确认当前安装包和日志'
}

$src = Join-Path $pkg.InstallLocation 'app\resources\cua_node'
$dst = Join-Path $env:LOCALAPPDATA 'OpenAI\Codex\runtimes\cua_node\1fb7137c46617d50'
$required = @('manifest.json','bin\node.exe','bin\node_repl.exe')

foreach ($name in $required) {
    if (-not (Test-Path -LiteralPath (Join-Path $src $name))) {
        throw "源安装包缺少：$name"
    }
}

if (Test-Path -LiteralPath $dst) {
    throw '目标目录已存在，先检查内容并备份，不要覆盖'
}

New-Item -ItemType Directory -Path $dst -Force | Out-Null
robocopy $src $dst /E /R:1 /W:1 /XJ /NP /NFL /NDL
$rc = $LASTEXITCODE
if ($rc -ge 8) { throw "Robocopy 失败，退出码：$rc" }

$sourceFiles = @(Get-ChildItem $src -File -Recurse -Force)
$targetFiles = @(Get-ChildItem $dst -File -Recurse -Force)
$sourceBytes = ($sourceFiles | Measure-Object Length -Sum).Sum
$targetBytes = ($targetFiles | Measure-Object Length -Sum).Sum

if ($sourceFiles.Count -ne $targetFiles.Count -or $sourceBytes -ne $targetBytes) {
    throw '文件数量或总大小不匹配'
}

foreach ($name in $required) {
    $a = (Get-FileHash (Join-Path $src $name) -Algorithm SHA256).Hash
    $b = (Get-FileHash (Join-Path $dst $name) -Algorithm SHA256).Hash
    if ($a -ne $b) { throw "SHA-256 校验失败：$name" }
}

Write-Host 'CUA Node 已复制，关键文件校验通过' -ForegroundColor Green
Write-Host "文件数量：$($targetFiles.Count)"
```

**实测 PowerShell 输出**：

```text
目录：总数 338；复制 337；跳过 1；不匹配 0；失败 0
文件：总数 2384；复制 2384；失败 0
字节：241.36 m；复制 241.36 m
CUA Node 已复制，关键文件校验通过
文件数量：2384
```

> 这里的「跳过 1」是目录项，不是文件丢失；文件已全部复制。校验涉及**文件总数、总字节数和 3 个关键文件的 SHA-256**，不是对 2,384 个文件逐一哈希，勿夸大校验覆盖度。

重开 Codex 后，用户实际反馈切回窗口不再出现之前的「未响应」，界面流畅。

---

## 5. 下次遇到相似问题：建议 Agent 使用的诊断决策树

```text
点击更新后 Codex 异常
  |
  +-- A. 无法进入主界面 / 无法加载组织设置循环
  |     |
  |     +-- 先查最近启动日志中的：
  |           source=override / source=missing
  |           bundled_executable_relocation_failed
  |           EPERM / rename_staging
  |           Desktop startup policy read failed
  |     |
  |     +-- 若 source=override：检查 CODEX_CLI_PATH，备份后排除旧 CLI 覆盖
  |     |
  |     +-- 若 codex.exe source=missing + EPERM：
  |           核对当前安装包版本、源文件和日志实际目标哈希目录
  |           先检查锁定/权限，再考虑由官方包恢复该版本 EXE
  |
  +-- B. 可以进入主界面，但切窗/切聊天明显卡顿
        |
        +-- 查 main_thread_jank_snapshot、thread_navigation、thread_hydration
        +-- 查是否反复部署 node.exe / node_repl.exe / rg.exe
        +-- 查 bundled_plugins_reconcile_failed、reason=focus
        +-- 若 cua_node source=missing + EPERM：
              核对当前包 app\resources\cua_node 与日志指定版本目录
              备份后复制整个匹配运行时目录，校验并回归测试
        +-- 若本地部署都正常却仍卡顿：
              继续排查 WebSocket/TLS/代理、插件冲突、Electron 性能
```

### 快速提取日志的 PowerShell 命令（只读）

先确保日志目录存在；这条命令不会修改系统文件：

```powershell
$pkg = Get-AppxPackage -Name OpenAI.Codex
if (-not $pkg) { throw '未找到 Codex 安装包' }

$logDir = Join-Path $env:LOCALAPPDATA (
    'Packages\{0}\LocalCache\Local\Codex\Logs' -f $pkg.PackageFamilyName
)

if (-not (Test-Path $logDir)) {
    throw "日志目录不存在：$logDir"
}

Get-ChildItem $logDir -Recurse -Filter '*.log' -File |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 10 Name,LastWriteTime,Length
```

如果要定位某份日志中的关键词，先把 `$file` 换成实际日志的完整路径：

```powershell
$file = 'C:\请替换为实际的日志路径\example.log'
$pattern = 'bundled_executable_relocation_failed|rename_staging|source=missing|source=override|main_thread_jank_snapshot|thread_navigation|thread_hydration|reconcile_failed|ETIMEDOUT|TLS_CERT'

Select-String -LiteralPath $file -Pattern $pattern |
    ForEach-Object {
        $short = $_.Line.Substring(0,[Math]::Min(300,$_.Line.Length))
        '{0}:{1} {2}' -f $_.Filename,$_.LineNumber,$short
    }
```

**日志时间注意**：文件行首常为 `Z`（UTC），而当天用户在中国标准时间 UTC+8。例如 `2026-10-24T12:48:xxZ` 对应当天 20:48 左右，不要误判成凌晨或前一天。

---

## 6. 验收标准与未完成项

### 本次已经得到的确认

- [x] 用户级 `CODEX_CLI_PATH` 已处理，后续主程序日志不再依赖外部 CLI 覆盖。
- [x] 4 个主程序 EXE 拷贝完成，关键哈希相符。
- [x] 日志中 `codex.exe` 改为 `source=copied`，应用可以进入主界面。
- [x] CUA Node 文件拷贝完成：2,384 个，约 241.36 MB；3 个关键文件哈希正确。
- [x] 用户自己测试后反馈：切窗不再出现可感知的「未响应」，使用流畅。

### 尚未验证 / 仍需谨慎

- [ ] 未上传 CUA Node 修复**之后**的完整新日志，因此未统计修复后的 `EPERM` / `jank` 是否彻底清零。
- [ ] 未确认 Windows 返回 `EPERM` 的底层具体原因；可能在未来新版本升级时复发。
- [ ] `rg.exe` 部署错误、插件 Marketplace 来源冲突和部分 TLS / WebSocket 超时未作独立闭环诊断；若目前功能正常，不建议仅因历史日志有警告就盲目修改。

### 建议的复发验证方式

1. 先确定当前应用版本、错误发生时间；退出正在运行的任务。
2. 重新触发一次故障，等待日志写入后再完全退出应用。
3. 找到最新 `.log`（优先事故时间附近、大小较大的主进程日志），提供给 Agent。
4. 让 Agent 用 **新版日志中明确出现的 `destinationPath`** 判断目标目录，而不是抄旧版 `2993767bd1dfb009` 或 `1fb7137c46617d50`。
5. 如果用户已经能正常启动且切窗流畅，没有必要重复执行任何复制命令。

**隐私提示**：日志可能包含 Windows 用户名、项目路径、邮箱、请求上下文及其他敏感数据。分享前先检查和脱敏；不要上传 Token、API Key 或完整的 `.codex` 私人数据目录。

---

## 7. 可以直接发给其他 Agent 的任务说明

> 我在 Windows 11 上使用 Codex 桌面客户端。曾在 2026-10-24 点击更新后遇到「无法加载组织设置」无限弹窗，后来又遇到切换窗口就短暂「未响应」。当时版本 `26.1021.3705.0` 的日志显示 `bundled_executable_relocation_failed`、`EPERM`、`operation=rename_staging`：先是 `codex.exe` 缺失，后是 `cua_node` 下的 `node.exe`/`node_repl.exe` 缺失。上次从当前 WindowsApps 官方安装包手动复制 `app\resources` 的 4 个核心 EXE，以及完整 `app\resources\cua_node` 目录至对应版本哈希目标目录，校验后恢复正常。旧 `CODEX_CLI_PATH` 也曾清理，但单独清理无效。
>
> 现在如果故障复发，请先读取我提供的**最新日志**，确认当前版本、实际报错、具体组件的 `source=missing`、`destinationPath`，并区分本地部署失败与网络/插件问题。不要预设一定是相同原因；不要照搬旧版哈希路径；不要直接删除整个 `.codex`、关闭 Defender 或改变 WindowsApps 目录权限。请优先给我只读诊断命令；明确确认可以修复的组件后，再给出有版本校验、备份和校验机制的最小变更 PowerShell 命令。

---

## 8. 隐私与证据说明

这是一篇根据用户现场排障记录撰写的公开案例。原始日志、ZIP 和屏幕截图**不随仓库公开**，因为它们可能包含操作系统账号、工作区和会话信息。示例路径已脱敏为 `C:\Users\<USERNAME>`；遇到相同错误时，使用本人环境的 `$env:LOCALAPPDATA` 和 `$env:USERPROFILE`，不要复制其他人的完整绝对路径。

日志中与故障直接相关的关键词：`bundled_executable_relocation_failed`、`rename_staging`、`EPERM`、`source=missing`、`main_thread_jank_snapshot`。日志计数为修复前的样本统计。修复后未另行收集长时间性能日志，**因此不能声称所有底层错误已永久解决**。

如果准备公开自己的日志，先删除或脱敏用户名、主机名、项目路径、邮箱、Token、Cookie、API Key、Authorization 头、会话 ID 等内容。

---

**本案例结论**：更新后内部运行组件部署失败（`rename_staging` → `EPERM` → `source=missing`）；从匹配版本的已安装官方包恢复对应 `bin` 与 `cua_node` 组件并核验后，分别恢复了正常启动和流畅切换窗口。**造成 `EPERM` 的底层原因仍未知**。
