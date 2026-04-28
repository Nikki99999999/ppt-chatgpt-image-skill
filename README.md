# ppt-chatgpt-image-skill

> AI IDE Skill: 一句话生成带高质量配图的 PPT — 串联 [ppt-master](https://github.com/hugohe3/ppt-master) + [chatgpt-image-cli](https://github.com/Nikki99999999/chatgpt-image-cli)

```
"做一个 Claude Code 教程 PPT，深色科技风" → 25 分钟后拿到可编辑的 .pptx
```

## What is this?

一个 **Claude Code / Cursor / VS Code Copilot** 的 workflow skill，把两个独立工具串联成一条自动化流水线：

1. **[ppt-master](https://github.com/hugohe3/ppt-master)**（8000+ stars）— AI 驱动的 PPT 生成，输出原生可编辑 PPTX
2. **[chatgpt-image-cli](https://github.com/Nikki99999999/chatgpt-image-cli)** — 通过浏览器自动化调用 ChatGPT Image 2.0 生图，零 API 费用

串联后的效果：你说一句话，AI 自动完成**设计规划 → 图片生成 → 逐页排版 → 导出 PPTX**，全程无需手动操作。

## Why?

| 痛点 | 解决 |
|------|------|
| ppt-master 默认用 Gemini API 生图，需要付费 API Key | 本 skill 用 ChatGPT Image 2.0，利用已有会员，零额外费用 |
| ChatGPT Image 2.0 的中文文字渲染远优于 Gemini | 中文 PPT 配图质量大幅提升 |
| 两个工具各自独立，手动串联繁琐 | 一个 skill 文件，AI 自动走完全流程 |

## Requirements

- **AI IDE**: Claude Code / Cursor / VS Code + Copilot（任一）
- **Node.js >= 22**
- **Python 3** + `pip install python-pptx`
- **Google Chrome**
- **ChatGPT Plus or Max membership**

## Install

### Step 1: Install dependencies

```bash
# ppt-master
git clone https://github.com/hugohe3/ppt-master.git
cd ppt-master && pip install -r requirements.txt

# chatgpt-image-cli
git clone https://github.com/Nikki99999999/chatgpt-image-cli.git
cd chatgpt-image-cli && npm link
```

### Step 2: Install this skill

```bash
git clone https://github.com/Nikki99999999/ppt-chatgpt-image-skill.git
```

Copy `SKILL.md` into your AI IDE's skill directory:

**Claude Code:**
```bash
cp ppt-chatgpt-image-skill/SKILL.md .claude/skills/workflow_ppt_with_chatgpt_image.md
```

**Cursor:**
```bash
cp ppt-chatgpt-image-skill/SKILL.md .cursor/rules/workflow_ppt_with_chatgpt_image.md
```

Then add a trigger in your project's `CLAUDE.md` (or equivalent):

```markdown
| 做 PPT 且要用 ChatGPT Image 2.0 配图 | `rules/skills/workflow_ppt_with_chatgpt_image.md` |
```

### Step 3: First-time setup (once)

```bash
chatgpt-image-setup
```

This launches Chrome with remote debugging and walks you through ChatGPT login. Cookies persist — you only need to do this once.

## Usage

In your AI IDE, just say:

```
做一个 Claude Code 功能介绍的 PPT，10 页，深色科技风，用 ChatGPT Image 2.0 配图
```

The AI will:
1. Auto-launch Chrome + CDP proxy (if not running)
2. Run ppt-master Strategist (design planning, asks you to confirm)
3. Generate images with ChatGPT Image 2.0 (60-120s each)
4. Run ppt-master Executor (SVG page generation)
5. Export editable `.pptx`

## How It Works

```
Your AI IDE
    │
    ├─ Phase 0: Environment Setup
    │   └─ Chrome (9222) ← CDP Proxy (3456) ← chatgpt-image-cli
    │
    ├─ Phase 1: ppt-master Strategist
    │   └─ 8 confirmations → design_spec.md + spec_lock.md
    │
    ├─ Phase 2: ChatGPT Image 2.0
    │   └─ For each image: chatgpt-image -p <prompt> -o images/<file>
    │
    ├─ Phase 3: ppt-master Executor
    │   └─ SVG pages (referencing generated images)
    │
    └─ Phase 4: Post-processing
        └─ finalize_svg → svg_to_pptx → .pptx
```

## Workflow Details

See [SKILL.md](SKILL.md) for the complete 4-phase workflow specification, including:
- Environment auto-setup commands
- Prompt writing rules for ChatGPT Image 2.0
- Failure handling strategies
- Chinese PPT garbled text fix
- Troubleshooting table

## FAQ

### Q: Do I need an OpenAI API key?

No. This skill uses your existing ChatGPT Plus/Max membership through browser automation. Zero API cost.

### Q: Can I use this without Claude Code?

Yes. The `SKILL.md` works as a workflow guide for any AI IDE that supports custom skills/rules — Cursor, VS Code + Copilot, Windsurf, etc.

### Q: What if ChatGPT Image 2.0 fails on some images?

The workflow has built-in failure handling: retry once with simplified prompt, then mark as `Needs-Manual` and continue. The PPT generation doesn't block on individual image failures.

### Q: Chinese text in generated images looks wrong

ChatGPT Image 2.0 is actually the **best** model for Chinese text rendering. Tips:
- Use Chinese prompts for Chinese content: `画一张图：标题'xxx'`
- Put Chinese text in quotes within the prompt
- Keep prompts concise

### Q: PPTX Chinese characters are garbled

This is a ppt-master rendering issue (svglib), not image generation. Fix:
```bash
python3 scripts/svg_to_pptx.py <project> -s final --only native
```

## Related Projects

- [ppt-master](https://github.com/hugohe3/ppt-master) — AI-driven natively editable PPTX generation
- [chatgpt-image-cli](https://github.com/Nikki99999999/chatgpt-image-cli) — ChatGPT Image 2.0 CLI tool

## License

MIT
