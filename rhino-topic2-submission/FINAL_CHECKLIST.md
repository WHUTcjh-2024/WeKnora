# 提交前检查清单

Tag：`rhino-2026-final-2`
分支：`feat/rhino-topic2-final`
仓库：https://github.com/WHUTcjh-2024/WeKnora

## 仓库侧

| 检查项 | 结果 |
|---|---|
| 仓库链接正确、可打开 | ✅ `https://github.com/WHUTcjh-2024/WeKnora` |
| 最终 Tag 已创建并推送 | ✅ `rhino-2026-final-2` |
| Tag 指向的完整 SHA 记录在 `submission.yaml` | ✅ 由 `git rev-parse` 实际读取 |
| `submission.yaml` 的字段与仓库实际状态一致 | ✅ |
| README 里的命令真实可跑 | ✅ 命令与结果见 `TEST_COMMANDS.md` |
| A1–A5 报告完整，未跑项已标 `NOT_RUN` | ✅ `ACCEPTANCE_REPORT.md` |
| PR 链接有效 | ✅ 见 `PR_CONTRIBUTIONS.md` |
| 已合并 / 已 revert / 仅集成，三种状态没有混淆 | ✅ #3181 明确标注为 merged 后被 revert |
| 证据里没有密钥、Token | ✅ 未包含任何凭据 |
| 公开文档里没有本机绝对路径 | ✅ CGO 路径以 `<path-with-sqlite3.h>` 占位 |
| 没有推断出来的 PASS | ✅ 端到端项一律 `NOT_RUN` |
| 已知限制已记录 | ✅ `KNOWN_LIMITATIONS.md` |

## 邮件侧

| 检查项 | 结果 |
|---|---|
| 收件邮箱 `wxg_prc_cpg@tencent.com` | ☐ 发送前确认 |
| 标题含选题编号与姓名 / GitHub ID | ☐ |
| 正文含 Tag 与完整 40 位 Commit SHA | ☐ 从 `submission.yaml` 复制 |
| 发送时间早于 2026-09-13 00:00（北京时间） | ☐ |
| 发送后保留已发送邮件 | ☐ |

## 说明

- `submission.yaml` 记录的是**已打 Tag 的那个 commit**。提交 YAML 之后分支会多一个"材料提交"commit，这是正常的，**不要移动、删除或覆盖最终 Tag**。
- 集成分支是各模块分支的合并结果；上游评审的单元是六个独立 PR，不是一个巨型分支。
