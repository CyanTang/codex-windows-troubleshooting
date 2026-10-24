# Codex Windows 故障排查与修复实录

这里分享一份**真实验证过的 Windows Codex 桌面端故障案例**：应用更新后无限弹出“无法加载组织设置”，修复启动后又出现从浏览器切回 Codex 时卡顿、未响应。

**[查看完整中文排障教程（含日志证据和 PowerShell 命令）](./docs/2026-10-24-rename-staging-eperm.md)** · [English README](./README.md)

## 核心发现

2026 年 10 月 24 日，Windows 11 上 Codex `26.1021.3705.0` 更新后，日志持续出现：

```text
bundled_executable_relocation_failed
errorCode=EPERM
operation=rename_staging
source=missing
```

故障分两个阶段解决：

1. **无法进入主界面：** 发现内置 `codex.exe` 部署缺失，从当前官方安装包恢复 4 个匹配 EXE、校验哈希后恢复启动。
2. **切窗未响应：** 日志仍有 CUA Node 运行时的 `EPERM` 与主线程卡顿，恢复完整匹配的 `cua_node` 文件夹（2384 文件，241.36 MB）后，用户反馈切换窗口恢复流畅。

**请不要直接套用本案例的版本号或目录哈希。** 每一次修复都必须先核对最新日志、实际安装版本、当前官方安装包中的源文件和目标目录。`EPERM` 的更深层原因（占用、权限、杀毒软件、应用自身并发 Bug 等）没有被最终证明；修复后未开展完整长期日志审计。

## 最短诊断路径

先完全退出 Codex，检查日志是否包含 `rename_staging`、`EPERM`、`source=missing`。如果实际错误不同，不要照搬修复步骤。避免删除 `.codex`、修改 WindowsApps 权限或者关闭杀毒防护。

欢迎在 Issues 分享**脱敏后的**错误关键词、Windows/Codex 版本和最小复现步骤，不要上传 API Key、Token、原始日志或个人项目路径。

本文是社区经验分享，并非 OpenAI 官方文档。协议：[MIT](./LICENSE)。