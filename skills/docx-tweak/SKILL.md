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

当用户需要润色/修改 .docx 格式的论文时激活:
- "帮我润色这篇论文"
- "调整论文格式"
- "统一论文中的标题样式"
- "修改学位论文的章节编号"
- "在这个 .docx 里查找替换 XXX"
- "帮我整理参考文献格式"

---

## 核心流程

```
1. 分析需求 → 2. 解包 → 3. 定位 XML → 4. 编辑 → 5. 重打包 → 6. 验证
```

---

## 核心约束

### ⚠️ 禁止打印 XML 到终端

`.docx` 中的 XML 文件通常包含大量冗长的 w:p/w:r/w:t 标签。**绝对禁止**使用以下命令:

```bash
# ❌ 禁止: 打印 XML 到终端
cat word/document.xml
echo word/document.xml    ← 大文件，拖慢终端
python -c "print(xml)"    ← 同理禁止
```

使用以下方式:

```bash
# ✅ 正确: 用 Read 工具选择性读取
# Read 工具不会将大量文本打印到终端
```

### 临时文件 vs 输出文件

- **临时文件**（解包目录、脚本、回滚验证）→ 存放在 `$CLAUDE_JOB_DIR` 下，避免与并行任务冲突
- **最终输出文件**（如 `thesis-polished.docx`）→ 写入**当前工作目录**，即用户运行此 skill 的所在目录

---

## 工作模式

### 一次性解包，增量编辑

**只解包一次，后续所有修改都在同一个解包目录中增量进行**，避免重复解包：

```
第 1 次: 解包 → 编辑 → 重打包 → 确认 → 提交
第 2 次:                           → 编辑 → 重打包 → 确认 → 提交
第 3 次:                                                → 编辑 → 重打包 → ...
```

解包目录 `$CLAUDE_JOB_DIR/polish-work/` 就是"工作副本"，编辑对应 XML 文件后即可重打包。只有在处理全新的 .docx 文件时才需要重新解包。

重打包产物是临时验证文件，**始终覆盖同名输出**，不再反复生成新文件。版本差异由解包目录中的 git 保留。

### 输出文件策略：迭代覆盖，按需保存

默认策略——**始终覆盖同一输出文件**，版本由解包目录中的 git 管理：

```
./thesis-polished.docx                   ← 始终同名，每次重打包覆盖（当前工作目录）
$CLAUDE_JOB_DIR/polish-work/             ← git 仓库，保留完整修改历史
  .git/ ← 每次确认后 commit，可回退到任意版本
```

当用户说"存一份"、"导出一份"时，再复制为命名文件：

```bash
cp "./thesis-polished.docx" "论文_第3版.docx"
```

这样既保留完整版本历史（git），又避免磁盘上堆满 `_v1`、`_v2`、`_final` 之类无意义的文件。

### 用户确认 ≠ 全流程结束

关键区分：

| | 含义 | 动作 |
|--|------|------|
| **单轮确认** | 这一轮修改没问题 | git commit 记录本次修改，等待下一轮需求 |
| **全流程结束** | 所有修改都完成了 | 询问用户是否可以清理解包目录 |

**每次"单轮确认"后不应退出流程**，应主动询问"还需要继续修改吗？"。
只有当用户明确说"不用了"、"完成了"、"就这样"时，才是全流程结束，再清理临时文件。

---

## Git 版本管理（默认）

解包目录中独立初始化一个 git 仓库，专门管理 XML 数据文件的版本历史。此仓库与用户的工程项目仓库完全独立，只追踪 `word/document.xml`、`word/styles.xml` 等数据文件。

```
$CLAUDE_JOB_DIR/polish-work/    ← 独立的 git 仓库
├── word/
│   ├── document.xml             ← 正文
│   ├── styles.xml               ← 样式
│   ├── header1.xml              ← 页眉
│   └── ...
└── ...
```

每次修改用户确认满意后提交一次，形成一条清晰的修改时间线，方便随时回退。

---

## 回滚

回滚本质上就是**把 XML 数据文件恢复到之前的状态**，然后重新打包让用户确认。完整流程：

```bash
cd "$CLAUDE_JOB_DIR/polish-work"

# 1. 查看修改历史，确认要回到哪一步
git log --oneline
git diff --stat <hash>^!   # 查看某次改了什么（不打印 XML 内容）

# 2. 回滚 XML 文件到目标状态
git checkout <hash> -- .

# 3. 重新打包，让用户确认回滚结果
zip -r -q "$CLAUDE_JOB_DIR/thesis-rolled-back.docx" .
```

用户打开 .docx 确认回滚是否成功：
- ✅ **满意** — 用 git 提交回滚操作
- ❌ **不满意** — `git checkout HEAD -- .` 恢复到回滚前

不需要 git 时可直接从原始 .docx 重新解包。

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

先理解用户要做什么修改:

- 格式调整? 字体、字号、行距、标题样式?
- 内容润色? 通顺语句、统一术语?
- 结构修改? 章节编号、目录?
- 文档规范? 页眉页脚、封面、声明页?

确认待处理的 .docx 文件路径。

### 2. 解包

创建临时工作目录并解包，然后初始化 git 用于版本管理:

```bash
mkdir -p "$CLAUDE_JOB_DIR/polish-work"
unzip -o "/path/to/thesis.docx" -d "$CLAUDE_JOB_DIR/polish-work"

# 初始化 git，记录每次修改以便回滚
cd "$CLAUDE_JOB_DIR/polish-work"
git init
git add -A
git commit -m "初始状态：解包后的原始文件"
```

查看文档结构:

```bash
ls -R "$CLAUDE_JOB_DIR/polish-work/"
```

主要 XML 文件及其用途:

| 文件 | 内容 |
|------|------|
| `word/document.xml` | 正文内容（段落、文本、表格） |
| `word/styles.xml` | 样式定义（标题 1/2/3、正文等） |
| `word/header*.xml` | 页眉 |
| `word/footer*.xml` | 页脚 |
| `word/numbering.xml` | 编号/列表定义 |
| `word/_rels/document.xml.rels` | 资源关系（图片、样式引用） |
| `[Content_Types].xml` | 内容类型声明（添加新资源时需修改） |

### 3. 定位 XML

用 Read 工具读取目标 XML 文件确认内容位置:

```bash
# 读取样式定义（查看标题样式、正文样式设置）
Read "$CLAUDE_JOB_DIR/polish-work/word/styles.xml"

# 读取正文（搜索关键段落确认 XML 结构）
Read "$CLAUDE_JOB_DIR/polish-work/word/document.xml"
```

### 4. 编辑

根据操作类型选择策略。**编辑后不要急于提交**，先进入下一步（重打包）给用户确认。

#### 4a. 全局查找替换

在 `<w:t>` 文本节点中替换内容:

```bash
# 示例：全文替换 "自注意力机制" → "Self-Attention"
# 脚本写到 $CLAUDE_JOB_DIR，不让脚本打印 XML
cat > "$CLAUDE_JOB_DIR/replace.sh" << 'SCRIPT'
cd "$CLAUDE_JOB_DIR/polish-work"
OLD="自注意力机制"
NEW="Self-Attention"
# 安全的 w:t 内文本替换（只替换标签内的纯文本内容）
sed -i '' "s|<w:t>[^<]*$OLD[^<]*</w:t>|<w:t>${NEW}</w:t>|g" word/document.xml
SCRIPT
```

对于跨 `<w:r>` 的文本（被格式变化拆分的文字），需要先合并再替换。

#### 4b. 样式统一

修改 `word/styles.xml` 中的样式定义:

```
<w:style w:styleId="Heading1">      ← 标题 1 样式
  <w:name w:val="heading 1"/>
  <w:rPr>
    <w:rFonts w:eastAsia="黑体"/>    ← 中文字体
    <w:sz w:val="32"/>               ← 字号（16pt = 半磅值×2）
    <w:b/>                           ← 粗体
  </w:rPr>
</w:style>
```

常见字号对照（半磅值）:

| 半磅值 | 实际字号 |
|--------|----------|
| 42 | 小初 (21pt) |
| 36 | 二号 (18pt) |
| 32 | 三号 (16pt) |
| 28 | 小三 (14pt) |
| 24 | 四号 (12pt) |
| 21 | 小四 (10.5pt) |
| 20 | 五号 (10pt) |

常见中文论文格式约定:

- **标题 1** (章): 黑体/三号(32)
- **标题 2** (节): 黑体/四号(24)
- **标题 3** (条): 黑体/小四(21)
- **正文**: 宋体/小四(21)，Times New Roman 西文
- **行距**: 1.5 倍行距 (360)

#### 4c. 段落属性修改

修改段落间距/行距（在 `<w:pPr>` 内）:

```
<w:spacing w:line="360" w:lineRule="auto" w:before="0" w:after="0"/>
```

- `w:line="360"` = 1.5 倍行距（240 = 单倍，360 = 1.5 倍，480 = 双倍）
- `w:before` / `w:after`: 段前段后距（单位: 缇 twips，1 磅 = 20 缇）

#### 4d. 复杂修改

对于复杂操作（添加表格/图片、大幅重构），编写辅助脚本到 `$CLAUDE_JOB_DIR`:

```bash
cat > "$CLAUDE_JOB_DIR/edit.py" << 'SCRIPT'
# 处理 xml.etree.ElementTree 或 lxml
SCRIPT
python "$CLAUDE_JOB_DIR/edit.py"
```

**不打印 XML 内容到终端**（可以打印进度或统计信息）。

### 5. 重打包 & 确认

始终覆盖当前工作目录下的同一输出文件：

```bash
# 进入解包目录，打包输出到当前工作目录
ORIG_DIR=$(pwd)
cd "$CLAUDE_JOB_DIR/polish-work"
zip -r -q "$ORIG_DIR/thesis-polished.docx" .
cd "$ORIG_DIR"

# 验证
unzip -l "./thesis-polished.docx" | head -20
```

**此时暂停，让用户确认修改效果。** 用户打开 .docx 查看后反馈：
- ✅ **满意** — 用 git 提交本次修改，记录历史
- ❌ **不满意** — 回到第 4 步（编辑），重新修改，**不提交**

```bash
# 用户确认满意后执行提交
cd "$CLAUDE_JOB_DIR/polish-work"
git add -A
git commit -m "修改：统一正文字体为宋体小四"
```

确认后进入下一轮修改时，重复"编辑 → 重打包 → 确认 → 提交"循环。

### 6. 验证

- 告知用户输出文件路径
- 列出所做的修改清单（如：统一了标题字体为黑体、替换了 8 处术语、调整了行距）
- 请用户用 Word / WPS 打开确认效果
- 主动询问**"还需要继续修改吗？"**——如果需要则回到第 4 步；如果用户明确表示全部完成，再清理解包目录和临时文件

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

1. **XML 转义** — `<w:t>` 中特殊字符: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`
2. **保留 XML 声明** — `<?xml version="1.0" encoding="UTF-8" standalone="yes"?>` 必须保留
3. **命名空间属性** — 根标签的 `xmlns:w` 等命名空间声明不能丢失
4. **合并 `<w:r>`** — 同一 <w:p> 内相邻 <w:r> 如果格式相同可以合并其中 <w:t> 内容，方便全文替换
5. **中文编码** — 确保处理工具正确处理 UTF-8 编码
6. **解包目录可复用** — 同一份文档的多次修改在同一个解包目录中增量进行，不需要重新解包。**仅在用户明确表示全部修改完成后**再删除 `$CLAUDE_JOB_DIR/polish-work`
7. **Git 回滚** — 解包后默认用 git 管理修改记录，`git log --oneline` 查看历史，`git checkout <hash> -- .` 回滚
8. **备份原文件** — 修改前先备份原始 .docx，不依赖 git 时可直接重新解包恢复
