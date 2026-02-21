DEEP STATE VECTOR

PROJECT: Go toolchain source tree (bootstrap + dist)

MAP:
- Build/test entrypoint fork is under `src/` scripts (`make.bash`, `all.bash`, `run.bash`, `race.bash`), not repo root scripts. _(WAC-1)_
- Real orchestration lives in `src/cmd/dist` (`cmdbootstrap`, `cmdtest`, `cmdlist`, `cmdenv`); shell scripts are thin wrappers. _(WAC-11)_
- Bootstrap workspace is generated in `$GOROOT/pkg/bootstrap/src/bootstrap`, then removed on exit. _(WAC-11)_
- Two module boundaries exist: `src/go.mod` (`module std`) and `src/cmd/go.mod` (`module cmd`); vendor behavior differs inside/outside those trees. _(WAC-4)_
- Vendored externals are resolved via `src/vendor` or `src/cmd/vendor` using import-path rewriting (`resolveVendor`). _(WAC-6)_
- Dist-generated artifacts are identified by `generatedHeader` and often `z*.go`; cleanup logic depends on that marker. _(WAC-3)_

ROUTES:
- Change bootstrap minimum version → CORRECT: `src/cmd/dist/buildtool.go` (`minBootstrap`), `src/cmd/dist/notgo124.go` build tag/package guard, `src/make.bash` (`bootgo`), `src/cmd/dist/README`; NOT: only docs or only `go.mod`. (+ guard: bootstrap failure message references Go 1.24.6 when too old) _(WAC-6)_
- Change which platforms are supported/broken → CORRECT: `src/cmd/dist/build.go` (`cgoEnabled`, `broken`, `firstClass`, `cmdlist`); NOT: editing `src/go.mod` or tests only. (+ guard: `go tool dist list` / `-broken`) _(WAC-11)_
- Change default dist test sharding/selection → CORRECT: `src/cmd/dist/test.go` (`registerTests`, `GO_TEST_SHARDS`, `-run`, `-list`), with wrapper awareness in `src/run.bash`; NOT: only `run.bash`. (+ guard: `go tool dist test -list`/`-run`) _(WAC-11)_
- Change environment emitted by dist or bootstrap env defaults → CORRECT: `src/cmd/dist/build.go` (`xinit`, `cmdenv`, `toolenv`); NOT: `go.env` alone. (+ guard: `go tool dist env -p`) _(WAC-2)_
- Change generation/cleanup behavior for produced source files → CORRECT: `src/cmd/dist/buildgo.go` (`generatedHeader`, generators) + `src/cmd/dist/build.go` (`clean`); NOT: deleting generated files manually. (+ guard: `go tool dist clean`) _(WAC-3)_
- Add/rename dist subcommands → CORRECT: `src/cmd/dist/main.go` (`commands`, `usage`) + `src/cmd/dist/doc.go`; NOT: docs only. (+ guard: `go tool dist [command]` usage) _(WAC-11)_
- Change bootstrap package import rewriting → CORRECT: `src/cmd/dist/buildtool.go` (`bootstrapFixImports`, `bootstrapDirs`) + `src/cmd/dist/imports.go`; NOT: editing only vendored trees. (+ guard: `bootstrap/cmd/...` build succeeds) _(WAC-4)_

RULES:
- `make.bash`, `all.bash`, `run.bash`, `race.bash` must be run from `$GOROOT/src`; wrong cwd hard-fails. _(WAC-1)_
- `make.bash` must not be used on Windows (explicit error path says use `make.bat`). _(WAC-9)_
- `GOROOT_BOOTSTRAP/bin/go` is required and must be `>= go1.24.6`; setting `GOROOT_BOOTSTRAP=$GOROOT` is a hard error. _(WAC-2)_
- `requiredBootstrapVersion` derives from `src/go.mod` via N-2 rounded down to even; for `go 1.27`, minimum bootstrap is `1.24`. _(WAC-6)_
- `notgo124.go` intentionally creates a package mismatch if building dist with too-old Go; do not “fix” that mismatch. _(WAC-7)_
- `cmd/dist` intentionally reimplements logic (build tags/import parsing/platform support) instead of importing unstable internals to avoid bootstrap version skew. _(WAC-7)_
- `runInstall` forbids go_bootstrap depending on cgo packages `net`, `os/user`, `crypto/x509` (fatal). _(WAC-6)_
- `toolenv` disables cgo (`CGO_ENABLED=0`) unless external linking is required, to keep shipped tools static/reproducible. _(WAC-6)_
- In `cmdbootstrap`, `GOPATH` is forced to `$GOROOT/pkg/obj/gopath` and `GOPROXY=off` to avoid readonly modcache pollution in GOROOT. _(WAC-10)_
- `GOEXPERIMENT` is forced to `none` for toolchain1/go_bootstrap, then restored for toolchain2+. _(WAC-5)_
- `clean` skips `vendor` and `testdata` directory walks for generated-source deletion logic. _(WAC-3)_
- `generatedHeader` in `buildgo.go` is a cross-revision cleanup key; changing it breaks cleanup across branch switches. _(WAC-3)_
- `all.bash` does build+test (`make.bash --no-banner`, `run.bash --no-rebuild`, then dist banner); `run.bash` alone expects existing `../bin/go`. _(WAC-1)_
- Broken ports (`freebsd/riscv64`, `linux/sparc64`, `openbsd/mips64`) are blocked unless `dist bootstrap -force`; `dist list` hides them unless `-broken`. _(WAC-2)_
- `GO_TEST_SHORT` is parsed as bool; `GO_TEST_SHARDS` defaults to 1 (or 10 when `GO_BUILDER_NAME` set). _(WAC-2)_
- Race tests may be marked skipped when output contains `unsupported VMA range` despite `raceDetectorSupported` allowing arm64. _(WAC-5)_
- FIPS test matrix is gated: `GOEXPERIMENT=boringcrypto` disables `GOFIPS140` path; unsupported for wasm/openbsd/aix/windows-386 and ASAN. _(WAC-5)_
- Cross-compile path diverges after host-tool install in `cmdbootstrap`; some install paths intentionally skip direct build when `goos/goarch != gohostos/gohostarch`. _(WAC-5)_
- `make.bash` contains explicit “DO NOT ADD ANY NEW CODE HERE” after final bootstrap+rm; post-bootstrap additions belong in dist code (`cmdbootstrap`). _(WAC-7)_

HANDLES:
- Bootstrap build from source: `cd src && ./make.bash` _(WAC-1)_
- Full local validation path: `cd src && ./all.bash` _(WAC-1)_
- Test-only after existing build artifacts: `cd src && ./run.bash --no-rebuild` _(WAC-1)_
- Race workflow (supported hosts only): `cd src && ./race.bash` _(WAC-9)_
- Dist env snapshot used by scripts: `cd src && ../bin/go tool dist env -p` _(WAC-2)_
- Enumerate test targets before filtering: `cd src && ../bin/go tool dist test -list` _(WAC-11)_
- Focus dist tests by regex: `cd src && ../bin/go tool dist test -run='<rx>'` (invert with leading `!`). _(WAC-11)_
- Show hidden broken ports too: `cd src && ../bin/go tool dist list -broken` _(WAC-2)_
- Cross-bootstrap pack generation: `cd src && GOOS=<os> GOARCH=<arch> ./bootstrap.bash [-force]` _(WAC-9)_

DECISIONS:
- Toolchain is intentionally rebuilt in stages (`toolchain1 -> go_bootstrap -> toolchain2 -> toolchain3`) to converge semantics and build IDs. _(WAC-7)_
- Dist keeps bootstrap-critical copies of platform/buildmode/race logic instead of importing internal packages. _(WAC-7)_
- Bootstrap source is rewritten into `bootstrap/...` import space and forbids accidental direct `internal/...` imports from bootstrap toolchain. _(WAC-4)_
- `GOBIN` is forced to `$GOROOT/bin` during dist init so dist-installed binaries are deterministic. _(WAC-6)_


DISTILLED STATE VECTOR (<=50 lines)

PROJECT: Go source bootstrap + dist control plane

MAP:
- Real build/test control plane is `src/cmd/dist`; `src/*.bash` are wrappers.
- Two module domains (`src/go.mod` = std, `src/cmd/go.mod` = cmd) + split vendoring (`src/vendor`, `src/cmd/vendor`).
- Generated-source lifecycle depends on `generatedHeader` + `z*` files (dist clean logic).

ROUTES:
- Bootstrap version changes → CORRECT: `src/cmd/dist/buildtool.go` (`minBootstrap`), `src/cmd/dist/notgo124.go`, `src/make.bash` (`bootgo`), `src/cmd/dist/README`; NOT: `go.mod` only.
- Platform support/broken-port changes → CORRECT: `src/cmd/dist/build.go` (`cgoEnabled`, `broken`, `firstClass`, `cmdlist`); NOT: tests/docs only.
- Dist test sharding/filter behavior → CORRECT: `src/cmd/dist/test.go`; NOT: `src/run.bash` only.
- Dist env/default build env changes → CORRECT: `src/cmd/dist/build.go` (`xinit`, `cmdenv`, `toolenv`); NOT: `go.env` only.
- Dist command surface changes → CORRECT: `src/cmd/dist/main.go` (+ `doc.go`); NOT: docs only.
- Generated-file policy changes → CORRECT: `src/cmd/dist/buildgo.go` + `src/cmd/dist/build.go:clean`; NOT: manual file edits in generated outputs.

RULES:
- Always run `make.bash`/`all.bash`/`run.bash` from `$GOROOT/src`; wrong cwd fails.
- `make.bash` rejects Windows; use `make.bat` there.
- `GOROOT_BOOTSTRAP` must provide `bin/go`, be `>= go1.24.6`, and must not equal `GOROOT`.
- `notgo124.go` package mismatch is intentional guard for too-old bootstrap compilers.
- `requiredBootstrapVersion` comes from `src/go.mod`; for `go 1.27`, floor is `1.24`.
- `runInstall` forbids go_bootstrap cgo deps on `net`, `os/user`, `crypto/x509`.
- `cmdbootstrap` forces `GOPATH=$GOROOT/pkg/obj/gopath` and `GOPROXY=off`.
- `toolenv` disables cgo unless external linking is required.
- `GOEXPERIMENT` is forced to `none` for early bootstrap, then restored later.
- Do not change `generatedHeader`; cleanup across revision switches depends on it.
- Broken ports are blocked unless `dist bootstrap -force`; hidden from `dist list` unless `-broken`.
- `GO_TEST_SHARDS` defaults to 1 (10 when `GO_BUILDER_NAME` is set).

HANDLES:
- Build toolchain: `cd src && ./make.bash`
- Full build+tests: `cd src && ./all.bash`
- Test existing build only: `cd src && ./run.bash --no-rebuild`
- Dist test discovery/filter: `cd src && ../bin/go tool dist test -list` / `-run='<rx>'`
- Platform matrix: `cd src && ../bin/go tool dist list -broken`
- Cross-bootstrap bundle: `cd src && GOOS=<os> GOARCH=<arch> ./bootstrap.bash [-force]`

DECISIONS:
- 3-stage toolchain rebuild (`toolchain1/2/3`) is deliberate for build-ID correctness.
- Dist duplicates bootstrap-critical logic (tags/import/platform support) to avoid bootstrap-version skew.
- Bootstrap build rewrites imports into `bootstrap/...` namespace instead of compiling directly from live tree.
