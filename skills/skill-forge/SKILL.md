---
name: skill-forge
description: >-
  Skill authoring standards and conventions for this project. Use when writing
  or reviewing a SKILL.md to ensure it follows project conventions: directory
  structure, writing style, type classification (silent/process), frontmatter
  format, progressive disclosure patterns. Not a skill generator — a reference
  document that defines what well-written skills look like. Consult when the
  user says "write a skill", "review this skill", "what should a skill look
  like", "skill conventions", "skill format".
---

# Skill Standards

This document defines the conventions and standards for skills in this project. Reference it when writing or reviewing a skill to ensure consistency.

## Skill Types

Every skill is one of two types:

### Silent type

Activates, executes autonomously, reports the result. No user interaction during execution.

**Fits when:** the skill does one thing end-to-end without needing user input mid-way; input and output are files or structured data; the user wants the result, not a guided experience.

**Examples:** file format conversion, data extraction, code generation, batch operations.

### Process type

Activates and then guides the user through a structured workflow. Asks questions, branches on decisions, controls the interaction rhythm.

**Fits when:** the skill needs to ask questions or make decisions during execution; the workflow has branching paths based on user choices; part of the value is the interaction itself — guiding, confirming, structuring.

**Examples:** seedpeak, docx-tweak (original version), iter-mini.

## Directory Structure

```
skill-name/
├── SKILL.md           # Required — the skill instructions
├── scripts/           # Optional — reusable scripts for deterministic tasks
├── references/        # Optional — docs loaded into context on demand
└── assets/            # Optional — templates, fonts, icons used in output
```

## SKILL.md Format

### Frontmatter

Every SKILL.md starts with YAML frontmatter:

```
---
name: skill-name            # kebab-case, matches directory name
description: >-
  What the skill does AND when to use it. Claude decides whether to invoke a
  skill based on this description. Include both the action and the triggering
  context. Be slightly pushy — Claude under-triggers by default. List related
  phrasings and scenarios.
allowed-tools: "Read, Write, Edit, Bash"   # optional
---
```

### Body Structure

**Silent type body:**

```markdown
# Skill Name

[One-paragraph overview of what the skill does and why it exists]

## Flow

1. [Step 1]
2. [Step 2]
3. [Step 3]

## Input

[What the skill expects: file format, parameters, constraints]

## Output

[What the skill produces and where it's saved]

## Edge cases

[Common failure modes and how to handle them]
```

**Process type body:**

```markdown
# Skill Name

[One-paragraph overview]

## Workflow

```
Step 1 → Step 2 → Step 3 → Done
```

### Step 1: [Name]

[What the skill does at this step]
[What it asks the user]
[What happens with the user's answer]

### Step 2: [Name]

...

## End state

[What the workflow delivers when complete]

## Interaction guidelines

[How to handle off-topic users, skip requests, confirmation patterns]
```

## Writing Style

These are the most important principles. Getting them right determines whether a skill is effective or frustrating to use.

### Explain why, not just what

The model reading your skill has good theory of mind. When you explain why a step matters, it can handle edge cases on its own rather than blindly following rules. If you find yourself writing MUST or ALWAYS in all caps, stop and reframe as an explanation instead.

**Instead of:**
> MUST check file exists before reading. ALWAYS use $CLAUDE_JOB_DIR for temp files.

**Write:**
> Files may not exist at the expected path. Checking first avoids confusing error messages. Use $CLAUDE_JOB_DIR for temp files so concurrent sessions don't clash.

The model understands the reasoning and can apply it appropriately even when the specific scenario differs.

### Keep it lean

Every line costs context and constrains behavior. Remove anything that isn't pulling its weight. If deleting a paragraph doesn't change output quality, delete it.

A good test: after writing the skill, re-read it and ask yourself "what breaks if I remove this section?" If nothing, remove it.

### Generalize from examples

Use examples to understand the pattern, not to hardcode. A skill that only works on its test cases is useless. If there's a persistent issue, try explaining the concept differently rather than adding more constraints.

### Bundle repeated work

If the skill will run the same multi-step pipeline or script every time it's invoked, put that script in `scripts/` instead of asking the model to reinvent it. This saves tokens and reduces errors.

### Use examples to define format

Models understand "here's what it should look like" better than "use X format with Y fields". When the output has a specific structure, show an example.

### Imperative voice

Write "Read the input file", not "The skill should read the input file" or "You should read the input file". Direct, clear, one step at a time.

### Process type: control the rhythm

A process-type skill guides the user. This means:
- Ask one question at a time, not a list
- Let the user's answer guide the next question
- Don't proceed to the next phase until the current one is complete
- Handle off-topic answers gracefully — redirect, don't ignore

### Silent type: no ambiguity

A silent-type skill acts on its own. This means:
- Define all inputs and their formats clearly
- Handle edge cases explicitly (missing file, empty input, wrong format)
- Output location and format must be unambiguous

## Progressive Disclosure

Skills use a three-level loading system:

1. **Frontmatter** (name + description) — always in context (~100 words). Make the description count — it's the trigger.
2. **SKILL.md body** — loaded when the skill triggers. Keep under 500 lines; if approaching that limit, push details into reference files and link to them clearly.
3. **Scripts and references** — loaded on demand. Tell the model exactly when to read each reference file or run each script.

### When to use reference files

Reference files should be used for:
- Large tables or lookup data (font sizes, color codes, API schemas)
- Platform/framework-specific instructions organized by variant
- Anything that would push SKILL.md past 500 lines

For multi-variant skills, organize by domain:

```
cloud-deploy/
├── SKILL.md      # workflow + selection logic
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

The model reads only the relevant reference file.

## Description Writing Guide

The description field in frontmatter is the primary trigger mechanism. Claude decides whether to invoke a skill by reading the descriptions of all available skills.

### Rules

- **Include both action and context.** Don't just say what the skill does — say when it should be used.
- **Be slightly pushy.** Claude tends to under-trigger. List related phrasings and scenarios even if they're not exact matches.
- **Be specific.** Vague descriptions lead to weak triggering. Use concrete examples.
- **Keep under ~150 words.** Long descriptions get truncated.

### Bad descriptions

> Process .docx files.

Too vague. Claude won't know when to use this vs. handle it directly.

> A skill for tweaking thesis formatting.

Better, but still weak on the "when" dimension.

### Good descriptions

> 论文微调技能。解包 .docx → 编辑 XML → 重新打包。专注于学位论文、学术文章的格式规范和内容微调。适用于修改章节编号、调整标题样式、统一字体字号、查找替换术语、整理参考文献格式等任务。

Clear action, specific use cases, covers multiple triggering scenarios.

## Version Convention

Skills in this project track version in commit messages. No separate VERSION file. Follow semver:

- **Major** — incompatible behavioral change (e.g., process → silent)
- **Minor** — new capability, significant refactor
- **Patch** — bug fix, small adjustment

Commit format: `skill-name: 修改说明，版本升级至 X.Y.Z`

## Example: Writing a Description

A well-crafted description bridges what the skill does and when it triggers:

```
docx-tweak: 论文微调技能。解包 .docx → 编辑 XML → 重新打包。
```

vs. a better version that helps triggering:

```
docx-tweak: 论文微调技能。解包 .docx → 编辑 XML → 重新打包。
专注于学位论文、学术文章的格式规范和内容微调。
适用于修改章节编号、调整标题样式、统一字体字号、查找替换术语、整理参考文献格式等任务。
```

The second version lists concrete use cases, making it easier for Claude to recognize when this skill applies.
