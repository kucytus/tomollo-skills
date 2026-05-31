---
name: docx-tweak
description: >-
  论文微调技能。解包 .docx → 编辑 XML → 重新打包。
  专注于学位论文、学术文章的格式规范和内容微调。
  适用于修改章节编号、调整标题样式、统一字体字号、查找替换术语、整理参考文献格式等任务。
allowed-tools: "Read, Write, Edit, Bash"
---

# Docx Tweak — 论文微调 Skill

## 触发条件

当用户需要润色/修改 .docx 格式的论文时激活：
- "帮我润色这篇论文"
- "调整论文格式"
- "统一论文中的标题样式"
- "修改学位论文的章节编号"
- "在这个 .docx 里查找替换 XXX"
- "帮我整理参考文献格式"

---

## 核心流程

```
分析需求 → 解包 → 编辑 → 重打包 → 输出
```

单次执行，所有修改一次性完成后输出，中间不等待用户确认。

---

## 核心约束

### 禁止打印 XML 到终端

`.docx` 中的 XML 文件通常包含大量冗长的 w:p/w:r/w:t 标签。**绝对禁止**使用以下命令：

```bash
# ❌ 禁止：打印 XML 到终端
cat word/document.xml
echo word/document.xml
python -c "print(xml)"
```

使用以下方式：

```bash
# ✅ 正确：用 Read 工具选择性读取
```

### 临时文件 vs 输出文件

- **临时文件**（解包目录、脚本、回滚验证）→ 存放在 `$CLAUDE_JOB_DIR` 下，避免与并行任务冲突
- **最终输出文件**（如 `thesis-polished.docx`）→ 写入**当前工作目录**，即用户运行此 skill 的所在目录

---

## 工作模式

### 一次性解包，增量编辑

**只解包一次，后续所有修改都在同一个解包目录中增量进行**。只有在处理全新的 .docx 文件时才需要重新解包。

### 提交与输出策略

git 仓库仅用于记录"用户确认可接受的状态"，不记录中间修改：

```
解包 → commit(初始状态) → 编辑 → 重打包 → 输出
                                               ↓ 用户反馈
                      ┌─────────────────────────┴────────────────┐
                      │ 不满意：git checkout HEAD → 重新编辑     │
                      │ 重打包 → 输出                            │
                      └─────────────────────────┬────────────────┘
                                                 │ 满意，无操作
下次有新修改请求时：
  检出未暂存的修改 → commit(记录上一次满意状态) → 编辑新内容

### 输出文件策略：覆盖

默认策略——**始终覆盖同一输出文件**。新请求产生的输出覆盖上一次的结果，避免磁盘上堆满无意义版本：

```bash
./thesis-polished.docx  ← 每次重打包覆盖
```

当用户说"存一份"、"导出一份"时，再复制为命名文件：

```bash
cp "./thesis-polished.docx" "论文_第3版.docx"
```

---

## Git 版本管理

解包目录中独立初始化一个 git 仓库，只追踪 `word/document.xml`、`word/styles.xml` 等数据文件。此仓库与用户的工程项目完全独立。

**提交时机：仅在两种情况下提交：**
1. **解包后** — 记录初始状态作为回滚锚点
2. **有新修改请求时** — 如果发现解包目录中有未暂存的修改（即上一次编辑的结果），先提交再应用新修改。这确保 git 历史中只保留用户实际可接受的状态

**提交信息：** 不要重读 XML 文件来凑提交信息。用 `git diff --stat` 快速查看改了哪些文件（只输出文件路径和改动行数，不输出内容），再结合对话中已知的修改意图来写：

```bash
# 解包后（第 1 次提交）
cd "$CLAUDE_JOB_DIR/polish-work"
git init
git add -A
git commit -m "初始状态：解包后的原始文件"
```

```bash
# 新修改请求前（检测未暂存修改 → 提交上一次结果）
cd "$CLAUDE_JOB_DIR/polish-work"
if ! git diff --quiet; then
  # git diff --stat 只输出文件路径和改动行数，不输出内容
  git diff --stat
  git add -A
  git commit -m "修改：[根据对话上下文和 git diff --stat 编写]"
fi
```

---

## 回滚

当用户对修改不满意要求回退时，git 上一次提交记录的是用户可接受的状态，直接恢复即可。不询问"是否确认"——直接执行。

```bash
cd "$CLAUDE_JOB_DIR/polish-work"

# 回滚到上一次提交（即最近一次"好"的状态）
git checkout HEAD -- .

# 重新打包，覆盖输出
ORIG_DIR=$(pwd)
cd "$CLAUDE_JOB_DIR/polish-work"
zip -r -q "$ORIG_DIR/thesis-polished.docx" .
```

**无 git 时**，从原始 .docx 重新解包后直接输出。

---

## 论文微调典型操作

| 操作 | 说明 | 涉及 XML |
|------|------|----------|
| 统一字体字号 | 正文宋体/Times New Roman、标题黑体 | `document.xml` 中 `<w:rPr>` |
| 章节编号修正 | 修正"一、1.1"等编号格式 | `document.xml` 中标题段落 |
| 查找替换术语 | 全文统一术语用词 | `<w:t>` 标签替换 |
| 调整标题样式 | 设置标题 1/2/3 对应的样式 ID | `document.xml` 中 `<w:pPr><w:pStyle>` |
| 行距/段距 | 调整全文间距 | `document.xml` 中 `<w:spacing>` |
| 目录更新 | 刷新目录字段 | `document.xml` 中 `<w:fldSimple>` 或 TOC 字段 |
| 页眉页脚 | 添加/修改页眉中的论文标题 | `header*.xml`, `footer*.xml` |
| 参考文献格式 | 整理引用样式 | `document.xml` 中尾注/脚注 |

---

## 详细步骤

### 1. 分析需求

从用户的请求中理解要做什么修改。一次性列出所有需要执行的操作，不拆分多轮：

- 格式调整？字体、字号、行距、标题样式？
- 内容润色？通顺语句、统一术语？
- 结构修改？章节编号、目录？
- 文档规范？页眉页脚、封面、声明页？

确认待处理的 .docx 文件路径。

### 2. 解包

```bash
mkdir -p "$CLAUDE_JOB_DIR/polish-work"
unzip -o "/path/to/thesis.docx" -d "$CLAUDE_JOB_DIR/polish-work"

cd "$CLAUDE_JOB_DIR/polish-work"
git init 2>/dev/null
git add -A
git commit -m "初始状态：解包后的原始文件"
```

查看文档结构：

```bash
ls -R "$CLAUDE_JOB_DIR/polish-work/"
```

主要 XML 文件及其用途：

| 文件 | 内容 |
|------|------|
| `word/document.xml` | 正文内容（段落、文本、表格） |
| `word/styles.xml` | 样式定义（标题 1/2/3、正文等） |
| `word/header*.xml` | 页眉 |
| `word/footer*.xml` | 页脚 |
| `word/numbering.xml` | 编号/列表定义 |
| `word/_rels/document.xml.rels` | 资源关系 |
| `[Content_Types].xml` | 内容类型声明 |

### 3. 编辑

开始编辑前，先检查解包目录中是否有未暂存的修改。如果有，说明这是用户带着新需求回来的——先提交上一次的修改结果，再应用新修改：

```bash
cd "$CLAUDE_JOB_DIR/polish-work"
if ! git diff --quiet 2>/dev/null; then
  git add -A
  git commit -m "修改：[列举主要修改项]"
fi
```

然后根据分析需求中确定的所有修改项，逐一执行编辑。全部编辑完成后再进入重打包步骤。

#### 3a. 全局查找替换

在 `<w:t>` 文本节点中替换内容：

```bash
cat > "$CLAUDE_JOB_DIR/replace.sh" << 'SCRIPT'
cd "$CLAUDE_JOB_DIR/polish-work"
OLD="自注意力机制"
NEW="Self-Attention"
sed -i '' "s|<w:t>[^<]*$OLD[^<]*</w:t>|<w:t>${NEW}</w:t>|g" word/document.xml
SCRIPT
```

#### 3b. 样式统一

修改 `word/styles.xml` 中的样式定义：

```
<w:style w:styleId="Heading1">
  <w:name w:val="heading 1"/>
  <w:rPr>
    <w:rFonts w:eastAsia="黑体"/>
    <w:sz w:val="32"/>
    <w:b/>
  </w:rPr>
</w:style>
```

常见字号对照（半磅值）：

| 半磅值 | 实际字号 |
|--------|----------|
| 42 | 小初 (21pt) |
| 36 | 二号 (18pt) |
| 32 | 三号 (16pt) |
| 28 | 小三 (14pt) |
| 24 | 四号 (12pt) |
| 21 | 小四 (10.5pt) |
| 20 | 五号 (10pt) |

常见中文论文格式约定：

- **标题 1** (章)：黑体/三号(32)
- **标题 2** (节)：黑体/四号(24)
- **标题 3** (条)：黑体/小四(21)
- **正文**：宋体/小四(21)，Times New Roman 西文
- **行距**：1.5 倍行距 (360)

#### 3c. 段落属性修改

修改段落间距/行距（在 `<w:pPr>` 内）：

```
<w:spacing w:line="360" w:lineRule="auto" w:before="0" w:after="0"/>
```

- `w:line="360"` = 1.5 倍行距（240 = 单倍，360 = 1.5 倍，480 = 双倍）
- `w:before` / `w:after`：段前段后距（单位：缇 twips，1 磅 = 20 缇）

#### 3d. 复杂修改

编写辅助脚本到 `$CLAUDE_JOB_DIR`：

```bash
cat > "$CLAUDE_JOB_DIR/edit.py" << 'SCRIPT'
# 处理 xml.etree.ElementTree 或 lxml
SCRIPT
python "$CLAUDE_JOB_DIR/edit.py"
```

### 4. 重打包

所有修改完成后，一次性打包输出：

```bash
ORIG_DIR=$(pwd)
cd "$CLAUDE_JOB_DIR/polish-work"

zip -r -q "$ORIG_DIR/thesis-polished.docx" .

cd "$ORIG_DIR"

# 验证
unzip -l "./thesis-polished.docx" | head -20
```

### 5. 输出

- 告知用户输出文件路径
- 列出所做的修改清单（如：统一了标题字体为黑体、替换了 8 处术语、调整了行距）
- 告知用户可以打开文件确认效果
- 如用户后续还有新需求，保留解包目录（`$CLAUDE_JOB_DIR/polish-work`）以供增量修改

---

## 常见 XML 命名空间前缀

| 前缀 | 命名空间 |
|------|----------|
| `w` | WordprocessingML 主命名空间 |
| `r` | 关系命名空间 |
| `pic` | 图片 |
| `wp` | 绘图对象定位 |
| `a` | 绘图ML |
| `mc` | Markup Compatibility |

---

## 注意事项

1. **XML 转义** — `<w:t>` 中特殊字符：`&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`
2. **保留 XML 声明** — `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>` 必须保留
3. **命名空间属性** — 根标签的 `xmlns:w` 等命名空间声明不能丢失
4. **合并 `<w:r>`** — 同一 `<w:p>` 内相邻 `<w:r>` 如果格式相同可以合并其中 `<w:t>` 内容，方便全文替换
5. **中文编码** — 确保处理工具正确处理 UTF-8 编码
6. **解包目录可复用** — 同一份文档的多次修改在同一个解包目录中增量进行，不需要重新解包。用户明确表示不再修改后删除 `$CLAUDE_JOB_DIR/polish-work`
7. **备份原文件** — 修改前先备份原始 .docx
