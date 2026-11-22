# pydata feedstock

Feedstock for **pydata** — the curated python data-science toolkit
(numpy + pandas + matplotlib), packaged as a tebako `kind: app` payload
riding the tebako python runtime. The second flagship payload of the v2
ecosystem (TODO.sota2/05): metanorma proved the calculus on the hardest
ruby case; this payload proves it generalizes across languages — native
wheels with BLAS linkage, data files, and the install pain python is
famous for, answered hermetically.

**Status: build + gate.** The three desktop-triplet legs stage the
closure, press the image, and run the boot smoke + the three named
conformance arms. Publishing (the release workflow + registry rows + the
public comparison page) is the next increment — named, not shipped.

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
  conformance / pypi_offers.py.
- `.github/workflows/build-payload.yml` — plan (matrix from the recipe) →
  the three per-triplet legs (POSIX pipstage host-native; windows via
  msys2) → boot-smoke + checks + the conformance arms. No publish job in
  this increment.
- `tpkg-registry.yaml` — this package's registry; filled at first
  publish.

## Local rehearsal (one POSIX triplet, e.g. aarch64-macos)

```sh
tools/stage_runtime                      # the pinned factory release -> out/runtime-mirror
TRIPLET=aarch64-macos tools/gen_closure  # resolve + hash the wheel closure
TRIPLET=aarch64-macos tools/build        # pipstage -> out/aarch64-macos/pydata-*.tfs
set -a; . out/aarch64-macos/runtime.env; set +a
tools/boot_smoke "$PAYLOAD_IMG"          # image assertions + tebako check + dispatch smoke
tools/conformance "$PAYLOAD_IMG"         # the three arms, under deadlines
```

## Next increment (named)

- The publish workflow (owner-dispatched, tag-gated — the ecosystem hard
  rule) + the `tpkg-registry.yaml` rows, signed per the recipe's
  `signing:` block.
- The wrong-line refusal gate (dispatching the published payload with
  only a different python line available must earn the named exit-69
  refusal — the xml2rfc acceptance pattern).
- The public comparison page (TODO.sota2/05 arc 3): install UX measured
  byte-for-byte against the incumbents (pip + venv, conda, docker).
- Remaining triplets (linux-musl ×2, aarch64-linux-gnu, x86_64-macos) as
  the matrix widens; the windows arm64 leg when the factory serves it.
