# Mobile, library phase: Logos libraries on iOS and Android while the core is frozen

Status: plan, 2026-11-06. Decisions in §6 were taken in a grilling session.
Vocabulary: `CONTEXT.md` § *Mobile phases*. Rationale for the scope cut: ADR 0011.

## 1. Goal

The libraries behind the delivery, storage and chat modules cross-compile for iOS and
Android from origin inputs, load as dynamic libraries into a Qt app on a real device,
and do real work there, with a dated ledger row per library, per platform.

Builds and evidence only. No module wrapper, no core.

## 2. Constraints

1. **Frozen repos are untouched.** `logos-liblogos`, `logos-protocol`, `logos-plugin-qt`,
   `logos-module-loader-qt`, `logos-container-subprocess`, `logos-package-manager`: the
   new core replaces them.
2. **Module wrappers are out of scope.** Under the new specification a module is a plain C
   ABI (`spec-module-interface.md`). Today's Qt-plugin wrappers get replaced, so work on
   them is thrown away.
3. **Libraries own no mobile policy** (Status ADR 0010: the library states what it is, the
   caller decides how). Library-side work is cross-compile hygiene only: no mobile targets.
4. **Origin uses origin inputs.** No fork URL or revision in any delivered PR, and no fork
   reference to origin issues or PRs (workspace rule).

## 3. Scope

| library | lang | C API | phase 1 |
|---|---|---|---|
| `liblogosdelivery` (logos-delivery, contains nim-sds) | Nim | nim-ffi generated | **in** |
| `libstorage` (logos-storage-nim) | Nim | nim-ffi generated | **in**: local, small quota, behind NAT/relays |
| `libchat` (libchat) + embedded delivery | Rust | none: new small C API crate | **in** |
| `liblogos_blockchain` (logos-blockchain) | Rust | cbindgen | **out**: a remote module on mobile; the phone never runs the node |

Wasm: a timeboxed feasibility spike only (§4, step 9). Wasm modules are the core phase.

## 4. Sequence

1. **Basecamp Shell preview stack on origin**: iOS simulator → Android APK → physical iOS
   device. Rebase on master, undraft and land in order. It is Basecamp with a mocked
   backend on mobile, and its pipeline is the one the smoke apps reuse.
2. **logos-nix**: pass `xcodeVersion` / `xcodeBuild` in as arguments through
   `mkMobileTargets` (needed for any hosted iOS job), then upstream the fork's Nim and Rust
   mobile cross-compile pieces. Add the library recipes under `mobile/libs/`; logos-nix is
   the **caller**.
   The iOS Qt `reduce_exports` rebuild is **not** needed: the libraries have no Qt.
3. **First vertical slice: delivery on Android.** Recipe → `.so` → Qt smoke app in
   `logos-basecamp/mobile/smoke/` → emulator CI job → leased-device run → ledger row.
4. **Delivery on iOS.** Simulator first (ad-hoc signature); device once step 1's
   device PR has landed (signing is the app's packaging step, never the library's).
5. **Storage** on both platforms, plus the **two-libraries-in-one-process** case
   (delivery + storage, both Nim) so that "no symbol collision" is measured, not assumed.
6. **Chat**: a small C API crate in libchat over the `logos-chat` facade with
   `embedded-logos-delivery`, then both platforms.
7. **`logos-mobile-ci`**: a reusable workflow on the pattern of `logos-windows-ci`,
   extracted once step 3's CI job is proven. Opt-ins: logos-delivery, nim-sds,
   logos-storage-nim, libchat.
8. **Suspend/resume scenario** per library (background N min → resume → recovers),
   batched into device leases.
9. **Wasm feasibility spike**, timeboxed, in parallel from step 3: compile delivery (Nim)
   and libchat (Rust) to wasm, list what breaks (nim-ffi's worker thread versus
   single-threaded wasm, sockets, chronos), and file library issues. No runtime.

Library hygiene PRs (plain `nim c` builds, test executables made optional, portability
guards) land as they surface.

## 5. Verification

- **Per library PR (CI)**: Android cross-compile on hosted Linux, iOS cross-compile
  (device + simulator) on hosted macOS. Compile and link only; no secrets needed, because
  only app packaging signs.
- **Smoke (CI)**: Android emulator runs the Basecamp smoke app against the fresh library.
- **Device (manual)**: through the device lease queue (`docs/mobile-venue.md`). Each run
  gives a dated `docs/evidence-ledger.md` row: command, device, OS, result.
- **Smoke assertions per library**:
  - delivery: start node, relay and receive a message
  - storage: upload and download a file via a devnet bootstrap, plus CPU/RSS/battery/data
    numbers for the local-versus-remote call
  - chat: send and receive over embedded delivery
  - all: suspend/resume recovery

## 6. Decisions

| # | Decision | Alternative rejected |
|---|---|---|
| L1 | Phase 1 proves libraries, not modules (ADR 0011) | Module wrappers built for mobile: replaced by the new core's C ABI |
| L2 | No mobile targets in libraries; logos-nix is the caller | Per-library mobile targets: contradicts the caller-decides contract |
| L3 | Recipes in logos-nix, smoke apps in logos-basecamp | A new harness repo: nothing in it outlives the phase |
| L4 | Smoke apps are Qt, on the Shell preview pipeline | Native Swift/Kotlin apps: two new build paths. Libraries wired into the Shell preview: mixes the library proof with UI and rebuilds what the core will own |
| L5 | Dynamic only, unsealed; `dlopen(RTLD_LOCAL)`, Android also `-Bsymbolic` | Sealing now: only static linking into one image needs it, and that waits for the new core's realization choice |
| L6 | Chat = libchat facade + embedded delivery, with a new C API crate | generic-chat with app-side transport glue: that glue is the core's job |
| L7 | Blockchain out: a remote module on mobile | Node cross-build or a C API over `wallet-http-client`: a parallel remote mechanism the core replaces, and nobody needs a mobile wallet before the new core |
| L8 | Storage local, small quota; the numbers decide local versus remote for the core phase | Client-only mode: a remote module is self-sufficient, so no library needs a client mode |
| L9 | Suspend/resume is a smoke scenario, not a spike | A separate spike: same test, more ceremony |
| L10 | Central `logos-mobile-ci`, libraries only | Module repos too: no new-architecture modules exist yet |
| L11 | Wasm: feasibility spike only | Full wasm phase now: networking libraries need the outbound door, which is core |

## 7. Existing origin PRs

- Shell preview stack (Basecamp): keep, rebase, land (step 1).
- `logos-protocol` and `logos-plugin-qt` shared-library opt-outs, `logos-module` iOS CLI
  guard, `process-stats` iOS guard: **parked**. They are left open and untouched, each with
  a comment that it waits for the new core. Only the core's cross-compile chain uses them.
- `docs/liblogos-mobile-delivery-plan.md`: superseded for this phase. Its Layer 3 waits for
  the core to unfreeze.

## 8. Core phase (after unfreeze, not planned here)

Core plus selected modules on mobile: delivery, chat, storage local or remote by L8's
numbers, blockchain remote. Wasm modules. The three blocking decisions in
`docs/handoffs/2026-09-28-mobile-migration.md` §5 apply then.
