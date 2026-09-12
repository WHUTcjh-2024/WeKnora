# 课题二：可视化沙箱工作台 — 提交材料

腾讯犀牛鸟开源人才培养计划 · WeKnora 课题二

| 项 | 值 |
|---|---|
| 选题编号 | 二 |
| 选题题目 | 可视化沙箱工作台 |
| 成果类型 | 代码 |
| 仓库 | https://github.com/WHUTcjh-2024/WeKnora |
| 集成分支 | `feat/rhino-topic2-final` |
| 最终 Tag | `rhino-2026-final-2` |
| 最终 Commit SHA | 见 `submission.yaml`（由 Git 实际读取，40 位完整 SHA） |
| 基线 | `Tencent/WeKnora` `upstream/main` @ `462999ec` |

## 项目目标

WeKnora 的 Agent 可以在沙箱里执行代码，但这一切对用户是不可见的：Agent 跑了什么命令、生成了什么文件，用户只能从一个下载抽屉里间接感知。课题二的目标是把沙箱做成看得见、能上手的工作台，让数据分析和文档生成这类任务变得可用。

本仓库在本课题下完成了六个独立模块，每个模块单独向上游提交了 PR；`feat/rhino-topic2-final` 分支是这些模块在最新 `upstream/main` 之上的集成版本，用于交付验收。

## 选题任务 → 交付映射

| 选题任务 | 交付模块 | 上游 PR |
|---|---|---|
| 设计统一的终端会话接口，屏蔽本地子进程 / Docker 容器 / 远端虚拟机差异 | Docker 作为 `RemoteTerminalManager` 的第三个 provider | [#3175](https://github.com/Tencent/WeKnora/pull/3175) |
| 实现交互式终端：页面内输入命令、查看实时输出、支持中断 | 终端桥接、实时输出与中断（含 Docker 实现） | [#3175](https://github.com/Tencent/WeKnora/pull/3175) |
| 实现文件管理器：浏览、上传下载、重命名、删除，沿用现有路径限制 | 会话实时文件管理与文件面板 | [#3181](https://github.com/Tencent/WeKnora/pull/3181)（已合并后被 revert，见下）/ `feat/topic2-safe-live-files-v2` |
| 为产物定义类型标记，前端据此选择展示方式 | 产物语义类型（`kind`） | [#3218](https://github.com/Tencent/WeKnora/pull/3218) |
| 新增可生成演示文稿的 Skill，打通生成 → 预览 → 下载 | 演示文稿生成 Skill | [#3219](https://github.com/Tencent/WeKnora/pull/3219) |
| 验收：每条终端命令可在审计日志中查询 | 终端命令审计与脱敏 | [#3190](https://github.com/Tencent/WeKnora/pull/3190) |
| 验收：超出 CPU / 内存 / 时长限额时自动终止 | Docker 沙箱硬限额 | [#3217](https://github.com/Tencent/WeKnora/pull/3217) |

## 架构

终端侧沿用仓库既有的 `RemoteTerminalSession` 抽象：本地进程、Cube、E2B 已有实现，本课题新增 Docker provider，使 Docker 容器内执行成为同一抽象下的第三个实现，而不是平行子系统。

```text
前端 SandboxTerminal / SandboxFilesPanel
        │  WebSocket（终端）+ HTTP（文件、产物）
        ▼
internal/handler/session
   sandbox_terminal_ws.go / sandbox_terminal_bridge.go   终端桥接、认证、空闲断开
   sandbox_terminal_audit.go                             命令捕获与脱敏
   sandbox_live_files.go                                 文件浏览 / 上传 / 下载 / 改名 / 删除
        ▼
internal/application/service
   SandboxTerminalService     终端会话编排、首次使用时按会话 provisioning
   SandboxLiveFilesService    文件操作，公开路径限制规则
        ▼
internal/sandbox
   remote_client.go           RemoteTerminalSession 抽象
   docker_terminal.go         Docker provider（新增）
   session_live_files.go      沙箱内文件原语
   docker_limit_watcher.go    资源限额看护（新增）
        ▼
Docker Engine / Cube / E2B / 本地子进程
```

产物链路：Agent 在沙箱产出文件 → `ArtifactCollector` 收集 → 按 `kind` 标记 → 前端按类型选择展示方式（演示文稿分页、网页在受限 iframe 内渲染、表格转表格视图）→ 预览或下载。

## 当前 `main` 已提供的能力（本课题没有重做）

为避免重复造子系统，以下能力复用仓库既有实现，本课题只做扩展：

- `RemoteTerminalManager` 与 Cube / E2B provider、终端 WebSocket 握手与认证
- 沙箱侧边面板、预览流水线（含 PPTX 预览预处理）
- Skill 运行时与 `ArtifactCollector` 产物收集
- 审计日志模型与查询接口
- 会话文件系统原语与既有路径限制规则

## 上游贡献

六个模块各自独立成 PR，避免一个巨大 PR 混在一起。详细清单、状态与每个模块的贡献说明见 [PR_CONTRIBUTIONS.md](PR_CONTRIBUTIONS.md)。

其中 **PR #3181 已在上游合并（squash commit `01205b3d`），随后被 PR #3188 revert**（revert 由另一位冲突 PR 的作者提出，属于合并冲突让路，不是质量缺陷）。该功能因此从 `upstream/main` 中消失，本课题按最新上游基线重新实现了一版，即 `feat/topic2-safe-live-files-v2`（`7280876f`），并纳入本次集成版本。

## 环境与运行

依赖、启动方式与 `upstream/main` 完全一致，请以仓库根目录的 [README.md](../README.md) 为准。下面是本课题相关的配置项。

### 沙箱后端配置

- **本地子进程**：默认可用，无需额外配置。
- **Docker**：需要可访问的 Docker Engine。终端在容器内执行，容器挂载会话工作目录，产物写入 `/workspace/output`。
- **E2B / CubeSandbox**：需要对应平台的凭据（API Key / 模板 ID）。

### 验证入口

本课题相关能力的测试命令与真实执行结果见：

- [TEST_COMMANDS.md](TEST_COMMANDS.md) — 只记录真实执行过的命令与结果
- [ACCEPTANCE_REPORT.md](ACCEPTANCE_REPORT.md) — 选题验收项 A1–A5 的逐项记录

## 安全边界

- 终端与文件接口沿用仓库既有的权限与租户隔离规则，跨租户不可见彼此进程与文件。
- 文件操作限制在会话沙箱的产物根目录内，越界路径（`..` 穿越、绝对路径、符号链接、改名逃逸）由服务端拒绝。
- 终端命令在写入审计日志前做敏感信息脱敏。
- 网页类产物运行在受限 iframe 内。

## 已知限制

见 [KNOWN_LIMITATIONS.md](KNOWN_LIMITATIONS.md)。所有未在本机真实跑通的验收项都按 `NOT_RUN` 记录，没有推断为 PASS。
