# `.claude/creds-manifest.yaml` schema

Per-repo sidecar that pairs each Environment variable with the metadata needed to rotate it, validate it, and explain it to future maintainers. Safe to commit — contains no secrets.

## Minimal schema

```yaml
provider: 1password              # Required. Future: infisical, vault, aws-secrets, ...
environment: myapp-local         # Required. Matches the 1P Environment name.
mount: .env.local                # Optional. Relative path to the FIFO mount. Default: .env.local
variables:
  VAR_NAME:                      # Required. Matches the variable name inside the Environment.
    system: <string>             # Required. Upstream service (e.g., n8n, directus, stripe).
    admin_url: <url>             # Required for rotation. The admin surface to generate a new token.
    account: <string>            # Optional. Specific account/user/client ID this cred authenticates as.
    scope: <string>              # Optional. Default: repo. Canonical values: repo|fleet-wide|provisioned.
                                 # Free-text descriptions also accepted ("user-scoped header token for X").
    classification: secret|public-config  # Optional. Default: secret.
    dependencies: [VAR_NAME, ...]  # Optional. Other vars that must rotate together.
    used_in: [<path>, ...]       # Optional. Where this variable is consumed. Documentation aid;
                                 # validate action cross-references this against reality.
    last_rotated: <ISO-8601>     # Optional. Tracked by the rotate action.
    expires_at: <ISO-8601 or unix-timestamp>  # Optional. If upstream service exposes an expiry.
    legacy_source: <op:// or file path>  # Optional. Pre-migration source that can be DELETED
                                 # post-verification. See `legacy_source` vs `vault_reference` below.
    vault_reference: <op://...>  # Optional. Pre-migration source that must be RETAINED (it holds
                                 # additional data beyond this variable). Never offered for deletion.
    notes: <string>              # Optional. Anything non-schema-able.
```

**Rule of thumb:** a variable can carry `legacy_source` OR `vault_reference`, never both. See the detailed section on this distinction below — it's the single most common source of "wait, I deleted the wrong vault item" accidents.

## Field definitions

### `provider`
Which backend serves the credentials. Today always `1password`. Adding Infisical or Vault later: a sibling skill (e.g., `infisical-creds`) would reuse this same manifest format with `provider: infisical`.

### `environment`
The name of the 1P Environment. Convention: `{repo-basename}-{tier}`, e.g., for a repo at `~/code/myapp`, the local-dev Environment is `myapp-local`; the hosted/production variant could be `myapp-production`.

### `variables.<NAME>.system`
Upstream service name in a form that's search-friendly. Used by the `rotate` action: the skill web-searches `how to rotate API key in {system} {current year}`. Examples:
- `n8n` — for n8n API keys
- `directus` — for Directus user tokens
- `openai` — for OpenAI keys
- `anthropic` — for Anthropic keys
- `github` — for GitHub PATs
- `stripe` — for Stripe keys
- `supabase` — for Supabase service_role keys
- `metamcp` — for MetaMCP tokens
- `n8n-header-auth` — for custom bearer tokens mounted as n8n webhook headers

Unknown system → put your best description. Web search handles novel values.

### `variables.<NAME>.admin_url`
The URL the user goes to in order to rotate this credential. Used for:
- Focusing the web search ("how to rotate API key at {admin_url domain}")
- `open <admin_url>` — skill can launch browser directly

If no web UI exists (e.g., some credentials are provisioned by admin APIs only), use `admin_url: provisioned` and set `scope: provisioned`.

### `variables.<NAME>.account`
Optional. Some credentials are tied to a specific account/user identity. Examples:
- `account: alice@example.com` — user-specific API token
- `account: team-service@example.com` — service account shared across a team

Helps rotation instructions ("log in as `{account}` to rotate").

### `variables.<NAME>.scope`

| Value | Meaning |
|---|---|
| `repo` (default) | Credential is used only by this repo. Rotate freely. |
| `fleet-wide` | Shared across multiple repos / team members. Rotation must be coordinated across all consumers. |
| `provisioned` | Can only be rotated by admin/infra, not the end user. Skill escalates instead of proceeding. |

### `variables.<NAME>.classification`

| Value | Meaning |
|---|---|
| `secret` (default) | True secret. Never logged, never displayed, always masked in output. |
| `public-config` | Not sensitive (e.g., public URLs, feature flags). Can be logged. |

Classification affects how skill tooling renders values in inventory / validation reports.

### `variables.<NAME>.dependencies`

Variables that must rotate together. Example: a refresh token + access token pair. Skill warns when rotating one but dependencies haven't been updated.

### `variables.<NAME>.last_rotated`

ISO-8601 timestamp. The `rotate` action updates this automatically on successful rotation. Enables compliance tracking ("warn if any cred is > 180 days old").

### `variables.<NAME>.expires_at`

ISO-8601 timestamp if the upstream service exposes expiration. Skill can warn on approaching expiry.

### `variables.<NAME>.legacy_source` vs `vault_reference` — critical distinction

This is the single most consequential schema choice, because it drives the `decommission-legacy` action's behavior. Choosing wrong can orphan an identity or leave plaintext behind.

| Field | When to use | What `decommission-legacy` does |
|---|---|---|
| **`legacy_source`** | The pre-migration source held ONLY this variable's value and is safe to destroy once the new Environment is verified. Examples: plaintext line in `credentials/*.env`, a single-field `op://` item, a plaintext entry in `.claude/settings.local.json`. | Offers to delete the source after length-match safety check. On confirm: `op item delete` (for vault items) or file strip (for plaintext sources). |
| **`vault_reference`** | The pre-migration source is a 1P item that holds additional data beyond this variable — typically a LOGIN item whose `username`/`password` fields are identity anchors used for account rotation, or whose other fields became different manifest variables. Deleting the item would orphan those other fields. | Records the pointer for traceability only. **Never offered for deletion.** Stays in the manifest indefinitely as a rotation reference. |

**Rule of thumb:** if unsure, use `vault_reference`. It's strictly safer — the skill won't propose deletion. You can downgrade `vault_reference` → `legacy_source` later if you verify the source is truly single-purpose.

### Worked example: API_TOKEN with identity-anchor vault item

A common pattern: `API_TOKEN` comes from a 1P LOGIN item whose fields are:

```
op://myvault/Alice - api.example.com/Section_xxx/token
```

That same 1P LOGIN item ALSO holds:
- `username` — `alice@example.com` (the account identity)
- `password` — the account password
- Other fields used for browser login + rotation coordination

If `API_TOKEN` were given `legacy_source:` and `decommission-legacy` ran, the skill would delete the entire item — erasing the identity anchor. Any future rotation would require recreating the upstream user from scratch.

Correct manifest entry:

```yaml
API_TOKEN:
  system: api.example.com
  admin_url: https://api.example.com/admin/users
  account: alice@example.com
  vault_reference: op://myvault/Alice - api.example.com/Section_xxx/token
  # NOT legacy_source: the item also holds username/password identity anchors
```

The comment line documents the reasoning for future maintainers.

### `legacy_source` examples

Use `legacy_source` when the source truly held only this variable:

```yaml
# Variable was in a plaintext env block — safe to strip the block
WEBHOOK_TOKEN:
  system: example-webhook
  legacy_source: .claude/settings.local.json#env.WEBHOOK_TOKEN

# Variable was in a single-field op:// item that serves no other purpose
SOME_ORPHAN_TOKEN:
  system: some-service
  legacy_source: op://myvault/SomeService Token/password
```

Omit both fields for variables that have no pre-migration source (e.g., newly-issued tokens that never lived anywhere but the Environment from day 1).

### `variables.<NAME>.notes`

Free-form text. Anything that helps future maintainers but doesn't fit the schema.

## Worked example — `myapp-local`

A hypothetical repo `~/code/myapp` integrating with three external services: a Directus-like API platform (`api.example.com`), an n8n workflow host (`n8n.example.com`), and a MetaMCP aggregator (`mcp-bundle.example.com`). Substitute your own service names + URLs when adapting.

```yaml
provider: 1password
environment: myapp-local
mount: .env.local

variables:

  # --- Directus-style API platform (identity-anchored; vault_reference) ---

  API_URL:
    system: api.example.com
    admin_url: https://api.example.com
    classification: public-config
    notes: Platform base URL. Not a secret.

  API_TOKEN:
    system: api.example.com
    admin_url: https://api.example.com/admin/users
    account: alice@example.com
    scope: repo
    vault_reference: op://myvault/Alice - api.example.com/token
    notes: Static access token. The 1P LOGIN item also holds username/password
      (identity anchors for rotation) — do NOT decommission.

  API_EMAIL:
    system: api.example.com
    admin_url: https://api.example.com/admin/users
    account: alice@example.com
    scope: repo
    classification: secret
    vault_reference: op://myvault/Alice - api.example.com/username
    notes: Login email — used with API_PASSWORD for browser/SDK auth.

  API_PASSWORD:
    system: api.example.com
    admin_url: https://api.example.com/admin/users
    account: alice@example.com
    scope: repo
    dependencies: [API_EMAIL]
    vault_reference: op://myvault/Alice - api.example.com/password
    notes: Matches API_EMAIL. Used when token-based auth is not viable.

  # --- n8n workflow host ---

  N8N_API_KEY:
    system: n8n
    admin_url: https://n8n.example.com/settings/api
    scope: repo
    legacy_source: .claude/settings.local.json#env.N8N_API_KEY
    notes: n8n REST API key for workflow CRUD via the n8n-mcp server.

  TEAM_SHARED_TOKEN:
    system: n8n-header-auth
    admin_url: https://n8n.example.com/admin/variables
    scope: fleet-wide
    legacy_source: .claude/settings.local.json#env.TEAM_SHARED_TOKEN
    notes: Bearer for multiple MCP servers hosted on n8n. Shared across a team;
      rotation requires coordinated update of every consumer's Environment.

  MYAPP_PRIVATE_TOKEN:
    system: n8n-header-auth
    admin_url: https://n8n.example.com/admin/variables
    scope: repo
    legacy_source: .claude/settings.local.json#env.MYAPP_PRIVATE_TOKEN
    notes: Bearer for a repo-scoped private MCP endpoint on n8n.

  # --- MetaMCP aggregator ---

  METAMCP_TOKEN:
    system: metamcp
    admin_url: https://mcp-bundle.example.com/admin
    scope: repo
    legacy_source: .claude/settings.local.json#env.METAMCP_TOKEN
    notes: Bearer for a MetaMCP aggregated bundle (multiple upstream MCPs
      behind one endpoint).
```

This example covers the three common field patterns you'll see in practice:

- **`vault_reference`** for identity-anchored credentials (LOGIN items holding token + username + password — deleting the item would orphan the identity)
- **`legacy_source`** for values that were in `settings.local.json` or a standalone plaintext file (safe to delete the source after verification)
- **`fleet-wide`** scope for shared tokens that require coordinated rotation across consumers

## Invariants the skill enforces

- Every variable in the 1P Environment has a manifest entry
- Every `${VAR}` in `.mcp.json` / repo code has either a manifest entry or a known-ignore marker
- No `value:` field anywhere — the manifest must never contain secret material
- `admin_url` is reachable (optional net check) and returns HTTP < 500

## Evolution path

- **v1** (today): `provider: 1password`, hand-maintained
- **v2**: Auto-populate `system` guesses from token prefix patterns during `migrate`
- **v3**: `last_rotated` auto-tracking, compliance warnings
- **v4**: Shared fleet-wide "credential catalog" with per-repo manifests referencing by ID

## Location rationale

`.claude/creds-manifest.yaml` — next to other Claude config, hidden folder. The name is provider-neutral so a future `infisical-creds` or `vault-creds` skill can read the same file without migration.
