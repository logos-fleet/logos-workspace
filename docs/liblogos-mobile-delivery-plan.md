# Delivering liblogos's mobile cross-build to origin

Status: plan, 2026-09-28. Decisions in §7 were taken in a grilling session; the
reasoning behind each is in the "alternative rejected" column, not elsewhere.
Vocabulary: `CONTEXT.md`, plus two terms this plan needs that are not in it yet (§8).

**Terms used throughout.** *origin* — the canonical repositories the products are
built from. *fork* — this workspace's `logos-fleet` forks, where all the work
below already exists. No origin repository may reference a fork; that is the
constraint the whole shape of this plan follows from (§3.1).

## 1. Goal

`liblogos_core` cross-compiles for iOS and Android, **as a static or a dynamic
library**, and can load a module — built from **origin inputs, with no patches
and no fork pins**. Delivered as a sequence of pull requests against origin
masters.

Out of scope, in the order that follows this:

- The smoke host app (`logos-basecamp/mobile/liblogos-smoke/`). It is the only
  thing that can demonstrate *running*, and it is a different kind of review —
  an app host, an Android manifest, a bundled-set runtime. It should not hold up
  ten library PRs. See §9 for what it would claim.
- Dynamic core linkage on iOS, and dynamic module realization against a static
  core on Android. Both are spikes (§9), not deliverables.
- CI, and any new check or gate (D3).

## 2. State of play

Almost none of this is new code. The work exists; it is in the wrong places.

**Already on origin.** The mobile toolchain landed with milestone 1:
`logos-nix` master has `mobileTargets`, `forAllMobileTargets`,
`androidBuildSystems`, `nix/android/`, `nix/ios/`, `mkIosCmakeStage` (in
`nix/ios/cross-overlay.nix`) and the version-gated `xcodeWrapper`. This plan
**extends a landed feature** rather than introducing one.

**On the fork, and small.** `logos-liblogos`'s entire mobile delta is 4 commits
of 66 ahead of origin; `logos-nix`'s relevant delta is 6 of 29. The rest of each
fork is unrelated work — the web transport, the wasm outbound door, variants,
token fixes.

**Nowhere at all.** Two stages are carried as nix patches under
`logos-liblogos/nix/mobile/patches/`, because when they were written the two
repositories they patch were flake inputs with no fork. Forks now exist and are
**byte-identical to origin** (0 ahead, 0 behind), so those changes must be
authored, not cherry-picked.

**The chain.** `nix/mobile/{ios,android}.nix` build nine stages — protocol,
qt-host, lgx, module, process-stats, container-subprocess, module-loader-qt,
package-manager, liblogos — from **source trees**, not from the inputs'
published packages. This matters twice below: it is why most Layer 1 PRs need
only a CMake option, and it is why Layer 2's published mobile archives are not
needed (D8).

**Verification reality.** No workflow in any repository mentions mobile. No
workflow has ever run on a fork at all — `total_runs=0` on
`logos-liblogos`, `logos-package` and `logos-nix`, with no repository and no
organisation secrets, so the existing `ci.yml` would die at its cache step.
Windows is the only cross target with CI, and its per-repository cost is ten
lines delegating to a central reusable workflow. The venue Mac's store is warm:
315 mobile stage outputs are present, so rebuilds are incremental.

## 3. Constraints

1. **Origin uses origin flakes only.** No fork URL and no fork revision appears
   in any delivered PR. Consequence: `logos-liblogos` keeps its declared
   `.url`s unchanged and simply relocks once the leaves have landed — there is
   no URL rewrite anywhere in this plan.
2. **Delivery is sequential.** A repository's PR may only be opened once every
   input it needs is on origin master.
3. **Verification is local.** CI is dead on the forks (§2), and iOS can never
   run on a hosted runner: the derivations are `__noChroot` and the Xcode
   wrapper asserts an exact installed version, which a hosted image changes
   without warning.
4. **The Xcode values are not a pin.** `iosXcodeVersion` / `iosXcodeBuild` in
   `logos-nix/flake.nix` are an assertion about the build machine — Xcode is
   never in the store, and the gate only compares the declared string to
   `/Applications/Xcode.app/Contents/version.plist`. They are never delivered
   (D9).

## 4. What is claimed

Two independent axes (§8): **core linkage** (static / dynamic) and **module
realization** (`direct_static` / `direct_dynamic`). Seven meaningful cells:

| # | platform | core | modules | status | evidence |
|---|---|---|---|---|---|
| 1 | iOS | static | `direct_static` | **claimed** | the current chain; `mkIosCmakeStage` fails any stage that installs a dynamic image |
| 2 | iOS | static | `direct_dynamic` | **claimed** | ADR 0006. `fc2342b8`'s three checks, mutation-tested; dlopen spike level1/2/3 on the iPhone 16 Pro simulator and a signed iPad Air 4 |
| 3 | iOS | dynamic | either | **spike** | none. Needs three Qt-free dylibs and a relaxed stage gate |
| 4 | Android | dynamic | `direct_dynamic` | **claimed** | the `DT_NEEDED` gate (IMPLEMENTED + MEASURED, named handset); a Bare module executing from the app's own `files/` on Android 15 |
| 5 | Android | dynamic | `direct_static` | **claimed, low risk** | not separately measured; a module linked into the app while the core is shared |
| 6 | Android | static | `direct_static` | **claimed, new** | one image, no exports needed |
| 7 | Android | static | `direct_dynamic` | **spike** | cell 4's evidence does **not** transfer: with a static core there is no `liblogos_protocol.so`, so `DT_NEEDED` does not apply and the module must resolve upward into the app's own image |

Cell 7 is the one to be careful about. It rests on an open caveat already in
`docs/evidence-ledger.md` — *"ELF's flat namespace genuinely collapses the
duplicates, and that is untested"* — and the failure mode is the worst kind:
duplicate `StoreRegistry` **fails open**, leaving an identity holding the ambient
ring instead of erroring.

## 5. The sequence

Each entry names what the PR carries and, where it matters, what it must not.
SHAs are fork revisions.

### Layer 0 — `logos-nix`

Gates Layers 2 and 3. Everything else in Layer 1 is independent of it.

| pick | what it is |
|---|---|
| `9572de2f` | the mobile dependency tail for both targets, and `mkMobileTargets` / `mkForAllMobileTargets` — a parameterised generalisation of origin's fixed `mobileTargets` / `forAllMobileTargets`, which survive as specialisations, so it is backwards compatible |
| `fc87e23b` | one Boost CMake source for both tails; hoisted iOS targets |
| `fc4b091b` | iOS cross re-roots `find_package`; names the three modes |
| `c71a2a81` | a 4 GB gradle heap for APK packaging |
| `fc2342b8` | **the heaviest item in the plan.** `FEATURE_reduce_exports=OFF` rebuilds all eight iOS Qt packages, and a `postPatch` rewrites Qt's own `qmetatype.h` visibility pragma with `--replace-fail`. Required by cell 2 |
| `3f2c237f` | splits that commit's exports check per assertion |
| part of `e8157797` | **hunk 2 only** — the gate's message, which now names where the two strings live. Never hunk 1 (D9) |
| new | thread `xcodeVersion` / `xcodeBuild` through `mkMobileTargets` / `mkForAllMobileTargets`, as `mkIosPkgs` already accepts them. Copy the anti-vacuity check at `flake.nix:809`, which asserts a bogus override changes `qtbase.drvPath` |

Consumer note that belongs in the PR body: an iOS app that does not call
`logos_ios_export_symbols()` after `fc2342b8` exports everything and pays a
measured 5.8 MB.

### Layer 1 — source-only, mutually independent, does not need Layer 0

The chain compiles these from source, so all each needs is for the option to exist.

| repo | pick |
|---|---|
| `logos-protocol` | `1902bedf` — `LOGOS_PROTOCOL_BUILD_SHARED`, so a static-only host can opt out |
| `logos-plugin-qt` | `595274c3` — **must be split**: it bundles `LOGOS_QT_HOST_BUILD_SHARED` with an unrelated admission-bound change |
| `logos-module` | `3120dd23` — do not build the `lm` CLI for iOS |
| `process-stats` | `776f2bdc` (guard the macOS-only path on `TARGET_OS_IPHONE`) + `1cf3220b` (`getImageStats` builds for Android) |
| `logos-container-subprocess` | **new** — a `LOGOS_BUILD_TESTS` option, from `nix/mobile/patches/logos-container-subprocess-tests-option.patch` |
| `logos-module-loader-qt` | **new** — from `nix/mobile/patches/logos-module-loader-qt-ios.patch`: the tests option, the iOS install gate for `logos_host_qt`, `src/host/ios_entry_stub.cpp`, and the bionic `<execinfo.h>` guard. Worth **two commits**: the `backtrace(3)`-before-API-33 guard is a plain portability bug with nothing to do with this chain |

### Layer 2 — needs Layer 0

| repo | pick | trimmed |
|---|---|---|
| `logos-package` | `35b36f76` (static C ABI, CoreFoundation Unicode backend) and whatever exposes `cpp-semver` | `b719fb89`, `723f9895` — published iOS/Android archives the chain never consumes (D8) |
| `logos-package-manager` | `e8170abd` — `LGPM_STATIC_LIB`, no CLI for iOS | `3ac6bbb6`, `72e7e3a4` — same reason |

### Layer 3 — `logos-liblogos`

- `ee0b0787` and `85e452ac` — the chain and the per-build-platform evaluation.
- **Minus** `f32c6915` and `3be537d0`: both are fork-pin bumps, replaced by an
  honest relock onto origin.
- **Minus** `nix/mobile/patches/` and the `patched` helper in both chain files,
  now that Layer 1 has landed the two sources.
- **Plus** the include-dir fix: `logos_core` includes `logos_module_loader/*.h`
  without linking the INTERFACE target that carries the include directory. A
  native build hides it — nix's cc-wrapper names every `buildInput`'s
  `include/`. Fix it in `CMakeLists.txt` and delete
  `-DCMAKE_CXX_FLAGS=-I${native.logosModuleLoader}/include` from **both**
  `nix/mobile/ios.nix` and `nix/mobile/android.nix`.
- **Plus** Android static core as a second configuration
  (`LOGOS_CORE_STATIC=ON` against shared Qt), claimed only for cell 6 (D12).

### Prerequisite — this workspace

`logos-container-subprocess` and `logos-module-loader-qt` are flake inputs only
and cannot be edited from here. They must become submodules **before** Layer 1
is worked, or the two new-commit PRs have nowhere to be written: `ws sync-graph`
with `FLEET_ORG` set, `dep-graph.nix` regenerated. The `add-repo` skill covers it.

## 6. Verification

Per PR: that repository's own native build and checks, locally
(`ws test <repo>`), since no CI will run.

After Layer 3, on the venue Mac, from a lock naming only origin revisions:

- Android chain — `androidBuildSystem = "aarch64-darwin"`, both core linkages.
- iOS chain — with `xcodeVersion` / `xcodeBuild` passed as arguments
  (Layer 0's threading is what makes this possible without a fork delta), static
  core.
- A dated row in `docs/evidence-ledger.md` per claimed cell, naming command,
  platform and date. Cells 1, 2, 4, 5 and 6 currently have no such row, so
  "liblogos cross-compiles for mobile" is today an ungraded claim.

## 7. Decisions

| # | Decision | Alternative rejected |
|---|---|---|
| D1 | Deliver to origin as sequential PRs; nothing on origin references a fork | Re-declare liblogos's inputs to the forks and make mobile durable there — inverts the dependency direction |
| D2 | Stop at Layer 3; the smoke host is a follow-on | One changeset including the host app |
| D3 | No CI, no new checks, no gate | Mirror the Windows reusable-workflow pattern — no workflow has ever run on a fork, there are no secrets, and iOS cannot use a hosted runner |
| D4 | Fold both patches into source in their own repositories; delete `nix/mobile/patches/` | Keep them as patches (a patch is a bet on line numbers, and the rewrite will churn exactly those files); fold only the loader-qt one |
| D5 | Fix the include-dir defect at source | Keep the `-I` flag in both chain files |
| D6 | Claim five of seven cells; 3 and 7 are spikes | Claim all seven — puts an untested ELF claim, failing open, on the critical path; or claim only that the core builds, which ships something the migration cannot use |
| D7 | `fc2342b8` + `3f2c237f` are in Layer 0 | Exclude as a later milestone — untenable once the claim includes loading a module, since `direct_dynamic` on iOS *is* that commit |
| D8 | Trim Layer 2's published mobile archives | Keep them: already written, already working, and a reviewer may ask why a repository gains an iOS option but no iOS output |
| D9 | Split `e8157797`: ship the gate message, thread the two values through `mkMobileTargets`, never ship the numbers | Land the numbers on origin — they are a fact about one Mac; or leave them as a permanent fork delta, which the threading removes the need for |
| D10 | The two patched repositories become workspace submodules before Layer 1 | Edit them in a clone outside the workspace |
| D11 | Exclude from Layer 0: the Rust/Nim cross and `DT_NEEDED` series, Emscripten/wasm Qt, iOS libcurl | Ship all 29 fork commits — none of these is reached by the C++ chain |
| D12 | Android static core's claim is limited to `direct_static` modules | Claim `direct_dynamic` against it too — that is cell 7 |

## 8. Vocabulary

Two terms this plan depends on, proposed for `CONTEXT.md` (not written there;
that file has uncommitted changes).

**Core linkage** — how `liblogos_core` and the runtime libraries it needs
(`logos_protocol`, `logos_qt_host`) are linked into a process. `static`: one
image owns `TokenManager` / `StoreRegistry` / `LogosAPI`. `dynamic`: real shared
libraries own them and the core imports the runtime. `LOGOS_CORE_STATIC` selects
between them, and it is not a packaging switch — `src/CMakeLists.txt` points
`logos_sdk` at `logos_qt_host_shared` or at the static archive, which is what
decides *which image owns the singletons*. iOS is static because Qt for iOS is
static-only; Android is dynamic today.
*Avoid*: "static build" / "shared build" — names the artifact kind, not the
ownership question that actually changes.

**Module realization** — `spec-module-loader.md`'s axis: `direct_static` (ABI
code already linked and registered in the caller), `direct_dynamic` (a dynamic
library loaded into the caller), `hosted_dynamic` (the same library in a Module
Host). Independent of core linkage: a statically linked core can load a
`direct_dynamic` module, which is exactly what ADR 0006 does on iOS.
*Avoid*: bare "static"/"dynamic" for this axis — it collides with core linkage.

## 9. Spikes

Neither blocks the sequence.

**Cell 3 — an iOS dynamic core.** Can the runtime trio (`logos_protocol`,
`logos_qt_host`, `logos_core`) be built as Qt-free dylibs for iOS and still keep
exactly one `TokenManager` / `StoreRegistry` per process? Needs
`mkIosCmakeStage`'s post-install dynamic-image gate parameterised, and Qt
resolved upward from the app. Note what iOS gains: no fork/exec, no
out-of-process host, one app image — so the only real consumer of "dynamic"
there is a dlopened module, which cell 2 already serves.

**Cell 7 — a static Android core with dynamic modules.** Does ELF's flat
namespace collapse duplicate runtime singletons, or must a static-core app
export them (`--export-dynamic` or a version script)? Cheap: the ledger records
a two-image harness that reproduces the darwin split in under a second; porting
it to a pair of Android `.so`s is a day. **Include the anti-vacuity control** —
have one image authorize successfully and the other flip to authorizing after
adopting the token, or "it refused" might just mean nothing authorizes.

**Follow-on, not a spike — the smoke host.** `mobile/liblogos-smoke/` is what
would let "and runs" be claimed rather than "and links".

## 10. Risks

| Risk | Why | Mitigation |
|---|---|---|
| The cherry-picks do not apply cleanly | origin has moved since the fork branched: `logos-protocol` behind 10, `logos-plugin-qt` behind 10, `logos-nix` behind 3, `logos-package` behind 3 | one worker per repository; `595274c3` needs splitting before any rebase |
| `fc2342b8` changes the iOS Qt build for every consumer | `FEATURE_reduce_exports=OFF` plus a patch to Qt's own header | its evidence is the strongest in the set; state the 5.8 MB consumer note in the PR body |
| A Qt bump breaks the `qmetatype.h` patch | `--replace-fail`, by design | it fails loudly at build; that is the intent |
| The venue cannot build iOS against an origin `logos-nix` | origin declares a different Xcode than this Mac has, and the gate exits 1 before a compiler runs | Layer 0's threading; until it lands, iOS verification needs a local override |
| Relocking liblogos pulls a wave | bumping one input here has repeatedly required bumping several | `ws update-order`, and relock only after all leaves are on origin master |
| Cell 7 gets assumed rather than measured | cell 4's evidence looks like it transfers and does not | D12 scopes the claim; the spike is the only thing that may widen it |

## 11. Definition of done

1. Every PR in §5 merged on origin master, in order.
2. `logos-liblogos` master's `flake.lock` names only origin revisions;
   `nix/mobile/patches/` is gone; neither chain file carries the `-I` workaround.
3. On the venue, from that lock: the Android chain builds for both core
   linkages, and the iOS chain builds with the Xcode values passed as arguments.
4. `docs/evidence-ledger.md` carries a dated row for each of cells 1, 2, 4, 5, 6.
5. Two spike issues exist, each stating its question and its anti-vacuity control.

## 12. References

- `docs/mobile-bring-up-plan.md` — milestone 1, whose toolchain is already on origin.
- `docs/evidence-ledger.md` — the graded claims this plan leans on, including the
  ELF caveat behind cell 7 and the `DT_NEEDED` row behind cell 4.
- `docs/adr/0006-bundled-modules-on-ios-are-protocol-free-embedded-frameworks.md` — cell 2.
- `docs/adr/0001-shell-static-on-ios-dynamic-on-android.md` — why Qt for iOS being
  static-only is a hard gate rather than a preference.
- `docs/handoffs/2026-09-28-mobile-migration.md` — the architecture migration this
  plan is the base for.
- `logos-liblogos/nix/mobile/ios.nix` — the chain's own header comment explains why
  it is one chain here rather than a mobile output per repository.
- `logos-liblogos/src/CMakeLists.txt:240-300` — the runtime-singleton rationale
  behind core linkage.
