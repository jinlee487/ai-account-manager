# PRD — Terminal Claude-account greeting

**Owner:** jay.lee@luladelivery.com
**Date:** 2026-05-03
**Status:** Draft v0.6
**Platform:** macOS (zsh)

---

## 1. Problem

When opening a new terminal, there's no immediate signal which Claude Code account is currently active. Discovering it today requires inspecting `~/.claude.json` or the macOS Keychain, or starting a Claude session and asking. With two accounts (personal + Lula Commerce) in rotation, mis-identifying the active account risks doing work under the wrong identity.

A parallel problem exists for npm: `~/.npmrc` may point at the **default public registry** or at the **AWS CodeArtifact** mirror used for Lula's private packages. Running `npm install` against the wrong registry either fails (private packages missing on public) or pulls a stale mirror. Today the only way to tell is `cat ~/.npmrc` or `npm config get registry`.

## 2. Goal

Every new interactive zsh shell prints a single orange box with two lines:
1. `◆ Claude  <email>` — the currently-active Claude Code account.
2. `◆ npm     <label>` — the currently-active npm registry as a glanceable label (`default` / `codeartifact` / `custom`).

Both lines live **inside the same box** so the two facts read as one banner. The box border auto-sizes to the wider of the two lines, and both lines are right-padded so the closing `│` aligns. The banner must **reflect the real current state** at shell-start time, even after `claude /login` or `npm config set registry`.

Non-goals (v1): cross-shell support (bash/fish), Gemini account display, dynamic re-render on account switch within an already-open shell, color/theming configuration, project-level `.npmrc` resolution (only user-level `~/.npmrc` is consulted).

## 3. Users

Single user, single Mac, zsh only.

## 4. Approach

### 4.1 Chosen implementation — dynamic read from `~/.claude.json` and `~/.npmrc`

On every new interactive zsh shell, parse `oauthAccount.emailAddress` from `~/.claude.json` and the active `registry=` from `~/.npmrc`, then print both. Append to `~/.zshrc`:

```zsh
# Shared renderer: single orange box, two lines (Claude account + npm registry)
_claude_npm_banner() {
  local email display registry npm_label
  local content1 content2 inner1 inner2 width border

  # Claude account
  if [[ -f "$HOME/.claude.json" ]]; then
    email=$(python3 -c 'import json,os; print(json.load(open(os.path.expanduser("~/.claude.json"))).get("oauthAccount",{}).get("emailAddress","unknown"))' 2>/dev/null)
    display="${email:-unknown}"
  else
    display="unknown"
  fi

  # npm registry — derived label (default | codeartifact | custom)
  npm_label="default"
  if [[ -f "$HOME/.npmrc" ]]; then
    registry=$(grep -E '^registry[[:space:]]*=' "$HOME/.npmrc" 2>/dev/null | tail -n1 | cut -d= -f2- | tr -d '[:space:]')
    if [[ "$registry" == *codeartifact* ]]; then
      npm_label="codeartifact"
    elif [[ -n "$registry" && "$registry" != *registry.npmjs.org* ]]; then
      npm_label="custom"
    fi
  fi

  # Two content rows with matching prefix structure ("◆ Claude  …" / "◆ npm     …")
  content1="  ◆ Claude  ${display}  "
  content2="  ◆ npm     ${npm_label}  "

  # Box width = wider of the two rows
  if (( ${#content1} >= ${#content2} )); then
    width=${#content1}
  else
    width=${#content2}
  fi

  # Right-pad both rows to the box width (zsh ${(r:N:: :)var} fills with spaces)
  inner1="${(r:$width:: :)content1}"
  inner2="${(r:$width:: :)content2}"
  border=$(printf '─%.0s' $(seq 1 $width))

  echo "\033[38;5;214m╭${border}╮\033[0m"
  echo "\033[38;5;214m│\033[0m${inner1}\033[38;5;214m│\033[0m"
  echo "\033[38;5;214m│\033[0m${inner2}\033[38;5;214m│\033[0m"
  echo "\033[38;5;214m╰${border}╯\033[0m"
}

# Print on every new interactive shell
if [[ -o interactive ]]; then
  _claude_npm_banner
fi

# On-demand reprint — useful after `claude /login` or `npm config set registry`
claude-who() { _claude_npm_banner; }
```

Key properties:
- **Interactive-only** (`[[ -o interactive ]]`) — no noise in scripted/SSH-exec shells.
- **File-guarded** — missing `~/.claude.json` shows `unknown` in the email slot; missing `~/.npmrc` falls through to `default` (correct: absence means npm uses its built-in default registry). The box always renders both lines.
- **Parse-guarded** (`2>/dev/null` + fallbacks) — malformed JSON degrades to `unknown`; an `~/.npmrc` without a `registry=` line degrades to `default`. Never breaks the shell.
- **Dynamic width, aligned closing border** — box width = wider of the two content rows; the shorter row is right-padded with spaces so the closing `│` lines up vertically.
- **Parallel row structure** — both rows lead with `◆` + a fixed-width label (`Claude` / `npm   `) + value, so the eye reads them as a stack of facts, not a heading + footnote.
- **Orange color** (ANSI 214) — matches Claude's brand tone; degrades gracefully in terminals without 256-color support.
- **Glanceable npm label** — `default` / `codeartifact` / `custom` instead of a 100-char URL. The dichotomy the user actually cares about (default vs CodeArtifact) is one word.
- **Single shared renderer** — `_claude_npm_banner` is called both at shell start and from `claude-who`, so the two paths cannot drift.
- **No external deps beyond system Python 3** (ships with macOS) and `grep` / `cut` / `tr` (POSIX baseline). The right-pad uses zsh's `${(r:N:: :)var}` parameter expansion, which is built-in (no fork).
- **No `npm` invocation** — parsing `~/.npmrc` directly avoids ~200–500 ms of npm CLI startup per shell.

### 4.2 Why dynamic read (vs. hardcoding)

- **Always correct** — after `claude /login` to a different account, the next new terminal shows the new email.
- **Zero manual maintenance** — no need to edit `.zshrc` on every account switch.
- **Self-healing** — if `~/.claude.json` is absent, banner reads `unknown` instead of lying.
- Trade-off: ~20–50 ms shell-startup cost for the Python invocation. Acceptable on modern Macs; imperceptible in practice.

### 4.3 Alternatives considered

**Claude account source:**

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| Hardcode in `.zshrc` | Trivial, instant | Stale after `claude /login` | Rejected — defeats the purpose |
| Read `~/.claude.json` via Python | Always current, stdlib only | ~20–50 ms per shell | **Chosen for v1** |
| `jq`-based parse | Faster than Python | Adds Homebrew dep | Rejected — avoid deps |
| Query Anthropic API with keychain token | Authoritative | Network call on every shell start | Rejected |

**npm registry source:**

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| `npm config get registry` | Resolves full chain (project + user + global) | ~200–500 ms per shell — dominates startup | Rejected for v1 |
| Parse `~/.npmrc` directly | Fast (single `grep`), no deps | Misses project-level `.npmrc` overrides | **Chosen for v1** — banner is a user-account signal, project overrides aren't relevant |
| Show full registry URL | Most precise | Long, noisy, hard to scan | Rejected — user asked for glanceable |
| `npm whoami` | Shows actual logged-in user | Network call; fails offline; doubles startup latency | Rejected |

## 5. User experience

```
$ # open new terminal — Lula account, CodeArtifact registry
╭────────────────────────────────────────────╮
│  ◆ Claude  jay.lee@luladelivery.com        │
│  ◆ npm     codeartifact                    │
╰────────────────────────────────────────────╯
jay@mac ~ %

$ claude /login   # switch to personal account
$ npm config set registry https://registry.npmjs.org/
$ claude-who       # reprint banner in the same shell
╭──────────────────────────────────────────╮
│  ◆ Claude  jay.personal@gmail.com        │
│  ◆ npm     default                       │
╰──────────────────────────────────────────╯

$ # or: open new terminal — picks up both changes automatically
╭──────────────────────────────────────────╮
│  ◆ Claude  jay.personal@gmail.com        │
│  ◆ npm     default                       │
╰──────────────────────────────────────────╯
jay@mac ~ %
```

Banner is orange (ANSI 214), auto-sized to the wider of the two content rows, and always reflects `~/.claude.json` and `~/.npmrc` at read time — automatically at shell start, or on demand via `claude-who`. Both facts live in one box so the eye reads them as a unit: "who am I as Claude, and which npm world am I in."

## 6. Implementation

1. Append the snippet from §4.1 to `~/.zshrc` (the `_claude_npm_banner` renderer, the interactive-shell call, and the `claude-who` alias).
2. Verify by opening a new terminal tab — banner should match current Claude account and current npm registry.
3. Run `claude-who` in an existing shell — same banner should print on demand.
4. Sanity-check fallbacks:
   - Rename `~/.claude.json` temporarily → box still renders, Claude row shows `unknown` in the email slot.
   - Corrupt `~/.claude.json` → same as above (`unknown`).
   - Rename `~/.npmrc` temporarily → npm row shows `default` (correct: absent npmrc means npm uses the built-in default registry).
   - Set registry to a non-CodeArtifact, non-public URL (e.g. Verdaccio) → npm row shows `custom`.
   - Set a very long email (e.g. test@a-very-long-corporate-domain.example.com) → box widens to fit; npm row right-pads to match.

**Rollback:** delete the `_claude_npm_banner` function, the interactive-shell call, and the `claude-who` alias from `~/.zshrc`.

## 7. Risks & constraints

| Risk | Mitigation |
|---|---|
| `~/.claude.json` schema changes upstream (`oauthAccount.emailAddress` moves/renames) | Python `.get(...)` chain returns `"unknown"` instead of throwing; user notices and updates the path |
| System Python 3 removed in a future macOS release | Fall back to a zsh-native parse (regex on `emailAddress`) in v1.1 if this happens |
| Non-interactive shell output parsing breaks | Guarded by `[[ -o interactive ]]` |
| Banner lies during the brief window between `claude /login` writing keychain and `~/.claude.json` | Accepted — race is sub-second and only affects a shell opened in that instant |
| `~/.claude.json` contains secrets (project state, OAuth blob) — accidentally echoed | Only `oauthAccount.emailAddress` is extracted; full file is never printed |
| `~/.npmrc` may contain auth tokens (`//registry/...:_authToken=`) — accidentally echoed | Only the `registry=` line is read, and only its derived label (`default`/`codeartifact`/`custom`) is printed; raw URL and tokens never leave the file |
| Project-level `.npmrc` overrides user-level — banner says `default` but `cd`'d project uses CodeArtifact | Accepted in v1 (banner is a user-account signal, not a per-cwd one). `claude-who` reprint after `cd` is not project-aware. v1.x can add `precmd` resolution if this becomes a recurring footgun |
| `registry=` line uses unusual whitespace or quoting | `grep -E '^registry[[:space:]]*='` + `tr -d '[:space:]'` handle leading/trailing whitespace; truly malformed lines fall through to `default` |

## 8. Exit criteria

- New zsh terminal prints a single orange box with two rows: `◆ Claude <email>` (from `~/.claude.json`) and `◆ npm <label>` (from `~/.npmrc`). Closing `│` of both rows aligns vertically.
- After `claude /login` to a different account, the next new terminal shows the new email.
- After `npm config set registry <url>`, the next new terminal shows the corresponding label (`default` / `codeartifact` / `custom`).
- `claude-who` reprints the full box on demand in an already-open shell.
- Missing or malformed `~/.claude.json` does not error or delay shell startup; the box still renders with `unknown` in the email slot.
- Missing `~/.npmrc` results in the npm row showing `default` (not an error, not an empty row).
- Non-interactive shells (`zsh -c '...'`) do not print the banner.

## 9. Future work

- v1.1: replace Python with a zsh-native parser if shell-startup latency becomes noticeable.
- v1.2: extend to Gemini CLI (`~/.gemini/settings.json` → active account) once the `aam` tool ([PRD.md](PRD.md)) lands.
- v1.3: integrate with `aam use` so switching accounts within an open shell refreshes the banner automatically via a `precmd` hook (today `claude-who` covers this manually).
- v1.4: project-aware npm label — resolve `.npmrc` from cwd up to `$HOME` so the banner reflects the registry the *next* `npm install` would actually use, not just the user-level default.
