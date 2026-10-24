# Contributing / 反馈规范

Thanks for helping Windows Codex users. Contributions are welcome when they provide **reproducible evidence**, describe **what was actually tested**, and distinguish **symptoms** from **root causes**.

请优先报告：Windows 版本、Codex 版本、复现步骤、日志中的错误类型、已做过的测试，以及对应结果。避免夸大单例成功经验。

## Privacy / 隐私

Before posting in a public Issue, **redact**:

- API keys, bearer tokens, auth headers, cookies, session tokens
- Email addresses, real usernames, account IDs, machine/host names
- Full personal file paths, work project names, proprietary code
- Conversation messages and configuration containing secrets

**Do not upload entire `.codex` folders, raw logs, ZIP archives of diagnostics, or screenshots containing personal data.** If log context is necessary, share a small sanitized excerpt only.

## Suggested issue format

1. Windows and Codex package versions
2. Exact symptom and reproduction steps
3. Relevant error codes / keywords (redacted excerpt)
4. Which commands were tested and their outcome
5. Whether the issue reproduces after relaunch/update