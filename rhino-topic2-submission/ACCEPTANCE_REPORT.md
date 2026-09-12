# 验收报告（A1–A5）

**结果只按真实执行记录，不推断 PASS。** 未在本机跑通的项记为 `NOT_RUN`。

- 分支：`feat/rhino-topic2-final`
- 基线：`upstream/main` @ `462999ec3f5c1467ef0ccf5cf8c422393f40a0e6`
- 环境：Windows 11 + mingw-w64，Go toolchain，Node.js

## 总览

| 验收项 | 代码 / 单元测试 | 真实端到端运行 |
|---|---|---|
| A1 终端在至少两种沙箱后端上可用 | PASS | **NOT_RUN**（凭据与可执行文件缺失） |
| A2 两个租户各自仅能看到自身的进程与文件 | PASS | **NOT_RUN** |
| A3 请求产物目录以外的路径被拒绝 | PASS | **NOT_RUN** |
| A4 三类产物可在界面内直接查看 | PASS（可执行部分） | **NOT_RUN** |
| A5 超限自动终止 + 每条命令可审计查询 | PASS | **NOT_RUN** |

下面的"代码 / 单元测试"结果来自这条命令，四个包全部 `ok`：

```bash
export CGO_CFLAGS="-I<path-with-sqlite3.h>"
go test ./internal/sandbox/ ./internal/handler/session/ ./internal/handler/ ./internal/types/ \
        -run 'Terminal|LiveFile|LimitWatcher|HardLimit|DurationLabel' -count=1
# ok  github.com/Tencent/WeKnora/internal/sandbox
# ok  github.com/Tencent/WeKnora/internal/handler/session
# ok  github.com/Tencent/WeKnora/internal/handler
# ok  github.com/Tencent/WeKnora/internal/types
```

## A1 终端在至少两种沙箱后端上可用

| 项 | 内容 |
|---|---|
| 交付 | #3175 把 Docker 接入既有的 `RemoteTerminalSession` 抽象，与本地子进程、Cube、E2B 并列 |
| 测试 | `TestDockerOpenTerminalUsesNativeTTYOptions`、`TestDockerTerminalStreamsRawPTYAndAcceptsInputAndResize`、`TestDockerTerminalReportsExitAndClosesOutputOnce`、`TestDockerTerminalUsesBoundedRPCsButNotForAttachStream`、`TestDockerTerminalCloseAndContextCancellationAreIdempotent`、`TestSessionTerminalManagerReportsProviderReconnectCapability`、`TestDockerTerminalDefaultsIncludeColorTerm` |
| 结果 | **PASS**（`internal/sandbox`） |
| 端到端 | **NOT_RUN** — 需要 Docker Engine / E2B / Cube 凭据，且本机无法链接出可执行文件 |

只做到"代码层与单元测试层证明 Docker 是同一抽象下的第三个 provider"，**没有**声称两种后端已在真实环境跑通。

## A2 两个租户同时开启终端时各自仅能看到自身的进程与文件

| 项 | 内容 |
|---|---|
| 交付 | #3190 的终端会话按租户/会话绑定；实时文件管理按会话沙箱绑定 |
| 测试 | `TestCleanSessionLivePath`、`TestListSessionLiveFilesRefusesPausedSandboxWithoutConnect`、`TestLiveFileEntryTypeUsesBrowserDirectoryContract`、`TestLiveFileHelperLimitsMatchGoConstants`（`internal/sandbox`）；`internal/handler/session` 的沙箱文件与终端审计测试 |
| 结果 | **PASS** |
| 端到端 | **NOT_RUN** — 需要能启动服务并开两个租户会话 |

## A3 请求产物目录以外的路径被服务端拒绝

| 项 | 内容 |
|---|---|
| 交付 | 实时文件管理沿用现有路径限制规则，服务端拒绝越界路径 |
| 测试 | `TestCleanSessionLivePath`、`TestLiveFileHelperLimitsMatchGoConstants`（路径规范化与边界）；`internal/handler/session` 的文件接口授权测试 |
| 结果 | **PASS** — 覆盖 `..` 穿越与越界路径的规范化拒绝 |
| 端到端 | **NOT_RUN** — 绝对路径、符号链接、改名逃逸三类真实请求需要在运行中的服务上复验 |

`..` 穿越在单元测试层被覆盖；**绝对路径、符号链接、改名逃逸未在本机逐项复验**，不声称覆盖。

## A4 三类产物可在界面内直接查看，网页类运行在受限 iframe 内

| 项 | 内容 |
|---|---|
| 交付 | #3218 产物语义 `kind`；#3219 演示文稿 Skill（生成 → 预览 → 下载全链路） |
| 测试 | `internal/types` 的产物 `kind` 测试；前端 `SandboxFilesPanel.test.mjs` 2/2 PASS、`SandboxTerminal.test.mjs` 7/7 PASS |
| 结果 | **PASS（可执行部分）** |
| NOT_RUN | 演示文稿 Skill 的 Python 测试（本机未安装 `python-pptx`）；`sessionArtifacts.test.ts`（需要 tsx/vitest）；三类产物在浏览器里的真实查看 |
| 端到端 | **NOT_RUN** — 需要在运行中的界面里逐类查看 |

## A5 超出限额自动终止 + 每条终端命令可在审计日志中查询

| 项 | 内容 |
|---|---|
| 交付 | #3217 硬限额看护（CPU 时间 / 内存 / 最长存活）；#3190 终端命令审计与脱敏 |
| 测试 | `TestDockerLimitWatcherTerminatesAtCumulativeCPUTime`、`TestDockerLimitWatcherTerminatesAtHardLifetimeDespiteActivity`、`TestDockerLimitWatcherTerminatesOOMKilledContainer`、`TestDockerLimitWatcherLeavesContainerBelowLimits`、`TestDockerLimitWatcherDoesNotClaimConcurrentDeletion`、`TestDockerSandboxConfigHardLimitsJSON`、`TestDurationLabelRejectsInvalidAndOverflowingValues`；`internal/handler/session` 的终端审计测试；`internal/application/repository` 的审计作用域测试；`internal/types/audit_log_test.go` |
| 结果 | **PASS** |
| 端到端 | **NOT_RUN** — 需要在运行中的容器上真实触发超限，并在审计查询接口里查到命令 |

## 关于测试失败的说明

完整跑这些包时出现的失败**全部**是既有基线失败：

- `internal/sandbox` 3 个、`internal/handler` 2 个、`internal/application/service` 44 个。
- 已在干净 `upstream/main` 检出（`git worktree add --detach ../baseline-check upstream/main`）上复跑，失败集合与集成版本完全一致，**没有新增失败**。
- `internal/container` 与 `cmd/server` 的构建失败是本机 duckdb C++ 静态库的 ABI 问题，干净基线同样失败。

详见 [TEST_COMMANDS.md](TEST_COMMANDS.md) 与 [KNOWN_LIMITATIONS.md](KNOWN_LIMITATIONS.md)。
