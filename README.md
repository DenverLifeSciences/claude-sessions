# claude-sessions

Find and resume any Claude Code session from any project — and rebuild your whole
terminal after a reboot — from one [fzf](https://github.com/junegunn/fzf) picker.

**What it does**

- **Cross-project picker.** `claude --resume` only lists sessions for the directory
  you run it from. `claude-sessions` lists *every* session from *every* project in
  one picker, newest first. Enter opens the highlighted session in a new iTerm tab
  (or tmux window) running `claude --resume <id>` in the right directory — and the
  picker stays open for the next one.
- **Crash recovery.** A timer snapshots your full iTerm layout every 5 minutes —
  every window/tab, its working directory, and the Claude session running inside it.
  After a reboot (or an unattended OS update), one command rebuilds the whole set.

**Why.** Recovering by hand means remembering every repo you had open, `cd`-ing into
each, and running `claude --resume` there — once per session. This collapses that
into: run `cs`, skim, hit Enter. And a reboot no longer costs you your working set.

## Install

Dependencies: `fzf` (`brew install fzf`) and `python3` (ships with macOS).

One command — installs the script to `~/.local/bin`, adds it to your PATH with
a `cs` alias, and loads the 5-minute snapshot timer:

```sh
curl -fsSL https://raw.githubusercontent.com/DenverLifeSciences/claude-sessions/main/claude-sessions | bash -s -- --install
```

Then open a new shell and run **`cs`** — the short alias the installer adds —
to launch the picker (`claude-sessions` works too; zsh only, bash/fish users
add their own alias).

Later, `claude-sessions --update` pulls the latest version. Both commands are
idempotent — rerun them freely.

## Keys

```
enter   open session in a new tab (picker stays open)
ctrl-a  hide/show agent-teammate sessions (⛭)
ctrl-s  full-text search of whole transcripts for what you typed in the
        query box; the list becomes the matching sessions only
ctrl-r  refresh the list / return from a ctrl-s search to the full list
esc     quit
```

Each line shows the session's age, project directory, account (only when it
isn't the default one), git branch (when not main/master), and its opening
message (what you typed, not Claude Code's injected wrappers; slash commands show as `/name args`), followed by the session title (`/rename` title, else Claude's auto-generated one) in dim text. Both are searchable; `ctrl-s` searches whole transcripts. A preview pane below the list shows the last few
user/assistant exchanges of the highlighted session. Resume runs interactively in
the new tab, so Claude Code's own "resume from summary or full session?" prompt
appears there.

## How it works

Claude Code stores every session as a JSONL transcript:

```
~/.claude/projects/<encoded-project-path>/<session-id>.jsonl
```

The early lines of each file carry `sessionId`, `cwd`, `gitBranch`, the first user message, and — for sessions spawned as agent teammates — an `agentName` field. That's a complete session index sitting on disk; this script just points fzf at it.

Task-tool subagent transcripts live in a `<session-id>/subagents/` subdirectory and are never listed. Teammate sessions (agent teams) are tagged with a magenta `⛭ <agent-name>` and can be toggled off with `ctrl-a`.

### Two accounts

`CLAUDE_CONFIG_DIR` lets you run a second Claude account with its own tree of
transcripts, and a picker that reads only `~/.claude` can't see any of it — the
session is simply absent from the list, with nothing to select. So the picker
scans every config dir in `CS_CONFIG_DIRS` (`:`-separated, default
`~/.claude:~/.claude-personal`) and labels sessions from a non-default account
with a blue `▸<name>`. Missing dirs are skipped, so the default is fine on a
one-account machine. Set `CLAUDE_SESSIONS_CONFIG_DIRS` to scan a different set:

```sh
export CLAUDE_SESSIONS_CONFIG_DIRS="$HOME/.claude:$HOME/.claude-work"
```

Resuming across accounts is Claude Code's business, not the picker's: it runs
`claude --resume <id>`, so if your `claude` is a shim that points
`CLAUDE_CONFIG_DIR` at whichever tree owns the id, cross-account resume works
as-is. Otherwise, launch the picker from a shell where `CLAUDE_CONFIG_DIR` is
already set for the account you want to resume into.

## Crash recovery: snapshot & restore the whole terminal

A machine crash kills every tab, not just Claude. `claude-sessions` can
snapshot your full iTerm layout — every window/tab, its working directory,
and the Claude session running inside it (if any) — and rebuild it all
after a reboot:

```sh
claude-sessions --snapshot        # snapshot iTerm state now (the timer does this every 5 min)
claude-sessions --restore-crash   # recreate every tab (as tabs of the current window); rerun `claude --resume` where Claude was running
claude-sessions --restore-pick    # browse snapshot history in fzf, restore any earlier state
```

Snapshots are kept as history (last 100 distinct states, in
`~/.claude/terminal-snapshots/`), so even if the timer runs after you reopen a
near-empty terminal, the pre-crash layout is still there. `--restore-crash`
defaults to the newest snapshot that still has live Claude sessions — a reboot
leaves iTerm's bare shells behind, and that session-less husk is skipped rather
than restored over your real state. `--restore-pick` shows each snapshot's age,
tab count, and projects so you can grab any earlier one by hand. Identical
consecutive states aren't re-saved.

`--install` sets up the 5-minute snapshot timer automatically (a launchd
agent, `com.claude.terminal-snapshot`; template in
[`contrib/`](contrib/com.claude.terminal-snapshot.plist) if you'd rather wire
it yourself).

Notes: an empty iTerm never overwrites the last good snapshot, so the
pre-crash state survives the reboot. The script installs to a local path
(`~/.local/bin`) rather than anywhere cloud-synced because macOS denies
launchd jobs access to cloud-provider paths (iCloud Drive, Dropbox).

## Running Claude in tmux? Use `claude-sessions tmux`

The snapshot above reads iTerm tabs, so it can't see windows inside a tmux
session — and tmux itself doesn't survive a reboot. `claude-sessions tmux` is the
tmux counterpart: it saves each window's name, folder and Claude session id to
`~/.claude/tmux-layout.tsv` and rebuilds the session afterwards, one
`claude --resume <id>` per window.

```sh
claude-sessions tmux install-timer   # snapshot every 5 min (launchd agent com.claude.tmux-layout)
claude-sessions tmux save            # snapshot now
claude-sessions tmux show            # print the saved layout
claude-sessions tmux restore         # after a reboot: recreate every window, resuming each session
claude-sessions tmux export          # printable list of session ids + folders, to save by hand
```

`export` also lists sessions running outside tmux, and (after a reboot) every
session in the saved layout that isn't running. It reads the live state into a
temp file, so it never overwrites the saved layout.

Run `restore` before starting any other tmux session: the 5-minute timer saves
whatever tmux is running, so a fresh one-window server would replace the saved
layout. Session ids come from Claude Code's live session registry
(`~/.claude/sessions/*.json`), not from the `--resume` argument, which goes stale
because resuming forks a new session id.

## Caveats

- The `~/.claude/projects` layout is undocumented internal storage — a Claude Code update could change it. The script only reads these files; worst case the picker breaks, never your sessions.
- Opening tabs uses AppleScript (iTerm2, Terminal.app fallback), so that path is macOS-only. Inside tmux it uses `tmux new-window`, which works on Linux too.
- Lists the 300 most recent sessions by default (`CLAUDE_SESSIONS_MAX` overrides). Scanning a second account pushes more sessions into that budget, so raise it if older ones start dropping off.

## License

MIT
