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

```sh
curl -fsSL https://raw.githubusercontent.com/daksh-dls/claude-sessions/main/claude-sessions -o /usr/local/bin/claude-sessions
chmod +x /usr/local/bin/claude-sessions
```

Then run `claude-sessions` (alias it to `cs` if you like).

## How it works

Claude Code stores every session as a JSONL transcript:

```
~/.claude/projects/<encoded-project-path>/<session-id>.jsonl
```

The early lines of each file carry `sessionId`, `cwd`, `gitBranch`, the first user message, and — for sessions spawned as agent teammates — an `agentName` field. That's a complete session index sitting on disk; this script just points fzf at it.

Task-tool subagent transcripts live in a `<session-id>/subagents/` subdirectory and are never listed. Teammate sessions (agent teams) are tagged with a magenta `⛭ <agent-name>` and can be toggled off with `ctrl-a`.

## Caveats

- The `~/.claude/projects` layout is undocumented internal storage — a Claude Code update could change it. The script only reads these files; worst case the picker breaks, never your sessions.
- Opening tabs uses AppleScript (iTerm2, Terminal.app fallback), so that path is macOS-only. Inside tmux it uses `tmux new-window`, which works on Linux too.
- Lists the 300 most recent sessions by default (`CLAUDE_SESSIONS_MAX` overrides).

## License

MIT
