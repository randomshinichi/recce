DEEP STATE VECTOR

PROJECT: Agave monorepo (validator/runtime/tooling)

LEGEND:
- D (Detectability): `silent` = likely wrongness passes normal validation; `caught` = likely wrongness fails a named guard. If guard is unspecified, default to `silent`.
- C (Confidence): `0.10–1.00` in `0.05` steps. `1.00` = directly verified in discovery/read files; `0.70–0.99` = strong evidence with inference; `<0.70` = inference-heavy.
- F (Freshness): `stable` = low drift risk; `verify` = medium drift risk; `volatile` = high drift risk.

MAP:
- Workspace is a large Rust monorepo (100+ crates) rooted at top-level `Cargo.toml`; several trees are intentionally excluded from workspace resolution (`ci/xtask`, `dev-bins`, `platform-tools-sdk`, `programs/sbf`, `svm/tests/example-programs`). _(WAC-4)_ [D:silent] [C:1.00] [F:verify] (Cargo.toml)
- `./cargo` is the primary build wrapper: it injects a rustup toolchain (`stable` default, `nightly` optional) via `ci/rust-version.sh` rather than using raw `cargo` behavior. _(WAC-1)_ [D:silent] [C:1.00] [F:stable] (cargo, ci/rust-version.sh)
- Root toolchain pin (`rust-toolchain.toml`) is independent from SBF platform-tools toolchain pin (`platform-tools-sdk/*`), creating two Rust-version domains in one repo. _(WAC-5)_ [D:silent] [C:1.00] [F:verify] (rust-toolchain.toml, platform-tools-sdk/cargo-build-sbf/src/toolchain.rs)
- Validator control plane is in `validator/src/main.rs` → `commands::run::execute`; shell scripts (`scripts/run.sh`) are local orchestration wrappers, not source-of-truth for runtime semantics. _(WAC-11)_ [D:silent] [C:1.00] [F:stable] (validator/src/main.rs, validator/src/commands/run/execute.rs, scripts/run.sh)
- Persistent operator/user state is externalized into home-directory config paths (`~/.config/solana/...`, `~/.local/share/solana/install`, `~/.cache/solana/...`). _(WAC-2)_ [D:silent] [C:1.00] [F:stable] (install/src/defaults.rs, cli-config/src/config.rs, platform-tools-sdk/cargo-build-sbf/src/toolchain.rs)
- Storage data-format boundary is split: `storage-proto` Rust types are build-generated from `.proto`, while `storage-bigtable` supports protobuf+bincode compatibility reads. _(WAC-6)_ [D:silent] [C:1.00] [F:verify] (storage-proto/README.md, storage-proto/build.rs, storage-bigtable/src/bigtable.rs)

ROUTES:
- Change validator startup behavior, node config derivation, or capability handling → CORRECT: `validator/src/commands/run/execute.rs`; NOT: `scripts/run.sh` only. (+ guard: `agave-validator run --help` + startup path) _(WAC-11)_ [D:silent] [C:1.00] [F:verify] (validator/src/commands/run/execute.rs)
- Change CLI flags/defaults/help for validator runtime → CORRECT: `validator/src/commands/run/args.rs` (flag definitions) and `validator/src/cli.rs` (`DefaultArgs`, app wiring); NOT: execute logic only. (+ guard: `agave-validator --help`) _(WAC-11)_ [D:caught] [C:1.00] [F:verify] (validator/src/commands/run/args.rs, validator/src/cli.rs)
- Change `init` vs `run` semantics for validator startup → CORRECT: `validator/src/main.rs` dispatch + `execute::Operation` handling; NOT: only CLI docs. (+ guard: `agave-validator init ...` exits after initialization) _(WAC-9)_ [D:caught] [C:0.95] [F:verify] (validator/src/main.rs, validator/src/commands/run/execute.rs)
- Change local single-node dev cluster behavior (genesis/bootstrap args, key generation, default ports) → CORRECT: `scripts/run.sh` plus matching validator flags in `validator/src/commands/run/args.rs`; NOT: one side only. (+ guard: `scripts/run.sh` bootstraps and starts validator) _(WAC-1)_ [D:silent] [C:1.00] [F:verify] (scripts/run.sh, validator/src/commands/run/args.rs)
- Change SBF toolchain installation/versioning/target triple resolution → CORRECT: `platform-tools-sdk/cargo-build-sbf/src/toolchain.rs` (+ `src/main.rs` parsing), and install script in `platform-tools-sdk/sbf/scripts/install.sh`; NOT: root `rust-toolchain.toml` only. (+ guard: `cargo build-sbf --version` / `--install-only`) _(WAC-4)_ [D:silent] [C:1.00] [F:volatile] (platform-tools-sdk/cargo-build-sbf/src/main.rs, platform-tools-sdk/cargo-build-sbf/src/toolchain.rs, platform-tools-sdk/sbf/scripts/install.sh)
- Change BigTable auth/proxy/cert behavior → CORRECT: `storage-bigtable/src/access_token.rs`, `storage-bigtable/src/bigtable.rs`, `storage-bigtable/src/root_ca_certificate.rs`; NOT: README only. (+ guard: emulator/proxy env paths) _(WAC-2)_ [D:silent] [C:1.00] [F:verify] (storage-bigtable/src/access_token.rs, storage-bigtable/src/bigtable.rs, storage-bigtable/src/root_ca_certificate.rs)
- Change hard-fork arg parsing utility used in validator and ledger tooling → CORRECT: both `validator/src/commands/run/execute.rs` and `ledger-tool/src/main.rs`; NOT: one binary only. (+ guard: both CLIs parse `--hard-fork`) _(WAC-5)_ [D:silent] [C:0.95] [F:verify] (validator/src/commands/run/execute.rs, ledger-tool/src/main.rs)
- Change production “build from source” install flow → CORRECT: `scripts/cargo-install-all.sh` (binary selection/profile/install root) and docs (`docs/src/cli/install.md`); NOT: README build snippet alone. (+ guard: built binaries in `<install>/bin`) _(WAC-9)_ [D:silent] [C:1.00] [F:verify] (scripts/cargo-install-all.sh, docs/src/cli/install.md)

RULES:
- Using raw `cargo` can silently diverge from CI/toolchain assumptions; wrapper `./cargo` selects repo-pinned stable/nightly via rustup unless `NO_RUSTUP_OVERRIDE` is set. _(WAC-1)_ [D:silent] [C:1.00] [F:stable] (cargo, ci/rust-version.sh)
- Root Rust pin is `1.93.0`, while SBF platform-tools defaults to `v1.53` with base rust `1.89.0`; SBF build logic may run a different rustc toolchain than host workspace builds. _(WAC-5)_ [D:silent] [C:1.00] [F:volatile] (rust-toolchain.toml, platform-tools-sdk/cargo-build-sbf/src/toolchain.rs)
- `validator` startup force-sets `RUST_BACKTRACE=1` if absent and logs full argv; behavior is in execute path, not shell wrappers. _(WAC-2)_ [D:silent] [C:1.00] [F:stable] (validator/src/commands/run/execute.rs)
- Linux capability reduction is deliberate: non-run subcommands drop permitted caps; XDP setup must occur before spawning threads to avoid capability leakage. _(WAC-10)_ [D:silent] [C:0.95] [F:verify] (validator/src/main.rs, validator/src/commands/run/execute.rs)
- In multihoming mode (`--bind-address` repeated), `--public-tvu-address`, `--public-tpu-address`, and `--advertised-ip` have explicit constraints/error paths; do not assume single-homing semantics apply. _(WAC-2)_ [D:caught] [C:0.95] [F:verify] (validator/src/commands/run/execute.rs, validator/src/commands/run/args.rs)
- `--restricted-repair-only-mode` implicitly disables voting and strips multiple published services; this is a runtime mode fork, not a minor tuning flag. _(WAC-9)_ [D:silent] [C:0.95] [F:verify] (validator/src/commands/run/args.rs, validator/src/commands/run/execute.rs)
- Snapshot interval semantics fork on incremental-fetch mode: `--snapshot-interval-slots` maps to incremental interval when enabled, otherwise full interval; `--full-snapshot-interval-slots` can be ignored with warning in disabled path. _(WAC-2)_ [D:silent] [C:0.95] [F:verify] (validator/src/commands/run/execute.rs, validator/src/commands/run/args.rs)
- `--enable-accounts-disk-index` is deprecated but still mapped to `IndexLimit::Minimal`; deprecation does not mean no-op. _(WAC-7)_ [D:silent] [C:0.95] [F:verify] (validator/src/cli.rs, validator/src/commands/run/execute.rs)
- If `--vote-account` is omitted, validator emits warning and disables voting at runtime even without `--no-voting`. _(WAC-2)_ [D:silent] [C:1.00] [F:stable] (validator/src/commands/run/execute.rs)
- Localnet helper (`scripts/run.sh`) enables `--require-tower` and `--no-os-network-limits-test`; copying this recipe to production contexts can bypass intended startup checks. _(WAC-10)_ [D:silent] [C:1.00] [F:verify] (scripts/run.sh)
- `lock_ledger()` enforces single-process access via `ledger.lock`; second validator run on same ledger exits with explicit lock error. _(WAC-10)_ [D:caught] [C:1.00] [F:stable] (validator/src/lib.rs)
- Protobuf Rust structs in `storage-proto` are generated from `proto/*.proto`; editing generated structs directly is the wrong locus. _(WAC-3)_ [D:silent] [C:1.00] [F:stable] (storage-proto/README.md, storage-proto/build.rs)
- BigTable connection mode is env-gated (`BIGTABLE_EMULATOR_HOST`, `BIGTABLE_PROXY`, `GOOGLE_APPLICATION_CREDENTIALS`, `GRPC_DEFAULT_SSL_ROOTS_FILE_PATH`); wrong env silently shifts auth/transport behavior. _(WAC-2)_ [D:silent] [C:1.00] [F:verify] (storage-bigtable/src/bigtable.rs, storage-bigtable/src/access_token.rs, storage-bigtable/src/root_ca_certificate.rs)
- `ledger-tool` is subcommand-required and has top-level aliases for blockstore subcommands; command shape differs from validator CLI assumptions. _(WAC-9)_ [D:caught] [C:0.95] [F:stable] (ledger-tool/src/main.rs)

HANDLES:
- Workspace build/test with repo-pinned toolchain: `./cargo build`, `./cargo test` _(WAC-1)_
- Force stable/nightly in wrapper path: `./cargo stable <cmd>`, `./cargo nightly <cmd>` _(WAC-1)_
- Run local single-node cluster: `./scripts/run.sh` (expects built binaries on `target/<profile>`) _(WAC-1)_
- CI-equivalent stable test bundle: `ci/test-stable.sh` _(WAC-1)_
- Install all distributable binaries from source: `./scripts/cargo-install-all.sh <install-dir>` _(WAC-9)_
- SBF builder auto-installer path: `./cargo-build-sbf ...` / `./cargo-test-sbf ...` _(WAC-1)_
- Recover/prepare platform tools only: `cargo build-sbf --install-only [--tools-version vX.Y]` _(WAC-4)_
- Validator initialize-only mode (no full run): `agave-validator init --ledger <DIR> ...` _(WAC-9)_

DECISIONS:
- Deprecated validator args are intentionally retained (hidden by default) to avoid breaking existing operational startup commands. _(WAC-7)_ [D:silent] [C:1.00] [F:verify] (validator/src/cli.rs)
- Monorepo keeps host and SBF toolchains decoupled; platform-tools can be installed/linked per-version under `~/.cache/solana/<version>/platform-tools`. _(WAC-7)_ [D:silent] [C:1.00] [F:verify] (platform-tools-sdk/cargo-build-sbf/src/toolchain.rs)
- BigTable deserialization intentionally supports protobuf first with bincode fallback for compatibility across historical storage encodings. _(WAC-6)_ [D:silent] [C:1.00] [F:stable] (storage-bigtable/src/bigtable.rs)
- Validator and ledger-tool each own overlapping CLI/data-path logic (e.g., hard-forks parsing), so some duplication is deliberate binary-level independence. _(WAC-4)_ [D:silent] [C:0.90] [F:verify] (validator/src/commands/run/execute.rs, ledger-tool/src/main.rs)
