# Alternative approaches

The hosting shapes that could carry a conforming module on a phone, judged on their merits
rather than inherited from what was built first.

**A vocabulary correction leads, because it reorganises the question.** "Runtime-staged" is not
a rival hosting shape — it is a **delivery axis crossing the bundled-dynamic shape**, a question
about where an artifact came from rather than how it is realized. Recognising that collapses
seven candidates into four tiers plus one delivery variable.

---

## Shape-by-shape verdict

| # | shape | iOS | Android | cost | grade |
|---|---|---|---|---|---|
| 1 | **static, linked in** | yes | yes | no release, ever; the app seal attests the **image, not the module**, so per-module attribution is gone | conformance argument asserted; platform not in doubt |
| 2 | **dynamic, in the bundle** | yes | yes | **admission is TCB admission**; release only where the implementation language permits | measured on both devices |
| 3 | **staged at runtime** | **conditional** — OS layer open, store policy forbids the *act*, ordinary-device trust **open** | **yes**, via the first-party channel | membership fixed at release; bounded module count and size | measured on both (one device each); policy from primary sources |
| 4 | **separate process** | **closed, kernel, permanent** | **absent today** — the platform permits it, but no selectable local-transport profile carries the endpoint | — | iOS closure measured; the Android gap is asserted **by our own rule**, and reachability is untested |
| 5 | **webview** | yes, **isolates**, ~93 MB/page | yes, **does not isolate** in the shipped shape, ~37 MB marginal | cannot block on its own outbound call | implemented and measured |
| 6 | **in-process interpreter** | plausible | plausible | **unmeasured** | placement name reserved and deliberately undefined |
| 7 | **remote** | yes | yes (deployment floor on the platform path) | cannot cover offline, latency-sensitive, or platform-access modules | decided; the recorded blocker was disproved |

**Shape 4 deserves its own note**, because it is the one case where *our own proposal* closed a
door the operating system left open. Android has three process shapes. What is missing is a
selectable local-transport profile: the strategy is welded to a Unix-socket profile, and our
anti-vacuity rule — a peer check counts only if its predicate *can fail* — voids that profile
inside one application sandbox, where every peer necessarily presents the same identity. **A
profile gap, not a platform one**, and the fix is one new profile rather than a concession.

---

## The recommended combination

Not a ranking. **Four tiers**, and the assignment rule matters: three findings cut the module
space along axes that **do not nest**, so a module's tier is the *strictest* answer across all
three.

- **Listening versus dialing** is the real line, not "networking".
- **Unloadability is fixed by implementation language**, derived per variant.
- **Offline, latency-sensitive and platform-access modules stay on the handset**, so the graph
  is mixed by construction.

### Tier 1 — static. The platform floor, and the home for the unreleasable.

Anything with platform access; anything that **listens**, does UDP, NAT traversal or WebRTC;
anything whose networking core has no browser transport; anything on the cold-launch path. **And
— the argued addition — every module the build-derived record marks unreleasable.**

That addition is the part worth defending. Putting an unreleasable module behind the dynamic
shape buys a release operation that can never succeed, and the Loader is then obliged to retain
a failed realization for the life of the process. **Static does not answer the release question
— it deletes it**: nothing was mapped to reclaim, and the capability record honestly omits the
force-release field.

The cost, stated plainly: the platform signature **attests the image, not this module's bytes**.
Per-module attribution is gone and one compromised member is indistinguishable from any other.
Keep the manifest commitment for **provenance only**, marked so it can never be mistaken for
evidence.

### Tier 2 — dynamic, bundled. Native modules that can actually be released.

C++ (one thread-local, ours, removable), light Rust with a dynamic standard library once our own
SDK thread-local is fixed, single-threaded Nim with the two required flags — **and only where
the signed per-variant record says so, never by intent**, because unloadability **rots
silently**: a transitive dependency bump lands a thread-local with no source or manifest change.
Per-variant, because one platform pins where the other unmaps.

Delivery may be first-party-deferred; that is the **delivery axis crossing this tier**, not a
tier of its own.

**What Tier 2 must not claim is isolation.** Admission — the decision to map bytes into the host
image — *is* admission to the trusted computing base. Two obligations follow: the duplicate-symbol
gate must cover every admission-authority type and a second copy must fail **closed** (one such
type does; another **fails open**, which is exactly the case the rule exists to catch), and the
Loader must return a failed realization rather than a success it cannot establish.

### Tier 3 — webview. Downloaded, third-party, untrusted.

On iOS this is **the only containment available for foreign code at all**, since the process is
closed permanently at the kernel, leaving exactly two boundaries: admission and the webview.

Scope it by the listening/dialing cut: **dial-only logic plus everything UI-bearing**. HTTP and
WebSocket *clients* belong here and were swept out wrongly — one module is native today because
a crate gates its browser backend on the target triple, not because of any sandbox. Disqualify
anything whose contract must **block on its own outbound call**; that is deadlock, not slowness,
and threads would not help even where they exist.

**The two platforms diverge in character while staying one shape**: on iOS ~93 MB per page makes
this a two-or-three-module ceiling **but it isolates**; on Android ~37 MB marginal is affordable
but buys **no isolation** in the shipped shape, while the per-app-process alternative isolates at
~114 MB per module and leaves **one live module and N−1 suspended**.

So **iOS Tier 3 is a security choice and Android Tier 3 is a cost choice** — same tier, opposite
reasons. That is itself the evidence that the divergence belongs in capability advertisement.

### Tier 4 — remote. Everything the network can carry.

Any module that is none of offline-required, latency-sensitive, or platform-access.

**This tier changes the arithmetic of the map rather than adding to it.** On iOS it is the only
way a native module obtains a real failure domain, since the process is closed at the kernel. And
it dissolves, in one move: code signing and the staged-load question, unload and the thread-local
pin that decides it, single-address-space isolation, the JIT prohibition, one-copy-per-process,
and the entire protocol-free module shape — **which exists only to make in-process hosting
work.** A large fraction of this map's difficulty removed by *placement* rather than by
specification text, and it needs the least new text of any shape.

**It has no blocker.** The channel-binding requirement recorded as one is reachable through
platform TLS on both phones — no bundled TLS stack, no second trust store, no bypass of platform
certificate handling. What remains is smaller and different: a deployment floor on one platform,
and one leg of the binding specification that must be **explicitly excluded** because one
vendor's implementation diverges there and would be silently wrong.

And its key material is **strictly better than in-process realization's**: mutual-TLS client
certificates are curve-compatible with both platforms' protected key stores, where call-evidence
identity is hardware-bindable **nowhere** on one of them. Worth stating plainly:

> **Remote transport has a better key-protection story on mobile than in-process realization
> does.**

What it costs: four specification gaps stand — no suspend/resume contract (a phone suspends
constantly, and every resume finds connections and subscriptions gone, indistinguishable from
peer failure); no model for a graph **spanning the network**, which is exactly the topology this
creates; nothing about which grants may cross a remote boundary or whether a consumer must be
told a provider is remote; and enrollment durability across update, reinstall and restore is
undefined. And it does **not** reduce the in-process work — Tiers 1–3 stay live for everything it
cannot carry.

---

## Shape 6: worth defining?

**Yes — and worth exactly one spike before anyone writes normative text.** It is the only
candidate that is neither native-in-process nor a webview, and nobody had given it an answer.

**The argument that makes it interesting** is not cost, it is this: *"one Worker, one event loop,
no ASYNCIFY"* — the cause of "a hosted image cannot block on its own outbound call" — is a
**webview property, not a wasm property**. A host-driven interpreter can **re-enter**, because
the host owns the call stack and can service an outbound Request while the guest's frame is still
live.

That would make our synchronous module dispatch **correct by construction** rather than the
implementation defect it currently is, and remove the reason `web` modules ship job-id pairs in
their own interfaces. **Shape 6 is the only candidate that turns that defect into a non-issue
instead of specifying around it.**

Two supporting facts, both measured: non-JIT in-process execution on iOS is real (an interpreter
did a full workload where a JIT died at the *call*, and a runtime-built foreign-function call
works with **no executable page**, on a device that refused writable-executable memory); and it
removes the webview's ~93 MB per page, its origin/frame/world binding, and the blast radius where
several modules share one image.

**The spike — four numbers and one yes/no**, running an *existing unmodified* module image on
device: cold instantiate including validation, against the launch watchdog; per-call latency
against a bridge round trip; resident memory per image, against ~93 MB — *if it is not
dramatically smaller, the main argument collapses*; **whether an outbound call from inside a
guest frame completes and returns into that frame**, which is the decisive claim and is currently
reasoning rather than measurement; and which import surface the image actually requires, because
our images are built against an emscripten libc and its filesystem, so **state persistence is
re-opened, not inherited**.

Do not pick the interpreter by reputation — a JIT-capable engine is a dead end here, so the
candidate set is interpreters only.

**If it passes, three small edits**, because the ground is already prepared: define the reserved
placement under the interpreted strategy, carrying the webview branch's verification rule
verbatim; mirror the capability sub-fields, adding one for re-entrant outbound calls; and
register a **third local-transport profile**, because neither existing one fits — one binds to an
origin, frame and script world that do not exist here, and the other explicitly claims no
containment, which is this shape's whole reason to exist. The substitute for a peer check is that
**the host owns the interpreter's import table**: the guest reaches exactly the functions the
host installed.

**If it fails on memory or on re-entrancy, the shape stays reserved and undefined** — which is
the outcome already chosen, and would then be chosen on evidence.

---

## What is a product decision, not a technical one

Four questions this work deliberately does not answer, because they are not ours to answer:

1. **Which module kinds tolerate remote hosting.** Tier 4's boundary is drawn by offline
   tolerance and latency budget — both product properties. The technical answer is "remote works
   and has no blocker"; *which* modules go there is a judgement about the experience.
2. **Whether to pursue an in-app loopback origin on iOS.** It buys cross-origin isolation, and
   with it threads and synchronous fetch. It costs per-module port management, because the
   persistent store is keyed by origin, and it cannot be validated on a simulator. A real trade.
3. **Whether to use one platform's extra isolation or keep one shape across both.** The
   recommendation is to keep one shape, argued on composition and cost — but the counterweight is
   real, since that platform's admission boundary is measured weak and process is its only
   enforcement.
4. **Whether runtime-staged native delivery is worth pursuing off-store at all.** It is
   technically open on one platform pending one experiment, forbidden by store policy regardless,
   and legal through the other platform's first-party channel. Whether an off-store channel is a
   product the team wants is not a technical question.
