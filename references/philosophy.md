# Context Retention Philosophy — v4

## Aim

Prevent predictable wrong first moves when an agent enters a scope cold, by front-loading what it cannot discover quickly with standard tools.

The dominant cost of agent-assisted development is wrong-direction execution — starting in the wrong file, running the wrong command, violating a hidden invariant. The State Vector is a curated cache of non-obvious project state that blocks these errors. Like any cache, its value depends on freshness.

---

# I. Wrong Action Classes (WAC)

An open taxonomy. Select the WACs that actually occur in the current scope.

WAC-1: Wrong build/test path
WAC-2: Ignored gating (flags, env vars, feature/config switches)
WAC-3: Edited generated/mirrored/vendored code
WAC-4: Wrong subsystem boundary
WAC-5: Cross-variant breakage
WAC-6: Contract violation (API, performance, compatibility)
WAC-7: Re-litigated architectural decision
WAC-8: Direction drift (lost hypothesis)
WAC-9: Wrong runtime mode (install/deploy/backend fork)
WAC-10: Safety/compliance violation (security, data loss, license)
WAC-11: Wrong or missing change locus (agent searches wrong place, or doesn't know to search at all; high token burn)

New classes may be added when a wrong action repeats and doesn't fit an existing class.

## I.a Orthogonal Risk Dimensions (optional, selective)

WAC explains **what can go wrong**. D/C/F explains **whether to act now or
verify first**.

- **D (Detectability):** `silent` | `caught`
  - `silent` = likely to pass normal validation while still wrong.
  - `caught` = likely to fail loudly in a **named guard** (compiler/test/CI check).
  - If no concrete guard is named, default to `D:silent`.
- **C (Confidence):** numeric `0.10–1.00` in `0.05` increments
  - `1.00` = directly verified from discovery/read files.
  - `0.70–0.99` = strong evidence with some inference.
  - `<0.70` = inference-heavy; validate before editing.
  - If evidence is thin, cap at `C:0.65`.
- **F (Freshness):** `stable` | `verify` | `volatile`
  - `stable` = low drift risk.
  - `verify` = medium drift risk; spot-check before non-trivial edits.
  - `volatile` = high drift risk; revalidate before relying.
  - If drift is unclear, default to `F:verify`.

These tags are optional. Show them only when they change user action (especially
for silent failures, low confidence, or volatile facts). If any are used in a
Deep State Vector, include a `LEGEND:` section defining each tag and explaining
what each value/state means for action.

---

# II. Selection Criteria

An entry belongs in the State Vector if:

1. It blocks a WAC.
2. An agent would not *think to look for it* within 60 seconds using standard tools (`ls`, `rg`, `--help`). Not just "could it find it?" but "would it know to search?"
3. It is not redundant with another entry in the same section. Cross-section overlap is acceptable when sections serve different functions.

**Pruning bias:** when unsure, omit. Deletion is the default; addition requires justification. Stop when dominant WACs are clearly blocked. A State Vector should rarely exceed 40–50 lines.

---

# III. The State Vector

A minimal, high-leverage snapshot per scope (repo, subsystem, or workspace).

All entries are short phrases. No prose. No trivia. High-impact entries may include a provenance anchor `(path | doc | PR#)`. When action would differ based on uncertainty, add compact tags such as `[D:silent] [C:0.60] [F:verify]`. Use `C` in `0.05` steps; claims tagged `C>=0.90` should include a provenance anchor.

### MAP
Topology outside the grep radius. External paths, runtime mode forks, sole-adapter boundaries (where NOT to look), invisible state locations. Omit anything that restates file structure.

ROUTES and HANDLES are complementary. ROUTES = where to EDIT source code. HANDLES = what to RUN as commands. Never put a command in ROUTES. Never put a source code path in HANDLES.

### ROUTES
Where to edit, not what to run. Each route maps a change-intent to the correct source code location AND names where an agent would wrongly look first. Include call chains only when they cross module boundaries or have surprising indirection. Annotate with the test/guard that must pass after the edit. Omit when there's only one plausible place to edit.

### RULES
Hard invariants, hidden switches, safety constraints. Only record what would silently succeed but produce wrong results — omit what the compiler, type system, or CI catches.

### HANDLES
What to run, not where to edit. Only commands where the naive attempt would fail or cause harm. Bootstrapping sequences, mode-specific entrypoints, safety-critical flags. Omit what `--help` explains clearly or what's already in AGENTS.md.

### DECISIONS
Short pointers to irreversible or costly-to-reverse tradeoffs. Purpose: prevent re-litigation. Omit decisions obvious from the dependency manifest or project structure.

---

# IV. Maintenance

### Start of Session

1. Read State Vector.
2. Identify work item and risk points (map to WACs).
3. Perform 1–3 targeted validation reads (prioritise low-confidence `C<0.70`, `F:volatile`, and `D:silent` entries; include spot-checking 1–2 entries for staleness).
4. Act.

### End of Session

* If a wrong action occurred: add a blocking entry.
* Delete entries that no longer block a relevant WAC or that have become discoverable through other means.
* If the State Vector exceeds ~50 lines, prune lowest-impact entries.

---

# V. State Vector Generation Prompt

Self-contained prompt for generating or updating a State Vector. Intentionally duplicates selection criteria and section guidance so it can be used standalone.

## Pre-step: Run Discovery

Before generating, run `sv-discover` from the repo root. This produces a deterministic snapshot of file layout, function definitions, test guards, CLI args, constants, and external paths. Pipe the output as the `<discovery>` block below.

```bash
python3 ~/.pi/agent/skills/recce/scripts/sv-discover [path]
```

The discovery output is the **starting ground truth**. Draft from it first, but you may perform targeted reads when they can add WAC-blocking signal. Every function, path, test, and flag you reference in the final State Vector must appear either in the discovery output or in files you actually read. Do not invent names. Do not guess at locations.

Discovery saturates most files in 15 lines. Files where line 15 is still imports/package metadata are common targeted-read candidates, but not the only ones. Use judgment: read additional files when the discovery view is ambiguous, routing is unclear, or a likely WAC hinge needs confirmation. Keep reads sparse and purposeful.

```
You are writing a State Vector for this scope (repo/subsystem/workspace).
Goal: make the correct first move obvious and prevent repeated wrong actions.

Here is the discovery output for this repo (generated by sv-discover):
<discovery>
{paste sv-discover output here}
</discovery>

Output ONLY the State Vector. No commentary, no file-reading plan.
In distilled/human-oriented outputs, hide D/C/F tags unless omitting them would likely induce a wrong move.
If Deep output contains D/C/F tags, include a `LEGEND:` block near the top.

Process (silent):
1) Identify the top 3–6 dominant Wrong Action Classes (WAC) for this scope.
2) Draft MAP/ROUTES/RULES/HANDLES/DECISIONS from <discovery> first, with no invented facts.
3) Targeted reads: use <discovery> to choose files that could materially improve WAC blocking. Boilerplate-only heads are strong candidates, but you may also read other files when needed to resolve ambiguity, confirm boundaries, or locate true edit loci. Keep reads minimal and purposeful.
4) Delete any entry that is descriptive identity, aspirational, or redundant.
5) Preserve high-impact entries: do not delete ROUTES/RULES that block severe WACs without replacement.
6) Cross-check: every function(), path, test name, and --flag you wrote must appear verbatim in <discovery> or in a file you read in step 3. Remove any that don't.
7) Add optional risk tags only where they change action: Detectability `[D:silent|caught]`, Confidence `[C:0.10-1.00]` (0.05 steps), Freshness `[F:stable|verify|volatile]`.
   - If no concrete guard is named, use `D:silent`.
   - If evidence is thin, cap confidence at `C:0.65`.
   - If drift is unclear, use `F:verify`.
8) Prefer refining existing ROUTES over adding many new ones.
9) Target ≤50 lines total. If over, prune lowest-impact entries.

Inclusion rules:
- Each entry must block a WAC.
- Omit anything an agent would think to search for and find within 60 seconds.
- Prefer short phrases naming forks: boundaries, modes, gates, invariants, first commands.
- If an entry is high-impact, add a short provenance anchor: (path | doc | PR#).
- Add D/C/F tags only when needed (when they would change the next action).
- Confidence should be numeric in 0.05 steps: `[C:0.10-1.00]`.
- If `C>=0.90`, include a provenance anchor `(path | doc | PR#)`.

Section guidance:
MAP: Topology outside the grep radius. External paths, runtime mode forks, sole-adapter boundaries (where NOT to look), invisible state locations. Omit anything that restates file structure.
ROUTES and HANDLES are complementary:
- ROUTES = where to EDIT source code. HANDLES = what to RUN as commands.
- Never put a command in ROUTES. Never put a source code path in HANDLES.

ROUTES: Where to edit, not what to run. Each route maps a change-intent to the correct source code location AND names where an agent would wrongly look first.
Format: intent → CORRECT: location, NOT: obvious-but-wrong location (+ guard)
  YES: "Fix write deduplication → CORRECT: merge_and_write_partition(), NOT: write_partition_atomic() (+ guard: test_merge_and_write_partition)"
  BAD: "Load CSV → data.py load-csv --symbol BTCUSD" ← this is a command, not a code path. Move to HANDLES.
  BAD: "Fix normalize → normalize_dataframe()" ← only one plausible edit location. Omit entirely.
RULES: Hard invariants, hidden switches, safety constraints. Only what would silently succeed but produce wrong results.
HANDLES: What to run, not where to edit. Only commands where the naive attempt would fail or cause harm.
  YES: "Bootstrap from CSVs: uv run python data.py load-csv data/csvs/*.csv --symbol BTCUSD --exchange INDEX --tf 1m — glob + three required flags, none guessable"
  BAD: "Run tests: uv run python -m unittest..." ← already in AGENTS.md
  BAD: "Verify: data.py verify --symbol BTCUSD --tf 1m" ← discoverable from --help
DECISIONS: Short do-not-revisit pointers, not essays.

Format:
PROJECT: <5–10 words>

LEGEND: (include only if D/C/F tags are used)
- D (Detectability): `silent` = likely wrongness passes normal validation; `caught` = likely wrongness fails a named guard. If guard is unspecified, default to `silent`.
- C (Confidence): `0.10–1.00` in `0.05` steps. `1.00` = directly verified in discovery/read files; `0.70–0.99` = strong evidence with inference; `<0.70` = inference-heavy (validate before editing). If evidence is thin, cap at `0.65`.
- F (Freshness): `stable` = low drift risk; `verify` = medium drift risk (spot-check); `volatile` = high drift risk (revalidate before relying). If drift is unclear, default to `verify`.

MAP:
- <phrase>

ROUTES:
- <intent> → CORRECT: <location>, NOT: <obvious-but-wrong> (+ <guard>)

RULES:
- <phrase>

HANDLES:
- <phrase>

DECISIONS:
- <phrase>
```
