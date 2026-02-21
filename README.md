# recce

Generates a **State Vector** — a structured project snapshot that prevents predictable wrong first moves when an AI agent enters a codebase cold. Two modes:

- **`--fast`** (default): one curated ≤50-line snapshot. Designed for immediate orientation — calibrated for human working memory and smaller models.
- **`--deep`**: the agent reads all source files and mines subsystem-level invariants, then outputs two artifacts: an unrestricted deep State Vector for frontier AI agents, and a distilled ≤50-line version for human/smaller-model handoff.

Packaged as a [pi](https://github.com/mariozechner/pi) skill, but the two components — `scripts/sv-discover` and the generation prompt in `references/philosophy.md` — work with any agent harness or model interface.

> **Model note:** `--fast` works well with any capable model; the discovery feed eliminates hallucinated names and keeps smaller models honest. `--deep` requires a frontier model — the mining step involves judgment calls (constraint inheritance, look-alike traps, buried thresholds) that smaller models consistently get wrong.

## The problem

Agents entering an unfamiliar repo make the same mistakes repeatedly: editing the wrong file, running the wrong command, violating a hidden invariant, re-litigating a settled decision. These aren't intelligence failures — they're orientation failures. The agent doesn't know what it doesn't know.

A State Vector is a targeted fix: a structured brief of the non-obvious facts that block the most costly wrong moves, generated periodically and mentioned from `AGENTS.md`.

## What it produces

Both modes use the same structure:

```
PROJECT: <5–10 words>

MAP:     topology outside the grep radius — external paths, runtime forks, sole-adapter boundaries
ROUTES:  where to edit, with the wrong-but-obvious location named explicitly
RULES:   invariants that would silently produce wrong results
HANDLES: commands where the naive attempt would fail
DECISIONS: irreversible tradeoffs, to prevent re-litigation
```

Every entry must block a Wrong Action Class (WAC). No orientation fluff.

**`--fast`** produces one State Vector (≤50 lines) from the discovery output alone, with targeted reads only where discovery left genuine gaps.

**`--deep`** produces two: an unrestricted State Vector covering subsystem-level invariants mined from full file reads, and a distilled ≤50-line version extracted from it. The deep version is calibrated for a frontier AI agent working an extended session; the distilled version is for human readers and smaller models.

See [`references/philosophy.md`](references/philosophy.md) for the full WAC taxonomy and generation prompt.

## Installation

**With pi:** copy into your skills directory and it triggers automatically on the keywords above:

```bash
cp -r . ~/.pi/agent/skills/recce
```

**Without pi:** run `sv-discover` manually and paste the output plus the prompt from `references/philosophy.md` Section V into any model interface.

## Usage

Say any of: **recce**, **sitrep**, **recon**, "run recce", "State Vector", "project snapshot" — pi will trigger the skill automatically. Add **`--fast`** or **`--deep`** to select the mode; default is `--fast`.

Both modes start with discovery:

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

**`--fast`:** the agent feeds discovery output into the prompt in `references/philosophy.md` Section V, does targeted reads where needed, and produces a single ≤50-line State Vector.

**`--deep`:** the agent reads all source files systematically — prioritising constraint inheritance, look-alike traps, buried thresholds, format invariants, API surface mismatches, auth mechanics, and test fixture coupling — then produces a deep State Vector followed by a distilled ≤50-line version.


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
### Deep mode
```
PROJECT

Rust library (alpha, crates.io: tradingview) for TradingView API access.
Two data paths: one-shot history (src/chart/history/) vs live streaming (src/live/).


MAP

- Auth fallback literal: no-token auth uses the string "unauthorized_user_token" — some public TV
  endpoints accept this. The library does not require auth for public data.
  (WAC-2: agent assumes auth is always required)

- DataServer enum default != function defaults: DataServer::default() → DataServer::Data
  (data.tradingview.com), but every retrieve() and WebSocketClient::new() defaults to
  DataServer::ProData (prodata.tradingview.com). Using DataServer::default() hits a different
  server. (WAC-2: wrong server silently)

- user feature is default-on: default = ["user", "rustls-tls"] — disabling it requires
  --no-default-features --features rustls-tls. (WAC-2: agent drops TLS when disabling user)

- src/client/fin_calendar.rs is // TODO: — one line, no implementation. (WAC-11)

- Five distinct API domains — symbol search, pine facade, charts storage, main site, WS:
    WS:             wss://{server}.tradingview.com/socket.io/websocket
    Symbol search:  https://symbol-search.tradingview.com/symbol_search/v3/
    Pine/indicators: https://pine-facade.tradingview.com/pine-facade/
    Chart drawings: https://charts-storage.tradingview.com/charts-storage/
    Auth/misc:      https://www.tradingview.com/

- Test fixture hardcode: test_parse_packet reads tests/data/socket_messages.txt and asserts
  exactly 42 messages — modifying that file breaks the test. (WAC-3: modifying test data)

- reqwest client: https_only(true) — no HTTP fallback ever. (WAC-6)


ROUTES

- Fetch historical bars (one-shot)
    CORRECT: src/chart/history/single.rs::retrieve()
    NOT:     src/live/websocket.rs
             (live is streaming; history internally also opens a WS but tears it down on completion)

- Fetch historical bars (multi-symbol)
    CORRECT: src/chart/history/batch.rs::retrieve()
    NOT:     calling single::retrieve() in a loop
             (no rate-limit coordination, no shared WS connection)

- Auth token resolution
    CORRECT: src/chart/history/mod.rs::resolve_auth_token() — private fn, checks TV_AUTH_TOKEN only
    NOT:     src/utils.rs (HTTP header/packet utilities, not auth)

- Quote field list
    CORRECT: src/quote/mod.rs::ALL_QUOTE_FIELDS (lazy_static vec)
    NOT:     src/client/misc.rs

- WS connection state / reconnect logic
    CORRECT: src/live/handler/command.rs (ConnectionState, CommandRunner)
    NOT:     src/live/websocket.rs — that's the socket I/O wrapper; state machine lives in command.rs

- Adding a new TradingView protocol event
    CORRECT: src/live/models.rs (TradingViewDataEvent::from(String) match arm)
             + src/live/handler/message.rs (TradingViewResponse variant)
    NOT:     src/live/handler/data.rs alone (data.rs dispatches, models.rs defines the mapping)

- Adding a new command to send to TV
    CORRECT: src/live/handler/message.rs (Command enum)
             + src/live/handler/command.rs (CommandRunner::run() dispatch)
    NOT:     src/live/websocket.rs directly

- Adding a new error variant
    CORRECT: src/error.rs — BUT: Error is Copy, so all payload fields must be Ustr, not String or Vec
    NOT:     using String payloads (compile error: Error: Copy constraint violated)

- Symbol search / list symbols / indicator functions
    CORRECT: src/client/misc.rs — all pub use'd to crate root via lib.rs
    NOT:     creating new HTTP clients in calling code
             (use the existing utils::get() or utils::build_request())


RULES

Type system
- Error derives Copy — ALL error payload fields must be Ustr, never String, Vec, or any heap type.
  This is the root cause of Ustr being used everywhere in the codebase.
  (WAC-6: adding String field to Error variant is a compile error)
- Ustr not String for symbol/exchange identifiers — interned, Copy, O(1) equality. (WAC-6)
- bon::Builder derive for config structs — adding a field without #[builder(...)] attribute breaks
  the generated builder API. (WAC-6)
- ChartOptions has two construction APIs — ChartOptions::builder().build() AND chained setters
  (.symbol(), .exchange(), etc.) — both valid, both produce the same struct. Don't create a
  third. (WAC-7)

Timeouts and thresholds
- Single history default timeout: 30 seconds (Duration::from_secs(30))
- Batch history default timeout: 180 seconds (Duration::from_secs(180))
- Error threshold: 5 errors triggers abort — note asymmetry: single uses > 5, batch uses >= 5
- Consecutive error reset: after 60 seconds of no errors (error_reset_interval)
- WS reconnect timeout: 30 seconds before reconnect is declared failed
- Task cleanup grace period: 2 seconds per task

Batch-specific
- batch_size is capped at min(input, 10) regardless of what you pass (WAC-2)
- 200ms delay between commands within a batch, 2s between batches
  (rate-limiting, not configurable at call site)
- Batch signals Success even if some symbols failed — check failed_symbols count if completeness
  matters (WAC-6)
- Symbols tracked by numeric index (sds_sym_{N}) before series ID is known — SymbolError is
  attributed by index, not by name

Protocol internals
- WS packet framing: ~m~{byte_length}~m~{json} — heartbeats use ~h~ prefix, cleaned before parsing
- Extended symbol spec format: ={"symbol":"EXCHANGE:SYMBOL","adjustment":"dividends",...}
  — leading = is mandatory (WAC-6: omitting it sends wrong symbol)
- Session ID format: {session_type}_{12 random alphanumeric} — e.g. qc_Ab3Xy9Mn2Pq
- Protocol ID naming conventions:
    Chart sessions: cs_*
    Series:         sds_* (series data store) / s* (series reference)
    Studies:        st* / *_st*
- study_loading and series_loading wire events both map to OnSeriesLoading — same handler
  (WAC-11: looking for separate study handler)
- Replay completion detection: string-contains check on JSON —
  msg_json.contains("replay") && msg_json.contains("data_completed")
  — fragile to TV protocol changes (WAC-6: don't strengthen this without a fixture test)

Auth
- Login POST: application/x-www-form-urlencoded, body username={}&password={}&remember=true
  — remember=true is always sent
- 2FA flow: first login → detect response["error"] == "2FA_required" → POST TOTP code to
  /accounts/two-factor/signin/totp/
- Cookie format for requests: sessionid={s}; sessionid_sign={sig}; device_t={token};
  — exact key names matter
- No-2FA accounts log a warn!() — not an error (WAC-2: agent might treat warning as failure)

bar_count default is 500,000 — ChartOptions default requests up to 500K bars. Reduce if memory
is a concern. (WAC-2)


HANDLES

- Tests (unit only, no network):        cargo test --lib
- Tests (integration, live TV, no auth): cargo test --test misc_test  [requires network]
- Tests (auth-required):                just test-user  [all currently commented out in tests/user_test.rs]
- All tests + features:                 just quick-test  (= cargo test --all-features)
                                        put auth tokens in .env — Justfile uses set dotenv-load
- Run example:                          just example <name>  (e.g. just example historical_data)
- Lint:                                 just clippy  (= cargo clippy --all-features --fix -- -D warnings, auto-fixes)
- Disable user feature cleanly:         cargo build --no-default-features --features rustls-tls
                                        (must explicitly keep TLS or it drops to no-TLS) (WAC-2)
- list_symbols() makes up to 30 concurrent HTTP requests — account for this in rate-limit-sensitive
  environments (WAC-6)
- get_builtin_indicators(BuiltinIndicators::All) makes 3 parallel HTTP requests
  (fundamental + standard + candlestick) (WAC-6)


DECISIONS

- rustls-tls over native-tls — default feature; compile-time switch via feature flags, not runtime
  config (WAC-7)
- History fetch uses WS internally, not HTTP — even one-shot retrieve() opens and tears down a
  WebSocket connection. This is TV's protocol design.
  (WAC-7: don't try to swap in an HTTP fetch)
- Cargo.lock is committed — atypical for a library but intentional (WAC-7)
- All public functions from client/misc.rs are re-exported at crate root via
  pub use crate::client::misc::* — callers use tradingview::advanced_search_symbol() not
  tradingview::client::misc::advanced_search_symbol() (WAC-11: looking in wrong namespace)
- single::retrieve and batch::retrieve share the same function name — disambiguate via full path
  history::single::retrieve(...) or history::batch::retrieve(...) (WAC-11)


APPENDIX: Protocol quick-reference
Not WAC-blocking — reference only for protocol work.

Wire event          TradingViewDataEvent    TradingViewResponse
------------------  ----------------------  ----------------------------------
timescale_update    OnChartData             ChartData(SeriesInfo, Vec<DataPoint>)
du                  OnChartDataUpdate       ChartData(...)
qsd                 OnQuoteData             QuoteData(QuoteValue)
quote_completed     OnQuoteCompleted        QuoteCompleted(Vec<Value>)
series_loading      OnSeriesLoading         SeriesLoading(LoadingMsg)
study_loading       OnSeriesLoading         StudyLoading(LoadingMsg)
series_completed    OnSeriesCompleted       SeriesCompleted(Vec<Value>)
symbol_resolved     OnSymbolResolved        SymbolInfo(SymbolInfo)
symbol_error        OnError(SymbolError)    Error(...)
critical_error      OnError(CriticalError)  Error(...)
replay_ok           OnReplayOk              ReplayOk(Vec<Value>)
replay_data_end     OnReplayDataEnd         ReplayDataEnd(Vec<Value>)

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
- **≤50 lines (`--fast`)**: a State Vector that grows without bound becomes the thing it was meant to replace. The ceiling is calibrated for human working memory.
- **No ceiling (`--deep`)**: frontier AI agents have context windows large enough that a 250-line State Vector costs nothing to hold. The deep version is mined for subsystem-level invariants that only matter once you're inside a specific area of the codebase — buried thresholds, constraint inheritance, look-alike traps — then distilled back to ≤50 lines for human and smaller-model handoff. The deep output reads like the braindump of someone who's been working on the codebase for months: the stuff you only know after hitting the third weird bug.

Full rationale in [`references/philosophy.md`](references/philosophy.md).
