# Handoff — migrating mobile onto the new module architecture

**Written** 2026-09-28, from the session that completed the specification review (issue #261, 30 of
31 tickets resolved). Repo `logos-workspace-fleet`, master at `678faf44`. Owner: Alex Jbanca.
Terse reporting preferred.
This document is orientation and judgement only — everything factual lives in the repo. Follow the
pointers; do not trust a summary of them, including this one.

---

## 1. What this session is for

Milestone 1 (#1) proved Bundled and Downloaded modules working on iOS and Android **under the old
architecture**. The architecture is being replaced. Your job is to bring mobile onto the new one.

A preceding effort read the twelve new specification documents against what mobile can actually
do, ran ~20 device spikes, and concluded **the new architecture is possible on mobile** — with five
specification defects that will bite you during implementation, and a handful of product decisions
that are not yours to make. That is what the rest of this document is about.

---

## 2. Read these first, in this order

| Path | Why |
|---|---|
| `CLAUDE.md` § *Specifications and evidence* | The map, the grading rubric, and the rule in §3 below |
| `docs/mobile-spec/README.md` | Index and the short answer |
| `docs/mobile-spec/03-what-mobile-cannot-do.md` | **Read before designing anything.** Six real impossibilities. Short |
| `docs/mobile-spec/04-alternative-approaches.md` | Eight placement shapes, four recommended tiers, five product decisions |
| `docs/mobile-spec/01-spec-proposals.md` | The amendments, organised by document and section |
| `docs/evidence-ledger.md` | Every load-bearing claim, graded and dated — **including the ones that were wrong** |
| `docs/spec/` | Dated snapshot of the twelve `LOGOS-MODULE-*` documents, so a §-citation is followable locally |
| `docs/mobile-spec/02-spike-results.md` | What was measured, on which device, when |

If the local memory index (`MEMORY.md`) is available to you, it carries ~170 one-line pointers to
device-level traps. It is not a substitute for the repo docs, which are the shared record.

---

## 3. The one rule that matters most

**Nine of roughly twenty-three inherited claims about mobile were false when measured — every one
of them over-constraining what mobile could do**, and several were written confidently into source
comments and ADRs. Examples you will encounter: a comment in `logos_core.h` asserting an Android
W^X restriction that does not exist (#288); ADR 0003 asserting an iOS/Play equivalence that is
false (#290); "no SharedArrayBuffer on iOS", which is wrong.

So:

- **If a doc, comment or ADR states a platform limit without a date and a device, treat it as
  unverified.** Check it before designing around it.
- **A check counts only if its predicate can fail.** A test that passes whether or not the claim
  holds proves nothing.
- **Proving the positive never proves the prohibition.** "X works" is not evidence for "Y is
  impossible."
- **A simulator result cannot evidence a device claim.**

Grade your own findings the same way (IMPLEMENTED / MEASURED / ASSERTED / INHERITED) and append
rows to `docs/evidence-ledger.md` as you go. That ledger is the deliverable that outlives the code.

---

## 4. Settled — do not re-litigate

Each of these is measured and recorded in the ledger with device and date.

- **The new architecture works on mobile.** No platform blocker was found that closes it.
- **iOS can unload a module.** A thread-local variable pins a Mach-O image; filetype, `.framework`
  packaging and `CFBundle` are irrelevant. `-femulated-tls` removes the pin and imports a SIGSEGV,
  so dyld's refusal is protective. Unloadability is **per-module, fixed by implementation language
  and build flags** — not a platform property.
- **Both platforms load a module staged after install.** On iOS the gate is *signature trust, not
  location*; on Android the app data dir works and the W^X claim was false. **Store policy, not the
  OS, is what forbids it** — argue those two layers separately, always.
- **iOS cannot create a process.** `posix_spawn` and `fork` return EPERM at the kernel, on every
  channel. This one is permanent; sideloading does not route around it.
- **Android can host a module in an isolated process** — reachable only by a Binder-passed
  socketpair, module delivered by `memfd`, peer attestation one-directional.
- **An in-process wasm interpreter works on iOS**, ~0.13 MB per live instance against the webview's
  ~93 MB per page, and a blocking outbound call **re-enters correctly** (measured at guest frame
  depth 2). No JIT, so no W^X problem.
- **Divergence goes in capability advertisement** (`LOADER §4`), never in normative text. The
  design target was *zero platform-conditional normative statements* and it was met. Keep it.

---

## 5. Open, and it **blocks** implementation

Escalate these; do not decide them yourself.

1. **Where core wasm modules run: interpreter or webview.** The recommendation is the interpreter,
   argued in `04-alternative-approaches.md` on memory, unloading, and — most importantly — because
   it is the only in-process shape where a module can block on its own outbound call without any
   toolchain or platform cooperation. This decision determines whether you need the outbound-import
   ABI edit, and whether cross-origin isolation matters at all.
2. **The compiled view-backend deferral problem.** A compiled UI in a webview cannot block on
   another module and cannot move to the interpreter (it needs a DOM). Three options, none free:
   cross-origin isolation *plus* proxying the entry point to a pthread; engine-level stack
   switching; or a toolchain stack rewrite. **Note:** the Tier 3 paragraph covering this has been
   wrong twice, both times making the webview sound more tractable than it is. Trust it least.
3. **Catalog shape — A, B or C** (`04`, product decision 5). Whether the catalog may gain module
   identities the app never shipped knowing about. This is the only decision store policy actually
   binds, and it has an architectural consequence: **native modules cap you at A on iOS and B on
   Android; only interpreted modules reach C on either store.**

The other two product decisions in `04` — which modules go remote, and off-store native delivery —
do not block early work.

---

## 6. The largest piece of work is not mobile-specific

**Neither execution path speaks deterministic CBOR today.** The native provider entry is
`char* dispatch(const char* method, const char* argsJson)`
(`repos/logos-liblogos/src/logos_core/bare_module_abi.h:31`); the web path is `JSON.stringify`
throughout (`repos/logos-js-sdk/src/client.js:71`). The specification's CBOR-plus-payload-commitment
model is ahead of the implementation on **both** paths equally.

Scope that as a shared migration, not a mobile task. If it lands as "the mobile agent's problem" the
estimate will be wrong by a large factor.

---

## 7. Five specification defects that will bite you

Detail and proposed replacement text are in `docs/mobile-spec/01-spec-proposals.md`. What you need
here is the implementation hazard, because **implementing these faithfully produces bugs.**

| | Defect | What happens if you implement it as written |
|---|---|---|
| **I** | `INTERFACE §2.8` mandates synchronous subscriber callbacks on the publishing thread, and **no document anywhere bounds a call's duration** | Unbounded blocking on the shared UI thread. The OS watchdog kills the app. The spec presents this as "natural backpressure" |
| **II** | `TRANSPORT` contains the word "timeout" **zero times**, while a `LOGOS_ERR_TIMEOUT` code exists with no specified emitter. Cancellation is fully specified and nothing triggers it | Calls that never complete. The only defined exit is closing the connection — which on a webview-hosted module destroys the module |
| **III** | `TRANSPORT §8.2` requires a peer-identity check whose predicate **cannot fail** in any shape we use; across a passed socketpair it reports the *creating* process, so it is worse than vacuous | You ship something that looks like authentication and is not. Use `SCM_CREDENTIALS`, not `SO_PEERCRED` |
| **IV** | `RUNTIME §3.4`'s state machine has **one actor** — Runtime. Zero mentions of suspension, resume or backgrounding across the four lifecycle documents | On resume you report `ready` for modules whose page the OS reclaimed, and callers route to them. Also: `unloaded` does not mean the image was unmapped |
| **V** | `RUNTIME §10.1` detects failure via Transport closure, realization status, or process supervision — **all three absent for a `direct` realization** | You infer failure from a call timeout and destroy a context with a thread still inside it. That is a use-after-free reached by following the spec's own recovery steps |

**Policy suggestion, to confirm with the user:** where implementing the specification as written
would produce a hang, a crash, or a false claim to a caller, implement the *amended* semantics and
record the divergence in the ledger. Do not ship defect I or V faithfully.

Two of these have already been filed as implementation defects against the current code: #296 (the
wasm ABI cannot defer, so web modules reshape their contracts) and #297 (web bridge allocation and
queueing). #276, #288, #290 are the others; all five were parked by scope, not difficulty.

---

## 8. Traps that cost days

- **iOS `Atomics.wait` is prohibited on a page's main thread**, and a view backend cannot move to a
  Worker because it needs a DOM. Cross-origin isolation alone does not fix compiled UI.
- **Cross-origin isolation is unobtainable on Android WebView** — measured; the WebView ignores
  COOP/COEP outright, with a same-device Chrome control proving the harness works. On iOS it is
  reachable **only from a loopback origin**, never from the custom scheme. So anything built on it
  is single-platform.
- **`ws test` hides output**, an empty verify log is normal, and a cached skip stub prints nothing.
  A PASS can mean "the check did not run."
- **Nix overrides must be `git+file:`, not `path:`**, and a sub-repo eval needs the dependency's own
  lock overridden too.
- **Agent wall-clock time is ~85% nix build wait.** Budget accordingly; use the fast-loop recipes in
  the memory index rather than full `ws build` cycles where one exists.
- **Device claims are exclusive** — see `docs/mobile-venue.md`. Claim with `mkdir`, release when
  done, and never hold a device idle.

---

## 9. Suggested first move

Do not start with code. Do this instead, in one session:

1. Read §2's first four documents.
2. Produce a **one-page implementation plan** mapping each of the four recommended tiers in
   `04-alternative-approaches.md` onto concrete repos and files, and identify the first vertical
   slice — the narrowest end-to-end path that exercises the new architecture on one device with one
   module.
3. Put the three blocking decisions in §5 to the user as a numbered list with your recommendation
   for each, and wait.

Rationale: the three blocking decisions each change what you build, and two of them change the ABI.
Starting implementation before they are answered means rework, not progress.

---

## 10. Standing constraints (inherited, non-negotiable)

- Work **only** on `logos-fleet` forks. **No references to upstream repos, issues or PRs anywhere in
  the fork** — the spec snapshot in `docs/spec/` exists precisely so citations stay local.
- **Never commit `.fleet/manifest.json`** — it is gitignored and does not survive a replan.
- **Leave `CONTEXT.md` alone**; it carries the user's uncommitted work.
- Several `repos/*` submodule pointers are dirty on a clean checkout. **Pre-existing — do not
  commit them.**
- **Report extremely concisely**, sacrificing grammar for concision. The user reads a lot of these.
- Read `~/.claude/cmux.md` before using cmux.
- Permission mode is `auto`. **Never `dangerously-skip-permissions`.** A blocked action is a
  question for the user, not an obstacle to route around.

---

## 11. Suggested skills

| Skill | When |
|---|---|
| `logos-module-development` | Any module work — project skill, knows the local lifecycle |
| `superpowers:brainstorming` | **Before** planning the migration. Invoke first, before plan mode |
| `codebase-design` / `design-an-interface` | Designing the ABI edits in §7 |
| `superpowers:systematic-debugging` | Any device failure. Do not guess at mobile symptoms |
| `mobile-devices`, `mobile-build`, `mobile-deploy`, `mobile-run`, `mobile-logs`, `mobile-control` | The device loop, in that order |
| `superpowers:test-driven-development` | The ABI work especially — and note §3's anti-vacuity rule when writing the tests |
| `grilling` + `domain-modeling` | Putting the §5 decisions to the user |
| `superpowers:verification-before-completion` | Before claiming anything works |
| `wayfinder` | Only if the migration needs its own map. #261 is the existing one |

---

## 12. Provenance and remaining gaps

The predecessor effort's own weakest points, stated so you can weigh them:

- **That a wasm guest cannot reach host memory is INHERITED, not probed.** It is the one
  load-bearing claim in the interpreter shape that no probe has tried to falsify. An adversarial
  guest spike is specified in `04-alternative-approaches.md` and is **unrun**.
- **Whether a real jetsam eviction reaches the same delegate path on iOS is untested.**
- **Engine-level stack-switching availability on either engine is unverified.** One cheap probe
  would collapse several options in §5.2.
- **Nothing has been through App Review.** Every store-policy conclusion is primary-source reading
  of the agreements, not precedent.
- **#292 needs hardware the venue lacks** — a non-Developer-Mode iOS device with a distribution
  profile.

No secrets, keys or credentials appear in this document or in the referenced docs.
