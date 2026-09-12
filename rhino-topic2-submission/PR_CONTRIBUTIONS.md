# 上游 PR 贡献清单

课题二拆成六个独立模块，每个模块单独向上游提 PR，避免一个巨型 PR 混合多个子系统。所有 PR 的分支都在 fork `WHUTcjh-2024/WeKnora` 上，目标分支为 `Tencent/WeKnora:main`。

基线：`upstream/main` @ `462999ec3f5c1467ef0ccf5cf8c422393f40a0e6`

## 状态总览

| PR | 标题 | 状态 | 分支 | Head SHA |
|---|---|---|---|---|
| [#3175](https://github.com/Tencent/WeKnora/pull/3175) | feat(sandbox): add Docker support to interactive session terminals | **Open upstream** | `feat/topic2-docker-terminal` | `4ddf7d9967ae2770cece63990b6a83be1d5a43ed` |
| [#3190](https://github.com/Tencent/WeKnora/pull/3190) | feat(sandbox): audit interactive session terminal commands | **Open upstream** | `feat/topic2-terminal-audit` | `3b7183fbfe6c1973084965b10de30e28fd250511` |
| [#3217](https://github.com/Tencent/WeKnora/pull/3217) | feat(sandbox): enforce Docker sandbox hard limits | **Open upstream** | `feat/topic2-hard-sandbox-limits` | `c89d5562916be7e77f71458d6045be23c4c3d963` |
| [#3218](https://github.com/Tencent/WeKnora/pull/3218) | feat(artifacts): expose semantic kinds for generated artifacts | **Open upstream** | `feat/topic2-artifact-kind` | `b9abcc9a8df81a4a0559da762292a815ba97795b` |
| [#3219](https://github.com/Tencent/WeKnora/pull/3219) | feat(skills): add a focused presentation generation skill | **Open upstream** | `feat/topic2-presentation-skill` | `7d3ca9e6f4f4b1b9a9b6803f0b8fd3534952f058` |
| [#3181](https://github.com/Tencent/WeKnora/pull/3181) | feat(sandbox): add safe live file management to session sandboxes | **Merged upstream，随后被 revert** | `feat/topic2-safe-live-files` | 合并为 `01205b3d56c1f007457c76be223f7aad568d9593` |
| — | 实时文件管理重做（最新基线版本，尚未单独提 PR） | **仅最终集成版本** | `feat/topic2-safe-live-files-v2` | `7280876f2d9569279984826ede59c1c2a4991c0a` |

## 每个模块的贡献

### #3175 Docker 交互式终端（Open upstream）

把 Docker 容器内执行接入仓库既有的 `RemoteTerminalSession` 抽象，成为本地子进程、Cube、E2B 之外的第三个 provider，而不是另起一套平行实现。

- 新增 `internal/sandbox/docker_terminal.go`：Docker Engine 原生 TTY 会话、RPC 超时、活动刷新、事件语义与并发处理。
- 前端 `useSandboxTerminal` 支持重连，并在 ready 之前阻断重连。
- 含真实 Docker Engine 的集成测试、重连场景测试与设计文档（`docs/plans/docker-interactive-terminal-design.md`、`docs/poc/docker-terminal-reattach-spike.md`）。

### #3190 终端命令审计（Open upstream）

让终端里执行的每一条命令都可在审计日志中查询，并在写入前脱敏。

- 新增 `internal/handler/session/sandbox_terminal_audit.go`：从终端输出流中捕获 shell 集成标记，提取命令与退出码。
- 敏感信息脱敏（密钥、Token 等模式）后才落库。
- 新增审计作用域查询与对应测试。

### #3217 Docker 沙箱硬限额（Open upstream）

补齐容器资源限额，使会话超出 CPU、内存或存活时长时被强制终止。

- `internal/sandbox/docker_limit_watcher.go`：看护容器，超过 CPU 时间预算、内存上限或最长存活时间后强制删除沙箱。
- 新增租户级配置项与校验（`internal/types/tenant_docker_limits_test.go`）。
- `docs/plans/docker-sandbox-limit-gap-report.md` 记录了开发前对现有能力缺口的实测盘点。

### #3218 产物语义类型（Open upstream）

为产物定义类型标记，前端据此选择展示方式：演示文稿按页浏览、网页类在受限 iframe 内渲染、表格转表格视图。

- `internal/types/message.go` 增加产物 `kind` 的语义定义与测试。
- 前端 `sessionArtifacts` 按类型分发展示。

### #3219 演示文稿生成 Skill（Open upstream）

打通从 Agent 生成到界面预览与下载的完整链路。

- `examples/skills/presentation-generation/`：受限的 JSON → PPTX 构建器，五种版式（`title`、`section`、`bullets`、`two_column`、`metrics`）。
- 运行时依赖固定为 `python-pptx==1.0.2`。
- 校验 OOXML 打包、页数、文本可编辑性、Unicode、版式边界，以及生成物能被仓库既有 PPTX 预览预处理器接受。
- 不引入办公套件平台，也不做 DOCX/XLSX/PDF 生成。

### #3181 会话沙箱实时文件管理（Merged upstream，随后被 revert）

文件管理器：浏览沙箱目录，支持上传下载、重命名与删除，权限沿用现有路径限制规则。

- 该 PR 于 2026-09-11 合并（squash commit `01205b3d`），同日被 PR #3188 revert，功能从 `upstream/main` 中移除。
- revert 由另一位在途冲突 PR 的作者提出，属于合并冲突让路（同一开发计划中记录 #3168 为"未合并的冲突草稿"），不是对本模块质量的否定。
- 因此本课题在最新 `upstream/main` 上重新实现了一版，见下。

### 实时文件管理重做（仅最终集成版本）

- 分支 `feat/topic2-safe-live-files-v2`，commit `7280876f`，基于 `upstream/main` @ `462999ec`。
- 重新实现 `SandboxLiveFilesService`、会话文件 HTTP 接口、`SandboxFilesPanel.vue` 与 `sandbox-files.ts`，并配套服务层、handler 层、沙箱层测试。
- 本模块是验收项 A2（租户隔离）与 A3（越界路径拒绝）的实现载体。

## 集成版本

`feat/rhino-topic2-final` 基于 `upstream/main` @ `462999ec`，依次合入上述六个模块中仍然需要的提交，作为本课题的最终交付版本。

已合并进上游的提交不重复 cherry-pick；被 revert 的 #3181 用重做版替代。

集成过程中出现过两处真实冲突，均已解决：

1. `internal/handler/session/sandbox_terminal_bridge_test.go` — #3175 与 #3190 各自改动了同一个测试辅助函数。保留 #3175 的重连感知辅助函数，并把 #3190 的 `configure` 钩子参数并进去。
2. `internal/handler/session/handler.go` 与五份 i18n 语言文件 — 两个模块各自注册依赖、各加 i18n 键。两侧全部保留。

（说明：#3175 的 PR head 是 `4ddf7d99`，比本地 `feat/topic2-docker-terminal` 分支多三个提交，含"address Docker terminal review findings"等 review 修复。集成版本合入的是 PR head，不是本地旧分支。）
