---
name: recce
description: >
  Generate or update a State Vector — a curated ≤50-line project snapshot that
  prevents predictable wrong first moves for agents entering a scope cold. Use
  when asked to: create a State Vector, update an existing State Vector, write
  the State Vector section of AGENTS.md, or summarize a project into a
  structured WAC-blocking snapshot. Triggers on phrases like "recce",
  "run recce", "sitrep", "recon", "State Vector", "project snapshot",
  "AGENTS.md state", or "what should the State Vector look like for this repo".
---

# State Vector Skill

## What this produces

A minimal structured snapshot (MAP / ROUTES / RULES / HANDLES / DECISIONS) that
blocks Wrong Action Classes (WACs) — the predictable wrong first moves agents
make when entering a scope cold.

## Workflow

**Always read `references/philosophy.md` before generating or updating a State
Vector.** Section V contains the self-contained generation prompt with full
selection criteria, section guidance, and anti-examples.

### 1. Run discovery

```bash
python3 ~/.pi/agent/skills/recce/scripts/sv-discover [path]
```

Auto-detects code vs content mode. Options:
- `--code` / `--content` — force mode
- `--max-files N` — cap walk (default 2000)
- `--max-depth N` — limit tree depth

Pipe output as the `<discovery>` block in the Section V prompt.

### 2. Generate

Follow the 8-step process in `references/philosophy.md` Section V exactly:
1. Identify WACs
2. Draft from discovery first (no invention)
3. Do targeted reads when useful (not just boilerplate heads); choose files that can add WAC-blocking signal
4. Delete non-WAC entries
5. Preserve high-impact entries
6. Cross-check every reference against discovery or files read in step 3
7. Refine ROUTES
8. Prune to ≤50 lines

### 3. Print the result

Output the State Vector verbatim. Do not write it to any file unless explicitly asked.

## Key constraints (memorise these — don't re-read philosophy.md for them)

- Every entry must block a WAC — no exceptions, no orientation entries
- ROUTES = where to EDIT source; HANDLES = what to RUN — never cross them
- Pruning bias: omit when unsure; deletion is the default
- ≤50 lines soft ceiling; >50 requires explicit justification
- No ACTIVE section — goes stale; use task tracker instead
