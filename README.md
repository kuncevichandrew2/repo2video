# repo2video

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that generates AI-narrated video tutorials from GitHub repositories.

Paste a repo URL, get a 1080p MP4 with syntax-highlighted code walkthroughs, architecture diagrams, and spoken narration. **No API keys required** — Claude does the analysis and scripting, Edge TTS (free) handles narration.

## Installation

Add the skill to your Claude Code setup:

```bash
# Copy the skill file to your project or global skills directory
mkdir -p .claude/skills/repo2video
curl -o .claude/skills/repo2video/SKILL.md \
  https://raw.githubusercontent.com/kuncevichandrew2/repo2video/main/repo2video/SKILL.md
```

Or clone the repo and symlink:

```bash
git clone https://github.com/kuncevichandrew2/repo2video.git ~/repo2video-skill
ln -s ~/repo2video-skill/repo2video .claude/skills/repo2video
```

## Prerequisites

- **Node.js** >= 18
- **FFmpeg** (with libmp3lame) — `brew install ffmpeg` / `apt install ffmpeg`
- **Git**

That's it. No API keys needed.

Optional:
- `ELEVENLABS_API_KEY` — for premium TTS voices (falls back to free Edge TTS)

## Usage

In Claude Code, just ask:

```
/repo2video https://github.com/someone/cool-project
```

Or describe what you want:

```
Create a 3-minute video tutorial for https://github.com/expressjs/express
```

## How it works

Unlike typical AI video tools, **Claude itself** does the creative work:

1. **Clone** — shallow clone of the target repo
2. **Analyze** — engine detects languages, frameworks, entry points, key files
3. **Read** — Claude reads key source files to deeply understand the codebase
4. **Plan** — Claude designs the tutorial structure (architecture first, then code)
5. **Script** — Claude writes slide content, narration, and code annotations for each section
6. **Audio** — Edge TTS generates spoken narration (free, no API key)
7. **Render** — Remotion renders animated React scenes to 1080p MP4 with FFmpeg audio mux

Output: a polished MP4 video tutorial with:
- Animated intro with repo name and detected frameworks
- File tree visualization
- **Architecture slides** with animated bullet points explaining the project structure
- **Mermaid diagrams** showing component relationships and data flow
- Syntax-highlighted code walkthroughs with comment annotations
- Spoken narration synced to slides and code reveals
- Outro scene

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `ELEVENLABS_API_KEY` | No | Premium TTS (falls back to free Edge TTS) |
| `ELEVENLABS_VOICE_ID` | No | Defaults to `JBFqnCBsd6RMkjVDRZzb` |
| `EDGE_TTS_VOICE` | No | Defaults to `en-US-GuyNeural` |

## Architecture

The video rendering engine lives at [kuncevichandrew2/repotutor](https://github.com/kuncevichandrew2/repotutor). On first use, the skill clones it to `~/.repo2video` and installs dependencies. Subsequent runs reuse the cached installation.

The skill splits work between Claude and the engine:
- **Claude** — analyzes code, plans tutorial structure, writes narration scripts
- **Engine** — generates TTS audio, renders animated video with Remotion/FFmpeg

Videos are rendered using [Remotion](https://remotion.dev) (React-to-video) with a fallback to pure FFmpeg if Remotion fails.

## License

MIT
