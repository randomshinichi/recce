# recce

Generates a **State Vector** — a curated ≤50-line project snapshot that prevents predictable wrong first moves when an AI agent enters a codebase cold.

Packaged as a [pi](https://github.com/mariozechner/pi) skill, but the two components — `scripts/sv-discover` and the generation prompt in `references/philosophy.md` — work with any agent harness or model interface.

> **Model note:** Use a frontier model for best results. Smaller models struggle to apply the selection criteria correctly and tend to over-include. The structured discovery feed helps considerably — smaller models fed `sv-discover` output produce usable State Vectors with light editing — but judgment calls (which entries block a real WAC, when to omit) still favour frontier models.

## The problem

Agents entering an unfamiliar repo make the same mistakes repeatedly: editing the wrong file, running the wrong command, violating a hidden invariant, re-litigating a settled decision. These aren't intelligence failures — they're orientation failures. The agent doesn't know what it doesn't know.

A State Vector is a targeted fix: a structured brief of the non-obvious facts that block the most costly wrong moves, generated periodically and mentioned from `AGENTS.md`.

## What it produces

```
PROJECT: <5–10 words>

MAP:     topology outside the grep radius — external paths, runtime forks, sole-adapter boundaries
ROUTES:  where to edit, with the wrong-but-obvious location named explicitly
RULES:   invariants that would silently produce wrong results
HANDLES: commands where the naive attempt would fail
DECISIONS: irreversible tradeoffs, to prevent re-litigation
```

Every entry must block a Wrong Action Class (WAC). No orientation fluff. See [`references/philosophy.md`](references/philosophy.md) for the full taxonomy and generation prompt.

## Installation

**With pi:** copy into your skills directory and it triggers automatically on the keywords above:

```bash
cp -r . ~/.pi/agent/skills/recce
```

**Without pi:** run `sv-discover` manually and paste the output plus the prompt from `references/philosophy.md` Section V into any model interface.

## Usage

Say any of: **recce**, **sitrep**, **recon**, "run recce", "State Vector", "project snapshot" — pi will trigger the skill automatically.

The skill runs in two phases:

**1. Discovery** — scan the repo to build a verified fact base:

```bash
python3 ~/.pi/agent/skills/recce/scripts/sv-discover [path]
```

Auto-detects code vs content mode. Useful flags:
```
--code          force code mode (file heads, function defs, test names, CLI args)
--content       force content mode (file tree, type inventory, text heads)
--max-files N   cap the walk (default: 2000)
--max-depth N   limit tree depth
```

**2. Generation** — the agent feeds discovery output into the prompt in `references/philosophy.md` Section V and produces the State Vector.


## Examples

### Code project (Python/Rust/etc.)
```
PROJECT: tradingview-rs — Rust library (alpha) for TradingView API access

MAP:
- Two data paths: one-shot history (`src/chart/history/`) vs live streaming (`src/live/`)
- Auth: `TV_AUTH_TOKEN` env var (token) OR `TV_USERNAME` + `TV_PASSWORD` + `TV_TOTP_SECRET` (cookie)
- `user` feature is default-on — cookie auth included unless explicitly disabled
- `src/client/fin_calendar.rs` is a `// TODO:` stub — no implementation

ROUTES:
- Fetch historical bars → CORRECT: `src/chart/history/single.rs` or `batch.rs`
                            NOT: `src/live/websocket.rs` (live is streaming, not one-shot)
- Auth token resolution → CORRECT: `src/chart/history/mod.rs::resolve_auth_token()` (private fn,
`TV_AUTH_TOKEN` only)
                            NOT: `src/utils.rs` (HTTP headers + packet parsing, not auth)
- Quote field list      → CORRECT: `src/quote/mod.rs::ALL_QUOTE_FIELDS` (lazy_static)
                            NOT: `src/client/misc.rs`
- WS reconnect/state   → CORRECT: `src/live/handler/command.rs::ConnectionState`
                            NOT: `src/live/websocket.rs` (client wrapper only)

RULES:
- `Ustr` not `String` for symbol/exchange identifiers — all existing models enforce this
- `bon::builder` derive for config structs — adding a field without it breaks the builder API
- `misc_test.rs` integration tests hit live TradingView API — require network, no auth
- `rustls-tls` is the default TLS backend — adding `native-tls` without disabling default creates conflict

HANDLES:
- Tests: `just quick-test` — auth tokens go in `.env` (Justfile uses `set dotenv-load`, not explicit env)
- Disable user feature: `cargo build --no-default-features --features rustls-tls` (must keep TLS explicit)
```
### Content project (documentation, archives, reference material)
```
 PROJECT: Power Mac G5 Quad FCode reference archive

 MAP:
 - Scope is documentation/disassembly artifacts only; no build/test/toolchain entrypoint (README.md)
 - Primary corpus is compiled fcode/my notes/All.txt (~29–31MB); zip contains same payload plus __MACOSX metadata noise
 - Most key files are classic-Mac CR line-ending text; line tools mislead unless normalized first
 - compiled fcode/Worksheet is MPW shell + Forth patch notes (not a spreadsheet, not Unix shell)
 - Two words dumps differ by firmware mode: Open Firmware fcode-debug? true exposes extra internals vs Open Firmware Stuff.txt
 - NVMe guidance is in path-to-nvme-support.md/.txt; not in disassembly blobs

 ROUTES:
 - Confirm ROM base/size/address map → CORRECT: Dumping the New World Rom, NOT: Open Firmware Stuff.txt
 - Change/verify G5 detok parameters → CORRECT: compiled fcode/Worksheet (and README tool block), NOT: How to get compiled
 fcode.txt
 - Analyze probe-fcode interception behavior → CORRECT: compiled fcode/Worksheet (domappatch, blpatch, 1bc4 get-token drop a8
 +), NOT: All.txt
 - Compare exported OF dictionary visibility → CORRECT: Open Firmware fcode-debug? true, NOT: Open Firmware Stuff.txt
 - Read full disassembly content → CORRECT: compiled fcode/my notes/All.txt, NOT: top-level word-list dumps

 RULES:
 - Normalize CR before grep/head/wc: tr '\r' '\n' (discovery + README)
 - G5 disassembly settings are specific: detok -t -v -a -n -o -i -m 4 -s $FF844B00 (Worksheet/README)
 - Do not reuse 8600 settings (-m 1) for G5 outputs; both modes appear in Worksheet
 - In domappatch v3, images > 20000 are intentionally reduced to 1000 before encode-bytes (Worksheet)
 - Patch locus is token-derived callsite (1bc4 get-token drop a8 + ' domappatch blpatch); wrong offset risks invalid patching
 (Worksheet)

 HANDLES:
 - Extract canonical disassembly payload from archive: unzip "G5 Quad (disassembly of compiled Fcode).zip" "compiled fcode/my
 notes/All.txt"
 - Normalize classic-Mac text before tooling: tr '\r' '\n' < file
 - OF capture sequence hinges on telnet + dump primitive: open 192.168.1.150 then ff844b00 here ff844b00 - dumpbytes

 DECISIONS:
 - Keep repo as reference archive (raw dumps + notes), not a reproducible build project (README.md)
 - Prefer domappatch variant that stores ROM bytes as property (encode-bytes " romdump" property) over size/address-only
 variants (Worksheet)
 - Preserve OF telnet capture workflow for provenance (" enet:telnet,192.168.1.150" io + dumpbytes) rather than re-deriving from modern tooling
```

### Updating a stale State Vector
It's probably not worth committing the state vector since it's not deterministic. But like all caches, we need to automate updates. Tell the agent when to run it and what to do with the output in `AGENTS.md`

## Design notes

- **Discovery-first**: the agent's job is selection and compression, not exploration. Discovery eliminates hallucinated file paths and function names.
- **WAC-only selection**: if an entry doesn't block a specific wrong action, it doesn't belong. This keeps the State Vector from becoming documentation.
- **ROUTES name the wrong location**: forcing `CORRECT: X, NOT: Y` makes surprising indirection explicit — the most common source of wrong-file edits.
- **≤50 lines**: a State Vector that grows without bound becomes the thing it was meant to replace.

Full rationale in [`references/philosophy.md`](references/philosophy.md).
