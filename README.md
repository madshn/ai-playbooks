# AI Playbooks

Reusable skills, prompts, and patterns for running AI agents in production — shared openly for other operators.

---

## About

Hi, I'm **Mads Nissen** — a builder in Oslo running [Right Aim](https://rightaim.ai), where I operate a team of always-on AI associates alongside my own work: executive assistance, engineering, operations, growth, and revenue.

This repo is where I publish the playbooks that keep that team running: reusable skills, prompts, and patterns I've battle-tested day-to-day. Cross-harness where it matters (Claude Code, Codex, and others), concept-first, no framework lock-in.

- 🌐 [rightaim.ai](https://rightaim.ai)
- 💼 [LinkedIn](https://www.linkedin.com/in/madsnissen)

---

## Index

Items appear here as they're published. Each entry links to its own folder with a README explaining what it is, when to use it, and how to drop it into your own setup.

<!-- INDEX:START -->

| Skill | What it does | Harnesses |
|---|---|---|
| [`1p-creds`](./skills/1p-creds/) | Move scattered repo credentials into a per-repo 1Password Environment, mounted as a FIFO-served `.env.local` auto-loaded by direnv. Cut biometric prompts from 10-20/day to 1-2/day — without wrapping the agent CLI. | [`claude`](./skills/1p-creds/claude/) |

<!-- INDEX:END -->

---

## How to use this repo

- **Browse** the folders above. Each one is self-contained.
- **Copy** what's useful — everything here is MIT-licensed.
- **Adapt** to your stack. These are playbooks, not frameworks; the value is in the shape, not the exact syntax.

If something sparks a question, open an issue or ping me on LinkedIn.

---

## License

[MIT](./LICENSE) — use it, remix it, ship it. Attribution appreciated but not required.
