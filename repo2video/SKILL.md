---
name: repo2video
description: >
  Generate AI-narrated video tutorials from GitHub repositories.
  Clones a repo, analyzes its structure, generates tutorial scripts via LLM,
  produces TTS narration audio, and renders animated 1080p MP4 with code walkthroughs.
  Trigger: user asks to create a video tutorial from a repo, or types /repo2video.
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
---

# repo2video

Generate an AI-narrated video tutorial from any GitHub repository. The pipeline clones the repo, analyzes its codebase, uses an LLM to plan and script a tutorial, generates TTS narration, and renders an animated 1080p MP4 with syntax-highlighted code walkthroughs.

## Step 1: Parse the user's request

Extract the GitHub repository URL from the user's message. If no URL is provided, ask the user:
> What GitHub repository would you like to turn into a video tutorial?

Also check if the user specified a target video length in minutes. Default to **5 minutes** if not specified. Valid range is 1-15 minutes.

## Step 2: Check prerequisites

Run these checks in parallel. If any fail, tell the user what to install and stop.

```bash
node --version   # Must be >= 18
ffmpeg -version  # Must be installed (brew install ffmpeg / apt install ffmpeg)
git --version    # Must be installed
```

Check for the required API key:

```bash
echo "OPENROUTER_API_KEY=${OPENROUTER_API_KEY:-(not set)}"
```

If `OPENROUTER_API_KEY` is not set, ask the user:
> This skill needs an OpenRouter API key for LLM access. You can get one at https://openrouter.ai/keys
> Please set it: `export OPENROUTER_API_KEY=your-key-here`

Optional keys (inform user but don't block):
- `ELEVENLABS_API_KEY` — Premium TTS voices. Without it, free Edge TTS is used automatically.

## Step 3: Setup (one-time)

Check if the repo2video engine is already installed:

```bash
ls ~/.repo2video/cli.ts 2>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"
```

If NOT_INSTALLED, clone and install:

```bash
git clone https://github.com/kuncevichandrew2/repotutor.git ~/.repo2video
cd ~/.repo2video && npm install
```

Tell the user: "Installing repo2video engine (one-time setup). The first run will also download Chromium for video rendering (~90MB)."

If already installed, optionally pull latest:

```bash
cd ~/.repo2video && git pull --ff-only 2>/dev/null || true
```

## Step 4: Run the pipeline

Execute the CLI with the user's repo URL and target minutes:

```bash
cd ~/.repo2video && OPENROUTER_API_KEY="$OPENROUTER_API_KEY" npx tsx cli.ts <REPO_URL> --minutes <MINUTES> 2>&1
```

Replace `<REPO_URL>` with the actual GitHub URL and `<MINUTES>` with the target duration.

The pipeline runs 6 stages:
1. **Clone** — `git clone --depth 1` the target repo
2. **Analyze** — Detect languages, frameworks, entry points, key files
3. **Plan** — LLM generates tutorial section structure
4. **Script** — LLM writes narration and code block references per section
5. **Audio** — TTS generates spoken narration (ElevenLabs or Edge TTS)
6. **Render** — Remotion renders animated React scenes to 1080p MP4, FFmpeg muxes audio

Progress is printed to stderr. The final line of stdout is the path to the generated MP4.

## Step 5: Deliver the result

Once the pipeline completes:

1. Extract the video path from the last line of stdout
2. Tell the user: "Video tutorial generated! File: `<path>`"
3. On macOS, offer to open it: `open <path>`
4. Report the total time taken

## Troubleshooting

If the pipeline fails, check these common issues:

- **"Chromium not found"** — First run needs internet to download Chrome Headless Shell. Run `cd ~/.repo2video && npx remotion browser ensure` manually.
- **FFmpeg errors** — Ensure FFmpeg is installed with libmp3lame: `ffmpeg -encoders | grep mp3`
- **LLM timeout** — OpenRouter may be slow. The pipeline has 60s timeouts. Retry usually works.
- **Edge TTS failures** — WebSocket connection issues. The pipeline falls back to silence automatically.
- **"Cannot find module"** — Run `cd ~/.repo2video && npm install` to reinstall dependencies.
- **Remotion render fails** — Falls back to FFmpeg-only rendering automatically. Video will be simpler but functional.
