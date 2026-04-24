# Migrate detection patterns

Scan a repo (and the files it's known to access) to find credentials that should move into the 1P Environment. Output is always a triage report + a consolidated import file.

## Detection heuristics

### Layer 1 — Known plaintext-on-disk locations (cast a wide net)

**Goal: find all keys in scope.** Different repos use different naming conventions for credential storage. Don't assume `credentials/` — scan by name-pattern broadly and let false positives surface in the Phase B summary (cheap to skip manually). False negatives — missed plaintext — are the expensive failure mode. Err toward over-inclusion.

**Directory-name patterns** — scan all files within, any extension:

| Directory name (regex) | Typical contents |
|---|---|
| `credentials/`, `creds/` | `.env`, `.env.op`, `.json`, `.sh`, `.txt` — mix of op:// refs, plaintext tokens, loader scripts |
| `secrets/`, `.secrets/` | Similar mix |
| `keys/`, `.keys/` | Often SSH keys, API key dumps |
| `auth/`, `.auth/` | Token bundles |
| `.env.d/` (convention in some frameworks) | Split dotenv files |

**File-basename patterns** (any directory):

| Pattern | Examples | What to expect |
|---|---|---|
| `.env`, `.env.*` (not `.env.local` if already mounted) | `.env`, `.env.production`, `.env.prod` | Classic dotenv plaintext |
| `*.env.op` | `platform.env.op` | Legacy op inject pattern |
| `*cred*.{json,yaml,yml,toml,conf}` | `api-creds.json`, `service-credentials.yaml` | Ad-hoc credential bundles |
| `*secret*`, `*token*`, `*auth*` (with config extensions) | `mcp-auth.json`, `service-token.yaml` | Same |
| `*-auth.{json,yaml,yml}` | `service-admin-auth.json`, `ingest-auth.yaml` | Service auth bundles |

**Known Claude Code / tooling hotspots:**

| File | Pattern | Why it matters |
|---|---|---|
| `.claude/settings.local.json` | `"env": { "VAR": "value" }` — literal values, not `${VAR}` refs | Claude-Code-specific secret injection. Plaintext on disk. |
| `.claude/settings.json` | Same as above | Same — but committed. Higher risk. |
| `.mcp.json` | Literal header values like `"Authorization": "Bearer <hex>"` (not `${VAR}`) | Hardcoded tokens in committed files. |
| `.vscode/tasks.json`, `.vscode/launch.json` | `env` blocks with literal values | IDE-specific leaks. |
| `.github/workflows/*.yml` | `env:` blocks with literal defaults | CI leaks into git history |
| Shell scripts (`*.sh`) | `export VAR="<hex-or-jwt>"` | Copy-paste artifacts |

**Rule of thumb:** if a file's path or name suggests "credential, secret, key, token, auth" — scan its contents. If contents look like `KEY=VALUE` with a non-trivial value or JSON/YAML with something named "token"/"key"/"secret" — it's a candidate.

### Layer 2 — Legacy reference patterns (migrate to Environment)

Not leaks, but the old pattern we're superseding. Still need moving:

| File | Pattern | Action |
|---|---|---|
| `*.env.op` files | `VAR=op://vault/item/field` | Values live in 1P vault items, resolved at `op inject` / `op run` time. Migrate into Environment variables (copy values from vault). |
| `credentials/platform-load.sh`-style loaders | `source <file>; op inject -i <file>` | Obsolete launcher. Delete after migration verified. |

### Layer 3 — Pattern-based string search

For hardcoded tokens that slipped through, grep the whole repo (excluding `node_modules`, `.git`, `tmp`, `.trees`):

| Regex | Matches | Likely type |
|---|---|---|
| `eyJ[A-Za-z0-9_-]{20,}\.` | JWT header | JWT (n8n, Auth0, Supabase anon keys, etc.) |
| `sk-[A-Za-z0-9]{20,}` | `sk-...` | OpenAI, Anthropic, Stripe |
| `sk_live_`, `sk_test_`, `pk_live_` | Stripe prefixes | Stripe |
| `sk_mt_[A-Za-z0-9]{20,}` | MetaMCP tokens | MetaMCP |
| `ghp_[A-Za-z0-9]{36,}` | GitHub personal access tokens | GitHub |
| `github_pat_[A-Za-z0-9_]{40,}` | GitHub fine-grained PATs | GitHub |
| `xox[bap]-[A-Za-z0-9-]{20,}` | Slack bot tokens | Slack |
| `AIza[0-9A-Za-z_-]{35}` | Google API keys | Google |
| `[0-9a-f]{64}` | 64-char hex string | Generic bearer token (common for webhook/header auth) |
| `AKIA[0-9A-Z]{16}` | AWS access key IDs | AWS |

Each match → triage manually. Hex strings have false positives (hashes, IDs). JWTs are almost always real secrets.

### Layer 4 — `${VAR}` reference extraction (what the repo *needs*)

Not detection of leaks — detection of **what variables the repo requires**. Used to cross-check against the 1P Environment:

| File | Extract |
|---|---|
| `.mcp.json` | All `${VAR}` strings in `env`, `headers`, `command`, `args` |
| `*.sh` | `$VAR`, `${VAR}`, `$ENV{VAR}` patterns |
| `Dockerfile` | `ENV VAR $VAR` chains |
| `.github/workflows/*.yml` | `${{ secrets.VAR }}` — GitHub secrets, parallel concern |

Result: a set of `NEEDED_VARS` the repo references. If `NEEDED_VARS \ ENVIRONMENT_VARS` is non-empty, the Environment is incomplete.

## Layer 5 — Value-based sweep across all text files (SECURITY-CRITICAL)

Layers 1–4 scan **config-shaped files** (`.claude/settings*.json`, `.env*`, `credentials/*.env`, `.mcp.json`). They produce a list of candidate credentials and, combined with Phase C resolution, give us concrete plaintext values.

Layers 1–4 are necessary but not sufficient. They miss copies of the same values that live in **non-config files** — and these are where real-world leaks hide. Add a fifth layer.

### The scan

After Phase C resolves all token values for the import file, grep each resolved value across every text file in the repo:

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
EXCLUDES='--glob=!.git --glob=!node_modules --glob=!.trees --glob=!tmp --glob=!/tmp/*-import.env'
MIN_LEN=20   # skip very short values to avoid false positives (URLs, versions, etc.)

for var_name in $MIGRATED_VARS; do
  val="${RESOLVED[$var_name]}"
  [ ${#val} -lt $MIN_LEN ] && continue
  hits=$(rg -Fn -- "$val" "$REPO_ROOT" $EXCLUDES 2>/dev/null)
  if [ -n "$hits" ]; then
    printf '⚠ Copy site(s) for %s:\n' "$var_name"
    printf '%s\n' "$hits" | sed 's/^/    /'
  fi
done
```

### Why this is the security-critical layer

A canonical worked example: Layers 1–4 found 10 scattered credentials in config files. After the migration and cleanup committed successfully, a broader grep revealed **two more copies** Layers 1–4 had missed:

- `.claude/next-calendar.py:18` — `MCP_TOKEN = "36859c..."` (module-level Python constant used by the statusline)
- `3-resources/learnings/ops-kiosk-webhook-auth.md:26,48` — the token quoted literally in a bash example AND in a "Token Source" section for reader convenience

Both were committed to git history (2+ and 5+ commits respectively). Neither was detected by the config-shaped scan. Both were found instantly by `rg -F "36859c..."` across the repo.

This is how real-world credential leaks happen:

- Dev writes a quick Python script for a statusline; hardcodes the token as a constant "to test"; never parameterizes; commits.
- Learning doc quotes the raw value for reader clarity.
- CI workflow defaults to a literal for local testing.
- Shell script exports a literal because it was the fastest way to run a one-off.

Scanning only `.env*` and `settings*.json` misses all of these. Layer 5 exists because Layers 1–4 are not enough.

### What the skill does with hits

For each copy site:

1. **Include it in the Phase C summary** — user sees it before Phase D
2. **Include rewriting it in Phase H's cleanup** — replace the hardcoded value with `${VAR_NAME}` (shell), `os.environ['VAR_NAME']` (Python), `process.env.VAR_NAME` (JS), etc., appropriate to the file language
3. **Stage the rewritten files in the migration commit** — no separate cleanup pass

### Git history warning

If any copy site is in a file tracked by git, the token is likely in historical commits too. The Phase J follow-ups include a line flagging this. Default remediation: **rotate the token** — historical plaintext is useless once the value changes. Only rewrite git history (`git filter-repo`, etc.) if the repo has been or will be public — it's destructive and forces re-clones across collaborators.

### DO-NOT-MIGRATE patterns (safety guardrails)

Some files that appear in a credential scan must **never be migrated or deleted** — they're not storage-model-compatible with 1P Environments, and removing them would break working infrastructure.

#### SSH private keys — detect + warn, never touch

SSH private keys are fundamentally different from tokens:

- They must exist **as files on disk** (`ssh`, `git`, `scp`, etc. read them from a path, often via config lookup)
- Environment variables can't replace them — SSH clients don't consume keys from env
- 1P Environments can't meaningfully store them

**The skill detects SSH keys purely to warn.** Never migrated, never deleted, never moved. The scan reports them; the user handles them.

**Detection — by filename (common patterns):**

- `id_rsa`, `id_ed25519`, `id_ecdsa`, `id_dsa` (no extension)
- `id_rsa.pub`, etc. (public keys — safe but still flag so the user sees what's around)
- `*.pem`, `*.key` (common private key extensions)
- `*_rsa`, `*_ed25519`, `*_ecdsa`, `*_dsa` (named variants)
- Anything inside a `.ssh/` directory
- Files with `deploy_key`, `private_key`, `ssh_key` in the name

**Detection — by content:**

First non-blank line matches any of:

```
-----BEGIN RSA PRIVATE KEY-----
-----BEGIN OPENSSH PRIVATE KEY-----
-----BEGIN EC PRIVATE KEY-----
-----BEGIN DSA PRIVATE KEY-----
-----BEGIN PGP PRIVATE KEY BLOCK-----
-----BEGIN PRIVATE KEY-----
-----BEGIN ENCRYPTED PRIVATE KEY-----
```

If filename OR content matches, the file is tagged **do-not-migrate** and added to the Phase B warning list.

**Phase B report block:**

```
⚠ SSH KEYS / NON-TOKEN CREDENTIALS DETECTED
   (these will NOT be migrated or deleted — safety guardrail)

    credentials/deploy_key       — OpenSSH private key
    .ssh/id_ed25519              — OpenSSH private key
    config/service-account.pem   — RSA private key

  These stay where they are. 1P Environments can't replace them (they
  expect files, not env vars). If you want SSH keys managed by 1P, see
  1Password's SSH agent: https://developer.1password.com/docs/ssh/
```

**Why this exists:** a first-run migration case (2026-04-24) had an SSH deploy key in `credentials/` alongside plaintext tokens. The skill's old Phase H did `rm -rf credentials/`, which took out the SSH key as collateral. The key was restored from backup but the incident exposed a class of failure: **folder-level deletion is unsafe because folders contain heterogeneous content**. Phase H is now file-level (see SKILL.md). Layer 1 detection is the redundant second guardrail.

#### Other non-migratable patterns (extend as encountered)

| Pattern | Why not migratable |
|---|---|
| `*.p12`, `*.pfx` | PKCS#12 bundles — file-based, not env-var-based |
| `*.crt`, `*.cer` | Certificates — typically file-referenced in config |
| `*.kubeconfig`, `kubeconfig` | kubectl expects a path, not inline YAML |
| `.netrc`, `_netrc` | tools expect a specific file at `$HOME/.netrc` |
| `.npmrc`, `.pypirc` with auth fields | tool-specific files; if they contain tokens, consider extracting just the token, but leaving the file untouched is safer |

Treat all of these the same way: detect, warn, never touch.

### False-positive discipline

- Skip values under 20 chars (URLs, version strings, email addresses → too many false positives)
- Skip the `/tmp/*-import.env` file itself (obvious self-hit)
- Skip `.git`, `node_modules`, `.trees`, `tmp` directories

Even with these guards, expect occasional false positives for 64-char hex values that happen to also be a real hash. Human triage each hit — one-line skim of the context is enough.

---

## Layer 6 — Documentation reference scan

After migration planning (Phases B+C), scan narrative prose for phrases that reference the **pre-migration launch pattern**. These don't block migration — they're docs that will mislead future readers if not updated.

### Patterns to catch

| Regex / string | What it indicates |
|---|---|
| `source\s+.*platform-load\.sh` | Doc still tells readers to use the old loader |
| `source\s+.*\.env\.op` | Same, different filename |
| `op\s+run\s+--env-file=` | Doc references the op-run wrapper launch pattern |
| `op\s+inject\s+-i` | Doc references the op-inject pattern as a launcher |
| Literal `credentials/platform-load\.sh` | Direct file reference that won't resolve post-migration |

### Files to scan

- `README.md`, `CONTRIBUTING.md`, `ONBOARDING*.md` at repo root
- All `.md` under `docs/`, `1-projects/*/README.md`, `1-projects/*/*.md`
- `CLAUDE.md` — though Phase H usually rewrites this

### What the skill does with hits

- **Include each hit in Phase J follow-ups** as "docs still reference pre-migration pattern — update when convenient"
- **Do NOT auto-rewrite.** Narrative prose needs human judgment — a historical reference ("we used to launch via `source credentials/platform-load.sh`") is different from a current instruction ("to start the agent, run `source credentials/platform-load.sh`"). Let the user decide.

Layer 6 is low-cost (one grep pass) and high-value (prevents "follow these stale onboarding instructions and wonder why nothing works" hours later).

---

## Auto-classification

When `migrate` finds a credential, it auto-classifies `system` + `classification` from the value pattern + variable name. This removes user input from the setup flow. The user can still edit the manifest if they disagree with an inference.

### By value pattern

| Pattern | Inferred `system` | Inferred `classification` |
|---|---|---|
| `eyJ…` JWT, payload has `role=anon` | `supabase-anon` | `public-config` |
| `eyJ…` JWT, payload has `iss=n8n` | `n8n` | `secret` |
| `eyJ…` JWT, payload has `iss=supabase` | `supabase-service` | `secret` |
| `eyJ…` JWT, other (decode `iss` claim) | `jwt` (or `iss` value) | `secret` |
| `sk-ant-…` | `anthropic` | `secret` |
| `sk-proj-…`, `sk-…` (50+ chars) | `openai` | `secret` |
| `sk_mt_…` | `metamcp` | `secret` |
| `sk_live_…`, `sk_test_…`, `pk_live_…`, `pk_test_…`, `whsec_…` | `stripe` | `secret` |
| `ghp_…`, `github_pat_…`, `gho_…`, `ghs_…` | `github` | `secret` |
| `xox[bapr]-…` | `slack` | `secret` |
| `AIza…` | `google` | `secret` |
| `AKIA…` | `aws` | `secret` |
| UUID form (`xxxxxxxx-xxxx-…`) | (from var name) | `secret` |
| 64-char hex | (from var name) | `secret` |

### By variable name

When value pattern is ambiguous or absent, infer from the name:

| Name suffix/prefix | Inferred `system` | Inferred `classification` |
|---|---|---|
| `*_URL`, `*_ENDPOINT`, `*_HOST` | (leave blank) | `public-config` |
| `*_REGION`, `*_PORT`, `*_BUCKET` | (leave blank) | `public-config` |
| `*_EXPIRES_AT`, `*_EXPIRES_IN`, `*_TIMESTAMP` | (leave blank) | `public-config` |
| `ANON_KEY`, `PUBLIC_KEY` | (from value if JWT) | `public-config` |
| `*_API_KEY`, `*_TOKEN`, `*_SECRET`, `*_PASSWORD` | (from value pattern) | `secret` |
| `*_WEBHOOK_SECRET`, `*_SIGNING_SECRET` | (from value pattern) | `secret` |

### By `${VAR}` context in `.mcp.json`

When a variable appears as `${VAR}` in `.mcp.json` under an MCP entry, infer `system` from the MCP's URL:

| MCP URL host/path | Inferred `system` |
|---|---|
| `mcp.supabase.com` | `supabase` |
| `*/mcp/n8n-*` or `{your-n8n-host}/mcp/*` | `n8n-header-auth` |
| `{your-directus-host}/mcp` | `directus` |
| `*/metamcp/*` | `metamcp` |

(Populate / override these with URL patterns from your own `.mcp.json` file.)

Still `# TODO: verify` the inferred `admin_url` — that's a human check.

### Orphan credentials

Variables present in `.claude/settings.local.json` (or any scanned plaintext source) but **not referenced** in `.mcp.json` / scripts / hooks / other scanned files.

Examples: a `SLIDESPEAK_TOKEN` left from an earlier experiment, a test token that never got wired up.

**Default: migrate them anyway.** Plaintext on disk carries the same risk whether or not a consumer exists today. Moving to the Environment is strictly better:
- If a consumer shows up later, it inherits via shell env with no rewiring.
- If it's truly dead, you can delete the variable from the Environment in one click (less effort than hunting down plaintext on every developer machine).

The migrate report flags orphans explicitly so the user knows:

```
[orphan] SLIDESPEAK_TOKEN — no consumer found in repo; migrating anyway (default)
```

Pass `--skip=SLIDESPEAK_TOKEN` to setup/migrate to exclude specific vars.

## Output format

Migration report:

```
Repo: ~/code/myapp
Environment: myapp-local

=== Layer 1 — plaintext on disk (MUST migrate) ===
.claude/settings.local.json env block:
  N8N_API_KEY=<207 chars>       ← likely JWT (eyJ prefix)
  WEBHOOK_TOKEN=<64 chars>      ← likely hex bearer
  PRIVATE_API_TOKEN=<64 chars>  ← likely hex bearer
  METAMCP_TOKEN=<70 chars>      ← MetaMCP token (sk_mt_ prefix)

=== Layer 2 — legacy references (migrate + delete source) ===
credentials/platform.env.op:
  DIRECTUS_URL=https://api.example.com              ← public, not a secret
  DIRECTUS_TOKEN=op://myvault/Alice - api/token     ← resolve + migrate
  DIRECTUS_EMAIL=op://myvault/Alice - api/username  ← resolve + migrate
  DIRECTUS_PASSWORD=op://myvault/Alice - api/password  ← resolve + migrate

=== Layer 3 — pattern matches (triage) ===
(none found in remaining files)

=== Layer 4 — variables referenced but not yet in Environment ===
All 8 NEEDED_VARS match planned migration set. ✓

=== Files to clean post-migration ===
- Strip `env` block from .claude/settings.local.json
- Delete credentials/platform.env.op
- Delete credentials/platform-load.sh
- rmdir credentials/ (if empty)
```

Then a consolidated `/tmp/{env}-import.env` file 600-perms, plus instructions to drag into 1P.

## Cleanup order (critical)

**Do not delete source files until the Environment is populated and FIFO reads verified.** Safe order:

1. Produce report + import file
2. User imports into 1P GUI
3. Verify FIFO serves all expected vars (`cat .env.local`)
4. Verify a new shell + `claude` + MCP works end-to-end
5. Strip plaintext from `settings.local.json` (keep `permissions` block)
6. Delete `credentials/` folder contents
7. Shred `/tmp/{env}-import.env`

Step 5 before step 6 because `settings.local.json` also contains useful `permissions` / `enableAllProjectMcpServers` that aren't credentials.

## False-positive hygiene

- **Hashes vs. tokens:** 64-char hex in a comment after `sha256:` is almost certainly a hash. Check context before treating as secret.
- **Public config vs. secret:** URLs, email addresses, role names, feature flags — not secrets. Mark `classification: public-config` in manifest if you want them managed but not redacted.
- **Dev-only placeholders:** `changeme`, `dummy`, `REPLACE_ME`, `password123` — flag as **fail**, not migrate. These should never reach production.

## Script skeleton

A minimal scanner can live at `.claude/skills/1p-creds/scripts/scan-repo.sh`. Signature:

```
scan-repo.sh <repo-root> <env-name>
→ prints report to stdout
→ writes /tmp/<env-name>-import.env
→ exit 0 if gaps found, 1 if clean
```

See the skill's repo for an implementation once written.
