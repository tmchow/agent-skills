---
name: yt-dlp
description: >-
  This skill should be used when a user wants media from a yt-dlp-supported URL
  such as YouTube, Twitter/X, TikTok, Instagram, Bilibili, Vimeo, Twitch, and
  similar sites downloaded, audio extracted, playlists or channels archived,
  subtitles pulled, clips cut, formats inspected, or yt-dlp 403, bot-check, or
  "only 360p" failures fixed. Do NOT use for editing or transcoding an existing
  local file, watching or summarizing a video without saving it, searching
  YouTube without a download, or generic file downloads that are not
  streaming-site URLs.
version: 0.1.0
license: MIT
metadata:
  hermes:
    tags: [yt-dlp, media, video, audio, subtitles, archiving]
    category: media
    requires_toolsets: [terminal]
---

# yt-dlp - agent workflow

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

## Baseline protocol

Use the installed `yt-dlp` CLI directly. Run `yt-dlp --help` or `yt-dlp -h`
before uncommon or version-sensitive syntax; live help owns flags, format
selectors, extractor arguments, and postprocessor options.

Proceed only when the user has rights or permission to save the content. Done
when the request is either within that boundary or stopped with "rights unclear"
as the blocker.

Quote every URL in shell commands. Done when every streaming URL appears as
`"<url>"` or `'<url>'`; unquoted `?`, `&`, and `#` can be eaten by the shell.

Treat a watch URL as one video unless the user explicitly asked for the
playlist. Add `--no-playlist` for single-video jobs. Done when playlist scope is
explicit in the command.

Inspect before any exotic format pick. Use `yt-dlp -F "<url>"` or
`yt-dlp --print ... "<url>"`; never invent format IDs. Done when the chosen
format ID or selector is supported by the inspection output or by live help.

Check `ffmpeg -version` before merging separate video/audio streams, extracting
or converting audio, embedding subtitles, clipping, or SponsorBlock removal.
Done when `ffmpeg` exists or the task is stopped with "ffmpeg missing".

Choose an output directory and template before downloading. Prefer `-P "<dir>"`
and `-o "%(title).200B [%(id)s].%(ext)s"` so reruns are findable and filenames
carry stable identity. Done when the command writes under the intended
directory and the final report names the exact path.

## Download conditions

For a single video at best reasonable quality, prefer the default selector and
request an mp4 merge when possible:

```bash
yt-dlp --no-playlist -P "<dir>" -o "%(title).200B [%(id)s].%(ext)s" --merge-output-format mp4 "<url>"
```

Verify with file existence, size, and video metadata. Done when the file is
non-empty and `ffprobe` or equivalent reports video streams with the expected
duration/resolution.

For a height cap such as 1080p or 720p, use a height constraint instead of a
guessed format ID:

```bash
yt-dlp --no-playlist -f "bv*[height<=1080]+ba/b[height<=1080]" --merge-output-format mp4 -P "<dir>" -o "%(title).200B [%(id)s].%(ext)s" "<url>"
```

Change only the numeric cap. Inspect with `-F` first when the site exposes
unusual formats. Done when verification reports video height at or below the
requested cap.

For audio-only output, check `ffmpeg`, then extract audio and specify the
requested format only when the user asked for one:

```bash
yt-dlp --no-playlist -x --audio-format mp3 -P "<dir>" -o "%(title).200B [%(id)s].%(ext)s" "<url>"
```

Done when the final file is audio-only and the extension/container matches the
request.

For subtitles, decide whether the user wants subtitle files, embedded subtitles,
or both:

```bash
yt-dlp --no-playlist --write-subs --write-auto-subs --sub-langs "en.*" --embed-subs -P "<dir>" -o "%(title).200B [%(id)s].%(ext)s" "<url>"
```

Omit `--write-auto-subs` when generated captions are not acceptable. Done when
the subtitle file exists or the media file contains an embedded subtitle stream,
according to the request.

For playlists or item ranges, confirm scope before any command. Use
`--playlist-items` for explicit ranges and `--download-archive` when reruns must
skip already-downloaded entries:

```bash
yt-dlp -I "1-10" --download-archive "<dir>/download-archive.txt" -P "<dir>" -o "%(playlist_title).120B/%(playlist_index)s - %(title).200B [%(id)s].%(ext)s" "<playlist-url>"
```

Do not download a full playlist or channel without a confirmed range, archive
file, or other bounded scope. Done when the downloaded count matches the agreed
scope and the archive file records completed IDs.

For a channel archive, require an archive file from the start:

```bash
yt-dlp --download-archive "<dir>/download-archive.txt" -P "<dir>" -o "%(uploader).120B/%(upload_date)s - %(title).200B [%(id)s].%(ext)s" "<channel-url>"
```

Done when rerunning the command would skip completed entries instead of
duplicating them.

For clips or time sections, verify `--download-sections` in live help, then use
one section expression and check the resulting duration:

```bash
yt-dlp --no-playlist --download-sections "*00:01:30-00:02:10" -P "<dir>" -o "%(title).200B [%(id)s] clip.%(ext)s" "<url>"
```

Done when the output duration matches the requested time span within expected
keyframe tolerance.

For SponsorBlock removal, first confirm the installed binary lists
`--sponsorblock-remove`; then name only the categories the user requested or
approved:

```bash
yt-dlp --no-playlist --sponsorblock-remove sponsor,selfpromo -P "<dir>" -o "%(title).200B [%(id)s].%(ext)s" "<url>"
```

Done when the resulting media exists and the report says which categories were
removed.

## Failure handling

When extraction fails, update `yt-dlp` before adding authentication, cookies, or
site-specific workarounds. Verify the new version with `yt-dlp --version`, then
retry a low-cost inspection (`-F` or `--print`) before downloading again.

For YouTube 403, "confirm you're not a bot", missing higher qualities, nsig
errors, PO-token paths, or header-sensitive HLS failures, read
`references/youtube-auth.md` in full before retrying. That reference is the
mandatory late route; do not inline or improvise those workarounds here.

Stop instead of pretending success when `yt-dlp` exits non-zero, `ffmpeg` is
missing for a postprocessing job, authentication is required and not approved,
the requested playlist/channel scope is too large or unclear, or rights are
unclear. Done when the blocker is named with the next safe action.

## Final report

Report the exact output path, file size, media identity (title and ID when
available), and verification performed. For video include resolution and
duration; for audio include audio-only confirmation; for playlists include item
count and archive-file use. Done when the user can find the file and a later
transcription/editing skill can consume it without re-discovering what happened.
