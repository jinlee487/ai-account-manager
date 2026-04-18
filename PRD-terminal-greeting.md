# PRD — Terminal Claude-account greeting

**Owner:** jay.lee@luladelivery.com
**Date:** 2026-04-18
**Status:** Draft v0.3
**Platform:** macOS (zsh)

---

## 1. Problem

When opening a new terminal, there's no immediate signal which Claude Code account is currently active. Discovering it today requires inspecting `~/.claude.json` or the macOS Keychain, or starting a Claude session and asking. With two accounts (personal + Lula Commerce) in rotation, mis-identifying the active account risks doing work under the wrong identity.

## 2. Goal

Every new interactive zsh shell prints a one-line banner showing the currently-active Claude Code account email, **reflecting the real current state** even after `claude /login` to a different account.

Non-goals (v1): cross-shell support (bash/fish), Gemini account display, dynamic re-render on account switch within an already-open shell, color/theming configuration.

## 3. Users

Single user, single Mac, zsh only.

## 4. Approach

### 4.1 Chosen implementation — dynamic read from `~/.claude.json`

On every new interactive zsh shell, parse `oauthAccount.emailAddress` from `~/.claude.json` and print it. Append to `~/.zshrc`:

```zsh
# Claude Code active-account reminder
if [[ -o interactive ]] && [[ -f "$HOME/.claude.json" ]]; then
  _claude_email=$(python3 -c 'import json,os; print(json.load(open(os.path.expanduser("~/.claude.json"))).get("oauthAccount",{}).get("emailAddress","unknown"))' 2>/dev/null)
  _claude_display="${_claude_email:-unknown}"
  _claude_inner="  ◆ Claude  ${_claude_display}  "
  _claude_width=${#_claude_inner}
  _claude_border=$(printf '─%.0s' $(seq 1 $_claude_width))
  echo "\033[38;5;214m╭${_claude_border}╮\033[0m"
  echo "\033[38;5;214m│\033[0m${_claude_inner}\033[38;5;214m│\033[0m"
  echo "\033[38;5;214m╰${_claude_border}╯\033[0m"
  unset _claude_display _claude_inner _claude_width _claude_border
  unset _claude_email
fi
```

Key properties:
- **Interactive-only** (`[[ -o interactive ]]`) — no noise in scripted/SSH-exec shells.
- **File-guarded** (`[[ -f ... ]]`) — no error if `~/.claude.json` is missing (e.g. fresh install).
- **Parse-guarded** (`2>/dev/null` + `:-unknown` fallback) — malformed JSON degrades to `Claude account: unknown`, never breaks the shell.
- **Dynamic width** — box border auto-sizes to fit any email length.
- **Orange color** (ANSI 214) — matches Claude's brand tone; degrades gracefully in terminals without 256-color support.
- **No external deps beyond system Python 3** (ships with macOS).

### 4.2 Why dynamic read (vs. hardcoding)

- **Always correct** — after `claude /login` to a different account, the next new terminal shows the new email.
- **Zero manual maintenance** — no need to edit `.zshrc` on every account switch.
- **Self-healing** — if `~/.claude.json` is absent, banner reads `unknown` instead of lying.
- Trade-off: ~20–50 ms shell-startup cost for the Python invocation. Acceptable on modern Macs; imperceptible in practice.

### 4.3 Alternatives considered

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| Hardcode in `.zshrc` | Trivial, instant | Stale after `claude /login` | Rejected — defeats the purpose |
| Read `~/.claude.json` via Python | Always current, stdlib only | ~20–50 ms per shell | **Chosen for v1** |
| `jq`-based parse | Faster than Python | Adds Homebrew dep | Rejected — avoid deps |
| Query Anthropic API with keychain token | Authoritative | Network call on every shell start | Rejected |

## 5. User experience

```
$ # open new terminal
╭────────────────────────────────────────────╮
│  ◆ Claude  jay.lee@luladelivery.com        │
╰────────────────────────────────────────────╯
jay@mac ~ %

$ claude /login   # switch to personal account
...

$ # open new terminal
╭──────────────────────────────────────────╮
│  ◆ Claude  jay.personal@gmail.com        │
╰──────────────────────────────────────────╯
jay@mac ~ %
```

Banner is orange (ANSI 214), auto-sized to email length, and always reflects `~/.claude.json` at shell-start time.

## 6. Implementation

1. Append the snippet from §4.1 to `~/.zshrc`.
2. Verify by opening a new terminal tab — banner should match current account.
3. Sanity-check fallbacks:
   - Rename `~/.claude.json` temporarily → new shell prints nothing (file-guard trips).
   - Corrupt the JSON → new shell prints `Claude account: unknown`.

**Rollback:** delete the snippet from `~/.zshrc`.

## 7. Risks & constraints

| Risk | Mitigation |
|---|---|
| `~/.claude.json` schema changes upstream (`oauthAccount.emailAddress` moves/renames) | Python `.get(...)` chain returns `"unknown"` instead of throwing; user notices and updates the path |
| System Python 3 removed in a future macOS release | Fall back to a zsh-native parse (regex on `emailAddress`) in v1.1 if this happens |
| Non-interactive shell output parsing breaks | Guarded by `[[ -o interactive ]]` |
| Banner lies during the brief window between `claude /login` writing keychain and `~/.claude.json` | Accepted — race is sub-second and only affects a shell opened in that instant |
| `~/.claude.json` contains secrets (project state, OAuth blob) — accidentally echoed | Only `oauthAccount.emailAddress` is extracted; full file is never printed |

## 8. Exit criteria

- New zsh terminal prints `Claude account: <current-email>` as the first line, matching `~/.claude.json`.
- After `claude /login` to a different account, the next new terminal shows the new email.
- Missing or malformed `~/.claude.json` does not error or delay shell startup.
- Non-interactive shells (`zsh -c '...'`) do not print the banner.

## 9. Future work

- v1.1: replace Python with a zsh-native parser if shell-startup latency becomes noticeable.
- v1.2: extend to Gemini CLI (`~/.gemini/settings.json` → active account) once the `aam` tool ([PRD.md](PRD.md)) lands.
- v1.3: integrate with `aam use` so switching accounts within an open shell refreshes the banner via a `precmd` hook.
