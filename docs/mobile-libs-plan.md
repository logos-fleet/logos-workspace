# Mobile, library phase: Logos libraries on iOS and Android while the core is frozen

Status: plan, 2026-11-06. Decisions in §7 were taken in a grilling session.
Vocabulary: `CONTEXT.md` § *Mobile phases*. Rationale for the scope cut: ADR 0011.

## 1. Goal

Two parts, in order:

1. **Migration.** Move the fork's mobile work that survives the core freeze to origin.
   The proof is one library, delivery, cross-compiled from origin inputs and running in a
   Qt app on a real device.
2. **Extras.** New work on the same path: chat, then storage, then mobile CI, plus an
   optional wasm spike.

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

## 3. Migration

What moves from the forks to origin, and the minimum new work that makes it build there.
This is the definition of done for the phase.

| # | Repo | What | Source |
|---|---|---|---|
| M1 | logos-basecamp | Shell preview stack, rebased and landed in order: iOS simulator → Android APK → physical iOS device. Basecamp with a mocked backend on mobile, and the pipeline the smoke app reuses | fork, already open on origin |
| M2 | logos-nix | Rust and Nim cross-compile for the mobile targets with its DT_NEEDED gate: `4e21065` plus fixes `587b3f3`, `dc266f5`, `34d1cfb`, `9944271`, `693aa1f`, `c841b8d`. One or two PRs | fork |
| M3 | logos-nix | iOS cross re-roots `find_package`: `fc4b091`. Needed for CMake-built native dependencies | fork |
| M4 | logos-nix | Pass `xcodeVersion` / `xcodeBuild` in as arguments through `mkMobileTargets`. Replaces fork commit `e815779`, which hard-codes one Mac's Xcode | new, replaces fork |
| M5 | logos-nix | Delivery recipe under `mobile/libs/`; logos-nix is the **caller** | the fork's "networking stack cross-built" result, re-homed |
| M6 | logos-delivery | Cross-compile hygiene: plain `nim c` build, plus whatever breaks | new, as it surfaces |
| M7 | logos-basecamp | Delivery smoke app in `mobile/smoke/`: start a node, relay and receive a message. Android first; iOS simulator next; iOS device after M1's device PR | new |

Order: M1 and M4 in parallel; M2 and M3; then M5–M7 (Android → iOS simulator → iOS device).
Signing belongs to app packaging (M1, M7), never to the library.

**Fork logos-nix work deliberately left behind:**

- Core phase: iOS Qt exports for Bare modules (`fc2342b`, `3f2c237`); liblogos_core's
  dependency tail (`9572de2`, `fc87e23`).
- Wasm phase: Emscripten pin and wasm Qt (`d0043d3`, `d89b850`, `9f3d4fa`, `53acb6c`).
- Wallet modules only: iOS libcurl (`a3a9605`).
- Never lands: the hard-coded Xcode version (`e815779`).
- Unclear: 4 GB gradle heap (`c71a2a8`). Taken only if M1's APK build runs out of memory
  against origin.

## 4. Extras, in priority order

| # | What | Repos | Why this order |
|---|---|---|---|
| E1 | **Chat**: a small C API crate in libchat over the `logos-chat` facade with `embedded-logos-delivery`, then a smoke check | libchat, logos-nix, logos-basecamp | In the core phase's module list, and it builds on delivery |
| E2 | **Storage**: recipe, hygiene, smoke (upload and download via a devnet bootstrap, plus CPU/RSS/battery/data numbers for the local-versus-remote call); and the two-libraries-in-one-process check (delivery + storage, both Nim) | logos-storage-nim, nim-leopard, logos-nix, logos-basecamp | Never on the phone in the forks; first chance to measure the collision claim |
| E3 | **`logos-mobile-ci`**: a reusable workflow on the pattern of `logos-windows-ci`, extracted once M7's Android job is proven. Android cross-compile on hosted Linux, iOS cross-compile on hosted macOS (no secrets: only app packaging signs), and an Android emulator smoke job | new repo, logos-basecamp, opt-ins in logos-delivery, nim-sds, logos-storage-nim, libchat | Wanted, but the migration doesn't need it |
| E4 | **Wasm feasibility spike**, timeboxed, optional: compile delivery (Nim) and libchat (Rust) to wasm, list what breaks (nim-ffi's worker thread versus single-threaded wasm, sockets, chronos). Output is library issues, not PRs | none | Learning for the core phase only |

## 5. Scope by library

| library | lang | C API | status |
|---|---|---|---|
| `liblogosdelivery` (logos-delivery, contains nim-sds) | Nim | nim-ffi generated | **migration** |
| `libchat` (libchat) + embedded delivery | Rust | none: new small C API crate | extra E1 |
| `libstorage` (logos-storage-nim) | Nim | nim-ffi generated | extra E2: local, small quota, behind NAT/relays |
| `liblogos_blockchain` (logos-blockchain) | Rust | cbindgen | **out**: a remote module on mobile; the phone never runs the node |

## 6. Verification

- **Device**: through the device lease queue (`docs/mobile-venue.md`). Each run gives a
  dated `docs/evidence-ledger.md` row: command, device, OS, result.
- **Migration done** when: M1–M7 are merged on origin; delivery's smoke app passes on an
  Android device and an iOS device, built only from origin revisions; ledger rows exist for
  both.
- **CI** arrives with E3. Until then each PR is verified locally.

## 7. Decisions

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
| L9 | No suspend/resume testing in this phase | A smoke scenario: not needed for the migration |
| L10 | Central `logos-mobile-ci`, libraries only, as an extra | Module repos too: no new-architecture modules exist yet. CI inside the migration: not needed to land it |
| L11 | Wasm: optional feasibility spike only | Full wasm phase now: networking libraries need the outbound door, which is core |
| L12 | Migration = what survives the freeze, proved on delivery alone | Port all 29 fork logos-nix commits: most serve the core, wasm or wallet modules |

## 8. Existing origin PRs

- Shell preview stack (Basecamp): keep, rebase, land (M1).
- `logos-protocol` and `logos-plugin-qt` shared-library opt-outs, `logos-module` iOS CLI
  guard, `process-stats` iOS guard: **parked**. They are left open and untouched, each with
  a comment that it waits for the new core. Only the core's cross-compile chain uses them.
- `docs/liblogos-mobile-delivery-plan.md`: superseded for this phase. Its Layer 3 waits for
  the core to unfreeze.

## 9. Core phase (after unfreeze, not planned here)

Core plus selected modules on mobile: delivery, chat, storage local or remote by L8's
numbers, blockchain remote. Wasm modules. The three blocking decisions in
`docs/handoffs/2026-09-28-mobile-migration.md` §5 apply then.
