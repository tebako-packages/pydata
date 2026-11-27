# pydata feedstock

Feedstock for **pydata** — the curated python data-science toolkit
(numpy + pandas + matplotlib), packaged as a tebako `kind: app` payload
riding the tebako python runtime. The second flagship payload of the v2
ecosystem (TODO.sota2/05): metanorma proved the calculus on the hardest
ruby case; this payload proves it generalizes across languages — native
wheels with BLAS linkage, data files, and the install pain python is
famous for, answered hermetically.

**Status: build + gate + publish pipeline (owner-dispatched).** The three
desktop-triplet legs stage the closure, press the image, and run the boot
smoke + the three named conformance arms; the wrong-line-refusal job
proves the named exit-69 ABI refusal on every run. The publish job (tag
dispatch with `publish=true`, the owner's explicit act — never automatic)
uploads the signed assets + writes the registry rows via bot PR. The first
real publish awaits the owner's go.

## Design decisions

1. **Per-triplet, never universal.** The closure is compiled members end
   to end (numpy's BLAS core, pandas' cython members, matplotlib's
   ft2font + kiwisolver + contourpy + pillow) — a universal image cannot
   serve that matrix.
2. **ABI-line runtime edge.** Every entrypoint's `runtime_requirement` is
   `engine: python, constraint: ~> 3.14.0, abi: <staging platform tag>` —
   spec 05 §5's native-extension law (wrong line = named error, never a
   segfault). The abi string flows VERBATIM from the runtime package's
   own `.abi` sidecar (e.g. `cpython-314-darwin.so` — suffix included;
   the factory is the single owner). The staging/exec line is python
   **3.14.7** (tebako runtime **0.3.5**, the post-tebako#716 naming era:
   `tebako-runtime-<ver>-python-<py>-<host>`) — the one line every leg
   rides, so the ABI edge stays uniform across the matrix (the factory's
   windows-ucrt64 runtime exists only on the 3.14 line).
3. **pipstage build, wheels only.** `tools/gen_closure` resolves the
   recipe's member pins (Tebakofile `upstream.members` — the
   requirements.in) with the STAGED tebako python runtime's OWN pip
   (dogfood — never the runner's python, never the user's machine) into
   `closure/<version>-<triplet>.txt` (wheel filenames + sha256, the
   metanorma/xml2rfc pattern), and `tools/build` installs that lock with
   `pip install --target --require-hashes` into the payload's
   `/lib/python/site-packages`. **Source builds are never a feedstock
   path** (the audience law: payload consumers need no compilation
   machinery): `--only-binary=:all:` makes an sdist resolution a named
   failure, and the generator then prints what PyPI actually offers for
   each native member (fail closed, never a compile). The imager is
   **`tfs mkimage`** (default limnifs — the format the runtime factory
   itself ships). pip never runs at dispatch time: the image is the
   install.
4. **Release channel (the factory's shard model).** Staging + CI test
   legs consume the pinned tebako-runtime-python RELEASE —
   `tools/stage_runtime` enumerates the pinned tag's per-package manifest
   shards (`gh release view --json assets`), fetches the assets each
   shard names (exe + env image + the windows `.dll` facet + the
   `.tfs.blksum` facet, each with its `.sha256` sidecar), verifies every
   staged byte three ways (sidecar ↔ staged bytes ↔ the shard's own
   digest fields + the filename identity), and derives the TRIMMED
   `manifest.json` + `SHA256SUMS.txt` consumer-side into a `file://`
   runtime MIRROR (spec 05 §2's download-base shape). The pin is
   `Tebakofile`'s `build.runtime` block (`channel: release` + `release:
   "v0.3.5"`).
5. **Windows-ucrt rides msys2 prebuilts, not PyPI.** PyPI's `win_amd64`
   wheels are MSVC-built (the cp314 win_amd64 ABI); the tebako windows
   runtime is an msys2/ucrt mingw cpython (`cp314-mingw_x86_64_ucrt_gnu`)
   that cannot load them, and PyPI publishes no mingw wheels — the
   xml2rfc feedstock's lxml lesson, here applied to the whole stack since
   numpy/pandas/matplotlib are ALL compiled. `tools/gen_closure` refuses
   the windows triplet by name; `tools/stage_msys2` stages the stack from
   msys2's signed PREBUILT ucrt64 packages via pacman (never a source
   build — the same audience-law reading as the runtime factory's own
   windows support set), collects the PE import closure of the staged
   pyds via objdump (a fixpoint walk; system DLLs excluded), and records
   the staged set in `closure/<version>-x86_64-windows-ucrt.txt`. The
   windows lock is a RECORD of prebuilt packages (pacman's signature
   verification is the trust anchor) where the POSIX locks are
   hash-INPUTS — the asymmetry is deliberate and documented here. The
   boot rides the factory's materialize tier (`windows_boot:
   materialize` on the msys env image — a windows payload boots on its
   own platform).
6. **Spec 32 spawn form.** The console script dispatches through the
   provider's own spec-17 dispatch — no host-tier exe materialization, no
   POSIX-only shell shim. The in-image `/bin/pydata` stub derives
   site-packages from its OWN location (the mount point is the
   dispatcher's choice).
7. **The three conformance arms are CI gates** (TODO.sota2/05 arc 2), run
   per leg by `tools/conformance` through the spec-17 wire, each under a
   hard deadline (a hang fails the arm by name, never a wedged run):
   - **mount layout** — a numpy import + BLAS compute inside the mounted
     image (the wheel's bundled shared libraries load through the VFS);
   - **__file__ data** — the bundled sample CSV
     (`pydata_toolkit/data/rivers.csv`) read through a `__file__`-derived
     path in the mounted context, byte-verified against the build-time
     digest constant;
   - **mp spawn** — a `multiprocessing` spawn-context `Pool(2).map` that
     must return, never hang (the interpreter re-exec edge: the child
     re-execs `sys.executable` with the driver-armed environment
     inherited).
8. **Payload checks (spec 26).** `version` (exit 0) + `analyze` (fixture
   sample.csv → summary + PNG), carried in the manifest — the payload
   ships its own compile test.

## Deviations (each with its owner and removal condition)

A. *(none yet — reserved to keep the lettering stable with the sibling
   feedstocks)*

B. **pandas ctypes guard (`patches/pandas-3.1.1-guard-ctypes-import.patch`).**
   The POSIX tebako python runtime's hermetic core ships no `_ctypes`
   (the factory's `DISABLED_MODULES`; windows-msys re-enables it), while
   pandas 3.1.1 imports `ctypes` at module level in
   `pandas/errors/__init__.py` (used only by `PyperclipWindowsException`
   — a windows-only clipboard path) and in
   `pandas/core/interchange/from_dataframe.py` (used only inside
   `buffer_to_ndarray` — the dataframe-interchange protocol, never on
   the payload's own paths). The patch guards both imports
   (`try/except ModuleNotFoundError`, `ctypes = None`), applied by
   tools/build ONLY when a probe of the staged runtime confirms the
   absence (zero fuzz — a drifted pandas fails the leg loudly). The
   patch never lands on the windows leg (msys cpython ships `_ctypes`,
   probe-verified). **Owner: the runtime factory — ship `_ctypes` on
   POSIX (static libffi, the windows-msys pattern) or prune the ctypes
   python side so the absence is a clean "no such module". Remove this
   patch when the feedstock's runtime floor ships `_ctypes` on POSIX
   (tools/build's probe then prints the removal notice).**

C. **Linux-gnu known blocker: tamatebako/tebako#1096 (the ELF header
   window).** libtfs-preload's dependency walk parses only the first 1 MiB
   of a loadable module (`crates/tfs/src/exec_closure.rs`,
   `HEADER_WINDOW = 1<<20`). numpy 2.5.5's manylinux
   `_multiarray_umath…so` (10,609,009 B) carries its PT_DYNAMIC at offset
   ~9.9 MB, so `parse_elf` bails, the RUNPATH `$ORIGIN/../numpy.libs`
   sibling never materializes, and every numpy import on x86_64-linux-gnu
   dies `libscipy_openblas64_-….so: cannot open shared object file`
   (CI run 45004111997). macOS is unaffected (the Mach-O headers sit
   inside the window — arm 1 passes there); windows does not ride PyPI
   wheels at all. The linux leg therefore runs its boot-smoke +
   conformance through `tools/blocker_gate`, a TRI-STATE wrapper that
   never fakes green: full pass → green (the full gates apply
   automatically the moment the blocker clears); the exact #1096
   signature (`libscipy_openblas64` + `cannot open shared object file`)
   → green by NAMED EXCEPTION with a loud `::warning` citing the issue;
   any other failure → red. **Owner: tamatebako/tebako (#1096 — widen or
   stream the dep-walk's header window). Removal: the consumption chain
   (tebako release → link unit → factory rebuild → the runtime pin bump
   here); the gate's PASS branch then asserts itself and the wrapper is
   deleted.**

D. **Windows arm-3 known blocker: tamatebako/tebako-runtime-python#105
   (the missing `sem_unlink` export).** cpython 3.14's
   `multiprocessing/synchronize.py` imports `sem_unlink` from
   `_multiprocessing` UNCONDITIONALLY, and upstream's win32 build always
   provides it (on `MS_WINDOWS` the semaphore shim defines
   `SEM_UNLINK(name)` as a no-op). The factory's windows-ucrt64 build —
   on the mingw/autoconf path — drops the export behind
   `POSIX_SEMAPHORES_NOT_ENABLED`, so `Lock`/`Queue`/`Pool` all die at
   the synchronize import on the published runtime (CI run 45007247506;
   msys2's `mingw-w64-python` carries the fix as patch 0038, which the
   factory does not). Arm 3 (mp spawn) therefore rides
   `tools/blocker_gate` on the windows leg while arms 1+2 stay HARD
   gates: full pass → green; the exact #105 signature (`cannot import
   name 'sem_unlink'` + `_multiprocessing`) → green by NAMED EXCEPTION
   with a loud `::warning`; any other failure → red. **Owner:
   tamatebako/tebako-runtime-python (#105 — carry the msys2-0038
   equivalent). Removal: the runtime pin bump to a factory release
   carrying the fix; the gate's PASS branch then asserts itself.**

## Operational note: the exec-cache dlcache and rebuilt images

On macOS the driver's persistent dlcache keys a mount's materialized
files on the image's `.sha256` sidecar when present, else on the image's
PATH (a dev boot). `tools/build` therefore writes
`<image>.tfs.sha256` (the store-sidecar shape) next to every image it
presses — a rebuilt image keys differently and never serves a stale
extraction. Without the sidecar, re-pressing the same path and
re-dispatching on the same host serves the previous build's cached
files (observed in rehearsal: an edited conformance arm ran its old
bytes until the cache was cleared).

## Layout (tebako-packages/index `templates/feedstock/` + conventions)

- `Tebakofile` — member pins, pipstage runtime block, native members,
  the msys2 staging block, platforms, the tools sha256 pin block, the ci
  host pins.
- `manifests/payload.yaml` — the spec 03 manifest template (`@@…@@`
  filled by tools/build).
- `fixtures/check/analyze/sample.csv` — the analyze check's table.
- `closure/` — locked closures per (version × triplet); POSIX locks are
  hash-inputs (generated by tools/gen_closure, committed), the windows
  lock is the msys2 staging record (tools/stage_msys2).
- `templates/` — the payload's own authored content: `bin/pydata` (the
  spec-32 console stub) + `lib/python/site-packages/pydata_toolkit/` (the
  CLI, the bundled sample table, the three conformance arms).
- `tools/` — pins.rb (recipe → CI env/matrix/payload-args) /
  stage_runtime / gen_closure / stage_msys2 / build / boot_smoke /
  conformance / blocker_gate (deviations C + D) / set_registry_default.rb
  (the publish job's default move) / compare_install (the README table's
  numbers) / pypi_offers.py.
- `.github/workflows/build-payload.yml` — plan (matrix from the recipe) →
  the three per-triplet legs (POSIX pipstage host-native; windows via
  msys2) → boot-smoke + checks + the conformance arms → the wrong-line
  refusal acceptance → the publish job (tag dispatch with
  `publish=true` only: signed release assets + the registry bot PR).
- `tpkg-registry.yaml` — this package's registry; upserted by the publish
  job (never hand-edited — main is branch-protected, the registry lands
  by bot PR + auto-merge).

## Local rehearsal (one POSIX triplet, e.g. aarch64-macos)

```sh
tools/stage_runtime                      # the pinned factory release -> out/runtime-mirror
TRIPLET=aarch64-macos tools/gen_closure  # resolve + hash the wheel closure
TRIPLET=aarch64-macos tools/build        # pipstage -> out/aarch64-macos/pydata-*.tfs
set -a; . out/aarch64-macos/runtime.env; set +a
tools/boot_smoke "$PAYLOAD_IMG"          # image assertions + tebako check + dispatch smoke
tools/conformance "$PAYLOAD_IMG"         # the three arms, under deadlines
tools/compare_install                    # the comparison table below (needs network)
```

## Install UX, measured (tebako vs pip+venv vs conda)

One host, one script — `tools/compare_install` prints the paste-ready
table; re-run it to reproduce every number. Measured 2026-11-22 on an
arm64 Apple M1 Max (macOS 14.1.1) with homebrew python 3.14.6, tebako
2.8.60, micromamba 2.10.0. The tebako row is THIS feedstock's 0.1.0
aarch64-macos image on the factory's v0.3.5 runtime pair; the pip/conda
rows install the same member pins (numpy 2.5.5 / pandas 3.1.1 /
matplotlib 3.11.2).

| metric | tebako (this payload) | pip + venv | conda (conda-forge) |
|---|---|---|---|
| install wall time | 0.11 s (store copy; + 175.2 MB download — 81.6 MB payload per package, 93.6 MB runtime pair once per machine) | 12.54 s | 15.52 s |
| bytes on disk after install | 175.2 MB (payload 81.6 MB + runtime pair 93.6 MB, the runtime shared) | 205.3 MB | 427.7 MB |
| first command (cold) | 29.61 s | 10.18 s | 15.77 s |
| same command (warm, median of 5) | 4.17 s | 0.36 s | 0.39 s |

How to read it, honestly:

- **The install is the win.** One file, atomic, content-addressed, no
  solver, no environment to manage, no compiler — and the runtime pair
  (93.6 MB of the tebako bytes) lands once per machine and is SHARED by
  every python payload on the same ABI line, so the second tebako python
  package installs in sub-second time and adds only its payload bytes.
  pip re-resolves into every venv; conda pays its solver plus the
  largest footprint of the three.
- **The command latency is the honest cost.** The tebako probe is the
  payload's real first command — `pydata demo` imports numpy + pandas +
  matplotlib and renders the bundled CSV to a PNG (this CLI imports the
  whole stack at startup by construction) — while the pip/conda probes
  run a bare `import numpy, pandas, matplotlib`; the tebako rows
  therefore do strictly MORE work and still show the tax. Cold start
  pays the first-run materialization into the driver's exec cache; even
  warm, the VFS import path (thousands of small reads through the
  mounted image) costs ~4 s where a host filesystem costs ~0.4 s. That
  tax buys hermetic, unmodifiable, byte-verified execution, and it
  narrows as the driver's caches improve. For a data tool whose real
  commands run seconds to minutes it is the right trade; for a
  `python -c` hot loop, today, it is not.
- **Methodology.** Install wall is the end-to-end install verb (pip/conda
  network-bound on the rehearsal host's connection; tebako measured as
  the store copy with the download bytes named beside it). Disk is `du`
  of everything the install left — the pip row rides an already-present
  homebrew python whose bytes are NOT counted, while conda and tebako
  both carry their interpreter. Cold is the first command after install
  with a scratch exec cache (page cache as-is — noted, not gamed); warm
  is the median of five repeats. Sizes are MiB (1,048,576 B); times are
  wall-clock seconds.

## Publishing (the owner's explicit act)

Never automatic (the ecosystem hard rule). The owner pushes a tag
(`<version>[-<buildno>]`, e.g. `0.1.0-1`) and dispatches build-payload
**on the tag** with `publish=true`. The publish job (gated on every leg +
the wrong-line refusal green): downloads the legs' dist artifacts,
verifies the toolchain pins, stages the release-channel runtime mirror
for the CLI's clean-cache verify, signs per the recipe's `signing:` block
(the org secret `TEBAKO_CI_SIGNING_KEY`; declared-signing + absent secret
fails closed before any upload), uploads the per-triplet `.tfs` +
`.sha256` sidecars + manifest mirror + `.asc` to the tag release, and
upserts `tpkg-registry.yaml` — which lands on main by **bot PR +
auto-merge** (direct pushes are GH006-refused). **Prerelease policy:** a
tag whose version carries a non-numeric segment after the numeric prefix
(`0.2.0rc1-1`) publishes its rows like any other but the release is
flagged `--prerelease` and the registry `default:` never moves off the
latest stable (`tools/set_registry_default.rb` runs on stable tags only).

## The SBOM lane (pydata#4 — the metanorma#168 feedstock template)

Every POSIX leg emits `pydata-<version>-<asset-platform>.cdx.json`
(CycloneDX 1.6) into its dist artifact, and the publish job ships it
beside the payload with the same trust treatment as every other served
name. tebako#1124's standing rules bind: components carry **purl +
sha256**.

- **Generation (per POSIX leg, tools/build).** The pinned CLI's `sbom`
  verb (tebako#1120) in COMPLETE mode: `tebako sbom <image.tfs>
  --closure closure/<version>-<triplet>.txt --ecosystem pypi --triplet
  <triplet>`. Deterministic — the serial derives from the document's
  identity content and the timestamp reads `SOURCE_DATE_EPOCH` pinned at
  the feedstock commit (the product release's own `sbom_epoch`
  convention, spec 13 §5a); two runs on unchanged inputs are
  byte-identical. **Activation is the pin, never a workflow edit**: the
  verb exists from the tebako release carrying tebako#1120, so an older
  tools pin probes out and the step skips LOUDLY until the automated
  pin roll brings a release carrying the verb.
- **The input of record is the hashed closure TEXT
  (`closure/<version>-<triplet>.txt`)** — the sha256-pinned wheel lock
  the leg itself stages from (the verb's grammar is `<artifact>
  <sha256>` / `<sha256> <artifact>` rows; the sha256sum spelling here
  parses directly). The ecosystem word is **pypi** (PEP 427 wheel
  names; the tag triplet rides as the `tebako:pypi:tags` property) —
  the metanorma template's `gem`.
- **The windows-ucrt leg skips loudly — no complete-mode input of
  record exists for its shape.** Its closure record is stage_msys2's
  `pacman -Q` of the prebuilt set: name+version rows, **no per-package
  hashes**, so the verb's closure grammar refuses it by name; and the
  verb's degraded inventory reads `specifications/*.gemspec` (a
  rubygems layout a python image never carries — verified locally: the
  refusal is the named exit 65). The honest annotation is the skip
  notice, not a fabricated inventory. Activation path: stage_msys2
  records the pacman cache archives' digests (the `.pkg.tar.zst` set
  pacman itself signature-verified) — then the same complete mode
  applies with `--ecosystem generic`.
- **Signing (the offline GPG lane).** The publish job produces a
  detached `.cdx.json.asc` with the SAME CI signing subkey material
  `tebako publish --sign` consumes (`TEBAKO_CI_SIGNING_KEY`, the
  recipe's `signing.keyid`) — gpg is the detached-sign instrument for
  files the CLI does not itself sign. Declared-signing + absent secret
  fails closed before any upload. This feedstock has **no sigstore
  attestation lane** for any served name (the CLI-derived `.tfs.asc`
  signatures never touch the disk) — the SBOM rides the same `.asc`
  treatment as every other name, and the attestation lane is a named
  follow-up for the publish job as a whole, not an SBOM-specific gap.
- **The registry sidecar.** Each platform entry gains the additive
  `sbom: {artifact, sha256}` pin, re-rendered from the release's OWN
  served digests by `tools/render_sbom_sidecars.rb` at every publish —
  the L3 mirror-by-value rule, because the CLI's registry upsert
  round-trips its serde model and drops unknown keys. Pre-SBOM readers
  ignore the key (the registry evolution law). A first-class `sbom:`
  sidecar class is a registry-schema candidate for the product — until
  it lands, the key is an additive unknown key by design.
- **The smoke (report-only, never fails).** The publish job runs
  osv-scanner over every staged SBOM — tebako#1124's universal
  acceptance proof (parse OK / findings reported). osv-scanner is
  pinned in Tebakofile's `sbom:` block, sha256-verified at fetch and
  cross-checked against the release's own `osv-scanner_SHA256SUMS`.

## Next increment (named)

- **The first real publish** — the owner's go: tag + dispatch. Until
  then the pipeline is proven on PR CI only.
- **Consume the upstream fixes** (the blocker gates reassert
  automatically): tebako#1097 (the ELF dep-walk fix, merged upstream) →
  a tebako release → the link unit → a tebako-runtime-python rebuild
  carrying tebako-runtime-python#106 (the sem_unlink patch) → a factory
  release → the pin bumps here (`tools.release` + `build.runtime`) —
  deviations C and D close on their PASS branches.
- The public comparison page on tebako.org (a separate repo — the
  measured table below is the content source).
- Remaining triplets (linux-musl ×2, aarch64-linux-gnu, x86_64-macos) as
  the matrix widens; the windows arm64 leg when the factory serves it.
