# 下游私有区（downstream）

本目录只存在于 `mine` 分支，是我个人 fork 的规范与笔记区。上游自带的 `docs/`、`AGENTS.md`、`CONTRIBUTING.md` 属于上游，我写的一切都不往那里放。

## 分支约定

| 分支 | 角色 | 追踪 | 允许谁提交 |
|------|------|------|------------|
| `main` | 上游发布版镜像 | `upstream/main` | 只 ff-only 同步，不放我的代码 |
| `develop` | 上游日常集成镜像 | `upstream/develop` | 同上 |
| `mine` | 我的私有主线 | `origin/mine` | 我全部的工作都在这 |

- `origin` = `https://github.com/yinjiangit/Octop`（可写）
- `upstream` = `https://github.com/TencentCloud/Octop`（push URL 已置为 `https://invalid.example/none`，防误推；裸 `git push` 在镜像分支上必然失败）

铁律：**镜像分支永不承载我的提交**。一旦把功能 commit 到 `main`/`develop`，`--ff-only` 同步就会失效，之后每次同步都是冲突地狱。

## 为什么用 merge 而不是 rebase

自用、不提上游 PR，所以不需要线性历史。

- rebase：每次同步把全部私有提交重放一遍，同一处冲突反复解，上次怎么解的不留记录。
- merge：冲突解果固化在那个 merge commit 里，下次同步只解**新增**冲突。跑半年之后这个差别是决定性的。

代价（可接受）：`mine` 的历史里有一串 "sync: upstream" merge commit，不好读。反正不提 PR，无人在意。

## 同步流程（四步）

```bash
git fetch upstream --prune --tags
git checkout develop && git merge --ff-only upstream/develop && git push origin develop
git checkout mine && git merge develop -m "sync: upstream develop"
# 解冲突 → git add <文件> → git commit
```

同步前先备份数据库（见下节）。同步后必须验证：

```bash
uv run pytest -m "not live"          # 全量后端测试
uv run mypy --strict src/octop       # 类型
cd dashboard && npx tsc -b           # 动了前端时
```

搞砸了一步回退：`git reset --hard ORIG_HEAD`（merge 前 git 自动记录）。

查看上游动向而不合并：

```bash
git log --oneline develop..upstream/develop     # 上游新增了什么
git diff --stat develop...mine                  # 我到底改了多少
git tag --sort=-creatordate | head              # 最近的发布版
```

## 动手前的三条判断

1. **能配置解决就别改代码。** 改代码是永久负债（每次同步都要重解那次冲突）；配置是零冲突。先看 `docs/configuration.md` 和 `src/octop/config.py`（`config.json` + `OCTOP_*` 三层覆盖）。
2. **改动尽量是加法。** 新增文件、新增 `infra/` 子包、新增 router、往 `connectors` / `skills` / `slash` / `experts` 这些既有扩展位里挂东西——冲突面接近零。
3. **不删上游代码。** 删掉"用不上的"（多用户 auth、IM 渠道、某个 locale）之后上游会在同文件持续改动，你每次同步都撞，还会丢掉上游在那个文件里的 bugfix。要关功能用配置关。

## 同步前必须备份数据库

`run_migrations(db)` 挂在 `src/octop/infra/server.py:323`，服务器启动即自动迁移，**无确认环节**。迁移文件已到 `src/octop/infra/db/migrations/015_*.sql`。

```bash
uv run octop backup        # 见 src/octop/cli/commands/backup.py
```

别拿唯一的真实库做实验。这是自用 fork 最容易一次性丢数据的地方。

## 依赖不受控提示

`harness-agent` / `harness-gateway` 等来自 PyPI（`pyproject.toml:24-36`），`uv sync` 即可跑，不需要源码。但这些包的破坏性变更不由我控制：同步上游时 `uv.lock` 的 diff 也要一起看、一起测。

## git 行为备忘

切到 `main`/`develop` 时本目录会从工作区消失（镜像分支里没有它），回到 `mine` 又出现。这是预期行为，不是丢了。

## 已知上游文档过期点（以代码为准）

- `AGENTS.md` §7 称 schema 版本"currently `v == 7`"，实际迁移已到 015。
- `AGENTS.md` §6 称 CLI 为 `cli/*_cmd.py`，实际是 `src/octop/cli/commands/*.py`。

## 待办

- [ ] `make` 未安装（`winget install GnuWin32.Make` 或 `choco install make`），装好后执行 `git config core.hooksPath .githooks` 启用 pre-commit 门禁。**在装 make 之前不要开这个配置**，`.githooks/pre-commit:29` 无条件调 `make precommit` 且脚本是 `set -e`，开了会让每次 `git commit` 直接失败。
- [ ] 本地私有忽略项已写入 `.git/info/exclude`（不进仓库、不影响上游 `.gitignore`）。
