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
| 4 | **separate process** | **closed, kernel, permanent** | **available**, via a Binder-delivered socketpair | a new transport profile; module delivery by memfd only | iOS closure measured; Android reachability **measured** |
| 5 | **webview** | yes, **isolates**, ~93 MB/page | yes, **does not isolate** in the shipped shape, ~37 MB marginal | cannot block on its own outbound call | implemented and measured |
| 6 | **in-process interpreter** | **yes, measured** | untested | **~0.13 MB per live instance**, 0.23 ms per call, 0.4 ms to re-instantiate; confines memory and authority but **contains no fault** | **define it** — both gates passed on device; the confinement claim is **inherited, unprobed** |
| 7 | **remote** | yes | yes (deployment floor on the platform path) | cannot cover offline, latency-sensitive, or platform-access modules | decided; the recorded blocker was disproved |

**Shape 4 deserves its own note**, because it was believed closed on both platforms and is not.

On iOS it is closed permanently at the kernel. On Android it is **available** — measured: an
isolated process reached a local endpoint over a socketpair whose descriptor was delivered by
Binder, and **loaded and executed a module delivered after install**. It was recorded as absent
because the strategy is welded to a Unix-socket profile that is unusable there — **a profile
gap, not a platform one** — and the fix is one new profile rather than a concession.

Two details from that measurement change how the profile must be written, and both invert the
obvious design:

- **No address-based local socket is reachable at all** — not filesystem (`connect()` returns
  **ENOENT**, because the app's data directory is *not in an isolated process's mount
  namespace*) and not abstract (separately denied by mandatory access control). This is a
  *second, independent* reason beyond the peer-check problem.
- **The obvious peer check is vacuous and a different one is not.** Peer credentials read across
  a passed socketpair report the **creating** process, not the actual peer — they are stamped at
  creation. The ancillary-credentials mechanism instead carries the true, kernel-attested
  identity, unforgeable by the sender. **That predicate can fail**, which is what the rule
  requires. But attestation is **one-directional**: the host can verify the module, and the
  module cannot verify the host, so it must rest on the descriptor's provenance.

And **module delivery must be by anonymous memory file, not a staged file** — a file in the
app's data directory is denied execute permission, while an anonymous one carries a different
label and maps cleanly. It must be mapped directly and never re-opened.

**Availability is not advisability.** This changes what is *possible* on Android, not the
recommendation below, which rests on composition and cost.

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

**The spike ran, and both gates passed on device.** An interpreter already vendored in this
tree, running the **shipped, unmodified** core image on an iPad:

| | |
|---|---|
| cold instantiate | **12.8–19.2 ms**, of which ~95% is validation — **paid once per image, not per instance** |
| re-instantiate a compiled image | **0.4 ms** |
| per call | **0.23 ms** mean (0.13 ms floor), with no process, thread or event-loop hop |
| memory per additional live instance | **~0.13 MB** — against the webview's ~93 MB per page, roughly **700×** |

Five extra instances were alive at once and all five still answered, so they are live rather
than empty; the image's declared 20.1 MB of linear memory never becomes resident.

**And the decisive question is answered YES, by measurement.** Inside a host import the guest
called *while its own dispatch frame was still on the stack*, the host made a **real blocking
outbound round trip**, then **re-entered the same instance** with a nested delivery the guest
answered **from frame depth 2**, then returned into the still-live outer frame, which produced
its own Result normally.

**It generalises.** A Rust-cored image has 31 imports against the C++ image's 18, **every one of
the same host-implementable kinds** — the extra thirteen are filesystem syscalls, and neither
image needs a JS engine. The small surface is a property of the **core-image host**, and it does
**not** grow with the module's language.

**Two real limits, both specific.** The Rust-cored image does not currently load, for a reason
**orthogonal to imports**: it carries legacy exception-handling opcodes the interpreter cannot
parse, originating in Rust's unwinder on this target — with a named, untested likely fix. And
**`view_backend` images are structurally out of scope**: they embed arbitrary JS and need a JS
engine, so the placement covers `web` **core** images while UI views stay in the webview.

**So: define the placement**, with three small edits — the reserved placement under the
interpreted strategy carrying the webview branch's verification rule verbatim; the capability
sub-fields plus one for re-entrant outbound calls; and a **third local-transport profile**,
because neither existing one fits — one binds to an origin, frame and script world that do not
exist here, and the other explicitly claims no containment, which is this shape's whole reason to
exist. The substitute for a peer check is that **the host owns the interpreter's import table**:
the guest reaches exactly the functions the host installed, and an unimplemented import is
reported by name rather than silently stubbed.

**One condition, and it must not be glossed.** The shipped host has **no outbound door linked in
at all**, so realising a synchronous outbound needs **one ABI edit** — an outbound import
replacing the job-id pair. The *mechanism* is measured; *"therefore the job-id pairs can be
removed"* is **asserted** until one image is compiled against such an import. Until then the
deferral defect remains live in the shipped artifacts.

### What isolation it actually provides

The placement's containment story is **not** one property, and the four parts do not move
together. Against Shape 5:

| | webview | in-process interpreter |
|---|---|---|
| **memory confinement** | MMU, separate process on iOS — **but measured *not* isolating in the shipped Android shape**, where several modules share a page | guest addresses only its own linear memory, no instruction forms a host pointer, separate `Store` = separate memory — **the same on both platforms** |
| **ambient authority** | the whole browser surface; we *subtract* | **none**; we *add*. 18 and 31 enumerable imports on the two measured images, an unimplemented one refused **by name** |
| **resource bounding** | the browser's scheduler | the primitives exist in the vendored crate — fuel metering (`OutOfFuel`) and `StoreLimits` (`memory_size`, `instances`, `tables`, `table_elements`) — **and we use none of them** |
| **fault containment** | a page crashes alone | **none — shared process, shared fate** |

**Grade, and it matters.** The webview rows are **measured in this effort**. The interpreter's
memory-confinement row is **inherited from wasm's design and wasmi's, not locally probed** — it is
the one load-bearing claim in this shape that no probe of ours has tried to falsify. The
resource-bounding row is **primary-source verified** (the APIs are present in wasmi 0.40 as
vendored) and **untested by us**.

**Three caveats on resource bounding**, before anyone leans on it. The limiter's own
documentation scopes it to *guest linear memory* — not wasmi's bookkeeping, not embedder
allocations — so it is not a total-footprint cap. **Fuel counts guest instructions, so time spent
inside a host import is unbounded**: a guest that spams the outbound door is not fuel-limited, and
the host must rate-limit its own doors. And fuel carries per-instruction cost, while the 0.23 ms
per call was measured **without** it.

**Fault containment is the structural gap and no in-process design closes it.** A guest *trap* is
caught and returned as an error, but a defect in the interpreter or in a host import takes the
process. This is Shapes 1 and 2's problem exactly, and a per-module failure domain is precisely
what Shape 4 supplies — which iOS cannot have at all, at the kernel. If failure domains are a
requirement, nothing available on that platform delivers them.

**The trusted base comparison cuts both ways.** wasmi is small, memory-safe, and **performs no
code generation at all** (verified against the vendored source), which removes the single largest
class of browser sandbox escape; and with no high-resolution timer and no shared memory unless the
host grants them, the side-channel work browsers needed COOP/COEP and timer coarsening for is
largely moot. Against that: WebKit is patched by the platform vendor out of band, **and is
re-sandboxed into its own process precisely because that vendor assumes it will be
compromised**. Our surface is genuinely smaller and it is genuinely ours to keep correct.

**The spike that would settle it** — same harness and device as the placement spike, and every
probe able to fail: a read outside linear memory; a grow past the `StoreLimits` cap; an infinite
loop against set fuel; a call to an ungranted import; a reach at a sibling instance's memory; a
**deliberate trap at guest depth 2 while a re-entrant outer frame is live** (the one that matters
most, since the re-entrancy result now rests on that unwind); and outbound-door spam, to confirm
fuel does *not* bound it and to size what the host must add. **Unrun.**

### Store policy: the interpreter is not a downgrade from the webview

Worth stating explicitly, because the assumption ran the other way for most of this effort.
Three questions separate cleanly, and only the third touches policy at all.

**The interpreter itself ships compiled into the app.** It is ordinary submitted code. Nothing
regulates shipping an interpreter on either store, and the one genuine platform prohibition —
JIT — does not reach it: this is a pure interpreter, measured doing a full workload on a device
where a JIT died at the call into generated code.

**Running images bundled in the app engages no store rule.** Nothing is downloaded, so there is
nothing to argue. The ~700× memory result stands on its own as an in-app isolation mechanism.

**Running images downloaded after install is permitted by the letter of both rules — the same
letter that permits the webview.** Apple's DPLA 3.3.1(B) bans downloading executable code and
then carves out *interpreted* code, subject to three conditions (advertised purpose; no bypass of
signing or sandbox; no storefront for other Applications). The pre-2017 restriction of that
carve-out to WebKit and JavaScriptCore is **not in force** — neither string appears in the
current agreement. **Under the text the engine does not matter**, so Shape 6 and Shape 5 are the
same legal object. Play's rule carves out "code that runs in a virtual machine or an interpreter"
in the same shape. Note the contrast with **Shape 3**: that one downloads *native* code, which is
what both rules actually prohibit. Shape 6 downloads interpreted code, which is what both rules
name as the exception.

**What constrains it is engine-agnostic.** Guideline 2.5.2 forbids downloaded code that
"introduces or changes features", and it binds Shapes 5 and 6 identically. The real question is
catalog scope after review, not engine choice — the choice between these two shapes cannot be
made on policy grounds.

**One place Shape 6 is stronger.** The historical rejection trigger is native reach, and 4.7.2
forbids exposing platform APIs to hosted plug-ins. A guest here has no ambient capability at all
and reaches exactly the imports the host installed — 18 and 31 on the two measured images, an
**enumerable** list. That is demonstrable rather than arguable, and it is the opposite shape to
the `dlopen`/reflection bridges that drew enforcement.

**Two places it is weaker, and one caveat on Play.** Downloaded QML and downloaded JS have years
of listed precedent; downloaded wasm through a non-WebKit interpreter has **none** — no
rejections, no acceptances, nothing. And Play's carve-out continues *"where either provides
indirect access to Android APIs"*; our guests reach the host import table, not Android, so
whether that clause reads as a requirement or an example is genuinely open, and a webview is
their named example. **Nothing of ours has been through App Review**; all of the above is reading
primary sources, which is a different thing from surviving a reviewer.

*Grade: rule text is* **primary-source verified**; *the equivalence of engines and the reading of
Play's trailing clause are* **inference**; *acceptance is* **untested**.

---

## What is a product decision, not a technical one

Five questions this work deliberately does not answer, because they are not ours to answer:

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
5. **Which catalog shape the product has** — the one decision that actually binds under store
   policy, and it is orthogonal to every shape above. A downloaded module adds functionality by
   definition, so guideline 2.5.2 read literally forbids *any* catalog; it cannot be read that
   way, because guideline 4.7 expressly permits downloadable plug-ins, and the DPLA's own
   condition is the far weaker *"does not change the primary purpose"* — which a modular platform
   satisfies by construction. So the question is which of three regimes we are in:

   | | what it is | what it costs |
   |---|---|---|
   | **A** | fixed membership, everything present at release | no rule engaged — the shipped Bundled set |
   | **B** | downloads only update module identities already reviewed at submission | defensible under 2.5.2 as not introducing features; the highest-precedent position available (the JS over-the-air pattern, ~1000 iOS builds/day, one reported rejection) |
   | **C** | new module identities appear after review | squarely guideline 4.7: the **host is responsible for every module's compliance with all guidelines**, plus per-module consent (4.7.3), an index with universal links (4.7.4), age gating with the host rated at its highest module (4.7.5), IAP for paid modules, and no native platform API exposure (4.7.2) |

   **C's real risk is silhouette, not mechanism.** The 2026 removals found in this effort were
   apps whose *visible purpose* was running or previewing other people's unreviewed content —
   and at least one was removed despite shipping **data-only** payloads. Technical compliance did
   not save them. This is a question about what the product looks like when a reviewer opens it,
   and it is answered identically whether the modules are wasm, QML or JS. **4.7.2 is the one
   clause where Shape 6 is materially better placed than Shape 5** — the import table is
   enumerable, so "no native platform API exposure" is demonstrable rather than argued.
