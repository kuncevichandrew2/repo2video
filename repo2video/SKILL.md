---
name: repo2video
description: >
  Generate AI-narrated video tutorials from GitHub repositories.
  You (Claude) analyze the repo, plan the tutorial, and write narration scripts with slides.
  The rendering engine handles TTS and video. No API keys required.
  Trigger: user asks to create a video tutorial from a repo, or types /repo2video.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
  - AskUserQuestion
---

# repo2video

Generate an AI-narrated video tutorial from any GitHub repository. **You** (Claude) do the creative work — analyzing code, planning the tutorial structure, writing slide content, and scripting narration. The rendering engine handles TTS audio and video rendering. No external API keys are required.

**Key principle: Architecture first, code second.** Every video should start by explaining *what* the project does and *how* it's structured before diving into any code. Use slides and diagrams to build understanding progressively.

## Step 1: Parse the user's request

Extract the GitHub repository URL from the user's message. If no URL is provided, ask the user:
> What GitHub repository would you like to turn into a video tutorial?

Also check if the user specified a target video length in minutes. Default to **5 minutes** if not specified. Valid range is 1-15 minutes.

## Step 2: Check prerequisites

Run these checks. If any fail, tell the user what to install and stop.

```bash
node --version   # Must be >= 18
ffmpeg -version  # Must be installed (brew install ffmpeg / apt install ffmpeg)
git --version    # Must be installed
```

No API keys are needed. TTS uses free Edge TTS (no key required).

Optional (inform user but don't block):
- `ELEVENLABS_API_KEY` — Premium TTS voices. Without it, free Edge TTS is used automatically.

## Step 3: Setup the rendering engine (one-time)

Check if the repo2video engine is already installed:

```bash
ls ~/.repo2video/render-cli.ts 2>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"
```

If NOT_INSTALLED, clone and install:

```bash
git clone https://github.com/kuncevichandrew2/repotutor.git ~/.repo2video
cd ~/.repo2video && npm install
```

Tell the user: "Installing repo2video rendering engine (one-time setup). The first run will also download Chromium for video rendering (~90MB)."

If already installed, optionally pull latest:

```bash
cd ~/.repo2video && git pull --ff-only 2>/dev/null || true
```

## Step 4: Clone and analyze the target repository

Clone the target repo and run the mechanical analyzer:

```bash
REPO_DIR=$(mktemp -d)/repo
git clone --depth 1 <REPO_URL> "$REPO_DIR"
cd ~/.repo2video && npx tsx analyze-cli.ts "$REPO_DIR" > /tmp/repo2video-analysis.json 2>&1
```

Read the analysis output to understand the codebase:

```bash
cat /tmp/repo2video-analysis.json
```

The analysis JSON contains: `repoName`, `description`, `primaryLanguage`, `languages`, `frameworks`, `entryPoints`, `fileTree`, `keyFiles` (with content), `dependencies`, `totalFiles`, `totalLines`.

## Step 5: Deep-read the codebase

Based on the analysis, read the most important source files to deeply understand the codebase. Use the Read tool to examine:
- Entry points identified in the analysis
- Core modules and key architectural files
- Configuration files (package.json, Cargo.toml, pyproject.toml, etc.)
- README.md for project context

Read at least 5-10 key files. You need to understand the code well enough to:
1. Explain the high-level architecture with diagrams
2. Describe each component's role and how they connect
3. Walk through the most important code paths

## Step 6: Plan the tutorial (architecture-first)

Design the tutorial structure. The video MUST follow this pattern:

1. **Overview slide** — What the project is, who it's for, what problem it solves
2. **Architecture slide(s)** — How the project is structured, component relationships, data flow (with Mermaid diagram)
3. **Code walkthroughs** — Deep dive into key files, in dependency order
4. **Summary slide** — How everything connects, how to get started

Calculate sections:
- `totalSections = max(3, min(10, ceil(targetMinutes * 1.5)))`
- Each section gets `targetDurationSec = round((targetMinutes * 60) / totalSections)`

Write the plan as JSON to `/tmp/repo2video-plan.json`:

```json
{
  "title": "Tutorial title — e.g. 'Deep Dive into ProjectName'",
  "sections": [
    {
      "id": "unique-section-id",
      "title": "Section title for chapter marker",
      "type": "overview | architecture | code_walkthrough | summary",
      "targetDurationSec": 40,
      "filePaths": ["src/index.ts", "src/core.ts"],
      "keyPoints": ["What this section teaches"]
    }
  ]
}
```

Rules:
1. At least the first 2 sections MUST be `overview` or `architecture` type (slides, not code)
2. Walk through code in dependency order — foundations before features
3. ONLY reference file paths that exist in the analysis `keyFiles`
4. End with a `summary` section

## Step 7: Write narration scripts with slides

For each section, write a script. **Slide sections** (overview, architecture, summary) get a `slides` object with bullets and description. **Code sections** get code blocks with comment hints.

Write ALL scripts as a JSON array to `/tmp/repo2video-scripts.json`:

### Slide section example (overview, architecture, summary):

```json
{
  "sectionId": "overview",
  "narration": "Full narration text spoken during this slide...",
  "slides": {
    "description": "Optional subtitle or context line shown below the title",
    "bullets": [
      "First key point about the project",
      "Second point — use `backticks` for code terms",
      "Third point explaining a concept"
    ]
  },
  "codeBlocks": [],
  "estimatedDurationSec": 40
}
```

### Architecture section example (with diagram + optional slide):

```json
{
  "sectionId": "architecture",
  "narration": "Narration explaining the architecture...",
  "slides": {
    "description": "How the components fit together",
    "bullets": [
      "`ComponentA` handles user requests and routing",
      "`ComponentB` manages data persistence",
      "Events flow from A → B → C via message queue"
    ]
  },
  "mermaidDiagram": "graph TD\n  A[Client] -->|HTTP| B[API Server]\n  B --> C[Database]\n  B --> D[Cache]",
  "codeBlocks": [],
  "estimatedDurationSec": 40
}
```

### Code walkthrough example:

```json
{
  "sectionId": "auth-module",
  "narration": "Narration explaining the code...",
  "codeBlocks": [
    {
      "filePath": "src/auth.ts",
      "startLine": 1,
      "endLine": 45,
      "focusStart": 10,
      "commentHints": [
        { "insertAfterLine": 5, "comment": "`createApp` initializes the Express server" },
        { "insertAfterLine": 12, "comment": "Middleware chain: auth → rate-limit → handler" }
      ],
      "revealOrder": 1
    }
  ],
  "estimatedDurationSec": 40
}
```

### Rules for slides:
1. **3-6 bullets per slide** — concise, scannable points
2. Use `backticks` around code terms (function names, variables, file names) — they render in accent color
3. **Description** is optional — use for a one-line context setter below the title
4. Bullets animate in one by one, so structure them as a progressive reveal of understanding
5. Keep bullets under 80 characters each

### Rules for code blocks:
1. Use generous line ranges (30-60 lines) for context
2. Set `focusStart` to the most interesting line
3. **Comment hints**: 6-12 per block, every 3-5 lines — these create pause points in the typing animation
4. Wrap variable/function names in backticks in comment hints
5. Narration must follow code order (top-to-bottom)

### General rules:
1. **Narration tone**: Conversational but technical, like a senior engineer explaining to a new team member
2. **Explain WHY** code is structured this way, not just what each line does
3. **Word count**: ~2.5 words per second (e.g., 40 sec = ~100 words)
4. **Mermaid diagrams**: REQUIRED for architecture sections. `graph TD` or `graph LR`, 5-10 nodes max. Just diagram syntax, no markdown fences
5. **Transitions**: Each section should flow naturally from the previous one

## Step 8: Render the video

Run the rendering engine with the generated JSON files:

```bash
cd ~/.repo2video && npx tsx render-cli.ts \
  --analysis /tmp/repo2video-analysis.json \
  --plan /tmp/repo2video-plan.json \
  --scripts /tmp/repo2video-scripts.json \
  --repo "$REPO_DIR" \
  2>&1
```

The `--repo` flag tells the renderer where to find source files for code block extraction. The renderer will:
1. Read code from the repo based on your startLine/endLine references
2. Inject your comment hints as visible annotations
3. Generate TTS audio from your narration (Edge TTS, free)
4. Render slides with animated bullet reveals
5. Render Mermaid diagrams for architecture sections
6. Render animated 1080p MP4 with Remotion

The final line of stdout is the path to the generated MP4.

## Step 9: Deliver the result

Once the pipeline completes:

1. Extract the video path from the last line of output
2. Tell the user: "Video tutorial generated! File: `<path>`"
3. On macOS, offer to open it: `open <path>`
4. Report the total time taken

## Step 10: Clean up

Remove the cloned repo:

```bash
rm -rf "$REPO_DIR"
```

## Troubleshooting

If the pipeline fails, check these common issues:

- **"Chromium not found"** — First run needs internet to download Chrome Headless Shell. Run `cd ~/.repo2video && npx remotion browser ensure` manually.
- **FFmpeg errors** — Ensure FFmpeg is installed with libmp3lame: `ffmpeg -encoders | grep mp3`
- **Edge TTS failures** — WebSocket connection issues. The pipeline falls back to silence automatically.
- **"Cannot find module"** — Run `cd ~/.repo2video && npm install` to reinstall dependencies.
- **Remotion render fails** — Falls back to FFmpeg-only rendering automatically. Video will be simpler but functional.
