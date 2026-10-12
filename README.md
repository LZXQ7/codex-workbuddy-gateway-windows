# codex-workbuddy-gateway — Windows desktop-queue 适配

本仓库保存一份针对 [codex-workbuddy-gateway](https://github.com/dudu555-djiao/codex-workbuddy-gateway) **v0.7.0** 的 Windows 适配补丁，以及对应的改动清单与实机验收记录。

- 📋 [改动清单与实机验收记录](CHANGES_WINDOWS.md)
- 🩹 [补丁文件 `windows-desktop-queue.patch`](windows-desktop-queue.patch)（13 个文件，+233/-21）

上游 v0.7.0 的 `desktop-queue` 原生路线只支持 macOS（launchd、`lsof/ps/osascript`、`#!/bin/sh` 启动器）。本补丁在**不改动 macOS 代码路径**的前提下增加 Windows 分支，并在 Windows 版 WorkBuddy **5.8.1** 上完成端到端验证：

Codex 派工 → 常驻服务经 CDP 原生唤醒 → 桌面 WorkBuddy 实际领取并用自身工具落盘 → helper 回报 → Codex 独立核对真实文件。

验证覆盖：小清单创建、同一会话返工、正常退出重开后自动领取。

## 应用

```bash
git clone https://github.com/dudu555-djiao/codex-workbuddy-gateway.git
cd codex-workbuddy-gateway
git apply /path/to/windows-desktop-queue.patch
npm ci && npm run build
```

细节、限制与验收证据见 [CHANGES_WINDOWS.md](CHANGES_WINDOWS.md)。
