---
name: 1p-creds
description: Manage a repository's credentials via 1Password Environments — setup (direnv + FIFO-mounted .env.local + Claude/MCP wiring), migrate (detect scattered plaintext tokens, op:// refs, JWT/hex patterns, and ${VAR} references across the repo), rotate (system + admin_url from manifest, AI+web-searched instructions), validate (mount health, Environment contents match .mcp.json needs), and inventory (list all creds the repo uses). Invoke for any of those lifecycle actions on a repo. Bound to 1Password (FIFO mount + biometric model); sibling skills cover other providers. Harness-portable — same manifest + mount mechanics for Claude Code today, Codex and Gemini CLI via per-harness reference docs.
allowed-tools:
  - Read
  - Edit
  - Write
  - Bash
  - WebSearch
  - WebFetch
---

# /1p-creds — Repository credentials via 1Password Environments

Put every repo's credentials behind one 1Password Environment per `{repo}-{tier}`. Mount them as a FIFO-backed `.env.local`, let `direnv` auto-source on shell init, let `claude` / `codex` / `gemini` inherit from the shell. Secrets never touch disk as plaintext. One biometric per 1P unlock cycle covers every terminal, every session, every tool.

## Why this exists

- **Plaintext tokens on disk** (e.g., `.claude/settings.local.json` `env` blocks, literal `.env` files) are the #1 credential hygiene failure in repos.
- **`op run --` wrapping** breaks interactive agents' TTY detection.
- **Per-shell `op inject`** costs ~10-20 biometric prompts per workday.
- **1P Environments' FIFO mount** solves all three: no disk plaintext, no TTY wrap, one biometric per 1P unlock.

## Prerequisites

| # | Requirement | Check | Install |
|---|---|---|---|
| 1 | 1Password desktop app, signed in | `pgrep -f 1Password` | `brew install --cask 1password` |
| 2 | `op` CLI, recent | `op --version` (≥ 2.33) | `brew install 1password-cli` |
| 3 | 1P ↔ CLI biometric integration | Settings → Developer → "Integrate with 1Password CLI" | — |
| 4 | `direnv` | `which direnv` | `brew install direnv` |
| 5 | Shell hook for direnv | `grep direnv ~/.zshrc` | `echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc` |

Setup runs these checks automatically and offers to auto-fix.

## Actions

| Action | What it does |
|---|---|
| **setup** | One-time wiring: check prereqs, create 1P Environment (guide GUI), mount `.env.local`, write `.envrc` + `.gitignore`, `direnv allow`, smoke-test MCP |
| **migrate** | Scan repo for scattered tokens (see `references/migrate.md`), produce import plan, generate consolidated `.env` import file in `/tmp`, wait for user import, shred |
| **rotate** | Read `{VAR}` entry from `.claude/creds-manifest.yaml`, web-search current rotation steps for that `system` at that `admin_url`, guide user, verify FIFO serves new value |
| **validate** | Mount FIFO exists, 1P is running, all `${VAR}`s in `.mcp.json` have entries in the 1P Environment, `.envrc` is allowed by direnv, manifest entry exists for every Environment variable |
| **inventory** | List all creds: name, system, admin URL, scope, last rotation if tracked. Good for onboarding docs and audit |
| **decommission-legacy** | After migration is verified, delete stale pre-migration vault items referenced by any `legacy_source:` field in the manifest. Dedupes at item level. Strips `legacy_source` from manifest on success. |

### setup

Deterministic, step-by-step. Starts with a **Phase 0 preamble** (prereq checklist + confirmation) so the user enters the flow with full context. Then the real interventions come at **Phase D** (1P GUI batch) and **Phase G** (E2E confirmation). Everything else is auto.

**Defaults — we do NOT ask you:**

| Question | Default | Override |
|---|---|---|
| Environment name | `{basename of repo root}-local` (e.g. for a repo at `~/code/myapp/`, default env is `myapp-local`) | Pass `--env=name` |
| Which credentials to migrate | **All** scattered creds found. Orphans too. Plaintext is plaintext. | Pass `--skip=VAR1,VAR2` |
| Classification per variable | Auto-inferred from value + name (see `references/migrate.md` § Auto-classification) | Edit manifest between Phase C and D |
| `system` / `admin_url` per variable | Auto-inferred from value pattern or best-effort from `${VAR}` usage | Edit manifest between Phase C and D |
| Legacy cleanup scope | **Full** — `settings.local.json` env block, `credentials/` folder, stale `.env.op` | Pass `--keep-legacy` |
| Mount path | `{repo-root}/.env.local` | Pass `--mount=/other/path` |

The principle: a half-migrated credential plane is worse than an unmigrated one. Don't invite decision paralysis — migrate everything the scanner finds, classify by pattern, let the user edit the manifest if they disagree with an inference.

#### Phase 0 — Preamble (paged wizard, 4 steps)

**Always run this first, before any checks or file touches.** Present a 4-page wizard. Show ONE page at a time, each under 20 lines of user-facing content. Wait for the user's selection between pages. Never advance on silence.

**Use `AskUserQuestion` for all page navigation.** Do NOT parse free-text replies. At the end of each page, call `AskUserQuestion` with the navigation options as structured choices — the user clicks to select; no typing, no typos. The `Options:` lines in each page template below describe the semantic options — each becomes an `AskUserQuestion` option with a clear label + short description.

**Sequence for any page transition:**

1. User selects an option on Page N (or first invocation → start on Page 1)
2. If Page 2 specifically, **run the live probes first** (Bash), capture results
3. Render the page's content block (markdown — breadcrumb + body + any conditional info like probe results)
4. Call `AskUserQuestion` with the page's options
5. Wait for selection → loop

**Every page starts with the same breadcrumb**, with `✓` on completed pages and the current page bracketed as `[N.Name]`:

```
Step:  1.Overview  →  2.Prereqs  →  3.Flow  →  4.Start
```

Universal navigation semantics (apply to all pages):
- `next` → advance one page
- `back` → return to previous page
- `why`  → print the "Design rationale" block below, then re-render same page + picker
- `cancel` → abort, make zero changes
- `recheck` (Page 2 only, Variant B) → re-run probes, re-render Page 2

**Why `AskUserQuestion` over free-text:** picker can't be typo'd; options are always exactly the set the skill supports; the interaction boundary is clear (the user knows it's their turn). Especially useful in Variant B where the user is reading a detailed remediation block — they don't have to scroll back down to find which words to type. They just pick.

---

**Visual style — used on all 4 pages.** Keep the frame consistent across pages so the user has a predictable visual anchor. Template:

```
╭─ 1p-creds setup · N.PageName ─────────────── {optional status} ─╮

  [{breadcrumb with current page bracketed}]

  » {SECTION HEADING}
      Body indented 4 spaces from the »
      Bullets use ✓ / • / ▢ depending on meaning

  » {NEXT SECTION}
      …

╰──────────────────────────────────────────────────────────────────╯
```

Status suffix on the title is blank by default; `✓ all green` on Page 2 Variant A; `⚠ action needed` on Page 2 Variant B. Box width corner-to-corner ≈ 70 characters — fits cleanly in standard terminals. Section headings use `»` as a sigil for visual grouping without noisy dividers.

---

**Page 1 — Overview**

```
╭─ 1p-creds setup · 1.Overview ────────────────────────────────────╮

  [1.Overview]  ·  2.Prereqs  ·  3.Flow  ·  4.Start

  Migrating:  {repo} → 1P Environment {env}

  » WHAT THIS DOES
      Moves this repo's scattered credentials into a per-repo 1P
      Environment, mounted as a FIFO-served .env.local that direnv
      auto-loads on shell init.

  » WHAT YOU GET
      ✓  No plaintext credentials on disk
      ✓  1–2 biometric prompts per workday (vs ≈10–20 today)
      ✓  No wrappers — claude / codex / gemini / npm inherit env
      ✓  Committed manifest for rotation + audit

  » TIME
      ≈5 minutes, with two short GUI interactions in 1Password

╰──────────────────────────────────────────────────────────────────╯
```

Then call `AskUserQuestion` with options:
- `next`   — continue to prerequisites
- `why`    — show design rationale (why FIFO? why direnv?)
- `cancel` — abort, nothing changes

---

**Page 2 — Prerequisites** (breadcrumb: `✓1.Overview → [2.Prereqs] → 3.Flow → 4.Start`)

**On entry, immediately run the live check — don't print a static checklist.** A static list forces the user to verify mentally and then discover failures later in Phase A. Running probes first means the user sees exactly what's green and exactly what needs attention.

**Probes** (all safe, read-only, <1s total):

```bash
# 1P desktop running
pgrep -f '1Password.app' > /dev/null 2>&1 || pgrep -f '1password' > /dev/null 2>&1

# 1P CLI installed
command -v op > /dev/null 2>&1

# 1P CLI version ≥ 2.33
op --version 2>/dev/null | grep -qE '^2\.(3[3-9]|[4-9][0-9]|[1-9][0-9]{2,})'

# 1P CLI signed in + desktop integration enabled (both in one call)
op account list > /dev/null 2>&1

# direnv installed
command -v direnv > /dev/null 2>&1

# direnv shell hook in ~/.zshrc or ~/.bashrc
grep -q 'direnv hook' ~/.zshrc 2>/dev/null || grep -q 'direnv hook' ~/.bashrc 2>/dev/null

# Git working tree present
git rev-parse --show-toplevel > /dev/null 2>&1
```

Then render one of two variants based on results:

**Variant A — all probes green:**

```
╭─ 1p-creds setup · 2.Prereqs ──────────────────────── ✓ all green ─╮

  ✓1.Overview  ·  [2.Prereqs]  ·  3.Flow  ·  4.Start

  » LIVE CHECK (just ran, all passing)
      ✓  1Password desktop app running
      ✓  1Password CLI {detected-version} + signed in + integrated
      ✓  direnv {detected-version} installed
      ✓  direnv shell hook in {~/.zshrc or ~/.bashrc}
      ✓  Git working tree accessible ({repo-root})

  No action needed. Ready to continue.

╰───────────────────────────────────────────────────────────────────╯
```

Then call `AskUserQuestion` with options:
- `next`   — continue to flow overview
- `back`   — return to overview
- `cancel` — abort

**Variant B — one or more probes failed:** Show green items briefly at the top, then ONLY the failing items with precise install/enable/verify/docs. Don't bury the user in items that are already done.

```
╭─ 1p-creds setup · 2.Prereqs ──────────────────── ⚠ action needed ─╮

  ✓1.Overview  ·  [2.Prereqs]  ·  3.Flow  ·  4.Start

  » LIVE CHECK (just ran)
      ✓  1Password desktop app running
      ✗  1Password CLI — not installed or not on PATH
      ✓  direnv 2.37.1 installed
      ✗  direnv shell hook — not found in ~/.zshrc or ~/.bashrc
      ✓  Git working tree accessible

  » ACTION NEEDED  (fix each, then pick `recheck`)

      ▢  1Password CLI 2.33+
             Install:  brew install 1password-cli
             Enable:   1P Desktop → Settings → Developer →
                       ☑ "Integrate with 1Password CLI"
             Verify:   op --version && op account list
             Docs:     https://developer.1password.com/docs/cli/get-started/

      ▢  direnv shell hook
             Fix:      echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
             Then:     exec zsh   (or open a new terminal)
             Verify:   grep 'direnv hook' ~/.zshrc
             Docs:     https://direnv.net/docs/hook.html

╰───────────────────────────────────────────────────────────────────╯
```

Then call `AskUserQuestion` with options:
- `recheck` — I've fixed the above; re-run the live check
- `back`    — return to overview
- `cancel`  — abort

**On `recheck`** — re-run all probes and re-render this page. Each pass is idempotent; the user can `recheck` as many times as needed as they fix items. When all green, proceed via `next`.

**Action content per failing item** — populate Variant B from this table:

| Failing probe | Install | Enable / Configure | Verify | Docs |
|---|---|---|---|---|
| 1P desktop not running | `brew install --cask 1password` then launch the app | Sign in, unlock (Touch ID) | `pgrep -f 1Password.app` | https://1password.com/downloads/ |
| 1P CLI not installed | `brew install 1password-cli` | — | `op --version` | https://developer.1password.com/docs/cli/get-started/ |
| 1P CLI installed but < 2.33 | `brew upgrade 1password-cli` | — | `op --version` | https://developer.1password.com/docs/cli/upgrade/ |
| 1P CLI not signed in / not integrated | — | 1P Desktop → Settings → Developer → ☑ "Integrate with 1Password CLI" | `op account list` | https://developer.1password.com/docs/cli/app-integration/ |
| direnv not installed | `brew install direnv` | — | `direnv --version` | https://direnv.net/ |
| direnv hook not in rc | `echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc` (or `~/.bashrc` + `bash` variant) | `exec zsh` (or open a new terminal) | `grep 'direnv hook' ~/.zshrc` | https://direnv.net/docs/hook.html |
| Not in a git repo | `git init` (or `cd` into the correct repo) | — | `git rev-parse --show-toplevel` | https://git-scm.com/book/en/v2/Getting-Started-Installing-Git |

**Why automatic probe beats static checklist:** early real-world runs showed the pattern — users skim a static list, click `ready`, Phase A catches a miss (or doesn't), and there's a round-trip. With live probe on Page 2 entry, the user never has to verify manually; they either see `all green, ready` or `fix these specific items`. Less cognitive work, fewer round-trips, stronger signal that the skill knows what it's doing.

---

**Page 3 — The flow**

```
╭─ 1p-creds setup · 3.Flow ────────────────────────────────────────╮

  ✓1.Overview  ·  ✓2.Prereqs  ·  [3.Flow]  ·  4.Start

  » YOU DO TWO THINGS  (≈3 minutes total)

      Phase D — 1Password Desktop, ≈2 min
          Create Environment, drag import file, mount destination.
          Skill gives click-by-click with macOS shortcuts.

      Phase G — fresh terminal, ≈1 min
          `claude` inherits env via direnv. Confirm /mcp shows ✓.

  » THE SKILL DOES  (auto)

      A: verify prereqs       F: verify FIFO serves values
      B: scan for creds       H: strip plaintext, cleanup, commit
      C: build import +       I: offer to delete stale vault items
         draft manifest       J: print completion report
      E: write .envrc + direnv allow

  » SAFETY

      •  Destructive actions reversible (1P trash 30d / git revert)
      •  Temp /tmp/{env}-import.env mode 600, shredded Phase H
      •  Only migration files touched — other changes untouched

╰──────────────────────────────────────────────────────────────────╯
```

Then call `AskUserQuestion` with options:
- `next`   — go to final start page
- `back`   — return to prereqs
- `cancel` — abort

---

**Page 4 — Start**

```
╭─ 1p-creds setup · 4.Start ───────────────────────────────────────╮

  ✓1.Overview  ·  ✓2.Prereqs  ·  ✓3.Flow  ·  [4.Start]

  » WHEN YOU PICK `go`

      1.  Phase A runs — verify prereqs one more time
      2.  If any missing: skill STOPS and shows the exact install
          line so you can fix and re-run /1p-creds setup
          (no halfway-installs mid-flow)
      3.  If all pass: Phase B scan begins. No more input needed
          until Phase D (1P GUI batch).

╰──────────────────────────────────────────────────────────────────╯
```

Then call `AskUserQuestion` with options:
- `go`     — start — Phase A runs next
- `back`   — return to flow overview
- `cancel` — abort

On `go`, transition to Phase A. On `cancel` at any point, exit cleanly without touching anything.

---

**Design rationale** (print on `why`; then re-show the current page):

```
WHY FIFO (not a real .env file)?
  A FIFO (named pipe) has no bytes at rest. The 1P desktop app
  streams current secret values on each read. If the laptop is
  stolen cold, there's nothing to exfiltrate from the filesystem.

WHY direnv (not a claude/codex/gemini wrapper)?
  Wrapping breaks interactive TTYs and means every CLI tool needs
  its own wrapper. Setting env in the shell lets every inheriting
  process (claude, codex, gemini, npm, curl) work naturally.

WHY `source` instead of direnv's `dotenv`?
  direnv's dotenv builtin does [ -f file ] which returns false for
  FIFOs. Shell `source` handles FIFOs natively.

WHY not op run -- claude?
  op run wraps stdio, which breaks claude's TTY detection — it flips
  Claude into --print mode. Same for codex and gemini.

WHY not the 1P shell plugin?
  Biometric-per-call. Incompatible with autonomous agents that
  fire many op invocations per session.

WHY /tmp for the import file?
  Self-cleans at reboot. Outside any git-tracked path. Unix
  convention for transient files. Phase D uses `open -R` to reveal
  it in Finder for the drag, so the location isn't a UX cost.
```

---

**Why paged, not one block?** Early runs showed users reading a wall of preamble text scrolled past the parts they needed (prereq list, safety notes). Pages force attention to each chunk before the next. The breadcrumb gives a permanent mental map — the user always knows where they are and how far is left.

**If the user deviates from the reply vocabulary** (e.g. types "ok" instead of "next"), interpret generously (treat "ok"/"yes"/"continue"/"y" as `next`; treat "no"/"stop"/"quit" as `cancel`). But on ambiguity, ask: "I heard X — did you mean `next` or something else?"

**If `go` comes and Phase A detects a missing prereq**, do NOT fall back to prompting or partial install. Print exactly which prereq is missing, paste the install line from Page 2, and stop. The user re-runs `/1p-creds setup` when ready. This avoids the halfway-installed mid-flow failure.

#### Phase A — Pre-flight (auto)

Check: 1P desktop running, `op` ≥ 2.33, `direnv` installed + hooked, `.gitignore` covers `.env*`. Print status table. If anything missing, offer exact fix (`brew install …`) and wait for user yes/no *once*.

#### Phase B — Scan (auto)

Run all detection layers from `references/migrate.md`. Output: categorized table of findings + auto-classification column + "used in" cross-reference to `.mcp.json` / scripts / hooks. **No questions.**

#### Phase C — Build + Layer 5 security sweep (auto)

Generate three artifacts AND run the value-based security scan:

1. `/tmp/{env}-import.env` (mode 600) — consolidated `KEY=VALUE` for drag-import into 1P
2. `.claude/creds-manifest.yaml` — draft with `system`, `admin_url`, `classification`, `legacy_source`/`vault_reference` populated where inferable; `# TODO: verify` annotations on uncertain values
3. One-page summary of what's going where

**Layer 5 — value-based cross-repo sweep (SECURITY-CRITICAL).** After resolving values for the import file, grep each resolved value (20+ chars to avoid false positives) across every text file in the repo, excluding `.git`, `node_modules`, `.trees`, `tmp`, and the `/tmp/*-import.env` file itself. Any hit is a **copy site** — a plaintext duplicate of a credential that Layers 1–4 didn't catch because it lives in a `.py`, `.md`, `.sh`, or other non-config file. Common patterns: module-level `TOKEN = "..."` constants, tokens quoted in learning docs, Bearer headers in code examples. See `references/migrate.md` § Layer 5 for the full rationale + scanner spec.

If Layer 5 hits: include each copy site in the Phase C summary as a follow-up that needs Phase H-style rewriting (to `${VAR}` references). Do NOT block Phase D on this — migration can proceed and the user can rewrite copy sites in parallel or after.

Print the manifest draft + any Layer-5 copy sites to the user. They can edit the manifest between now and Phase D if they disagree with any inference.

#### Phase D — User GUI batch (ONE intervention)

Print this exact three-step recipe, with the two macOS tricks inline. Do NOT omit the `open -R` or `Cmd+Shift+G` instructions — they're the difference between this phase being 30 seconds and being 3 minutes of clicking through file dialogs.

```
=== Phase D — 3 steps in 1Password Desktop ===

STEP 1 — Create the Environment
  1Password Desktop → sidebar: Developer → Environments → + New environment
  Name it exactly:  {env}

STEP 2 — Import the variables (drag a file in)
  In this terminal, run:
      open -R /tmp/{env}-import.env
  → Finder opens with the import file already selected.
  In 1Password → Variables tab → drag that selected file onto the drop zone.
  Click Save.
  Expect: all {N} variables appear with placeholders (dots) for values.

STEP 3a — Configure the destination path
  Destinations tab → "Local .env file" card → Configure destination.
  In the native "Select .env file destination" dialog:
      1. Press Cmd+Shift+G  (macOS "Go to folder" shortcut)
      2. Paste:  {repo-root}
      3. Press Enter
      4. File name field:  .env.local    (usually pre-filled; confirm it)
      5. Click Select
  ⚠ macOS will warn: "Are you sure you want to use a name that begins
     with a period '.'? Finder hides files that begin with a period."
     → Click  Use "."  to confirm. This warning is normal for dotfiles.

STEP 3b — ⚠ TOGGLE THE DESTINATION TO ENABLED  ← easy to miss
  Back in the Destinations tab, you should now see the "Local .env file"
  card showing the path you just set. There's a toggle at the top-right
  of the card.

      ►►►  FLIP THE TOGGLE TO ENABLED  ◄◄◄

  Without this flip, 1P does NOT create the FIFO at the path. Phase F
  will fail with "No FIFO exists at expected path" and the skill will
  ask you to come back and enable it. Save yourself the round-trip:
  toggle it now, before saying "done".

Say "done" when all three steps (1, 2, 3a, 3b) are complete AND the
destination card reads Enabled.
```

**Do not ask any questions during this phase.** Wait for "done."

**macOS tricks this recipe depends on:**

- `open -R <path>` reveals a file in Finder with it already selected — makes Step 2's drag a single gesture instead of a navigation hunt. Without this, the user has to open a Finder window, Cmd+Shift+G, type `/tmp`, find the file, then drag.
- `Cmd+Shift+G` inside any native file dialog opens the "Go to folder" prompt — paste an absolute path, press Enter, and the dialog jumps there. Without this, the user clicks through sidebar favorites (which may not include `~/team`) or drills down through `Users/{you}/team/{repo}` manually.

**Why the temporary file + GUI drag, not a CLI command?**

Include this paragraph under the recipe so the user understands why we're briefly writing plaintext to disk:

> **Why a temp file:** 1Password Environments is still in beta, and the `op` CLI does not yet support adding variables or configuring destinations programmatically — only reading. Until that API ships, the only way to populate an Environment is the GUI drag-and-drop importer. That's why we build `/tmp/{env}-import.env` (mode 600, owner-only) and ask you to drag it in.
>
> **Why `/tmp` specifically:** it auto-cleans on reboot, lives outside any git-tracked path, and matches the Unix convention for transient files. `open -R` above makes the drag step as fast as a file on your Desktop without the cleanup risk.
>
> **Is this adding risk?** No. Every value in that temp file was already plaintext on this same disk — inside `.claude/settings.local.json`, inside `credentials/*.env`, or resolvable via `op://` refs that any process with your 1P session could read. We're consolidating existing plaintext into one short-lived file, not introducing new secrets to disk. After the import, Phase H shreds the temp file and strips the original plaintext locations. Net plaintext on disk *decreases* as a result of this step — it doesn't increase.
>
> The file will also disappear from `/tmp/` at next reboot regardless; it's explicitly removed in Phase H regardless of where we are in the flow.

This framing matters: without it, a careful user mid-flow can reasonably think "you spent the whole skill avoiding plaintext on disk, and now you're writing it to /tmp?" The paragraph pre-empts that concern with the concrete reason.

**Why the temporary file + GUI drag, not a CLI command?**

Include this paragraph under the recipe so the user understands why we're briefly writing plaintext to disk:

> **Why a temp file:** 1Password Environments is still in beta, and the `op` CLI does not yet support adding variables or configuring destinations programmatically — only reading. Until that API ships, the only way to populate an Environment is the GUI drag-and-drop importer. That's why we build `/tmp/{env}-import.env` (mode 600, owner-only) and ask you to drag it in.
>
> **Is this adding risk?** No. Every value in that temp file was already plaintext on this same disk — inside `.claude/settings.local.json`, inside `credentials/*.env`, or resolvable via `op://` refs that any process with your 1P session could read. We're consolidating existing plaintext into one short-lived file, not introducing new secrets to disk. After the import, Phase H shreds the temp file and strips the original plaintext locations. Net plaintext on disk *decreases* as a result of this step — it doesn't increase.
>
> The file will also disappear from `/tmp/` at next reboot regardless; it's explicitly removed in Phase H regardless of where we are in the flow.

This framing matters: without it, a careful user mid-flow can reasonably think "you spent the whole skill avoiding plaintext on disk, and now you're writing it to /tmp?" The paragraph pre-empts that concern with the concrete reason.

#### Phase E — Wire (auto)

- Write `.envrc` with the **guarded FIFO pattern** (see below)
- Ensure `.gitignore` has `.env`, `.env.local`, `.env.*.local`
- `direnv allow` in the repo
- Print the single line needed to verify: `env | grep {FIRST_VAR}` from a new shell

**`.envrc` template — use exactly this, including the guard and the read-loop:**

```bash
# .envrc — managed by the 1p-creds skill
# .env.local is a FIFO served on-demand by the 1Password desktop app.
# See .claude/creds-manifest.yaml for the variables this loads.
#
# Why a read loop, not `source` or direnv's `dotenv` builtin:
#   - dotenv does [ -f file ] which returns false for FIFOs (named pipes)
#   - `source .env.local` re-parses values as shell — values containing
#     `|`, `$`, backticks, spaces, etc. trigger command-not-found errors
#     (e.g., a Coolify token shaped like "2987|tokenvalue" pipes into a
#     non-existent command). `export "KEY=$value"` treats the value as
#     a literal string.
#
# Why the guard: tolerate .env.local being absent (1P quit, destination
# toggled off, first run before setup completes). Without the guard,
# direnv would print a noisy error on every shell init in those states.

if [ -p "$PWD/.env.local" ] || [ -f "$PWD/.env.local" ]; then
  while IFS='=' read -r key value; do
    case "$key" in
      ''|'#'*|*' '*) continue ;;
    esac
    # Strip optional surrounding single or double quotes
    case "$value" in
      \"*\") value="${value:1:${#value}-2}" ;;
      \'*\') value="${value:1:${#value}-2}" ;;
    esac
    export "$key=$value"
  done < "$PWD/.env.local"
fi
```

Two guards, two reasons:

1. **Outer `[ -p … ] || [ -f … ]`** — handles absent mount (1P desktop quit, destination disabled, first boot before setup). Without it, direnv spews errors every shell init.
2. **`while read` loop instead of `source`** — handles shell-metacharacter-safe assignment. Without it, tokens containing `|`, `$`, `` ` ``, `&`, `(`, `)`, `;` get parsed as script syntax by `source` and either truncate or produce "command not found" noise at shell init. Discovered in the wild on a Coolify token containing a `|` pipe.

The quote-stripping cases (`\"*\"` and `\'*\'`) are defensive — 1P's FIFO output is unquoted, but other import paths (manual `.env` files, future 1P format changes, variants) may quote values.

#### Phase F — Verify FIFO (auto)

Read `.env.local` **once** into a variable (FIFOs are one-shot per read). Assert all N expected variables present with lengths matching what was in the import file. Block on mismatch with specific var + expected-vs-actual length.

**Do NOT use `timeout` in the read command.** macOS doesn't ship GNU `timeout` by default (it's in `coreutils` as `gtimeout`); unconditional use causes `command not found`. The read is fast when 1P is unlocked (< 100ms) — no timeout needed. If you want defense-in-depth against a hung FIFO, use a pure-bash pattern:

```bash
# Portable — works on macOS and Linux without coreutils
content=$(cat "$PWD/.env.local" &)
```

But in practice, Phase A already verified 1P is running + unlocked, so reads will return promptly.

#### Phase G — User E2E (ONE intervention)

Print:

```
Open a fresh VS Code integrated terminal in this repo.
Expect: `direnv: loading .envrc` + export list of all {N} vars.
Run: claude
Inside: /mcp
Expect: all token-dependent servers ✓ connected.

Say "green" or paste the /mcp output.
```

Wait for user. No other questions.

#### Phase H — Cleanup (auto on "green")

**File-level aware, not folder-level.** This is the critical safety principle: only delete files that Phase B explicitly identified as migrated credential sources. **Never nuke a folder wholesale** — folders contain heterogeneous content (SSH keys, certs, unrelated configs) and a `rm -rf` is how collateral damage happens.

Operations, in order:

- **Strip ONLY the `env` block** from `.claude/settings.local.json` — **the FILE STAYS on disk** with its other Claude Code settings intact (`permissions`, `enableAllProjectMcpServers`, `enabledMcpjsonServers`, `disabledMcpjsonServers`, `statusLine`). Block-level JSON edit, NOT a file deletion.

- **Delete specific files that Phase B confirmed as migrated credential sources**, one by one. Typical list:
  - `credentials/*.env.op` (if present and its `op://` refs were resolved + migrated)
  - `credentials/platform-load.sh` / `credentials/*-load.sh` (legacy loader scripts)
  - `.env`, `.env.op` at repo root (only if ALL their variables migrated — if any variables remain unmigrated, leave the file and report)
  - Any other file Phase B listed as a "layer 1 plaintext-on-disk" source

- **Never delete files flagged as DO-NOT-MIGRATE** (SSH keys, certs, `.p12`, `.kubeconfig`, etc. — see `references/migrate.md` § DO-NOT-MIGRATE patterns). These stay untouched regardless of which folder they live in.

- **After per-file deletions, check if `credentials/` (or any similar folder) is now empty.** If empty → `rmdir` it. If non-empty → leave the folder in place and include its remaining contents in Phase J follow-ups. Usually remaining content is SSH keys, certificates, or other non-migratable assets.

- **Shred `/tmp/{env}-import.env`** — full deletion of the temporary import file.

- **Rewrite any Layer-5 copy sites** flagged in Phase C's summary. For each: replace the hardcoded value with a `${VAR_NAME}` reference or `os.environ['VAR_NAME']` / equivalent language construct. Include these edits in the commit.

- **Stage + commit the migration files** (scoped — don't touch unrelated `M` files). The commit includes: `.envrc`, `.gitignore` (if changed), `.claude/creds-manifest.yaml`, any files rewritten for Layer-5 copy sites, and any doc updates (CLAUDE.md / READMEs that still reference the old launch pattern).

**The folder-level-nuke-is-unsafe rule was learned from a real incident:** an early run had an SSH deploy key in `credentials/` alongside the plaintext tokens. The skill's old Phase H did `rm -rf credentials/`, which took out the SSH key as collateral. The key was restored from backup but the incident exposed the failure class. Phase H is now file-aware + guarded by Layer 1's DO-NOT-MIGRATE detection.

#### Phase I — Offer `decommission-legacy` (optional destructive step)

If manifest has any `legacy_source:` entries: print "Run decommission-legacy now? This deletes N vault items (recoverable from 1P trash for 30 days)." Wait for explicit yes. On yes → run the action + commit manifest strip.

#### Phase J — Completion report (auto, always runs)

The report is the last thing the user sees. Make it feel like a small celebration — the user just moved N credentials off disk into a safer home, and they earned a clean, polished summary. Structure:

1. A banner with a small furry animal (the bunny below) delivering the top-line stat
2. The "moved from / to" hero stat — **most important number front and center**
3. Detail sections (what changed, risks mitigated, biometric behavior, verify path, day-to-day, follow-ups)

**Substitute all `{placeholders}` with real values from this run** — no unresolved braces. Width ≈ 72 chars.

```
╭─ 1p-creds setup · ✓ complete ───────────────────────────────────────╮

                       (\(\
                       ( -.-)        Nicely done! {N} credential{s}
                       o_("_)(_)     are now tucked safely inside
                                     your 1Password vault.

  » THE HEADLINE

      {N} credential{s} removed from disk  →  stored in 1P Environment
      `{env}` (mounted at {mount-path})

      {K} plaintext entries stripped from .claude/settings.local.json
      {M} files deleted from credentials/ (folder removed)
      {P} Layer-5 copy sites rewritten to use `${VAR}` references
      {Q} legacy vault item{s} deleted from {vault-name}/  (if any)

  » WHAT CHANGED

      Created   {repo-root}/.envrc
      Created   {repo-root}/.claude/creds-manifest.yaml
      Modified  {repo-root}/.gitignore         (+3 env patterns)
      Stripped  {repo-root}/.claude/settings.local.json  env block
                (FILE stays; the `env` block is gone — permissions
                 and other native Claude settings preserved intact)
      Deleted   {repo-root}/credentials/       ({M} files)
      Shredded  /tmp/{env}-import.env
      Commits   {hash-1}  {short-summary}
                {hash-2}  {short-summary}   (if applicable)

  » RISKS MITIGATED

      Plaintext credentials on disk   :  {before}  →  0
      Biometric prompts per workday   :  ~10–20   →  ~1–2
      Credential source of truth      :  scattered ({X} locations)
                                         →  {env} 1P Environment
      Secrets in settings.local.json  :  {before}  →  0
      Secrets in credentials/ folder  :  {before}  →  0  (folder gone)

  » BIOMETRIC BEHAVIOR NOW

      •  First read of .env.local after 1P unlocks  →  ONE Touch ID
      •  Every subsequent read (any shell, any tool) →  silent until
                                                        1P locks
      •  Typical workday                             →  1–2 prompts
      •  If 1P is fully quit                         →  reads block
                                                        until 1P restarts

  » MANUALLY VERIFY IN 1PASSWORD DESKTOP

      1.  Sidebar → Developer → Environments → {env}
      2.  Variables tab — {N} rows; click ⋯ → "Show value by default"
          to reveal; edit inline, next FIFO read returns the new value
      3.  Destinations tab → "Local .env file" card
          •  Path:    {mount-path}
          •  Toggle:  Enabled (green)
      4.  Manage environment (top-right) → Export — one-click backup

  » DAY-TO-DAY

      Rotate:     Skill("1p-creds", "rotate {VAR_NAME}")
      Validate:   Skill("1p-creds", "validate")
      Inventory:  Skill("1p-creds", "inventory")
      Migrate:    Skill("1p-creds", "migrate")  (re-scan for new scatter)

  » FOLLOW-UPS  (your call whether / where to track these)

      {Only include lines that apply to this run. Emit as a clean list;
       do NOT save to memory, task trackers, or external note systems.
       The user decides where persistence goes. Candidate lines:}

      ⚠  Manifest entries with `# TODO: verify`:  {count}
         (edit .claude/creds-manifest.yaml when convenient)

      ⚠  1P item still referenced (vault_reference):
         {item-path} — intentionally kept (holds identity anchors);
         decommission-legacy will never offer to delete it

      ⚠  Fleet-wide tokens needing coordinated rotation:
         {list of vars with scope: fleet-wide}

      ✓  Vault item(s) deleted:
         {list} — recoverable from 1P trash for 30 days

      ⓘ  Layer 5 rewrote {N} copy sites in this commit:
         {file list}

      ⓘ  Layer 6 found stale doc references (not auto-rewritten):
         {file:line list} — update when you next touch those docs

      ⓘ  SSH keys / non-migratable files preserved:
         {list of paths detected by Layer 1 DO-NOT-MIGRATE scan}
         Left in place. SSH keys can't be migrated to 1P Environments
         (file-based, not env-var-based). Consider 1Password SSH agent
         (https://developer.1password.com/docs/ssh/) if you want them
         managed. If this list is non-empty and {credentials/} folder
         was NOT auto-removed, that's why.

      ⓘ  Newly-visible failures may surface. Migrating away from
         plaintext often removes one suspect from ambiguous errors;
         anything that was flaky before may now fail loudly with
         its real cause (stale URLs, expired certs, removed endpoints).

╰─────────────────────────────────────────────────────────────────────╯
```

**Rules for generating the bunny banner:**
- Pluralize `credential` vs `credentials` based on N ({N} == 1 → singular).
- The bunny ASCII is load-bearing for the vibe — keep it exactly as shown. Small, familiar, low-token. It's the skill's signature.
- If `N` is unusually high (say > 20), the bunny's line still fits: "N credentials are now tucked safely inside your 1Password vault."
- Substitute `{s}` with `s` when N > 1, blank when N == 1.

**Do NOT auto-save follow-ups to memory, task trackers, or external note systems.** The skill's job is to emit the list cleanly. Persistence is the user's call — different users use different systems.

Print the report verbatim with real substitutions. This is what makes setup *feel done* — the user sees concretely what's now true about their repo, how many secrets they just locked up, and a small furry animal congratulating them.

### migrate

Standalone invocation of Phases A–C from setup. Produces the import file + manifest draft, prints summary, exits. Use when a repo already has an Environment mounted and you want to pull in newly-scattered creds (e.g., someone added a plaintext token to `settings.local.json` after the initial setup).

See `references/migrate.md` for the detection rules and the auto-classification table.

### rotate

```
/1p-creds rotate N8N_API_KEY
```

1. Read `.claude/creds-manifest.yaml` → extract `system`, `admin_url`, optional `account`
2. WebSearch: `how to rotate API key in {system} {current-year}` scoped to `admin_url`'s domain
3. Synthesize concise rotation steps with cited sources
4. Offer to `open` admin_url in browser
5. Wait for user to generate new value and paste it
6. Update variable in 1P Environment (user pastes in GUI — can't do via CLI in beta)
7. `cat .env.local` via FIFO, confirm new value serves
8. If `last_rotated` tracking enabled in manifest, update timestamp

### validate

Fast health check. Returns a table of pass/fail for:
- `~/Applications/1Password.app` running
- FIFO at expected path is a named pipe (not a regular file)
- `cat {fifo}` returns non-empty content within 5s
- Every `${VAR}` in `.mcp.json` has a matching entry in the Environment
- `.envrc` is allowed (check `direnv status`)
- Every Environment variable has a manifest entry (flag "silent creds")

### decommission-legacy

Post-migration cleanup for variables that came from 1P vault items (not from plaintext files). Reads `legacy_source:` fields from the manifest and offers to delete the originating items.

Flow:
1. Collect all `legacy_source` values from `.claude/creds-manifest.yaml`
2. Parse each `op://vault/item/[section/]field` — dedupe at `{vault, item}` level
3. For each unique item, show: which manifest variables reference it, the item's metadata from `op item get`
4. Safety check: compare field lengths in the vault item against current Environment values. Block with a warning if they diverge ("vault still has different data — migration may not be complete")
5. On user confirmation: `op item delete <item> --vault <vault>`
6. Strip matching `legacy_source` fields from the manifest (idempotent — next run finds nothing)

**When to run:** only after E2E verified (new shell → `claude` → `/mcp` all ✓). Never before. The vault item is your rollback point.

### inventory

```
/1p-creds inventory
```

Renders the manifest + cross-referenced usage:

```
Variable             System           Scope        Used in
API_KEY              acme             repo         .mcp.json (acme-mcp)
TEAM_SHARED_TOKEN    acme-webhook     fleet-wide   .mcp.json (4 servers)
DB_TOKEN             postgres         repo         .mcp.json (db-mcp), scripts/deploy.sh
...
```

## Manifest: `.claude/creds-manifest.yaml`

Per-repo sidecar. Safe to commit — no secrets, only metadata. Schema + worked examples: `references/manifest-schema.md`.

## Security invariants

- **No plaintext secrets on disk** — `.env.local` is always a FIFO served on-demand by the desktop app
- **No service account tokens on developer workstations** — service accounts belong in hosted/cloud contexts only; developer machines gate credentials via biometric on the desktop 1P app
- **No wrapping of `claude` / `codex` / `gemini`** — env gets into the shell via direnv, then inherits normally
- **Manifest contains no secrets** — only system names, admin URLs, optional scope/account metadata

## Harness adapters

Current state and coverage:

| Harness | Status | Reference |
|---|---|---|
| Claude Code | Implemented | `references/harness-claude-code.md` (covers `.mcp.json` `${VAR}` expansion + direnv composition) |
| Codex | TBD | `references/harness-codex.md` (TODO when first Codex repo needs creds) |
| Gemini CLI | TBD | `references/harness-gemini-cli.md` (TODO) |

The FIFO + direnv pattern is harness-agnostic — only the "how the harness reads env vars" differs.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `cat .env.local` hangs indefinitely | 1P desktop app is quit. Launch it. (FIFO has no writer; reader blocks forever.) |
| `cat .env.local` returns "Environment is empty" placeholder | Variables not yet added to the Environment in 1P GUI, OR destination is toggled Disabled |
| Biometric prompts every shell | direnv not hooked. Check `grep direnv ~/.zshrc`, then new shell |
| Biometric every `op` call (not FIFO) | You're using the old `op inject` path, not the FIFO mount. Stop the launcher script from calling `op`; rely on `direnv` + `.env.local` instead |
| `/mcp` shows ✗ on a server expecting `${VAR}` | Shell didn't have the var when you ran `claude`. Confirm: new shell → `env \| grep {VAR}`. If empty, run `direnv reload` or `direnv allow` |
| direnv says "Disallowed" | `cd` into repo → `direnv allow` — a security gate on first use of every `.envrc` |
| direnv says `.env at .env.local not found` but the file exists | You used `dotenv .env.local` in `.envrc`. direnv's `dotenv` builtin does `[ -f file ]` which returns false for FIFOs (named pipes). Use `set -a; source "$PWD/.env.local"; set +a` instead — shell `source` handles FIFOs natively. |
| Second `cat .env.local` in the same pipeline blocks or returns garbled output | FIFOs are **one-shot per read**. Each `cat` triggers a fresh render from 1P; you can't read twice from the same open. Read once into a variable, then iterate: `content=$(cat "$PWD/.env.local"); echo "$content" \| while IFS='=' read -r k v; do ...`. Don't chain `cat` calls or use `... \| tee` to split the stream. |
| Mount shows in 1P but FIFO doesn't exist on disk | The destination toggle is off (Configured ≠ Enabled). Go to 1P → Developer → Environments → {env} → Destinations tab → flip the "Local .env file" toggle to Enabled. See Phase D step 3b. |
| Variable shows in 1P but not in FIFO | Hit Save in the Environment variable form (edits are pending until saved) |
| 1P Environment locked to personal scope | Known limitation in current beta. Share-vault-scoped Environments not yet available |

## References

- `references/migrate.md` — detection patterns for scattered tokens
- `references/manifest-schema.md` — `.claude/creds-manifest.yaml` schema + examples
- `references/harness-claude-code.md` — Claude-Code-specific wiring details
- [1Password Environments docs](https://developer.1password.com/docs/environments/)
- [1Password local .env mount](https://developer.1password.com/docs/environments/local-env-file/)
- [1Password agent hooks (optional validation layer)](https://developer.1password.com/docs/agent-hooks)
- [direnv docs](https://direnv.net/)

## When to invoke

- New repo onboarding — run `setup`
- Repo has scattered plaintext tokens — run `migrate`
- Rotating any credential — run `rotate <VAR>`
- MCP connection failure involving `${VAR}` expansion — run `validate`
- Audit / documentation pass — run `inventory`

## Not covered by this skill

- Creating the underlying secret at the upstream service (Directus user, n8n factory account, etc.)
- Scoped service account tokens for cloud-hosted workloads (see `references/cloud-provisioning.md` when it lands)
- Secret material rotation at the issuing service (skill guides via web search but does not automate)
