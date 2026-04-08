# repo2video

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that generates AI-narrated video tutorials from GitHub repositories.

Paste a repo URL, get a 1080p MP4 with syntax-highlighted code walkthroughs, architecture diagrams, and spoken narration.

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
- **OpenRouter API key** — get one at [openrouter.ai/keys](https://openrouter.ai/keys)

```bash
export OPENROUTER_API_KEY=your-key-here
```

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

## What it does

The skill runs a 6-stage pipeline:

1. **Clone** — shallow clone of the target repo
2. **Analyze** — detect languages, frameworks, entry points, key files
3. **Plan** — LLM generates tutorial section structure
4. **Script** — LLM writes narration + code block references per section
5. **Audio** — TTS generates spoken narration (ElevenLabs → Edge TTS → silence fallback)
6. **Render** — Remotion renders animated React scenes to 1080p MP4 with FFmpeg audio mux

Output: a polished MP4 video tutorial with:
- Animated intro with repo name and detected frameworks
- File tree visualization
- Syntax-highlighted code walkthroughs with comment annotations
- Architecture diagrams (Mermaid)
- Spoken narration synced to code reveals
- Outro scene

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `OPENROUTER_API_KEY` | Yes | LLM access via OpenRouter |
| `OPENROUTER_MODEL` | No | Defaults to `google/gemini-2.0-flash-001` |
| `ELEVENLABS_API_KEY` | No | Premium TTS (falls back to Edge TTS) |
| `ELEVENLABS_VOICE_ID` | No | Defaults to `JBFqnCBsd6RMkjVDRZzb` |
| `EDGE_TTS_VOICE` | No | Defaults to `en-US-GuyNeural` |

## How it works

The video rendering engine lives at [kuncevichandrew2/repotutor](https://github.com/kuncevichandrew2/repotutor). On first use, the skill clones it to `~/.repo2video` and installs dependencies. Subsequent runs reuse the cached installation.

Videos are rendered using [Remotion](https://remotion.dev) (React-to-video) with a fallback to pure FFmpeg if Remotion fails.

## License

MIT
