---
name: yt-dlp-cli
description: >-
  yt-dlp. Use when the user wants to download, extract audio, pull subtitles,
  clip, or archive a YouTube or other streaming-site URL, or when yt-dlp hits
  403, bot-check, or only-360p. Not for local-file edit/transcode,
  watch/summarize without saving, search, or generic downloads.
version: 0.1.2
license: MIT
metadata:
  hermes:
    tags: [yt-dlp, media, video, audio, subtitles, archiving]
    category: media
    requires_toolsets: [terminal]
---

# yt-dlp-cli - agent workflow

**Result:** After a user asks to download, extract, clip, subtitle, or archive
media from a yt-dlp-supported URL, the agent has either produced the correct
file(s) on disk and reported path/size/identity, or stopped on an explicit
blocker (missing binary, auth required, rights unclear, playlist too large,
YouTube bot wall).

**Next consumer:** The user (file on disk) and any later skill that
transcribes/edits that file.

**Done:** File exists at the stated path, matches the requested job (video vs
audio vs clip vs playlist range), and the agent reported what it did and how it
verified. Or a named blocker with the next safe action.

**Intent (changes the approach):** Agents can already invoke `yt-dlp <url>` and
they do it badly. This skill exists to close those failure modes, not to teach
the CLI. Do not write a flag encyclopedia. Do not hide yt-dlp behind a Python
wrapper.

## Autonomy envelope

Invoking this skill authorizes local inspection, downloading the requested
streaming-site URL, writing the resulting file(s) under the chosen destination,
and running required local `ffmpeg` postprocessing for the requested media
shape. It does not authorize downloading an unbounded playlist/channel, reading
browser cookies or auth files, bypassing unclear rights, or supplying user-only
inputs. Stop for confirmation only when rights are unclear, scope expands to a
full playlist/channel without an agreed range/archive, cookies/auth are needed,
or the next input can only come from the user.

Quote every URL in shell commands. Treat a watch URL as one video unless the
user explicitly asked for the playlist; keep `--no-playlist` on single-video
jobs. Check `ffmpeg -version` before merged video/audio, audio extraction or
conversion, subtitle embedding, clips, or SponsorBlock postprocessing; missing
`ffmpeg` is a blocker for those shapes.

## Default download

For a single requested video, run this command first:

```bash
yt-dlp --no-playlist -P "<dir>" -o "%(title).200B [%(id)s].%(ext)s" --merge-output-format mp4 --print after_move:filepath "<url>"
```

Use the path printed by `--print after_move:filepath` as the authoritative
handoff path. Verify that file exists and is non-empty; for video, verify
duration and resolution with `ffprobe` or equivalent.

## Deltas from the default

- Height cap: add `-f "bv*[height<=1080]+ba/b[height<=1080]"` and change only
  the numeric cap. Use `yt-dlp --no-playlist -F "<url>"` first only when the
  user requested a specific format ID or the first run returns the wrong shape;
  never invent format IDs.
- Audio-only: replace `--merge-output-format mp4` with `-x`; add
  `--audio-format <format>` only when the user requested a container/codec such
  as mp3 or m4a.
- Subtitles: for subtitle-only jobs, replace media download/postprocessing with
  `--skip-download --write-subs --sub-langs "<langs>"`. For video plus
  subtitles, keep the default download and add
  `--write-subs --sub-langs "<langs>"`. Add `--write-auto-subs` only when generated captions are
  acceptable, and `--embed-subs` only when embedded subtitles are requested.
- Output layout: change only `-P "<dir>"` and `-o "<template>"`. Keep `%(id)s`
  in the template unless the user requested a different stable naming scheme.
- SponsorBlock: add `--sponsorblock-remove <categories>` only for categories
  the user requested or approved.

## Pinned fragile routes

For playlist or channel archive work, first confirm the range or archive scope.
Then use an archive file so reruns skip completed entries:

```bash
yt-dlp -I "1-10" --download-archive "<dir>/download-archive.txt" -P "<dir>" -o "%(playlist_title).120B/%(playlist_index)s - %(title).200B [%(id)s].%(ext)s" --print after_move:filepath "<playlist-or-channel-url>"
```

Keep `-I` for item ranges; remove it only after the user confirmed a full
archive. The archive file is part of the handoff because it prevents duplicate
downloads on rerun.

For clips or time sections, use `--download-sections` and verify the resulting
duration against the requested span:

```bash
yt-dlp --no-playlist --download-sections "*00:01:30-00:02:10" -P "<dir>" -o "%(title).200B [%(id)s] clip.%(ext)s" --merge-output-format mp4 --print after_move:filepath "<url>"
```

Change only the timestamps unless the user requested a different section.

## Ordered hatch

Run the pinned default or fragile route first. If it exits non-zero, returns the
wrong media shape, reports a bot/auth/version mismatch, or shows only 360p where
higher formats should exist, take this order:

1. Preserve the command, exit status, and stderr.
2. Update `yt-dlp`, verify `yt-dlp --version`, then retry a low-cost inspection
   with `yt-dlp --no-playlist -F "<url>"` or
   `yt-dlp --no-playlist --print ... "<url>"`.
3. For wrong shape or site-specific syntax drift, consult live `yt-dlp --help`
   / `yt-dlp -h` and adjust the smallest delta needed.
4. For YouTube 403, "confirm you're not a bot", missing higher qualities, nsig
   errors, PO-token paths, or header-sensitive HLS failures, read
   `references/youtube-auth.md` in full before retrying.
5. Stop with a blocker when rights, scope, auth, user-only input, or a missing
   binary prevents safe completion.

## Report

The report may omit secondary detail, but it must preserve command, exit
status, output path and size, media identity when available, verification
performed, and stderr or blocker. For video include duration/resolution; for
audio include audio-only confirmation; for playlists/channels include item count
and archive-file path.
