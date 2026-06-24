---
name: vhs-terminal-recorder
description: >-
  This skill should be used when the user names Charmbracelet VHS, asks to use the `vhs` CLI, wants a terminal recording, terminal GIF, terminal MP4/WebM, `.tape` file, scripted terminal demo, or a reproducible command-line screencast. Do NOT use for generic browser recording, full desktop screen capture, or unscripted secret-bearing interactive sessions.
version: 0.1.0
author: Trevin Chow
license: MIT
metadata:
  hermes:
    tags: [terminal, recording, vhs, charmbracelet, screencast, gif, mp4, webm, cli]
    category: media
    requires_toolsets: [terminal]
  openclaw:
    emoji: "📼"
    homepage: https://github.com/charmbracelet/vhs
    requires:
      bins: [vhs, ttyd, ffmpeg]
    install:
      - id: homebrew
        kind: brew
        formula: vhs
        bins: [vhs, ttyd, ffmpeg]
        label: Install VHS with Homebrew
---

# VHS Terminal Recorder

## When to use

Use this skill only when `vhs` or Charmbracelet VHS is actually the right surface:

- user says `vhs`, Charmbracelet VHS, `.tape`, terminal GIF, terminal recording, command-line screencast, or scripted terminal demo
- docs need a reproducible recording of CLI behavior
- an existing VHS tape needs authoring, validation, rendering, or debugging

Do **not** use it for arbitrary desktop capture, browser product tours, video editing, or one-off interactive work that cannot be made deterministic. Use a video/screen-capture workflow instead when the subject is not a terminal.

## Mental model

VHS runs a tape file in a pseudo-terminal and renders the terminal session to GIF, MP4, or WebM. Treat a tape like a small executable test:

- the tape must create or point at deterministic state
- commands shown to viewers should be short and intentional
- setup belongs behind `Hide` / `Show`
- `vhs validate` checks tape syntax but does not prove the commands work
- `vhs <file>` executes the tape and can mutate files, hit the network, or run project commands

This skill is not the VHS language reference. For exact syntax, generate or inspect live docs:

```bash
vhs --help
vhs manual
vhs new /tmp/example.tape
vhs validate /tmp/example.tape
```

## Setup and verification

VHS is the Charmbracelet CLI from https://github.com/charmbracelet/vhs. Before depending on it, verify the local binary and help:

```bash
vhs --version
vhs --help
```

Use command-specific help for current flags:

```bash
vhs record --help
vhs validate --help
vhs publish --help
```

Rendering also depends on the local VHS render stack, especially `ttyd` and `ffmpeg`. If `vhs validate` passes but rendering fails before running the tape, check those binaries before changing tape logic.

If `vhs` is missing and the user wants it installed, prefer the upstream-supported package manager for the current machine. The common macOS/Linux path is:

```bash
brew install vhs
```

For Windows, Arch, Nix, Docker, Debian/RPM packages, or Go source installs, use the upstream installation section instead of copying commands from memory: https://github.com/charmbracelet/vhs#installation

First render can be slow because VHS may initialize browser/rendering caches. Do not misdiagnose that as a bad tape until validation passes and the render actually fails.

## Authoring rules

Start from a hand-written tape for agent-created demos. Use `vhs record` only as a rough draft, then clean the tape before committing it.

Keep tapes deterministic:

- keep `Output`, `Require`, and most `Set` commands at the top of the tape before visible actions; VHS docs call out top-of-file ordering for dependencies and settings
- set `Output`, `Set Shell`, `Set Width`, `Set Height`, `Set FontSize`, `Set TypingSpeed`, and `Set Theme` explicitly
- add `Require <program>` for every external command the tape depends on
- create fixtures or temp files in hidden setup instead of relying on the user's current shell state
- prefer fake/sample values over real accounts, tokens, private paths, or production URLs
- constrain noisy commands with flags, known fixtures, or small output windows
- keep `Type` lines simple; put complex setup in a script or separate hidden commands instead of relying on nested quotes or command substitution
- use `Wait+Screen@<timeout> /pattern/` or `Wait@<timeout> /prompt-or-line/` for readiness; reserve `Sleep` for presentation timing after readiness has happened
- when waiting on a server, file watcher, or network dependency, prefer a hidden shell readiness loop or a visible command followed by a VHS `Wait+Screen`; do not rely on a bare fixed sleep

Minimal skeleton:

```text
Output demo.gif

Require bash
Require your-cli

Set Shell "bash"
Set Width 1200
Set Height 700
Set FontSize 28
Set Theme "Builtin Dark"

Hide
Type "mkdir -p /tmp/vhs-demo && cd /tmp/vhs-demo"
Enter
Type "your-cli init --fixture small >/dev/null"
Enter
Show

Type "your-cli status"
Enter
Wait+Screen@5s /ready|ok|complete/
Sleep 1s
```

## Best-practice scenarios

**Create a docs GIF.** Author a tape beside the docs or examples it supports, use a small deterministic fixture, validate it, render it, then inspect the output file before reporting success:

```bash
vhs validate docs/demo.tape
vhs docs/demo.tape
```

**Render another format.** Prefer GIF for lightweight README embeds. Use MP4 or WebM when quality or file size matters:

```text
Output demo.mp4
Output demo.webm
```

Use `Output frames/` for a PNG sequence when post-processing is needed. Use text outputs such as `Output golden.ascii` only for integration-test/golden-file workflows, not as the normal docs artifact.

**Choose a theme.** Do not guess theme names from memory. List them and copy the theme string exactly:

```bash
vhs themes
vhs themes --markdown
```

**Validate several tapes.** Quote globs so the shell does not expand or reject them before VHS sees the pattern, especially in `zsh`:

```bash
vhs validate 'docs/*.tape'
```

**Debug a failing tape.** Validate first. If validation passes, run each viewer-facing command directly in a clean shell or temp directory. Most VHS failures are ordinary command failures hidden inside a recording.

**Use recording mode.** Reach for `vhs record` only to capture an exploratory interaction or learn a rough sequence. Rewrite the result into a short scripted tape with hidden setup, explicit dimensions, and no accidental local state.

## Safety and publishing

Rendering executes the tape. Read untrusted tapes before running them.

Never capture or publish secrets. Before rendering or committing, scan the tape for tokens, real customer data, private paths, `.env` output, shell history, git remotes, and account-specific prompts.

Publishing uploads a GIF to `vhs.charm.sh`. Treat it as an external side effect: only run `vhs publish <file>.gif` or `vhs <file> --publish` when the user has explicitly approved that exact file for public sharing.

## Pitfalls

- `vhs validate` is parse-only; it does not run `Require` checks or prove the demo commands succeed.
- Unquoted globs can fail in the shell before VHS runs.
- Long command output makes unreadable recordings. Shrink the scenario instead of increasing terminal height until everything fits.
- Nested quoting and shell command substitution inside `Type` lines can produce confusing parse errors. Simplify the line or move setup into a small script.
- Plain `Wait /pattern/` watches the current line. Use `Wait+Screen /pattern/` when matching command output anywhere on the terminal.
- VHS supports useful commands beyond the common demo path, including `Screenshot`, `Source`, `Copy`, `Paste`, and version-dependent `Env`; check `vhs manual` before using them, and never use `Env` as a way to smuggle real secrets into recordings.
- Default shell, prompt, theme, size, and font can vary by machine. Set the recording surface explicitly.
- `Sleep` hides races in the final video. Use hidden readiness checks for anything that depends on a server, file watcher, background job, or network.
- Do not commit generated videos unless the repo expects binary demo assets. If unsure, ask before adding large GIF/MP4/WebM files to version control.
