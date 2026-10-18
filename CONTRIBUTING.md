# Contributing / 维护流程

本仓库是 [`dlsdl/metanum`](https://github.com/dlsdl/metanum) 的维护 fork。

| 远程 | 用途 |
|---|---|
| `upstream` → `dlsdl/metanum` | 上游发布源 |
| `origin` → `MarchBeta2087/metanum` | 本 fork |

## 分支模型

| 分支 | 用途 | 更新方式 |
|---|---|---|
| `main` | 与上游发布对齐 | **只能通过 PR** |
| `dev` | 集成分支，日常改动先落这里 | **只能通过 PR** |
| `fix/*`、`docs/*`、`ci/*` | 短期工作分支 | 从 `dev` 开出，PR 回 `dev` |

`main` 和 `dev` 都受 ruleset「请使用PR」保护：禁止直接 push / 强推 / 删除。
合并前 CI 会运行 `npm test`（`node test_mixed.js`，当前 1149 条）。

## 日常改动

```bash
git fetch origin
git checkout dev
git pull --ff-only

git checkout -b fix/short-topic
# ... 修改 ...
npm test

git add -A
git commit -m "fix(parser): ..."   # 建议 conventional commits
git push -u origin fix/short-topic
gh pr create --base dev --head fix/short-topic
```

CI 变绿后再合并。合并后同步本地：

```bash
git checkout dev && git pull --ff-only
```

## 同步上游

上游发布新版本时，**不要**直接在 `main` 上 merge/push：

```bash
git fetch upstream --tags
git checkout main && git pull --ff-only

git checkout -b sync/upstream-2.1.0
git merge upstream/main
# 解决冲突并 npm test
git push -u origin sync/upstream-2.1.0
gh pr create --base main --head sync/upstream-2.1.0
```

`main` 合并后，再把 `main` 同步回 `dev`（同样走 PR）：

```bash
git checkout -b sync/dev-after-2.1.0 main
git push -u origin sync/dev-after-2.1.0
gh pr create --base dev --head sync/dev-after-2.1.0
```

需要给 fork 打标签时，在 `main` 同步完成后执行：

```bash
git checkout main && git pull --ff-only
git tag v2.1.0
git push origin v2.1.0
```

## 测试与文档

- 运行测试：`npm test`（Node 22）。
- 复现 Issue #17 的历史缺陷：`node docs/repro/issue-17.js`。
- 记号语义的规范来源是 [`docs/notation-semantics.md`](docs/notation-semantics.md)；
  改变 parser/formatter 行为时，必须同时更新该文档与 `test_mixed.js` 的回归向量。
- 后续项状态记录在 [`docs/followups.md`](docs/followups.md)。

## Issue 与上游 PR

- 关闭 issue 前逐条核对验收项，并在评论里给出可运行的复现/测试向量与提交号。
- 明确「不做」的项用 `not planned` 关闭，并链接设计记录。
- 向上游提交时，从 `upstream/main` 开一个干净分支，只带必要文件（不要带 fork 专用的 CI / 维护文档）：

```bash
git checkout -b upstream/short-topic upstream/main
# cherry-pick 或 checkout 需要的文件
npm test
git push -u origin upstream/short-topic
gh pr create --repo dlsdl/metanum --base main --head MarchBeta2087:upstream/short-topic
```
