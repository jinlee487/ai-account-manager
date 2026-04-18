# PRD — `ai-account-manager` (aam)

**Owner:** jay.lee@luladelivery.com
**Date:** 2026-04-18
**Status:** Draft v0.1
**Platform:** macOS (Darwin 25.x, Apple Silicon + Intel)

---

## 1. Problem

The user runs multiple AI CLI tools (Claude Code, Gemini CLI) and has two accounts per tool — one **personal**, one **business** (Lula Commerce). Today, each CLI only remembers one logged-in account at a time. Switching means logging out, logging back in, and losing session state — high friction, done many times a day.

## 2. Goal

A local macOS CLI, `aam`, that lets the user:

1. **Register** each existing AI account under a friendly label (email).
2. **List** registered accounts across tools.
3. **Switch** the active account for a given tool with a single command (`aam use claude personal`), such that the tool's CLI, when next launched, operates under that account with no re-login.

Non-goals (v1): GUI, cross-machine sync, managing accounts for tools other than Claude Code and Gemini CLI, proxying API calls.

## 3. Users

Single user, single Mac. Multi-user support is out of scope.

## 4. Technical feasibility analysis (macOS)

### 4.1 Where each tool stores its identity

| Tool | Credential storage | Profile metadata | API-key alt |
|---|---|---|---|
| **Claude Code** | macOS Keychain — `security` service `Claude Code-credentials`, account = unix username. Contains OAuth access + refresh tokens (JSON blob). | `~/.claude.json` — top-level `oauthAccount` object (`emailAddress`, `organizationUuid`, `organizationName`, `accountUuid`, etc.). Per-project state in `~/.claude/projects/`. | `ANTHROPIC_API_KEY` env var |
| **Gemini CLI** | `~/.gemini/oauth_creds.json` (OAuth refresh/access tokens, plain JSON on disk). | `~/.gemini/settings.json`, plus cached `~/.gemini/google_accounts.json` (email → account id). | `GEMINI_API_KEY` env var |

Both tools read these paths/entries on startup; neither provides a first-class "profile" concept. Observed on this machine: Claude Keychain entry exists under `Claude Code-credentials` / acct `jay`, and `~/.claude.json` shows `oauthAccount.emailAddress = jay.lee@luladelivery.com`.

### 4.2 Core approach — snapshot & swap

For each (tool, account) pair, store a **profile bundle** containing that tool's credential blob(s) and relevant metadata. "Switching" writes the bundle back to the tool's canonical location. Before each switch, re-snapshot the currently-active account so rotated/refreshed tokens are preserved.

- **Claude Code switch**:
  1. Re-snapshot: read current Keychain entry via `security find-generic-password -s "Claude Code-credentials" -a <user> -w` and read `oauthAccount` from `~/.claude.json`; update the currently-active profile bundle.
  2. Write target: `security add-generic-password -U -s "Claude Code-credentials" -a <user> -w <blob>` and patch `oauthAccount` in `~/.claude.json` (preserving unrelated keys).
- **Gemini CLI switch**: same pattern against `~/.gemini/oauth_creds.json` and `~/.gemini/settings.json`.
- **API-key accounts** (optional): profile stores the key value; switching exports it into a managed shell snippet sourced by the user's shell, or writes it to a per-tool env file the user sources.

### 4.3 Why this works

- macOS Keychain supports scripted read/write of generic passwords via `/usr/bin/security` — first access per session prompts the user; subsequent access within the same session is silent once "Always Allow" is granted for the `aam` binary.
- OAuth refresh tokens are long-lived (weeks to months). As long as the refresh token is valid at switch time, the CLI will not prompt re-login.
- Neither Claude Code nor Gemini CLI pins credentials to hardware-bound identifiers; the credential blobs are portable across invocations on the same machine.

### 4.4 Risks & constraints

| Risk | Mitigation |
|---|---|
| Switch while a CLI is running → mid-flight writes corrupt state | Detect running `claude` / `gemini` processes; refuse or warn. |
| Token rotation drift — tokens refresh during use, stored bundle goes stale | Always re-snapshot the active profile **before** overwriting on switch. |
| macOS Keychain ACL prompts interrupt scripted use | Ship instructions to grant `aam` "Always Allow" on first run; fall back to interactive prompt. |
| Upstream tools change file schemas (`.claude.json` shape, `oauth_creds.json`) | Treat profile bundles as opaque JSON; patch only known keys (`oauthAccount`); version the profile format. |
| User wipes/reinstalls a CLI → schema diverges from stored bundle | `aam doctor` verifies each profile still parses against live schema; offer re-capture. |
| Multiple shells open with stale `ANTHROPIC_API_KEY` in env | Document that API-key mode requires new shell / re-source; OAuth mode is unaffected. |
| Secrets at rest — Gemini stores OAuth in a plain file | Store `aam` profile bundles under `~/.ai-account-manager/profiles/` with mode `0600`; optionally wrap with Keychain-encrypted generic items in v1.1. |

### 4.5 Feasibility verdict

**Feasible for v1** using `security` CLI + file swaps. No private APIs, no code signing gymnastics, no kernel extensions. The main ongoing cost is tracking upstream schema changes in Claude Code and Gemini CLI.

## 5. User experience

### 5.1 Commands (v1)

```
aam register claude                  # capture the currently-active Claude account as a profile
aam register gemini --label work     # same, with explicit label override
aam list                             # show all profiles across tools, with active marker
aam use claude personal              # switch Claude Code to the "personal" profile
aam use gemini work
aam whoami                           # print active account per tool
aam remove claude personal
aam doctor                           # verify each profile still loads; detect running CLIs
```

Labels default to the email's local-part (`jay.lee` → `jay-lee`) with a `--label` override. Profiles are addressable by label **or** full email.

### 5.2 Example flow

```
$ aam register claude
Captured Claude account: jay.lee@luladelivery.com (Lula Commerce) → label "work"

$ aam register claude                  # after logging in with personal account via `claude /login`
Captured Claude account: jay.personal@gmail.com → label "personal"

$ aam list
claude    * work       jay.lee@luladelivery.com   (Lula Commerce)
claude      personal   jay.personal@gmail.com
gemini    * work       jay.lee@luladelivery.com
gemini      personal   jay.personal@gmail.com

$ aam use claude personal
Switched Claude Code → jay.personal@gmail.com
(run `claude` in a new terminal to use)
```

## 6. Architecture

```
aam (Go or Node 20 CLI, single binary)
├── cmd/            register, list, use, whoami, remove, doctor
├── providers/
│   ├── claude/     read/write Keychain + ~/.claude.json oauthAccount
│   └── gemini/     read/write ~/.gemini/oauth_creds.json + settings.json
├── store/          ~/.ai-account-manager/ (profiles + index, 0600)
└── keychain/       wrapper around /usr/bin/security
```

**Profile bundle format** (`~/.ai-account-manager/profiles/<tool>/<label>.json`):

```json
{
  "schemaVersion": 1,
  "tool": "claude",
  "label": "work",
  "email": "jay.lee@luladelivery.com",
  "capturedAt": "2026-04-18T10:00:00Z",
  "payload": {
    "keychain": { "service": "Claude Code-credentials", "account": "jay", "blob": "<opaque>" },
    "profileJsonPatch": { "oauthAccount": { /* ... */ } }
  }
}
```

**Index** (`~/.ai-account-manager/index.json`): maps `(tool → label)` and records the currently-active label per tool, used to decide which profile to re-snapshot before a switch.

## 7. Milestones

| M | Scope | Exit criteria |
|---|---|---|
| M0 | Repo scaffold, `aam --version`, provider interface | Runs on macOS 14+; CI green |
| M1 | Claude Code provider: register / list / use / whoami | Round-trip switch between two Claude accounts without re-login |
| M2 | Gemini CLI provider: same surface | Round-trip switch between two Gemini accounts |
| M3 | `doctor`, running-process detection, re-snapshot-on-switch | Safe-switch guarantees documented & tested |
| M4 | Homebrew tap / signed release | `brew install jay/tap/aam` works |

## 8. Open questions

1. Language — Go (single static binary, easy Homebrew distribution) or Node/TS (ecosystem familiarity). **Recommendation: Go.**
2. Should `aam use` auto-launch the tool in a new terminal, or leave that to the user? **v1: leave to user.**
3. API-key-only accounts — in scope for v1 or v1.1? **Recommendation: v1.1.**
4. Should we support MCP server config swapping alongside accounts (some MCPs are org-scoped)? **Out of scope for v1.**

## 9. Success metrics

- Time-to-switch: < 2 seconds (excluding first-run Keychain ACL prompt).
- Re-login rate after switch: 0 (tokens remain valid across swaps for the refresh-token lifetime).
- User-reported friction: qualitative — "I stopped logging in and out."
