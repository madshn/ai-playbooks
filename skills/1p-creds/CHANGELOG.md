# Changelog

All notable changes to this playbook.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning: [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — 2026-04-24

### Added

- Initial public release of the Claude Code variant (`claude-code/SKILL.md` + `references/migrate.md` + `references/manifest-schema.md`)
- `setup` action — 11-phase deterministic flow (Phase 0 wizard → Phase J completion report)
- `migrate` action — standalone credential scan across 6 detection layers
- `rotate <VAR>` action — AI+web-searched rotation instructions per manifest `system` + `admin_url`
- `validate` action — FIFO mount + Environment contents health check
- `inventory` action — tabulate all credentials by variable, system, scope
- `decommission-legacy` action — safe cleanup of stale vault items referenced by manifest `legacy_source:` fields
- `.claude/creds-manifest.yaml` schema with `system` / `admin_url` / `classification` / `scope` / `legacy_source` vs `vault_reference` distinction
- Layer 5 cross-repo value sweep (security-critical — catches hardcoded token copies in scripts/docs that config-shaped scanners miss)
- Layer 6 documentation reference scanner
- Guarded `.envrc` template (`if [ -p … ] || [ -f … ]; then …`) — tolerates the mount being absent
- Auto-classification table for common token patterns (JWT `role=anon` → `supabase-anon`/`public-config`, `sk-ant-` → `anthropic`, `sk_mt_` → `metamcp`, etc.)
- Completion report with a small bunny 🐰 delivering the headline credential count

### Origin

Extracted from a real production migration of a small developer fleet. Three pilot runs (each exposing ~7, ~4, and ~4 refinements in turn) stabilized the flow before this public release.

## [Unreleased]

- Codex variant (placeholder in `codex/README.md`)
- Gemini CLI variant (not yet scoped)

[0.1.0]: https://github.com/madshn/ai-playbooks/releases/tag/1p-creds-v0.1.0
