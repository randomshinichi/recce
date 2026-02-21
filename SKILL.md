---
name: recce
description: >
  Generate or update a State Vector — a structured project snapshot that prevents
  predictable wrong first moves for agents entering a scope cold. Two modes:
  --fast (default) produces a curated ≤50-line snapshot; --deep reads all source
  files, mines subsystem-level invariants, and outputs both a comprehensive deep
  State Vector and a distilled ≤50-line version for human/smaller-model handoff.
  Use when asked to: create a State Vector, update an existing State Vector, write
  the State Vector section of AGENTS.md, or summarize a project. Triggers on
  phrases like "recce", "run recce", "sitrep", "recon", "State Vector", "project
  snapshot", "AGENTS.md state", "recce --fast", "recce --deep", or "what should
  the State Vector look like for this repo".
---

# State Vector Skill

## What this produces

A structured snapshot (MAP / ROUTES / RULES / HANDLES / DECISIONS) that blocks
Wrong Action Classes (WACs) — the predictable wrong first moves agents make when
entering a scope cold.

**`--fast` (default):** A single ≤50-line State Vector. Calibrated for immediate
orientation — readable in 30 seconds, holdable in human working memory.

**`--deep`:** Two outputs:
1. **Deep State Vector** — no line limit; covers subsystem-level invariants,
   protocol internals, type system constraints, silent limits; annotate each
   entry with the WAC it blocks. Optionally add risk metadata only when it
   changes action: `[D:silent|caught] [C:0.10-1.00] [F:stable|verify|volatile]`.
   Use `C` in 0.05 increments; default uncertain cases to `D:silent`, `F:verify`,
   and `C<=0.65`. Calibrated for frontier AI agents with large context windows
   working an extended session.
2. **Distilled State Vector** — ≤50 lines mined from the deep version; the
   subset that blocks the first 3–5 hours of wrong moves. Hide metadata tags by
   default; include only when omitting them would likely cause a wrong move.
   Calibrated for human readers and smaller models.

Always read `references/philosophy.md` before generating. Section V contains the
full generation prompt, selection criteria, and anti-examples.

---

## --fast workflow

### 1. Run discovery

```bash
python3 ~/.pi/agent/skills/recce/scripts/sv-discover [path]
```

Auto-detects code vs content mode. Options:
- `--code` / `--content` — force mode
- `--max-files N` — cap walk (default 2000)
- `--max-depth N` — limit tree depth

### 2. Generate

Follow the process in `references/philosophy.md` Section V exactly:
1. Identify WACs
2. Draft from discovery only (no invention)
3. Targeted reads: files whose heads bottomed out on boilerplate, and any file
   whose content would likely block a WAC not yet covered
4. Delete non-WAC entries
5. Preserve high-impact entries
6. Cross-check every reference against discovery or files read in step 3
7. Refine ROUTES
8. Prune to ≤50 lines

### 3. Print

Output the State Vector verbatim. Do not write to any file unless asked.

---

## --deep workflow

### 1. Run discovery (same as --fast)

Establishes the verified fact base and project topology.

### 2. Systematic file reading

Do not limit yourself to truncated heads. Read source files in this priority order,
stopping when a file yields no new WAC-blocking facts:

1. **Constraint inheritance** — base classes, core interfaces, error contracts,
   serialization requirements, or any design decision in one place that forces a
   pattern everywhere else. These explain *why* the whole codebase made certain
   choices that otherwise look arbitrary.
2. **Look-alike traps** — things that appear to be X but are actually Y: the same
   function name in two modules, a default value that differs from caller
   expectation, an enum default that differs from function defaults, a fallback
   literal that silently changes behaviour. The gap between appearance and reality
   is the highest-value class of fact.
3. **Buried thresholds** — hardcoded limits, timeouts, retry counts, rate limits,
   batch caps, and delays embedded in business logic. Invisible at the call site,
   only visible in the implementation.
4. **Format invariants** — wire formats, serialization field names, URL structure
   per endpoint domain, framing protocols, extended parameter encodings. Anything
   where the format looks arbitrary but is actually mandatory.
5. **API surface vs internal layout** — re-exports, star imports, name collisions
   across modules, functions with identical names in sibling modules, public
   facades that hide internal structure.
6. **Auth and session mechanics** — login endpoints, credential key names,
   fallback token literals, MFA detection patterns, session cookie formats.
7. **Test fixture coupling** — hardcoded counts or specific values in assertions
   that break silently if fixture data is modified.

Cross-reference everything against discovery output. Any name not in discovery or
a read file must be removed.

### 3. Generate the deep State Vector

No line limit. Every entry must still block a WAC — annotate each with `_(WAC-N)_`.
For entries that are action-driving, ambiguous, or high-risk, add metadata tags:
`[D:silent|caught]` (detectability), `[C:0.10-1.00]` (confidence, 0.05 steps),
`[F:stable|verify|volatile]` (freshness).
Assignment defaults: if no concrete guard is named use `D:silent`; if evidence
is thin cap at `C:0.65`; if drift is unclear use `F:verify`.
If any D/C/F tags are used, include a `LEGEND:` block near the top of the deep
State Vector defining each tag, each value, and what each value means in action
terms.
Subsystem-specific appendices are acceptable (e.g. protocol wire event tables)
but label them "reference only, not WAC-blocking" so they don't inflate the WAC
signal.

### 4. Distil to the standard State Vector

From the deep version, select the entries that block wrong moves in the first
3–5 hours of working in the codebase — the ones that apply regardless of which
subsystem you enter. Leave subsystem-specific entries (protocol internals, batch
rate-limiting details) in the deep version only.

Apply the ≤50-line ceiling and the WAC-only selection criteria from
`references/philosophy.md`. Keep D/C/F tags only when they change what the next
actor should do (for example: low-confidence, silent-failure, or volatile facts).
The distilled version must stand alone — a reader who has not seen the deep
version should be fully oriented by it.

### 5. Print both

Output the deep State Vector first, then the distilled State Vector. Label them
clearly. Do not write to any file unless asked.

---

## Key constraints

These apply to both modes. Memorise — don't re-read philosophy.md for them.

- Every entry must block a WAC — no exceptions, no orientation entries
- ROUTES = where to EDIT source; HANDLES = what to RUN — never cross them
- Pruning bias: omit when unsure; deletion is the default
- ≤50 lines applies to the distilled output only; deep has no ceiling
- No ACTIVE section in either output — goes stale; use task tracker instead
- The deep State Vector is calibrated for frontier AI agents; the distilled for
  humans and smaller models — do not conflate the audiences
- D/C/F metadata is optional and selective: show it when it changes action,
  otherwise omit to preserve signal density
- If no concrete guard is named, default detectability to `D:silent`
- If evidence is thin, cap confidence at `C:0.65`; if drift is unclear, default
  freshness to `F:verify`
- If any D/C/F tags appear in deep output, include a `LEGEND:` section that
  defines D, C, F and explains each value/state in action semantics
- Confidence is numeric (`C:0.10-1.00`) in 0.05 increments
- Claims with `C>=0.90` should include a provenance anchor `(path | doc | PR#)`
