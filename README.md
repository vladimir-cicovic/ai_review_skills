# ai_review_skill

**`phased-code-review`** — a Claude Code skill for deep code review and technical bug hunting on large or multi-component codebases (thousands of files; Windows services, RPC/IPC, daemons, libraries, external integrations).

It digs out **technical, logical, security, concurrency, lifecycle, resource-leak, error-handling, and cross-file bugs**, and is engineered so that token/session limits never lose progress: every intermediate result is checkpointed to disk, and any interrupted run can be resumed from where it stopped.

## What's in this repository

| File | Purpose |
|------|---------|
| `REVIEW_PLAYBOOK.md` | The full review method — phases, prompt templates, JSON schemas, gotchas |
| `SKILL.md` | The skill entry point Claude reads when the skill triggers (lives in the installed skill folder) |

## REVIEW_PLAYBOOK.md in short

The playbook is a battle-tested, 10-phase method built on one core idea: **disk is the source of truth, context is disposable.** The orchestrator never reads code itself — it launches review agents, parses their structured JSON output with scripts, and writes every result to durable artifacts (`00_inventory.json` → `05_plan.json`, `MANIFEST.md`, `HANDOFF.md`).

The phases:

0. **Setup & inventory** — script-list every source file with line counts.
1. **Cluster** — group files by subsystem into ~15–60 clusters small enough for one agent each, plus an `integration-spine` cluster that hunts cross-file contract bugs.
2. **Known baseline** — a one-line-per-issue index of everything already known, so nothing gets re-reported across passes.
3. **Fan-out review** — one agent per cluster, structured output via JSON schema, run in resumable waves.
4. **Adversarial verification** — a skeptical verifier re-reads the actual code for every candidate; only *real and new* findings survive (this filters out ~25% false/duplicate findings).
5. **Consolidate** — a script (not an agent) merges duplicates, sorts by severity, and assigns IDs.
6. **Draft prose** — parallel agents write finding write-ups to separate files, with a mandatory agent-free fallback.
7. **Splice into docs** — scripted insertion with backups and "anchor appears exactly once" safety checks.
8. **Deliverables** — optional PDF/zip generation, verified textually.
9. **Handoff** — additive session notes so the next session (or next pass) continues seamlessly.

It ends with a **resume protocol** (what to read and in which order after an interruption), a hard-won gotchas checklist, and guidance for scaling to thousands of files.

## Installation

Copy the skill folder (containing `SKILL.md` and `REVIEW_PLAYBOOK.md`) to one of:

- **Personal** (available in all your projects): `~/.claude/skills/phased-code-review/`
  (on Windows: `C:\Users\<you>\.claude\skills\phased-code-review\`)
- **Project-level** (available to everyone working in that repo): `<project>/.claude/skills/phased-code-review/`

Claude Code discovers the skill automatically on the next session — no restart or registration needed.

## Usage from the command line (Claude Code CLI)

1. Open a terminal in the repository you want to review and start Claude Code:

   ```bash
   cd path/to/your/project
   claude
   ```

2. Invoke the skill explicitly:

   ```
   /phased-code-review
   ```

   or just ask naturally — the skill triggers automatically on prompts like:

   - "Review this entire codebase in phases and dig out technical bugs."
   - "Do a comprehensive multi-pass review of the `services/` subsystem."
   - "Continue the code review from where the last session stopped."

3. If a session hits a token limit mid-review, simply start a new session and say "continue the review" — the skill picks up from the on-disk checkpoints.

## Usage from Claude Desktop (Claude Code desktop app)

1. Open the Claude Code desktop app and open the project folder you want reviewed.
2. The skill is picked up from the same locations as the CLI (`~/.claude/skills/` or the project's `.claude/skills/`).
3. Type `/phased-code-review` in the chat, or describe what you want in plain language ("audit this codebase for bugs, in phases") — the skill triggers on the same phrases as in the CLI.
4. Long reviews behave the same way: progress is checkpointed to disk, so you can close the app and resume later.

## Notes

- The review writes its working artifacts to a scratch directory **outside** your repo plus a `review/` directory for durable results — your source tree is never modified by the review itself.
- Findings are adversarially verified before being reported, so the final list contains only confirmed, non-duplicate bugs with concrete failure scenarios and exact `file:line` references.
