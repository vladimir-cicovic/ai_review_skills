---
name: phased-code-review
description: Systematic, resumable, multi-phase deep review of a large or multi-component codebase (thousands of files; Windows services, RPC/IPC, daemons, libraries, external integrations) to dig out technical, logical, security, concurrency, lifecycle, resource-leak and cross-file bugs — engineered so token/session limits never lose progress. Use when the user wants to audit or review an entire codebase or a large subsystem in phases, "dig out bugs" across many files, do a comprehensive multi-pass review, or continue such a review after an interruption. Also use when the user references REVIEW_PLAYBOOK or asks to review "in phases/parts".
---

# Phased, resumable deep code review

The full method with prompt templates, JSON schemas, and gotchas is in **`REVIEW_PLAYBOOK.md`** bundled next to this file — read it before starting a real run.

## Core principle (why this survives token limits)

**Disk is the source of truth; context is disposable.** Every intermediate result is a durable artifact on disk, and every phase is independently resumable. If a token/session limit cuts you off anywhere, the next session reads the artifacts and continues. Never drag large data through the conversation; the orchestrator launches agents and parses their JSON with scripts, but never ingests full file dumps.

## The 10 phases (each ends in an on-disk checkpoint)

0. **Setup & inventory** — script-list every source file with line counts → `00_inventory.json`.
1. **Cluster** — group files by subsystem/component into ~15–60 clusters (~5–10 files each); add an `integration-spine` cluster for cross-file flows → `01_clusters.json`.
2. **Known baseline** — condense already-known issues to one line each → `02_known.md` (grows every pass; injected into every agent so nothing is re-reported).
3. **Fan-out review** — Workflow `pipeline()`; one agent per cluster with a `schema`; run in waves of ~15–20; `resumeFromRunId` on interruption → `03_candidates.json`.
4. **Adversarial verify** — one skeptical verifier per candidate; keep only `is_real && is_new` → `04_verified.json` (+ `04_rejected.json`). Removes ~25% false/dup.
5. **Consolidate** — Python merges dups, routes by category, sorts, assigns IDs continuing existing numbering → `05_plan.json` + a one-line-per-finding digest.
6. **Draft prose** — one agent per group writes its own `out_X.md` (writes from verified findings, does not re-read code). **Fallback mandatory:** validate which groups are complete; write missing ones yourself.
7. **Splice into docs** — back up first; script with **anchor-assert exactly once**; re-read before write; preserve format; validate after.
8. **Deliverables** (optional) — generate PDFs/zip into scratch then copy over; verify textually.
9. **Handoff** — update `HANDOFF.md` additively; append new findings to `02_known.md`.

## Structured-output schemas (always pass these to agents)

Findings (review agent):
```json
{"type":"object","properties":{"findings":{"type":"array","items":{"type":"object",
"properties":{"title":{"type":"string"},"file":{"type":"string"},"line":{"type":"integer"},
"severity":{"enum":["critical","high","medium","low"]},
"category":{"enum":["technical","logical","security","concurrency","lifecycle","resource-leak","error-handling","cross-file"]},
"description":{"type":"string"},"failure_scenario":{"type":"string"},"cross_file":{"type":"string"}},
"required":["title","file","line","severity","category","description","failure_scenario"]}}},"required":["findings"]}
```
Verdict (verify agent):
```json
{"type":"object","properties":{"is_real":{"type":"boolean"},"is_new":{"type":"boolean"},
"confidence":{"enum":["high","medium","low"]},"corrected_severity":{"enum":["critical","high","medium","low"]},
"corrected_line":{"type":"integer"},"dup_of":{"type":"string"},"reasoning":{"type":"string"}},
"required":["is_real","is_new","confidence","reasoning"]}
```

## Resume protocol (when cut off)

1. Read `HANDOFF.md` + `MANIFEST.md`. 2. Find the first unfinished artifact (`00`→`09`). 3. Workflow phase: `resumeFromRunId`. 4. Draft phase: validate missing `out_*.md`, finish them yourself. 5. Continue.

## Non-negotiable rules

- **Agent-free fallback is mandatory** — helper agents can die on token limits; each writes a separate file so partial work survives, and the main loop must be able to finish alone.
- **Re-read before writing** any doc — expect concurrent edits; reconcile ID collisions by taking the next free number; keep handoff edits additive.
- **Scripts for everything deterministic** (merge/number/splice/tables); agents only for judgment (find + verify).
- **Anchor-assert "exactly once"** before every scripted splice; back up docs first.
- Checkpoint to disk after every non-trivial step — assume you could be cut off right after any tool action.

See `REVIEW_PLAYBOOK.md` (bundled) for the full phase detail, prompt templates, scaling guidance, and the Windows/PDF gotchas.
