# Skill: PPT + ChatGPT Image 2.0 One-Stop Workflow

> 将 **ppt-master**（PPT 生成）和 **chatgpt-image-cli**（ChatGPT Image 2.0 生图）串联为端到端流程。
> 一句话输入主题，自动完成：设计 → AI 生图 → 导出 PPTX。

## When to Use

- 需要用 ChatGPT Image 2.0 做高质量配图的 PPT
- 需要中文文字渲染精准的配图（GPT-Image-2 的强项）
- 用户说"做 PPT"、"生成演示文稿"并希望用 ChatGPT 生图

## Prerequisites

| 依赖 | 说明 | 安装 |
|------|------|------|
| **ppt-master** | PPT 生成 skill | `git clone https://github.com/hugohe3/ppt-master.git` |
| **chatgpt-image-cli** | ChatGPT Image 2.0 生图 | `git clone https://github.com/Nikki99999999/chatgpt-image-cli.git && cd chatgpt-image-cli && npm link` |
| **Node.js >= 22** | chatgpt-image-cli 依赖 | https://nodejs.org/ |
| **Python 3 + python-pptx** | ppt-master 依赖 | `pip install python-pptx` |
| **Google Chrome** | 浏览器自动化目标 | https://www.google.com/chrome/ |
| **ChatGPT Plus/Max** | 图片生成需要会员 | https://chatgpt.com/ |

## Overview

```
用户输入主题 / 素材
    │
    ▼
Phase 0: 环境准备（Chrome + CDP Proxy + ChatGPT 登录检查 + Smoke Test）
    │
    ▼
Phase 1: ppt-master Strategist（八项确认 → design_spec + spec_lock）
    │
    ▼
Phase 2: ChatGPT Image 2.0 批量生图（逐张生成，快速失败，自动重试）
    │
    ▼
Phase 3: ppt-master Executor（逐页生成 SVG，引用 Phase 2 的图片）
    │
    ▼
Phase 4: 后处理 + 导出 PPTX
    │
    ▼
输出: 可编辑的 .pptx 文件
```

---

## Phase 0: Environment Setup

> Phase 0 全程自动，不需要用户操作（首次除外需登录一次 ChatGPT）。

### 0.1 Launch Chrome with Remote Debugging

```bash
# macOS
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/tmp/chatgpt-image-cli-profile" &

# Windows
start "" "C:\Program Files\Google\Chrome\Application\chrome.exe" \
  --remote-debugging-port=9222 \
  --user-data-dir="%LOCALAPPDATA%\Google\Chrome\User Data\ChromeDebug"

# Linux
google-chrome --remote-debugging-port=9222 \
  --user-data-dir="$HOME/tmp/chatgpt-image-cli-profile" &
```

Or use the built-in setup command:
```bash
chatgpt-image-setup
```

### 0.2 Verify Environment

```bash
# Chrome debug port alive?
curl -sf http://127.0.0.1:9222/json/version > /dev/null && echo "OK: Chrome" || echo "FAIL: Chrome"

# CDP proxy alive?
curl -sf http://127.0.0.1:3456/targets > /dev/null && echo "OK: Proxy" || echo "FAIL: Proxy"
```

### 0.3 Login Check + Keepalive Tab

```bash
# Create keepalive tab (prevents Chrome auto-exit on Windows)
curl -s "http://127.0.0.1:3456/new?url=about:blank" > /dev/null

# Open ChatGPT and verify login
CHATGPT_TID=$(curl -s "http://127.0.0.1:3456/new?url=https://chatgpt.com/" | \
  python3 -c "import sys,json;print(json.load(sys.stdin).get('targetId',''))")
sleep 4

LOGIN=$(curl -s -X POST -H "Content-Type: text/plain" \
  --data 'JSON.stringify({editor:!!document.getElementById("prompt-textarea"),loginBtn:document.querySelectorAll("[data-testid=login-button]").length})' \
  "http://127.0.0.1:3456/eval?target=$CHATGPT_TID")

echo "$LOGIN" | grep -q '"loginBtn":0' && echo "OK: Logged in" || echo "WARN: Not logged in - please login in Chrome window"
```

### 0.4 Smoke Test (Required Before Batch)

```bash
chatgpt-image -p "Generate an image: a red circle on white background" -o /tmp/smoke_test.png
```

Pass criteria: file generated within 120s and size > 10KB.

---

## Phase 1: ppt-master Strategist

Follow ppt-master's SKILL.md Steps 1-4:

1. **Source Processing** — convert user's PDF/URL/text to Markdown
2. **Project Init** — `python3 scripts/project_manager.py init <name> --format ppt169`
3. **Template Option** — default: free design
4. **Strategist Eight Confirmations** (BLOCKING — wait for user)

### Key Modification for This Workflow

In the 8th confirmation item **Image usage approach**, specify:

> Image generation: **ChatGPT Image 2.0** via chatgpt-image-cli (browser automation)
> Estimated: 60-120s per image, 5s interval between batch calls

Output: `design_spec.md` + `spec_lock.md`

---

## Phase 2: ChatGPT Image 2.0 Batch Generation

### 2.1 Extract Pending Images

From `design_spec.md`, collect all images with status `Pending`.

### 2.2 Create Prompt Document

Write to `<project_path>/images/image_prompts.md`:

```markdown
| # | Filename | Prompt | Aspect | Status |
|---|----------|--------|--------|--------|
| 1 | hero_cover.jpg | Generate an image (16:9): ... | 16:9 | Pending |
| 2 | bg_tech.jpg | Generate an image (16:9): ... | 16:9 | Pending |
```

### 2.3 Prompt Rules

- **Must start with a generation verb**: `Generate an image`, `Create an illustration`, `画一张图`
- **Must include aspect ratio**: `(16:9)` or `(1:1)` or `(9:16)`
- **Chinese content uses Chinese prompt**: `画一张16:9封面图：标题'xxx'，金色字体`
- **Be specific**: describe composition, color palette, elements — avoid vague abstractions

### 2.4 Generate One by One

```bash
chatgpt-image -p "<prompt>" -o <project_path>/images/<filename> --proxy http://127.0.0.1:3456
sleep 5  # pacing between images
```

### 2.5 Failure Handling

- Single failure → simplify prompt, retry once
- Still fails → mark `Needs-Manual`, continue to next
- **Never block the pipeline** for a single image failure
- Update `image_prompts.md` status for each row

### 2.6 Phase Complete Checklist

```markdown
## Phase 2 Complete
- [x] Prompt document created
- [x] Each image: Generated or Needs-Manual
- [x] No Pending rows remaining
```

---

## Phase 3: ppt-master Executor

Follow ppt-master's SKILL.md Step 6:

1. Read `executor-base.md` + style-specific reference
2. Confirm design parameters
3. **Generate SVG pages sequentially** (main agent only, no sub-agents)
4. Re-read `spec_lock.md` before each page
5. Run `svg_quality_checker.py`
6. Generate speaker notes → `notes/total.md`

**Key difference from standalone ppt-master**: images are already generated in Phase 2 under `images/`. Executor references them directly — no need to call `image_gen.py`.

---

## Phase 4: Post-processing & Export

Follow ppt-master's SKILL.md Step 7, **strictly sequential**:

```bash
# 7.1 Split speaker notes
python3 scripts/total_md_split.py <project_path>

# 7.2 SVG post-processing
python3 scripts/finalize_svg.py <project_path>

# 7.3 Export PPTX
python3 scripts/svg_to_pptx.py <project_path> -s final
```

### Chinese PPT Fix

If Chinese characters appear garbled in the exported PPTX:
```bash
python3 scripts/svg_to_pptx.py <project_path> -s final --only native
```

---

## Time Estimate

| Phase | Duration |
|-------|----------|
| Phase 0: Environment | 1-2 min |
| Phase 1: Strategist | 5-10 min (includes user confirmation) |
| Phase 2: Image gen (5 images) | 5-12 min |
| Phase 3: Executor (10 pages) | 10-20 min |
| Phase 4: Export | 1-2 min |
| **Total** | **~25-45 min** |

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| All image generation fails | Chrome/CDP/login issue | Re-run Phase 0 checks step by step |
| Image size doesn't match SVG slot | Prompt missing aspect ratio | Add `(16:9)` to prompt |
| Chinese garbled in PPTX | svglib PNG renderer | Export with `--only native` |
| Repeated "Stopped thinking" no image | ChatGPT rate limit or content filter | Shorten prompt, or mark Needs-Manual |
| Image path error in SVG | Filename mismatch | Ensure spec_lock filenames match actual files in images/ |
| CDP proxy timeout | DevToolsActivePort stale cache | Restart Chrome + proxy |
| Chrome exits after tab close | Windows: last tab closed | Ensure keepalive tab exists |

---

## Comparison

| Dimension | ppt-master alone | chatgpt-image-cli alone | This integrated skill |
|-----------|-----------------|------------------------|----------------------|
| Image backend | Gemini API / generic | ChatGPT Image 2.0 | ChatGPT Image 2.0 |
| Environment | No Chrome needed | Chrome required | Auto-launches Chrome |
| Pipeline | Manual role switching | No PPT involved | Fully automated |
| Chinese text in images | Depends on backend | Accurate rendering | Accurate rendering |
| Best for | Quick PPT | Standalone image gen | High-quality illustrated PPT |
