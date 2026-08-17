# YouTube auth and extractor failure late route

Read this before retrying after YouTube returns 403, "confirm you're not a bot",
only low-quality formats, nsig failures, or header-sensitive HLS errors. This
route exists to prevent blind cookie use, leaked credentials, and stale
extractor workarounds.

## Order of operations

1. Update `yt-dlp` first, then confirm the version:

   ```bash
   yt-dlp -U
   yt-dlp --version
   ```

   If the install method does not support self-update, update through the
   package or binary source that installed `yt-dlp`. Done when a newer or
   already-current version is confirmed.

2. Retry inspection, not a full download:

   ```bash
   yt-dlp --no-playlist -F "<url>"
   ```

   Done when the format list proves the blocker is gone or the same auth/bot
   failure is reproduced cheaply.

3. Ask before using browser cookies. The prompt must name the site, why cookies
   are needed, and which browser/profile to read. Do not print, paste, commit,
   or log cookie contents.

4. Prefer browser-cookie access over raw cookie files when the user approves:

   ```bash
   yt-dlp --no-playlist --cookies-from-browser "<browser>" -F "<url>"
   ```

   Copy the browser/profile syntax exactly from `yt-dlp --help` for the
   installed version. Done when inspection succeeds without exposing cookie
   values.

5. Use a Netscape cookie file only when the user provides a path and approves
   that path for this download:

   ```bash
   yt-dlp --no-playlist --cookies "<cookie-file>" -F "<url>"
   ```

   Do not open the cookie file to inspect its contents. Done when the path is
   used only as an argument and no secret value appears in logs or messages.

## PO token, nsig, and plugin routes

Use PO-token or plugin-based routes only after update plus approved cookies
still fail, and only against current upstream yt-dlp guidance. Verify
`--extractor-args` in live help before passing any extractor argument:

```bash
yt-dlp --help
```

Treat PO tokens and visitor/session material as secrets. Do not place them in
commands, repo files, PR bodies, transcripts, screenshots, or reusable skill
text. If a workaround requires a token in a CLI argument, stop and ask for a
safer user-run command or a local config path supported by the current
upstream tooling.

Done when the retry path is either documented by current yt-dlp upstream
materials and does not expose secrets, or the blocker is reported as "YouTube
bot wall requires user-run auth/PO-token setup".

## Header-sensitive HLS

Use `--add-headers` only for non-secret headers that the user explicitly
provided or that yt-dlp upstream documentation names as safe. Do not pass
Authorization, Cookie, or other credential-bearing header values on the command
line; command arguments can leak through process lists, shell history, and
agent transcripts.

Done when header use is non-secret and verified by a low-cost inspection, or
the job stops with a named secret-handling blocker.
