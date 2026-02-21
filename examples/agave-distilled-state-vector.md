DISTILLED STATE VECTOR (<=50 lines)

PROJECT: Agave validator/runtime monorepo

MAP:
- Primary build entrypoint is `./cargo` (toolchain wrapper), not raw `cargo`.
- Runtime control plane lives in `validator/src/commands/run/execute.rs`; scripts are wrappers.
- Workspace excludes important trees (`ci/xtask`, `dev-bins`, `platform-tools-sdk`, `programs/sbf`, `svm/tests/example-programs`).
- Two toolchain domains exist: root Rust pin (`rust-toolchain.toml`) and SBF platform-tools pin.
- Critical state/config paths are outside repo (`~/.config/solana/...`, `~/.local/share/solana/install`, `~/.cache/solana/...`).

ROUTES:
- Validator startup behavior/capability logic → `validator/src/commands/run/execute.rs`, NOT `scripts/run.sh` only.
- Validator CLI flags/defaults/help → `validator/src/commands/run/args.rs` + `validator/src/cli.rs`, NOT execute.rs only.
- `init` vs `run` behavior → `validator/src/main.rs` dispatch + `execute::Operation` handling.
- Local cluster defaults/genesis bootstrap script behavior → `scripts/run.sh` plus matching runtime flags in validator args.
- SBF toolchain/version/target logic → `platform-tools-sdk/cargo-build-sbf/src/{main,toolchain}.rs` (+ `platform-tools-sdk/sbf/scripts/install.sh`), NOT root toolchain file only.
- BigTable auth/proxy/cert/env behavior → `storage-bigtable/src/{access_token,bigtable,root_ca_certificate}.rs`.
- `--hard-fork` parser/flow changes used by both binaries → touch both `validator/.../execute.rs` and `ledger-tool/src/main.rs`.
- Generated storage proto updates → edit `storage-proto/proto/*.proto`, not generated Rust structs directly.

RULES:
- `./cargo` injects rustup toolchain unless `NO_RUSTUP_OVERRIDE`; raw `cargo` can silently diverge.
- Root Rust is pinned (`1.93.0`) but SBF platform-tools default uses its own Rust (`1.89.0` via `v1.53`).
- Linux validator startup aggressively trims capabilities; XDP setup must happen before thread spawn.
- In multihoming, `--public-tvu-address` and related public-address flags have strict constraints.
- `--restricted-repair-only-mode` implies nonstandard runtime mode (including no voting).
- Snapshot interval semantics fork on incremental snapshot fetch mode; `--full-snapshot-interval-slots` can be ignored in one branch.
- Deprecated `--enable-accounts-disk-index` still forces `IndexLimit::Minimal` (not a no-op).
- Omitting `--vote-account` disables voting at runtime.
- `scripts/run.sh` uses local/dev assumptions (`--require-tower`, `--no-os-network-limits-test`).
- Ledger locking (`ledger.lock`) enforces single validator process per ledger dir.
- BigTable behavior is heavily env-gated (`BIGTABLE_EMULATOR_HOST`, `BIGTABLE_PROXY`, `GOOGLE_APPLICATION_CREDENTIALS`, `GRPC_DEFAULT_SSL_ROOTS_FILE_PATH`).

HANDLES:
- Build/test with repo toolchain: `./cargo build`, `./cargo test`
- Explicit channel wrapper: `./cargo stable <cmd>` / `./cargo nightly <cmd>`
- Local single-node cluster: `./scripts/run.sh`
- Stable CI-like suite: `ci/test-stable.sh`
- Source install of distributable binaries: `./scripts/cargo-install-all.sh <install-dir>`
- SBF toolchain prep/recovery: `cargo build-sbf --install-only [--tools-version vX.Y]`
- Initialize ledger only (no full validator run): `agave-validator init --ledger <DIR> ...`

DECISIONS:
- Hidden deprecated validator flags are intentionally kept for operator compatibility.
- Host toolchain and SBF toolchain are intentionally decoupled.
- BigTable reads intentionally support protobuf + bincode compatibility.
- Some validator/ledger-tool CLI logic duplication is deliberate binary independence.
