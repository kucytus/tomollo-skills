# tomollo-skills

A collection of custom skills for Claude Code.

## Available Skills

### docx-tweak

论文微调技能 —— 解包 .docx → 编辑 XML → 重新打包。

**触发条件：** 当用户需要微调/修改 .docx 格式的论文时激活。

**流程：** 分析需求 → 解包 → 定位 XML → 编辑 → 重打包 → 验证

**适用场景：**
- 学位论文格式微调（字体、字号、行距、标题样式）
- 学术文章内容微调（统一术语、通顺语句）
- 参考文献格式整理
- 合同模板填充、报告修改

### seedpeak

种子驱动的工作流程 —— 将模糊灵感转化为可执行的目标和计划。

**触发条件：** 当用户输入以 `种子：` 或 `seed:` 开头时激活。

**流程：** 明确目标 → 制定方案 → 补充上下文 → 执行 → 整合 → 反馈 → 迭代

**适用场景：**
- 学习计划制定
- 项目规划
- 文档写作
- 论文撰写

## Installation

```bash
/plugin install tomollo-skills
```

## License

MIT
