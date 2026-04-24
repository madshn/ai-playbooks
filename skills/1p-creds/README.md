# 1p-creds

Move a repo's scattered credentials into a per-repo **[1Password Environment](https://developer.1password.com/docs/environments/)**, mounted as a FIFO-served `.env.local` that **[direnv](https://direnv.net/)** auto-loads on shell init. Cut biometric prompts from 10–20 per workday to 1–2, without wrapping the agent CLI.

## The problem

If you use 1Password CLI (`op`) to inject secrets into shells that run AI coding agents (`claude`, `codex`, `gemini`, etc.), you probably have one of these patterns:

1. **`source credentials/loader.sh && claude`** — one biometric prompt per shell. Fresh terminals get fresh prompts; VS Code integrated terminals compound this into 10–20 prompts per workday.
2. **Plaintext tokens in `.claude/settings.local.json`** or similar — no prompts, but secrets sit unencrypted on disk.
3. **`op run -- claude`** — breaks interactive TTYs (`claude` flips into `--print` mode).
4. **A wrapper around `claude`** — couples credential plumbing to agent launch; often rejected for that reason.

All four have downsides. This playbook picks a different architecture.

## The solution

**[1Password Environments](https://developer.1password.com/docs/environments/)** (currently in beta) provide a destination type called **["Local `.env` file"](https://developer.1password.com/docs/environments/local-env-file/)** that mounts as a **FIFO** (named pipe). The file looks like a regular file to any reader, but contains no bytes at rest — the 1Password desktop app streams the current variable values into the pipe on demand, and authorization is gated by the **1P desktop unlock state**, not per-shell.

Paired with **direnv**, which auto-sources `.env.local` on shell init (no `cd` needed — works in VS Code integrated terminals), the result is:

- **Zero plaintext credentials on disk** — the "file" holds no bytes
- **One biometric per 1P unlock cycle** — covers every subsequent shell, process, tool
- **No CLI wrapping** — `claude` / `codex` / `gemini` / npm scripts all inherit env from the shell
- **A committed manifest** (`creds-manifest.yaml`) pairing each variable with its upstream service, admin URL, and (for identity-anchored credentials) its vault reference — enables guided rotation, validation, and audit

## What the playbook does

The `setup` action is an 11-phase deterministic flow with only 2 user interventions:

| Phase | What happens |
|---|---|
| **0** | Paged preamble wizard (4 pages) — overview, live prerequisite probe, flow summary, start confirmation |
| **A** | Re-verify prereqs at run time — fail-stop on missing rather than half-installing mid-flow |
| **B** | Scan the repo for scattered credentials across 6 detection layers (plaintext env blocks, legacy `.env.op` refs, pattern-matched tokens, `${VAR}` references, **value-based sweep across all text files**, doc references) |
| **C** | Build a consolidated import file + draft manifest; run the security-critical Layer 5 cross-repo value sweep |
| **D** | **User intervention #1:** drag the import file into 1Password → configure destination → toggle Enabled. Skill gives click-by-click with macOS shortcuts. |
| **E** | Auto-wire `.envrc` + `.gitignore` + `direnv allow` |
| **F** | Verify the FIFO serves all expected variables with correct lengths |
| **G** | **User intervention #2:** open a fresh terminal → run `claude` (or your harness) → confirm the env loads + dependent MCP servers connect |
| **H** | Auto-cleanup: strip plaintext from `.claude/settings.local.json`, delete legacy `credentials/` folder, shred the temp import file, rewrite any Layer-5 copy sites, commit |
| **I** | Offer to delete stale vault items via `decommission-legacy` (destructive — explicit confirm) |
| **J** | Celebratory completion report with a small bunny 🐰 and a tally of how many credentials are now tucked safely in 1Password |

Other actions: `migrate` (re-scan for new scatter), `rotate VAR_NAME` (AI+web-searched instructions), `validate` (mount health check), `inventory` (tabulate all creds), `decommission-legacy` (cleanup of stale vault items).

## Variants

| Harness | Status | Folder |
|---|---|---|
| **Claude Code** | ✅ Implemented | [`claude/`](./claude/) |
| Codex, Gemini CLI, others | Not yet ported | — |

Ports welcome as PRs. The underlying architecture (FIFO mount + direnv + manifest) is harness-agnostic; only the invocation mechanism differs. See `claude/SKILL.md` as the reference implementation.

## Security notes

- **Net plaintext on disk decreases.** The migration temporarily writes a consolidated import file to `/tmp` (mode 600, shredded in Phase H), but every value in it was already plaintext elsewhere on the same disk. End state: less plaintext than before.
- **Reversible.** Vault item deletions land in 1P's Recently Deleted for 30 days. Commits are `git revert`-able. Destination mount can be toggled off to stop serving.
- **Does not touch unrelated files.** Commits are scoped to the migration artifacts — your other uncommitted changes stay as they are.

## Prerequisites

- 1Password desktop app (Mac or Linux), signed in, unlocked
- 1Password CLI ≥ 2.33, with desktop integration enabled
- `direnv` with shell hook in `~/.zshrc` or `~/.bashrc`
- A git working tree (any state)

The setup flow's Phase 0 wizard verifies all of these automatically — if anything's missing you'll see the exact install line before it proceeds.

## Not covered

- **Windows** — 1P Environments' local `.env` destination is Mac/Linux only
- **Cloud/hosted headless runs** — use the 1P SDK + a service account token for that; this playbook targets developer workstations where service accounts would be a security downgrade
- **Creating the upstream credentials** — if you don't have a token yet, go generate one at the service first

## References

### 1Password

- **[1Password Environments](https://developer.1password.com/docs/environments/)** — the feature this skill is built on (currently in beta)
- **[Local `.env` file destination](https://developer.1password.com/docs/environments/local-env-file/)** — the FIFO mount architecture that makes the whole thing work
- **[Programmatically read Environments](https://developer.1password.com/docs/environments/read-environment-variables/)** — `op environment read` + SDK access for headless contexts (hosted workers, CI)
- **[1Password CLI (`op`)](https://developer.1password.com/docs/cli/)** — install + general usage
- **[1P CLI desktop-app integration](https://developer.1password.com/docs/cli/app-integration-security/)** — how the biometric-gated trust model works
- **[1Password Agent Hooks](https://developer.1password.com/docs/agent-hooks)** — optional validation layer that can verify the `.env.local` mount before an agent tool runs (Claude Code, Cursor, etc.)
- **[1Password SSH Agent](https://developer.1password.com/docs/ssh/)** — the right home for SSH private keys, which this skill explicitly does **not** migrate (see `claude/references/migrate.md` § DO-NOT-MIGRATE patterns)
- **[1Password desktop app downloads](https://1password.com/downloads/)** — prerequisite

### Other tools

- **[direnv](https://direnv.net/)** — the per-directory env auto-loader this skill composes with
- **[`man fifo`](https://linux.die.net/man/7/fifo)** — Unix named pipes, the architectural primitive 1P Environments use under the hood

## License

MIT — see [`../LICENSE`](../LICENSE).
