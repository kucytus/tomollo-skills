---
name: iter-mini
description: >-
  Git 分支与版本管理迭代工作流。解决 AI 迭代项目时的三个核心问题：
  (1) 不知道该不该建分支 (2) 不知道该基于哪个分支 (3) 忘记更新版本号。
  需显式调用触发：iter-mini: / 迷你迭代:
allowed-tools: "Read, Write, Edit, Bash"
---

# Iter Mini — 分支与版本管理迭代 Skill

## 触发条件

**仅当用户显式调用时激活**，不接受模糊匹配或关键词猜测：

- `iter-mini: <迭代需求>`
- `迷你迭代: <迭代需求>`

示例：
- `迷你迭代: 给 CLI 工具增加 --verbose 参数`
- `iter-mini: 修复解析器传入空字符串时报错的问题`
- `iter-mini: 重构数据库查询模块，把原始 SQL 改为 ORM`

**不会在以下场景自动触发：**
- "帮我写个 X" — 这不是迭代，请用常规对话
- "修改 X 功能" — 没有明确前缀不算
- "加一个 Y 方法" — 同上

> 本 skill 不依赖项目中的其它任何 skill，是独立的工作流。

---

## 流程概览

```
1. 需求解析 ── 理解迭代内容，评估类型和规模
                    ↓
2. 分支决策 ── ╭ 需要新分支吗？→ 基于哪个分支？
               ╰ 记录决策原因
                    ↓
3. 版本评估 ── ╭ 检测项目版本文件
               ╰ 确定版本更新幅度（major/minor/patch）
                    ↓
4. 迭代实施 ── 在正确的分支上开发
                    ↓
5. 收尾确认 ── ╭ 版本号复核
               ╰ 提交状态总结
```

---

## 步骤说明

### 1. 需求解析

解读用户以 `iter-mini:` 或 `迷你迭代:` 前缀传入的需求。

**输出格式：**

```
## 迭代需求

迭代内容: [用户需求原文]

项目性质: [根据项目文件推断]
迭代类型: [新功能 | 缺陷修复 | 重构 | 优化 | 文档 | 其他]
规模评估: [小型（< 5 文件）| 中型（5-15 文件）| 大型（> 15 文件）]
影响范围: [内部实现 | 接口变更 | 用户可见]
破坏性: [是 / 否] — 是否涉及不兼容变更
```

项目性质通过扫描项目根目录的文件推断：
- `*.py / *.js / *.rs / *.go` 等代码文件占多数 → **软件项目**
- `*.md / *.docx / *.txt / *.rst` 等文档占多数 → **文档项目**
- 混合情况或不确定 → **综合**

在此基础上直接进入下一步。不需要用户确认，除非需求有歧义。

---

### 2. 分支决策

**核心功能：解决前两个痛点**

#### 2a. 检查当前 Git 状态

迭代开始前，先确认本地仓库与远程同步，并检测项目的稳定发布分支。

**检测稳定发布分支名称：**

```bash
# 从远程 HEAD 检测稳定发布分支（即项目的默认分支）
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
# 如果检测失败（无远程或未设置 HEAD），回退到本地常见名称
if [ -z "$DEFAULT_BRANCH" ]; then
  for b in main master release stable; do
    git rev-parse --verify "$b" &>/dev/null && DEFAULT_BRANCH="$b" && break
  done
fi
echo "稳定发布分支: $DEFAULT_BRANCH"
```

**同步到远程最新状态：**

```bash
# 保存当前分支名
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)

# 获取远程最新数据
git fetch origin

# 检查本地稳定分支是否落后于远程
git rev-list --count --left-right origin/$DEFAULT_BRANCH...$DEFAULT_BRANCH
# 返回格式: behind ahead
# 如果 behind > 0，说明本地稳定分支落后，需要先更新

# 如果落后，拉取最新代码
git checkout $DEFAULT_BRANCH && git pull origin $DEFAULT_BRANCH

# 回到当前分支
git checkout $CURRENT_BRANCH
```

如果同步过程中发现冲突，暂停并告知用户。

同步完成后，获取当前分支信息：

```bash
git status                      # 当前工作区状态
git branch -a                   # 所有分支概览
git log --oneline -10           # 最近的提交历史
```

#### 2b. 分支决策矩阵

| 当前分支 | 迭代类型 | 是否需要新分支 | 基于哪个分支 |
|----------|---------|---------------|-------------|
| 稳定发布分支（$DEFAULT_BRANCH） | 任何 | **是**（禁止直接在稳定分支开发） | 稳定分支自身 |
| 特性/迭代分支 | 同一需求的迭代 | 可复用当前分支 | — |
| 特性/迭代分支 | 不相关的新需求 | **是** | 稳定分支 |
| 特性/迭代分支 | 紧急缺陷修复 | **是** | 稳定分支 |
| 任何分支 | typo/注释/只改 1-2 行 | 当前分支即可 | — |

> **稳定发布分支**即项目的默认分支（通常是 `main` 或 `master`），代表当前稳定的发布版本。所有特性开发和修复都应该在独立的分支上进行，通过 PR/MR 合并回稳定分支。

#### 2c. 分支命名标准

分支命名没有通用标准，**应该根据项目性质选择合适的命名体系**。AI 不应在文档项目上套用 `feat/`，也不应在软件项目上套用 `add/`。

根据步骤 1 确定的「项目性质」选择对应模板：

##### 模板 A：软件项目

适用于包含代码逻辑的项目（库、工具、应用、后端、前端等）。

| 迭代类型 | 前缀 | 示例 |
|---------|------|------|
| 新功能/特性 | `feat/` | `feat/增加递归扫描参数` |
| Bug 修复 | `fix/` | `fix/空指针异常处理` |
| 紧急修复 | `hotfix/` | `hotfix/支付接口超时` |
| 重构 | `refactor/` | `refactor/数据库ORM迁移` |
| 杂项/配置 | `chore/` | `chore/升级eslint到v9` |
| 文档/注释 | `docs/` | `docs/更新API使用说明` |
| 性能优化 | `perf/` | `perf/缓存热点数据查询` |
| 测试 | `test/` | `test/增加登录单元测试` |

##### 模板 B：文档项目

适用于以文章、文档、知识库为主的项目（博客、wiki、技术文档、书籍等）。

| 迭代类型 | 前缀 | 示例 |
|---------|------|------|
| 新增内容 | `add/` | `add/部署指南章节` |
| 修订更新 | `update/` | `update/升级说明适配v3` |
| 结构调整 | `restructure/` | `restructure/拆分入门教程` |
| 勘误纠错 | `fix/` | `fix/cURL命令参数错误` |
| 校对润色 | `proof/` | `proof/架构章节语句通顺` |

##### 模板 C：综合/自定义

项目性质不属于上述两类，或用户有自己的命名习惯时，在此处与用户确认：

> "当前项目没有匹配的命名模板。请简单说一下你的分支命名习惯，或指定一个模板调整。"

常见自定义场景：
- **Skill 项目**（如本仓库）：以操作为主 → `add/新增skill`、`update/完善文档`、`fix/修复描述`
- **配置/模板项目**：`add/`、`update/`、`remove/`
- **数据项目**：`add/`、`update/`、`migrate/`、`clean/`

**通用命名规则：**
- 格式：`<前缀>/<描述>`
- 描述用中文短语，**简洁表意**，10 字以内
- 描述用 `-` 连接英文单词，或直接写中文短语

```bash
# 软件项目示例
git checkout -b feat/增加递归扫描参数 $DEFAULT_BRANCH

# 文档项目示例
git checkout -b add/部署指南章节 $DEFAULT_BRANCH

# Skill 项目示例
git checkout -b add/iter-mini-skill $DEFAULT_BRANCH
git checkout -b update/完善分支决策说明 $DEFAULT_BRANCH
```

#### 2d. 输出并执行

分支决策完成后，输出决策方案并立即执行分支操作：

```
## 分支决策

当前分支: [当前分支名]
未提交改动: [有 / 无]
项目性质: [根据步骤 1 确定]
命名模板: [根据项目性质选择]

决策: [新建分支 / 复用当前分支]
基于分支: [分支名]
新建分支: [分支名]
决策原因: [决策依据]
```

```bash
# 新建分支
git checkout -b [新建分支名] [基分支]
```

---

### 3. 版本评估

**核心功能：解决第三个痛点**

#### 3a. 检测版本文件

检测当前项目中可能存在的版本文件：

| 文件 | 类型 | 版本字段 |
|------|------|----------|
| `package.json` | Node.js | `"version": "x.y.z"` |
| `Cargo.toml` | Rust | `version = "x.y.z"` |
| `pyproject.toml` | Python | `version = "x.y.z"` |
| `VERSION` | 纯文本 | 文件内容即为版本号 |
| `version.txt` | 纯文本 | 文件内容即为版本号 |
| `version.py` | Python 模块 | `__version__ = "x.y.z"` |
| `__version__.py` | Python 模块 | `__version__ = "x.y.z"` |
| `__init__.py` | Python 包 | `__version__ = "x.y.z"` |
| `project/__init__.py` | Python 包 | `__version__ = "x.y.z"` |
| `CMakeLists.txt` | CMake | `project(VERSION x.y.z)` |
| `setup.cfg` | Python | `version = x.y.z` |
| `setup.py` | Python | `version="x.y.z"` |

查找方式：

```bash
# 在当前目录下搜索常见版本文件
ls package.json 2>/dev/null && head -30 package.json | grep '"version"'
ls Cargo.toml 2>/dev/null && head -10 Cargo.toml | grep '^version'
ls pyproject.toml 2>/dev/null && head -20 pyproject.toml | grep '^version'
ls VERSION version.txt 2>/dev/null
# ...etc
```

如果找到多个版本文件，列出所有找到的。后续更新时同步修改所有文件。

如果**没有找到**任何版本文件，跳过此步，通知用户。

#### 3b. 确定版本更新幅度

| 迭代类型 | 默认更新 | 说明 |
|----------|---------|------|
| 新功能/新特性 | **minor** (x.y+1.0) | 新增功能不破坏兼容 |
| 缺陷修复 | **patch** (x.y.z+1) | 只修 bug，不改变 API |
| 重构（接口不变） | **patch** (x.y.z+1) | 内部重写，对外不变 |
| 破坏性变更 | **major** (x+1.0.0) | 不兼容的 API 变更 |
| 文档/注释 | **不更新** | 不影响运行 |
| 优化/性能 | **patch** (x.y.z+1) | 不改功能只改效率 |

#### 3c. 执行版本更新

输出版本评估结果并直接执行版本更新：

```
## 版本评估

检测到版本文件:
- package.json → 当前版本 1.2.3

建议更新: 1.2.3 → 1.3.0 (minor)
依据: 新增功能，向下兼容

更新内容:
- package.json: "version": "1.3.0"
```

执行版本文件修改：

```bash
# 示例：修改 package.json 版本
# 使用 sed 或编辑工具修改对应字段
```

---

### 4. 迭代实施

按分支决策和版本评估的结果执行开发。

实施过程中的规范：

1. **频繁提交** — 完成一个有意义的步骤就提交一次，commit message 用中文扼要描述改动内容
2. **提交信息 ≠ 分支命名** — 分支前缀（feat/fix/hotfix 等）只用于分支名称，不要套用到提交信息上
3. **不做计划外的事** — 有新想法记下来，但不要在当前迭代中实现
4. **遇到困难主动同步** — 卡住超过 5 分钟就问用户意见

提交格式：

```bash
git add <files>
git commit -m "$(cat <<'EOF'
增加 --verbose 参数

- 添加 argparse 对 --verbose/-v 的支持
- 根据 verbose 级别控制日志输出
- 更新 README 添加参数说明
EOF
)"
```

---

### 5. 收尾确认

迭代实施完成后，做最后复核并输出报告：

```
## 迭代收尾

### 确认清单
- 分支状态: [当前所在分支]
- 未提交改动: [有 / 无]

### 版本复核
- 版本文件版本号: [当前值]
- 是否已更新: [是 / 否]
- 与迭代规模匹配: [是 / 否]

### 提交记录
- [提交 1]
- [提交 2]
...
```

### 用户待办

以下事项交由用户自行处理：

1. **提交代码** — 审核代码后，由用户执行 `git add` / `git commit`
2. **推送代码** — 提交后由用户执行 `git push`
3. **创建 PR** — 如有需要由用户在远程仓库创建 Pull Request

输出报告后流程结束。用户可自行发起新一轮迭代。

---

## 注意事项

### 分支 vs 工作树

本 skill **始终使用分支进行迭代**，不使用工作树（worktree）：

| 方式 | 适用场景 | 本 skill 的选择 |
|------|---------|---------------|
| 分支 | 单人顺序迭代，一次只做一个任务 | **✅ 使用** |
| 工作树 | 并行开发、多会话同时进行 | ❌ 不适合 |

**如果当前已在工作树环境中**（如 `.claude/worktrees/`），迭代开始时先退出工作树，回到主仓库目录，再按分支工作流执行：

```bash
# 退出工作树（保留目录）
exit-worktree keep

# 确认已回到主仓库目录
pwd
```

单人迭代中所有工作在一条时间线上顺序进行，分支切换 (`git checkout -b` / `git checkout $DEFAULT_BRANCH`) 足够满足需求。无需工作树来增加磁盘占用和清理负担。

---

## 完整流程示例

```
用户: 迷你迭代: 给图片压缩工具增加递归扫描子目录的功能

你:

## 迭代需求

迭代内容: 给图片压缩工具增加递归扫描子目录的功能
迭代类型: 新功能
规模评估: 小型（涉及 2-3 个文件）
影响范围: 内部实现
破坏性: 否（新增参数 --recursive/-r，不影响原有功能）

---

## 分支决策

DEFAULT_BRANCH 检测 → main（稳定发布分支）

当前在 main，工作区干净

决策: 新建分支
基于分支: main
新建分支: feat/recursive-scan
决策原因: 新功能开发，禁止直接在稳定发布分支上开发
```

这样三个核心问题全部在流程中被强制处理，不会遗漏。
