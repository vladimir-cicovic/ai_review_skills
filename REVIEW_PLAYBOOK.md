# REVIEW_PLAYBOOK — Phased, Resumable Deep Code Review for Any Codebase

> A battle-tested method for auditing large or multi-component codebases (thousands of small files; Windows services, RPC/IPC, background daemons, libraries, external integrations) to dig out technical, logical, security, concurrency, lifecycle, resource-leak and **cross-file** bugs — **designed so token limits never lose progress**.
>
> Core idea: **disk is the source of truth, context is disposable.** Every intermediate result is a durable artifact on disk. Each phase is independently resumable. If a token/session limit cuts you off anywhere, the next session reads the artifacts and continues.

---

## 0. Core principles (why this survives token limits)

1. **Disk is truth, context is disposable.** Inventory, clusters, raw findings, verified findings, and drafted prose all live as files in a scratch dir. Never drag 100k+ chars through the conversation.
2. **The orchestrator never reads code.** The main loop only launches agents, parses their JSON output **with a script** (Python), and assembles. Subagents read code; their results are written to files, not into your context.
3. **Everything is resumable (idempotent).** A manifest records what is done. A restart skips completed work.
4. **Bounded work per turn.** Never "do it all at once." One cluster / one group per step, each with a disk checkpoint.
5. **Always have an agent-free fallback.** If helper agents die (token limit), the main loop must be able to finish the remaining work from the on-disk artifacts alone.
6. **Machines do the deterministic parts.** Merging, numbering, splicing, table-building = Python scripts. Agents are only for reasoning (finding + verifying bugs).

**Artifact naming convention** (in `scratch/` or `review/`):

```
00_inventory.json     files + line counts
01_clusters.json      subsystem clusters
02_known.md           baseline of already-known issues (grows each pass)
03_candidates.json    raw findings from review fan-out
04_verified.json      real + new findings (after adversarial verify)
04_rejected.json      rejected / duplicate (kept for audit)
05_plan.json          consolidated, merged, numbered final list
out_*.md              drafted prose sections (one file per group)
backup_*.md           backups of docs before splicing
MANIFEST.md           progress tracker: which phases/clusters are done
```

---

## Phase 0 — Setup & inventory (cheap, always first)

Goal: know exactly what you are reviewing; create working dirs.

- Create a scratch dir **outside** the repo, plus a `review/` dir for durable artifacts.
- **Inventory with a script, not by hand:** list every source file with its line count, sorted by size (this maps where code density is). Separate test files from source files.
- Write `00_inventory.json`: `[{path, lines, ext}]`.

> Checkpoint: `00_inventory.json`. Trivial to regenerate if interrupted.

---

## Phase 1 — Cluster into components

Goal: split thousands of files into ~15–60 **clusters**, each small enough for one agent to fully read in one pass.

- Group by **subsystem/component**, not alphabetically: `windows-service`, `rpc-ipc`, `external-services`, `libraries`, `network-protocols`, `config`, `ui`, `crypto`, `update`, etc.
- Add a dedicated **`integration-spine`** cluster that traces interactions *between* files (protocol/contract on both sides, threading of ctx/IDs, lifecycle events, dropped events). This catches cross-file bugs single-file review misses.
- For re-reviewing already-covered files, make **recheck** clusters that look for **new** issues only.
- Keep each cluster to ~5–10 files or ~2–4k lines. For thousands of files you will have 30–60 clusters — that is fine; you run them in **waves**.

> Checkpoint: `01_clusters.json` — `[{name, files[], focus}]`.

---

## Phase 2 — Baseline of known issues (avoid re-reporting)

Goal: a compact index of everything **already known**, so agents don't report duplicates.

- Condense existing findings (or issue tracker / prior docs) into one text file `02_known.md`: one line each — `file:line — short description (ID)`.
- First pass → this is empty. After each pass you **append** to it, so every later pass finds only new things.

> Checkpoint: `02_known.md`. This is the project's "memory" between passes.

---

## Phase 3 — Fan-out review (Workflow, resumable)

Goal: each cluster → one agent that finds candidates, with **structured output**.

- Use the **Workflow tool** with `pipeline()`: stage 1 = review a cluster, stage 2 = verify (Phase 4) — no barrier, each cluster flows independently.
- **Always pass a `schema`** so output is machine-parseable (no free-text parsing). See `FINDINGS_SCHEMA` below.
- Inject `02_known.md` into every agent prompt + explicit instruction: "Do NOT report anything on this list."
- **Waves for scale:** if you have 60 clusters, run 15–20 per Workflow call. Each call journals to disk. If a limit cuts you off → `resumeFromRunId` returns cached agents instantly; only new work runs live.
- Cap findings per agent (~12) to keep output bounded.

> Checkpoint: `03_candidates.json` (all raw findings). Workflow journals itself → resumable.

### FINDINGS_SCHEMA (review agent output)

```json
{
  "type": "object",
  "properties": {
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "title": {"type": "string"},
          "file": {"type": "string"},
          "line": {"type": "integer"},
          "severity": {"enum": ["critical","high","medium","low"]},
          "category": {"enum": ["technical","logical","security","concurrency","lifecycle","resource-leak","error-handling","cross-file"]},
          "description": {"type": "string"},
          "failure_scenario": {"type": "string"},
          "cross_file": {"type": "string"}
        },
        "required": ["title","file","line","severity","category","description","failure_scenario"]
      }
    }
  },
  "required": ["findings"]
}
```

### Review-agent prompt template

```
You are a senior engineer doing a rigorous file-level review of <APP> — <one-line what it is + how components connect>.
Your cluster: <name>. Focus: <focus>.
Read these files FULLY (repo root <ROOT>): <file list>.
Also read adjacent files needed to understand how these connect, but only REPORT bugs anchored in this cluster.

ALREADY-DOCUMENTED (do NOT re-report): <paste 02_known.md>

Find NEW bugs — technical, logical, concurrency, lifecycle, resource-leak, error-handling, security, and especially CROSS-FILE contract/interaction bugs. Think about: untrusted input without length/ok/error checks; goroutine/thread leaks, deadlocks, races, ctx never/early cancelled; resources leaked on error paths; swallowed errors / return-nil-on-failure; wrong logic (off-by-one, inverted condition, wrong default, path traversal, missing timeout); how these files connect to the service lifecycle, UI, RPC, external systems.
Rules: cite exact file+line; give a concrete failure scenario; skip pure style nits; quality over quantity but be thorough (up to ~12). Return via the schema.
```

---

## Phase 4 — Adversarial verification (the key filter)

Goal: keep only **real** and **new**. This is the difference between "151 candidates" and "116 real" (~25% are false/dup).

- Each candidate → a separate verifier, skeptical stance: "prove it from the code or reject it."
- The verifier **re-reads the actual code** around the line + adjacent files, and compares against `02_known.md`.
- Keep only `is_real && is_new`. Everything else goes to `04_rejected.json` (useful for audit).
- Optional stronger variant: N independent verifiers per finding, each with a distinct lens (correctness / security / does-it-reproduce); keep if majority confirm.

> Checkpoint: `04_verified.json` + `04_rejected.json`.

### VERDICT_SCHEMA (verify agent output)

```json
{
  "type": "object",
  "properties": {
    "is_real": {"type": "boolean"},
    "is_new": {"type": "boolean"},
    "confidence": {"enum": ["high","medium","low"]},
    "corrected_severity": {"enum": ["critical","high","medium","low"]},
    "corrected_line": {"type": "integer"},
    "dup_of": {"type": "string"},
    "reasoning": {"type": "string"}
  },
  "required": ["is_real","is_new","confidence","reasoning"]
}
```

### Verify-agent prompt template

```
You are an adversarial verifier. A reviewer claims a bug (repo root <ROOT>). Default stance: SKEPTICAL — prove it real from the actual code or reject it.
CLAIM: <finding JSON>
1. Read the actual file around the cited line + any adjacent file needed. Confirm the control flow truly does what the claim says and the failure path is reachable.
2. is_real = true ONLY if the code as written manifests the defect. If the reviewer misread, a guard exists elsewhere, or the path is unreachable → is_real=false.
3. is_new: compare against this ALREADY-DOCUMENTED list; set is_new=false + dup_of if it's the same underlying issue: <paste 02_known.md>
4. Set corrected_line, corrected_severity, confidence, reasoning (cite concrete code). Return via the schema.
```

---

## Phase 5 — Consolidate / merge / number (script, not an agent)

Goal: deterministically tidy the findings without spending tokens.

- Python script: merge obvious duplicates (same file + near lines); route by category (security → security doc, rest → technical); sort by severity; **assign IDs continuing the existing numbering**.
- Emit a compact **digest** (one line per finding) — that is all you look at in context, never the full descriptions.

> Checkpoint: `05_plan.json` (final numbered list) + a printed digest.

---

## Phase 6 — Draft prose (parallel agents + MANDATORY fallback)

Goal: turn verified findings into prose in the target document's style.

- Split the plan into groups (~15–20 findings) → one agent per group, **each writes its own `out_X.md` file** (does not return prose into context).
- The agent takes verified findings as input — it **writes from them; it does NOT invent or re-read code**.
- **Critical (learned the hard way):** agents can die on a token limit mid-write. So:
  - Each agent writes a **separate file** → partial success is still useful.
  - After each wave, **validate with a script** which groups are complete (every ID has a heading + a table row).
  - Missing groups → the **main loop writes them itself** from `05_plan.json`. Never depend on agents to finish.

> Checkpoints: `out_A.md … out_N.md`, validated for completeness.

---

## Phase 7 — Splice into docs (script + safety checks)

Goal: insert the prose into the real docs **without clobbering existing content**.

- **Back up first** (`backup_*.md`).
- Splice with a script using **anchor assertions**: the anchor (e.g. `## Summary`) must appear **exactly once**, else the script aborts *before* writing (prevents double/wrong inserts).
- **Re-read before writing.** If a file changed meanwhile (parallel session) → re-read, reconcile, then write. Resolve ID collisions by assigning the next free number.
- Respect the existing format (LF vs CRLF, table style, numbering scheme).
- Validate after: all IDs present, one intro section, table complete.

> This is where most silent errors happen — always script + verify, never hand-paste large text.

---

## Phase 8 — Deliverables (PDF / zip, optional)

- Generate into scratch, then **copy over** the target (avoids file-lock if a PDF is open in a viewer).
- Gotcha: don't keep a `.py` file named like a stdlib module (e.g. `inspect.py`) in the generator's dir — it shadows stdlib and breaks imports.
- Verify **textually** (extract PDF text, grep for key IDs, check zero replacement/tofu chars) instead of visual inspection.
- Rebuild any archive/zip from the final files; verify the entry list.

---

## Phase 9 — Handoff (bridge between sessions)

- Update `HANDOFF.md` **additively**: add a new session note, never delete prior ones.
- Record: what was covered, what remains, new numbering, and **session-specific gotchas**.
- Append new findings to `02_known.md` → the next pass automatically finds only new things.

---

## Resume protocol (when a limit cuts you off)

The next session does this in order:
1. Read `HANDOFF.md` + `MANIFEST.md` (which phases/clusters are done).
2. Find the first unfinished artifact (`00` → `09`).
3. Workflow phase: `resumeFromRunId` (cached agents = instant), rest runs live.
4. Draft phase: validate which `out_*.md` are missing; finish them yourself.
5. Continue from there.

**Rule:** checkpoint to disk after every non-trivial step. Assume you could be cut off right after any tool action; the artifacts must be enough to continue.

---

## Gotchas checklist (hard-won)

- Helper agents die on token limits → an **agent-free fallback is mandatory**; write partial results to separate files.
- Expect **concurrent edits** to files (parallel session) → always re-read before write; reconcile ID collisions by taking the next free number; keep handoff edits additive.
- **Adversarial verification removes ~25%** false/duplicate findings — do not skip it.
- `schema` on every agent = no free-text parsing.
- Scripts for everything deterministic (merge/number/splice); agents only for judgment.
- Anchor-assert "exactly once" before every scripted splice.
- On Windows: detect LF/CRLF and preserve it; use the platform's real Python launcher (not a cygwin shim) for PDF/reportlab; `reportlab.lib.units.mm` may be `None` → define `mm = 72/25.4`.

---

## Scaling to thousands of files / many components

- **Divide by component first, then by wave.** 3000 files → ~60 clusters → 3–4 Workflow waves of ~16 clusters. Each wave is a resumable checkpoint.
- Treat each **component type** with a tailored focus line: Windows service (handles, job objects, session/token, syscalls), RPC/IPC (contract match on both sides, auth, timeouts, streaming vs polling), external services (untrusted responses, TLS/host-key trust, timeouts, injection), libraries (API misuse, resource leaks, thread-safety), config (atomicity, global state, validation).
- Keep **one `integration-spine` cluster per major flow** (e.g. UI↔daemon↔service, message-bus↔handlers) — cross-file bugs are the highest-value and least-found.
- Grow `02_known.md` every wave so later waves never re-cover ground.
