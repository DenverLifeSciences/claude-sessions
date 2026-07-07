# claude-sessions

A cross-project session picker for [Claude Code](https://claude.com/claude-code) CLI.

Your machine crashes with seven Claude Code sessions open across five repos. `claude --resume` only lists sessions for the directory you run it from — so recovery means remembering every repo you were in, `cd`-ing into each one, and picking from a list, seven times.

`claude-sessions` instead lists **every session from every project** in one [fzf](https://github.com/junegunn/fzf) picker, newest first. Hit Enter and the session opens in a new iTerm tab (or tmux window) running `claude --resume <id>` in the right directory — while the picker stays open for the next one.

```
enter   open session in a new tab (picker stays open)
ctrl-a  hide/show agent-teammate sessions (⛭)
ctrl-r  refresh the list
esc     quit
```

Each line shows the session's age, project directory, git branch (when not main/master), and its opening message. A preview pane shows the last few user/assistant exchanges of the highlighted session. Resume runs interactively in the new tab, so Claude Code's own "resume from summary or full session?" prompt appears there.

## Install

Dependencies: `fzf` (`brew install fzf`) and `python3` (ships with macOS).

One command — installs the script to `~/.local/bin`, adds it to your PATH with
a `cs` alias, and loads the 5-minute snapshot timer (see below):

```sh
curl -fsSL https://raw.githubusercontent.com/DenverLifeSciences/claude-sessions/main/claude-sessions | bash -s -- --install
```

Later, `claude-sessions --update` pulls the latest version. Both commands are
idempotent — rerun them freely.

## How it works

Claude Code stores every session as a JSONL transcript:

```
~/.claude/projects/<encoded-project-path>/<session-id>.jsonl
```

The early lines of each file carry `sessionId`, `cwd`, `gitBranch`, the first user message, and — for sessions spawned as agent teammates — an `agentName` field. That's a complete session index sitting on disk; this script just points fzf at it.

Task-tool subagent transcripts live in a `<session-id>/subagents/` subdirectory and are never listed. Teammate sessions (agent teams) are tagged with a magenta `⛭ <agent-name>` and can be toggled off with `ctrl-a`.


## Crash recovery: snapshot & restore the whole terminal

A machine crash kills every tab, not just Claude. `claude-sessions` can
snapshot your full iTerm layout — every window/tab, its working directory,
and the Claude session running inside it (if any) — and rebuild it all
after a reboot:

```sh
claude-sessions --snapshot        # dump current iTerm state to ~/.claude/terminal-snapshot.json
claude-sessions --restore-crash   # recreate every tab (as tabs of the current window); rerun `claude --resume` where Claude was running
claude-sessions --restore-pick    # browse snapshot history in fzf, restore any earlier state
```

Snapshots are kept as history (last 100 distinct states, in
`~/.claude/terminal-snapshots/`), so even if the timer runs after you reopen a
near-empty terminal, the pre-crash layout is still there — `--restore-pick`
shows each snapshot's age, tab count, and projects so you can grab the right
one. Identical consecutive states aren't re-saved.

`--install` sets up the 5-minute snapshot timer automatically (a launchd
agent, `com.claude.terminal-snapshot`; template in
[`contrib/`](contrib/com.claude.terminal-snapshot.plist) if you'd rather wire
it yourself).

Notes: an empty iTerm never overwrites the last good snapshot, so the
pre-crash state survives the reboot. The script installs to a local path
(`~/.local/bin`) rather than anywhere cloud-synced because macOS denies
launchd jobs access to cloud-provider paths (iCloud Drive, Dropbox).

## Caveats

- The `~/.claude/projects` layout is undocumented internal storage — a Claude Code update could change it. The script only reads these files; worst case the picker breaks, never your sessions.
- Opening tabs uses AppleScript (iTerm2, Terminal.app fallback), so that path is macOS-only. Inside tmux it uses `tmux new-window`, which works on Linux too.
- Lists the 300 most recent sessions by default (`CLAUDE_SESSIONS_MAX` overrides).

## License

MIT
