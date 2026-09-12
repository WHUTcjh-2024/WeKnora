# 测试命令与真实结果

只记录真实执行过的命令与真实结果。未执行的记为 `NOT_RUN`。

环境：Windows 11 + mingw-w64 (gcc/g++ 15.2.0)、Go toolchain、Node.js。分支 `feat/rhino-topic2-final`。

## 前置：本机 CGO 头文件

```bash
export CGO_CFLAGS="-I<path-with-sqlite3.h>"
```

不含这一项时 `./internal/...` 会因 `sqlite-vec-go-bindings` 找不到 `sqlite3.h` 而无法构建。这是本机环境问题，仓库无关。

## 编译与静态检查

| 命令 | 结果 |
|---|---|
| `go build ./internal/...` | **PASS** |
| `go vet ./cmd/server/` | **PASS**（类型层面确认依赖接线正确） |
| `go vet ./internal/handler/ ./internal/sandbox/` | **PASS** |
| `go build ./cmd/server/... ./cmd/desktop/...` | **FAIL（环境）** — duckdb C++ 静态库与本机 libstdc++ ABI 不匹配，链接期失败；干净 `upstream/main` 上同样失败 |

## Go 单元测试

```bash
go test ./internal/sandbox/ ./internal/handler/session/ ./internal/handler/ \
        ./internal/application/service/ ./internal/application/repository/ \
        ./internal/sandbox/ ./internal/types/ ./internal/router/ ./internal/container/ -count=1
```

| 包 | 结果 |
|---|---|
| `internal/handler/session` | **PASS** |
| `internal/application/repository` | **PASS** |
| `internal/types` | **PASS** |
| `internal/router` | **PASS** |
| `internal/sandbox` | FAIL — 3 个失败，**在干净 `upstream/main` 上同样失败**（见下） |
| `internal/handler` | FAIL — 2 个失败，**均为既有基线失败**（见下） |
| `internal/application/service` | FAIL — **在干净 `upstream/main` 上同样失败**（见下） |
| `internal/container` | **build failed** — duckdb 链接失败，环境问题 |

### 既有基线失败（干净 `upstream/main` 复现，与本课题改动无关）

在 `d:/Desktop/WeKnora-do/.worktrees/baseline-check`（`upstream/main` @ `462999ec` 的干净检出）上复跑同一批测试，逐一确认：

```bash
git worktree add --detach ../baseline-check upstream/main
cd ../baseline-check
export CGO_CFLAGS="-I<path-with-sqlite3.h>"
go test ./internal/sandbox/ -count=1
go test ./internal/application/service/ -count=1
```

| 包 | 失败测试 | 定性 |
|---|---|---|
| `internal/sandbox` | `TestWorkspaceBootstrapDoesNotRemoveOrMoveFiles` | 基线复现 |
| `internal/sandbox` | `TestResolveEffectiveConfigMapsDockerNoneToDeniedEgress` | 基线复现 |
| `internal/sandbox` | `TestPolicyAllowsPublicHostname` | 基线复现（涉及公网主机名解析） |
| `internal/handler` | `TestDeploymentCapabilityKeysMatchFrontend` | 基线：`core.autocrlf=true`，前端 `*.ts` 按 CRLF 检出，测试解析出的 key 带尾随 `',` |
| `internal/handler` | `TestPutTenantParserConfigAdminPreservesRedactedSecrets` | 基线：期望 200 实得 400 |

## 前端测试

本机 Node 可直接执行的测试文件：

```bash
cd frontend
node --test src/api/tenant/audit-log.test.mjs
node --test src/components/chat/SandboxSidePanel.test.mjs
node --test src/views/chat/components/SandboxFilesPanel.test.mjs
node --test src/views/chat/components/SandboxTerminal.test.mjs
```

| 测试文件 | 结果 |
|---|---|
| `src/api/tenant/audit-log.test.mjs` | **PASS** 2/2 |
| `src/components/chat/SandboxSidePanel.test.mjs` | **PASS** 2/2 |
| `src/views/chat/components/SandboxFilesPanel.test.mjs` | **PASS** 2/2 |
| `src/views/chat/components/SandboxTerminal.test.mjs` | **PASS** 7/7 |

合计 **13/13 PASS**。

## NOT_RUN

| 项 | 命令 / 入口 | 原因 |
|---|---|---|
| 真实 Docker Engine 集成测试 | `internal/sandbox/docker_integration_test.go` | 需要能启动服务与可访问的 Docker Engine，本机无法链接出可执行文件 |
| E2B 后端验证 | — | 本机无 E2B 凭据 |
| CubeSandbox 后端验证 | — | 本机无 Cube 凭据 |
| 前端完整构建与全量测试 | `npm --prefix frontend test` | 只跑了与改动相关的测试文件 |
| 演示文稿 Skill 的 Python 测试 | `python -m unittest discover -s examples/skills/presentation-generation/tests` | 本机未安装 `python-pptx` |
| 全仓库测试套件 | `go test ./...` | 只跑了与模块相关的包 |
| 浏览器端真实操作（终端输入、文件面板） | — | 需要能启动服务，本机无法链接 |
