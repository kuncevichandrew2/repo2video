---
name: repo2video
description: >
  Generate AI-narrated video tutorials from GitHub repositories.
  You (Claude) analyze the repo, plan the tutorial, and write narration scripts.
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

Generate an AI-narrated video tutorial from any GitHub repository. **You** (Claude) do the creative work — analyzing code, planning the tutorial structure, and writing narration scripts. The rendering engine handles TTS audio and video rendering. No external API keys are required.

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

## Step 5: Read key files from the cloned repo

Based on the analysis, read the most important source files to deeply understand the codebase. Use the Read tool to examine:
- Entry points identified in the analysis
- Core modules and key architectural files
- Configuration files (package.json, Cargo.toml, etc.)
- README.md for project context

Read at least 5-10 key files. You need to understand the code well enough to explain it in a tutorial.

## Step 6: Plan the tutorial

Based on your analysis, create a tutorial plan. Calculate the number of sections:
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

Rules for planning:
1. Start with a high-level **overview** section (what the project does, who it's for)
2. Second section should be **architecture** (project structure, how components connect)
3. Walk through code in dependency order — foundations before features
4. Highlight the most important files (up to 8)
5. End with a **summary** section (how everything connects, how to contribute)
6. ONLY reference file paths that exist in the analysis `keyFiles`
7. Use section types: `overview`, `architecture`, `code_walkthrough`, `summary`

## Step 7: Write narration scripts

For each section in the plan, write a narration script. Write ALL scripts as a JSON array to `/tmp/repo2video-scripts.json`:

```json
[
  {
    "sectionId": "matches section.id from plan",
    "narration": "Full narration text, conversational but technical...",
    "codeBlocks": [
      {
        "filePath": "src/index.ts",
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
    "mermaidDiagram": "graph TD\n  A[Client] -->|HTTP| B[Server]\n  B --> C[DB]",
    "estimatedDurationSec": 40
  }
]
```

Rules for scripting:
1. **Narration tone**: Conversational but technical, like a senior engineer explaining to a new team member
2. **Explain WHY** the code is structured this way, not just what each line does
3. **Code blocks**: Use generous line ranges (30-60 lines) for context. Set `focusStart` to the most interesting line
4. **Comment hints**: 6-12 per code block, every 3-5 lines. These create PAUSE POINTS in the video where the narrator explains what just appeared. Write them as short teaching comments (max 80 chars). Wrap variable/function names in backticks
5. **Narration must follow code order**: The spoken narration should match the top-to-bottom reveal of code and comment hints
6. **Word count**: Target ~2.5 words per second of narration (e.g., 40 sec section = ~100 words)
7. **Mermaid diagram**: REQUIRED for `architecture` sections. Use `graph TD` or `graph LR`, 5-10 nodes max. Just the diagram syntax, no markdown fences
8. **Overview section**: Start with a warm welcome and project overview
9. **Summary section**: End with how everything connects and encouragement to explore
10. **Transitions**: Each section should transition naturally from the previous one

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

The `--repo` flag tells the renderer where to find the actual source files for code block extraction. The renderer will:
1. Read code from the repo based on your startLine/endLine references
2. Inject your comment hints as visible annotations
3. Generate TTS audio from your narration text (Edge TTS, free)
4. Render animated 1080p MP4 with Remotion (syntax-highlighted code, file trees, architecture diagrams)

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
