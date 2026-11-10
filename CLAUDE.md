# 项目约定

个人 Agent 开发知识库（Markdown 笔记 + 资料 PDF），托管于个人 GitHub。

## 权限

- 本仓库内 Claude 的所有操作均已授权，直接执行，不要询问确认。
- 本规则覆盖全局 CLAUDE.md 中的 Git 操作纪律：commit、push 等 Git 操作可直接执行。

## Git 提交

- 远程：`origin` → https://github.com/EliaukUsagi/eliauk-agent-dev-knowledge.git ，分支 `main`。
- 提交身份必须使用个人 GitHub 账号（已写入仓库级 `.git/config`，不使用公司账号 `changliu35`）：
  - `user.name`：`EliaukRender`
  - `user.email`：`113354699+EliaukRender@users.noreply.github.com`
- 提交前用 `git config user.email` 确认身份；若被重置，按上面的值重新设置。
- 内容有新增或修改时，默认流程：`git add` → `git commit` → `git push origin main`，一次完成。
- 提交信息格式：`docs: 中文描述`（Conventional Commits 前缀 + 中文描述）。

## 大文件

- GitHub 单文件硬上限 100 MB，超过 50 MB 会告警。
- 新增文件接近或超过 100 MB 时，先改用 Git LFS 或不入库，再提交。
