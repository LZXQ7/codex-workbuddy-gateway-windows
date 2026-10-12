# Windows 支持：Codex → 桌面 WorkBuddy 的 desktop-queue 路线

针对 `codex-workbuddy-gateway` v0.7.0 的 Windows 适配改动清单与实机验收记录。

结论：上游 v0.7.0 的 desktop-queue 原生路线只支持 macOS。本改动在**不改动 macOS 代码路径**的前提下补上 Windows 分支，并在 Windows 版 WorkBuddy **5.8.1** 上完成端到端实机验证（Codex 派工 → 常驻服务原生唤醒 → 桌面 WorkBuddy 实际领取并用自身工具落盘 → 回报 → 独立核对真实文件）。

补丁文件：`windows-desktop-queue.patch`（13 个文件，+233/-21）。

---

## 1. 上游为什么在 Windows 上跑不起来

`desktop-queue` 的持久服务负责「有任务就原生唤醒绑定的桌面会话」，这条链路在 v0.7.0 里是 macOS 专用的：

| 位置 | 问题 |
| --- | --- |
| `src/native-desktop-trigger.ts` | 端口归属用 `/usr/sbin/lsof`、进程枚举用 `/bin/ps`、优雅退出用 `/usr/bin/osascript`（`com.tencent.workbuddy.mac`）；渲染器目标按 `Resources/app.asar` 拼接 |
| `src/service-cli.ts` | `installService()` 非 `darwin` 直接抛错，机制写死为 launchd plist + `launchctl` |
| `src/platform/cli.ts` | Agent Hub 服务安装同样要求 macOS |
| `src/config.ts` | `cliElectronPath` 默认 `/Applications/WorkBuddy.app/...`；`workbuddyConfigDir` 依赖 `process.env.HOME`（Windows 为空） |
| `bin/workbuddy-*`、`bin/agent-hub` | 启动器是 `#!/bin/sh`，Windows 上连 MCP 客户端都起不来（`'sh' is not recognized`） |
| `src/desktop-queue.ts` | `assertDesktopCaller()` 用 `ps -o ppid=` 走父进程链验证「调用方来自桌面应用」 |

在这些之上，实机还暴露了 3 个与平台强相关的行为差异，属于会「静默失效」的那一类：

1. **新建会话必须先 load。** `session:create` 之后会话在渲染器里并未打开，此时 `session:sendMessage` 会被丢弃（不报错、不写入原生历史）。必须先 `session:load` 再发。
2. **相对路径会被业务目录带偏。** 唤醒提示让 WorkBuddy 用自己的 Bash 运行 helper，而桌面会话的 cwd 是业务目录，于是 `./data/tasks.sqlite`、`./data/runs` 会落到业务目录里，claim 读到空队列。
3. **Windows 拒绝 `fsync`。** 目录句柄和只读文件句柄上调用 `fsync` 会返回 `EPERM`，导致「提交唤醒」的意图文件写入抛错、唤醒永远停在 `submitting`。

另外 Windows 版把 `cwd` 以反斜杠形式返回，而桥接器用字符串相等判断，会被误判成「创建返回了不同身份」。

## 2. 改动清单

### 新增文件

| 文件 | 作用 |
| --- | --- |
| `src/native-process.ts` | 只读读取一次 Windows 进程表（`Win32_Process` → pid/ppid/ExecutablePath/CommandLine），供端口归属、根进程枚举、父链校验共用 |
| `src/env.ts` | Node 侧加载安装目录的 `.env`（`bin/*` 的 `cd $ROOT` + `source .env` 的等价物）；已存在的环境变量优先，永不覆盖宿主配置 |
| `bin/workbuddy-gateway.cmd` | Windows 下可直接作为 MCP 命令的启动器 |

### 修改文件

| 文件 | 改动 |
| --- | --- |
| `src/native-desktop-trigger.ts` | 新增 Windows 分支：`netstat -ano` 定位端口 + 进程表校验归属；`taskkill`（WM_CLOSE，必要时一次强制）实现优雅重启；按 `resources/app.asar/renderer` 选择渲染器目标；唤醒前先确认会话已在渲染器打开（必要时 `session:load`）；创建结果改为规范化路径比较 |
| `src/desktop-queue.ts` | Windows 用父进程链校验调用方确为 WorkBuddy 桌面进程（排除 `--type=`、`cli\bin` 等子进程）；唤醒指令在 Windows 改为显式 `node` 命令（正斜杠路径，bash/cmd 均可执行） |
| `src/service-cli.ts` | 新增 `installServiceWindows`：注册 JSON + **启动文件夹条目**（登录自启）+ 立即以分离进程启动监督循环；`status` 用状态文件里的 pid 判断存活；`uninstall` 结束进程并清理条目；`run` 不再依赖 `launchctl` |
| `src/persistent-service.ts` | 唤醒意图落盘改为「尽力而为」flush：Windows 上跳过目录/只读句柄的 `fsync`，其余平台行为不变 |
| `src/config.ts` | 相对路径锚定到**安装根目录**而非 cwd；默认配置目录改用 `os.homedir()`；Windows 默认可执行文件位置 |
| `src/env.ts` 调用点 | `src/server.ts`、`src/service-cli.ts`、`src/desktop-worker-cli.ts`、`src/doctor.ts` 在读取配置前加载 `.env` |
| `scripts/workbuddy-call.mjs` | Windows 下改用 `node dist/server.js` 启动，而非 sh 启动器 |

macOS 代码路径保持原样：所有平台判断都走显式分支，POSIX 分支的逻辑与上游一致。

## 3. Windows 常驻服务模型

macOS 用 launchd（`RunAtLoad` / `KeepAlive`）。Windows 等价物：

- **监督进程**：`node dist/service-cli.js run --config <registration.json>`，以分离进程运行，把状态写入 `data/persistent-service-status.json`
- **登录自启**：`%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\codex-workbuddy-service-<hash>.cmd`
- **端口启用**：监督循环发现端口不存在时，用 `WORKBUDDY_REMOTE_DEBUGGING_PORT` 启动 WorkBuddy；若应用已按普通方式运行且无端口，会在确认所有会话空闲后正常重启一次

> 注：WorkBuddy 自身读取 `WORKBUDDY_REMOTE_DEBUGGING_PORT` 并 `appendSwitch('remote-debugging-port', ...)`，这与平台无关，是 Windows 版同样可用的前提。

## 4. 本机配置步骤

```ini
# .env（保留原文件，只追加本机项）
WORKBUDDY_BACKEND=desktop-queue
WORKBUDDY_NODE_BIN=<node 绝对路径>
WORKBUDDY_CLI_ELECTRON=<WorkBuddy.exe 绝对路径>
WORKBUDDY_CONFIG_DIR=<用户目录>\.workbuddy
```

```bash
npm ci && npm run build
# 按业务项目目录原生创建/复用执行会话（无需 Computer Use、无需手动建聊天）
# workbuddy_ensure_workspace({workspace: "<业务项目绝对目录>"})
# 之后每次派工都显式传 workspace：
# workbuddy_send_task({workspace: "<业务项目绝对目录>", objective: "..."})
codex mcp add workbuddy -- node <gateway>\dist\server.js
```

注册 MCP 时使用 `node` + 绝对路径即可：配置路径已与 cwd 解耦，宿主从任何目录启动都能正确读取队列。

## 5. 实机验收记录

环境：Windows x64、Node v25、Codex CLI、WorkBuddy 5.8.1（上游文档实测版本为 macOS 5.7.6）。

三个场景全部以「桌面 WorkBuddy 实际领取 + 真实文件」为准，不以排队或历史记录为准：

| 场景 | 观察到的原生行为 | Codex 独立核对 |
| --- | --- | --- |
| 小清单 | 队列事件出现 `session claimed`，`runtimeSource=desktop_queue`，`desktopPid` 为 WorkBuddy 进程 | 文件 48 bytes，内容逐字节相符 |
| 同一会话返工 | 同一 `sessionId` 的第 2 轮再次被原生领取 | 仅第 3 行变化，其余行与末尾换行不变，SHA-256 与回报一致 |
| 正常退出重开后自动领取 | 先关闭应用并以普通方式重开（确认调试端口不存在、进程 7 个），仅排入队列不做任何手工操作；服务自行启用端口并唤醒 | 新文件 14 bytes，内容相符；上一轮文件哈希未变（回归） |

辅助核对方式（不依赖回报文本）：

- 直接读取产物原始字节并计算 SHA-256
- 读取 `data/runs/<attempt>/events.jsonl`，确认 `queued → working → needs_review` 的原生领取链
- 读取本轮 `claim.json` 中的 `desktopPid`，确认领取者是 WorkBuddy 桌面进程
- 读取 `%USERPROFILE%\.workbuddy\projects\<...>\<sessionId>.jsonl`，确认唤醒提示确实进入原生会话、且 WorkBuddy 真的调用了 helper

## 6. 已知限制

- **平台差异**：上游文档实测的是 macOS WorkBuddy 5.7.6；本改动针对 Windows 5.8.1 的原生接口行为（反斜杠 cwd、需先 load 会话）编写。WorkBuddy 升级后仍需重新核对原生 RPC 兼容性。
- **执行器侧钩子**：WorkBuddy 自带的安全钩子曾拦下「文本里带 helper 字样」的一整行 report 命令，执行器自行改为写临时脚本再执行（约多 10 秒）。结果不受影响，属执行器侧行为，未在桥接器内做变通。
- **未覆盖**：业务搜索、视频、复杂长任务、用户登出再登录、系统重启；以及 Windows 下的 Agent Hub（`agent-hub install`）仍未适配，本次只处理了 desktop-queue 路线。
- **单元测试**：`npm run build` 通过；上游测试集中仍有约 30 条为 macOS/Unix 专用用例（launchd、Unix domain socket、`sh` 启动器、CRLF 归档哈希），在 Windows 上本来就失败，本改动未修复也不声称通过；本改动的依据是上述实机端到端验证。

## 7. 如何应用

```bash
git clone https://github.com/dudu555-djiao/codex-workbuddy-gateway.git
cd codex-workbuddy-gateway
git apply /path/to/windows-desktop-queue.patch
npm ci && npm run build
```

补丁基于上游 v0.7.0（`package.json` version `0.7.0`）。
