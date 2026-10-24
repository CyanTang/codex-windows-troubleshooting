# Codex Windows Troubleshooting

**Practical recovery notes for Codex / ChatGPT desktop on Windows.**

> **Verified case study:** after a Windows desktop app update, Codex repeatedly displayed **“Unable to load organization settings.”** Once launch was recovered, switching back to the application caused long freezes / *Not Responding*. Detailed logs identified repeated `rename_staging` failures with `EPERM` while relocating bundled runtime files. Restoring the matching components from the **installed official Codex package** resolved the visible symptoms in this one case.

**[中文说明](./README.zh-CN.md) | [Complete step-by-step case study (中文)](./docs/2026-10-24-rename-staging-eperm.md)**

## Symptoms and evidence

| Symptom | Log evidence | Verified local recovery |
| --- | --- | --- |
| Endless **“Unable to load organization settings”** after an update | `bundled_executable_relocation_failed`, `errorCode=EPERM`, `operation=rename_staging`, `codex.exe` with `source=missing` | Copy the four required Codex executables from the matching *official installed package*, verify SHA-256, relaunch |
| UI becomes unresponsive when returning from another window or switching threads | Repeated CUA Node `node.exe` / `node_repl.exe` relocation failures plus `main_thread_jank_snapshot` | Copy the **entire matching `cua_node` runtime** (not just individual EXEs), verify files, relaunch |

**Observed setup:** Windows 11 x64, Codex Store/WindowsApps build `26.1021.3705.0`, October 24, 2026.

**Outcome:** main window restored after copying the CLI binaries; the reporter then confirmed smooth window switching after restoring CUA Node. This establishes a successful user-level workaround, **not** the ultimate explanation for why Windows returned `EPERM`. No complete post-fix performance-log audit was performed.

## Fast, read-only diagnostic checks

```powershell
Get-AppxPackage -Name OpenAI.Codex | Select-Object Name, Version, InstallLocation, PackageFamilyName

foreach ($scope in 'Process','User','Machine') {
    [pscustomobject]@{
        Scope = $scope
        CODEX_CLI_PATH = [Environment]::GetEnvironmentVariable('CODEX_CLI_PATH', $scope)
    }
}
```

Search the current desktop logs for:

```text
bundled_executable_relocation_failed
rename_staging
errorCode=EPERM
source=missing
source=override
main_thread_jank_snapshot
```

A generic “organization settings” modal does **not** prove that the account, network or organization policy is at fault. Likewise, a freeze alone does **not** prove an executable relocation error. **Match your own logs before applying the workaround.**

For the version-checked recovery PowerShell commands, findings, and validation results, see **[the complete Chinese walkthrough](./docs/2026-10-24-rename-staging-eperm.md)**.

## Safety first

- Stop active Codex tasks and quit the app before changing bundled runtime copies.
- **Never copy the example's version-specific folder hashes to an unrelated Codex build.** Inspect `destinationPath` in the current logs and validate the actual package version and contents.
- Work from the installed official Codex package; do not download unknown EXEs.
- Do not delete `%USERPROFILE%\.codex`, modify `C:\Program Files\WindowsApps` permissions, disable Windows Defender, or bypass TLS validation.
- Back up existing folders before replacing files; the scripts in the guide are recorded from a **single, version-locked success case**, not guaranteed to work everywhere.
- Do not upload raw logs, screenshots, token values, session data, or personal project directories in public Issues.

## Other possible causes

This case also contained WebSocket / TLS warnings, plugin reconciliation warnings, and an `rg.exe` relocation error. Those were **not** individually confirmed as the cause of the remaining symptoms. If your logs do not match the confirmed `EPERM` pattern, diagnose other causes instead of running these recovery steps blindly.

## Relevant Codex issue reports

- [openai/codex #45858 — post-update EPERM during bundled CLI/Node relocation](https://github.com/openai/codex/issues/45858)
- [openai/codex #49611 — freezes on focus / thread switching with CUA Node relocation failures](https://github.com/openai/codex/issues/49611)

## Contributing

Open an Issue to share the **Codex version, Windows version, symptoms, and sanitized error keywords**. Never attach unsanitized logs or credentials. See [CONTRIBUTING.md](./CONTRIBUTING.md).

## License and affiliation

Original case-study text and sample scripts: [MIT](./LICENSE). Independent community documentation, **not an official OpenAI support project**. No original user log, screenshot, API key, or personal path is included.