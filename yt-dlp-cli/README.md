# yt-dlp-cli

An agent skill for using [`yt-dlp`](https://github.com/yt-dlp/yt-dlp) safely and
reliably when a user asks an agent to save streaming-site media, extract audio,
pull subtitles, cut clips, or archive bounded playlists/channels. The skill
does not install or wrap `yt-dlp`; it teaches the agent how to drive the real
CLI without common failures like accidental full-playlist downloads, guessed
format IDs, missing `ffmpeg`, or leaked cookie material.

## Prerequisites

- `yt-dlp`, installed from the official upstream project and kept current.
  Verify with `yt-dlp --version` and `yt-dlp --help`.
- `ffmpeg` for merged video/audio downloads, audio extraction or conversion,
  subtitle embedding, clips, and SponsorBlock postprocessing. Verify with
  `ffmpeg -version`.
- Permission or rights to save the requested media.

## Install

Install only this skill from the repo:

```bash
npx skills add tmchow/agent-skills --skill yt-dlp-cli
```

Add `--global` to install it at the user level instead of the current project:

```bash
npx skills add tmchow/agent-skills --skill yt-dlp-cli --global
```

Hermes CLI:

```bash
hermes skills install tmchow/agent-skills/yt-dlp-cli
```

Hermes interactive slash command:

```text
/skills install tmchow/agent-skills/yt-dlp-cli
```

OpenClaw ClawHub lane:

```bash
openclaw skills install yt-dlp-cli
```

## What the skill adds

With it installed, your agent knows how to:

- Save single videos while avoiding accidental playlist downloads.
- Pick sane output paths and filename templates for later handoff.
- Inspect formats before choosing format IDs or height caps.
- Require `ffmpeg` for jobs that need postprocessing.
- Extract audio, subtitles, clips, and bounded playlist/channel archives.
- Route YouTube bot-check, cookies, PO-token, and header-sensitive failures
  through a late safety reference instead of guessing.
- Report the file path, size, media identity, and verification that ran.

SKILL.md is the agent-facing instructions; you don't need to read it to use the skill.

## Upstream

- yt-dlp: https://github.com/yt-dlp/yt-dlp
- This skill: [MIT](../LICENSE) (frontmatter license: MIT)
