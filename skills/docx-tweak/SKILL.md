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

### 使用 $CLAUDE_JOB_DIR 存储临时文件

解包和编辑过程中的所有临时文件必须存放在 `$CLAUDE_JOB_DIR` 下，避免与并行任务的文件冲突。

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

创建临时工作目录并解包:

```bash
mkdir -p "$CLAUDE_JOB_DIR/polish-work"
unzip -o "/path/to/thesis.docx" -d "$CLAUDE_JOB_DIR/polish-work"
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

根据操作类型选择策略:

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

### 5. 重打包

```bash
cd "$CLAUDE_JOB_DIR/polish-work"
zip -r -q "$CLAUDE_JOB_DIR/thesis-polished.docx" .
cd -

# 验证
unzip -l "$CLAUDE_JOB_DIR/thesis-polished.docx" | head -20

# 复制到目标路径
cp "$CLAUDE_JOB_DIR/thesis-polished.docx" "/path/to/output.docx"
```

### 6. 验证

- 告知用户输出文件路径
- 列出所做的修改清单（如：统一了标题字体为黑体、替换了 8 处术语、调整了行距）
- 请用户用 Word / WPS 打开确认效果

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
6. **清理临时文件** — 完成后删除 `$CLAUDE_JOB_DIR/polish-work` 和临时输出文件
7. **备份原文件** — 修改前先备份原始 .docx
