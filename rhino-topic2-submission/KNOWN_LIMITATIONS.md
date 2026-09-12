# 已知限制

本文件记录真实的限制与未验证项。**未在本机真实跑通的项一律按 `NOT_RUN` 记录，不推断为 PASS。**

## 环境限制

### 无法在本机链接出可执行文件

`cmd/server` 与 `cmd/desktop` 都依赖 `github.com/duckdb/duckdb-go-bindings` 的预编译 C++ 静态库。在本机 Windows + mingw-w64 工具链下，链接阶段失败：

```text
undefined reference to `std::basic_streambuf<char, std::char_traits<char> >::seekpos(...)'
undefined reference to `__stdio_common_vsnprintf_s'
undefined reference to `__stdio_common_vswprintf'
```

这是第三方 C++ 静态库与本机 libstdc++ 的 ABI 不匹配，与本次改动无关：

- `go build ./internal/...` 在本机通过（需按下方说明提供 sqlite CGO 头文件路径）。
- `go vet ./cmd/server/` 通过，说明改动后的依赖接线在类型层面是正确的。
- 二进制级冒烟测试（真实启动服务、浏览器里点开终端与文件面板）**本机 `NOT_RUN`**，需要在 Linux/Docker 环境执行。

### sqlite-vec CGO 头文件

本机裸跑 `go build ./internal/...` 会因 `github.com/asg017/sqlite-vec-go-bindings` 找不到 `sqlite3.h` 失败。需要显式提供头文件路径：

```bash
export CGO_CFLAGS="-I<path-with-sqlite3.h>"
```

这是本机环境问题，不是仓库问题。

### 既有的基线测试失败

以下失败在干净的 `upstream/main` 检出上同样复现，与本课题改动无关：

| 包 | 测试 | 原因 |
|---|---|---|
| `internal/handler` | `TestDeploymentCapabilityKeysMatchFrontend` | 本机 `core.autocrlf=true`，而 `.gitattributes` 只对 `*.sh`、`*.go` 等强制 `eol=lf`，不含 `*.ts`。前端文件按 CRLF 检出，测试解析出的 key 带尾随 `',`。 |
| `internal/handler` | `TestPutTenantParserConfigAdminPreservesRedactedSecrets` | 期望 HTTP 200，实际 400；同为既有失败。 |

## 未验证的后端

| 后端 | 状态 | 说明 |
|---|---|---|
| 本地子进程 | 未在本机端到端验证 | 需要能启动服务（见上） |
| Docker | 未在本机端到端验证 | 需要能启动服务 + 可访问的 Docker Engine |
| E2B | `NOT_RUN` | 本机无 E2B 凭据 |
| CubeSandbox | `NOT_RUN` | 本机无 Cube 凭据 |

因此验收项 A1 中"终端在至少两种沙箱后端上可用"这一条，本机只做了代码与单元测试层面的验证，**两种后端的真实运行验证 `NOT_RUN`**，需要在具备凭据与 Docker 的环境复跑 `internal/sandbox/docker_integration_test.go` 等集成测试。

## 设计限制

- **Docker 终端重连**：重连能力按后端的 `Reattachable` 能力协商。Cube / E2B 支持重连；Docker provider 的重连支持范围见 `docs/poc/docker-terminal-reattach-spike.md` 的实测记录，未覆盖的场景不会声称支持。
- **集成分支的性质**：`feat/rhino-topic2-final` 是各模块分支在最新 `upstream/main` 上的**合并**结果，用于交付验收；上游评审的单元仍是六个独立 PR，不是这个大分支。
- **前端完整构建未跑**：集成版本未执行完整前端构建与全量前端测试，只跑了与本次改动相关的测试文件。
- **全仓库测试未跑**：只跑了与模块相关的包，未跑完整仓库测试套件。
