# Specification update proposals

Organised by document and section, so each item is actionable in place. Cited against the
snapshot in `docs/spec/`.

**Two design rules govern everything below**, and they were arrived at rather than assumed:

1. **Platform divergence belongs in capability advertisement**, not in the execution mode, not
   in a placement name, and not in a new platform-profile document. `LOGOS-MODULE-LOADER §4`
   already exists and already carries the right rule — *"A field MUST be present only when the
   Loader can satisfy every requirement of that strategy or operation in the current
   deployment."* The test this design had to pass is **zero platform-conditional normative
   statements**, and it passes. Evidence that it sits at the right layer: two measurements that
   inverted a platform's behaviour mid-effort changed **no normative text**, only what a
   deployment advertises.
2. **Where divergence would otherwise touch the contract, pin the contract to the weaker
   platform** and treat the stronger platform's capability as implementation freedom.

---

## LOGOS-MODULE-INTERFACE

**`§2.1` / naming** — a runtime module name is unique within one Runtime instance; an ABI caller
MUST resolve module symbols only from the binding accepted for that name and MUST NOT resolve
them from the process-global namespace; on a platform with a global namespace a Bare artifact
MUST be mapped privately; two bindings for one name MUST NOT be accepted, and a platform where
they cannot coexist MUST report a load failure rather than silently bind the first.

**`§2.6`** — an implementation binding MUST NOT define the Runtime's protocol, capability-token
or store-isolation symbols, leaving them undefined for upward resolution. *(This is the first
specification basis the undefined-symbol contract has ever had; our build gate enforces a rule
no document states.)*

**`§2.6`, exactly one copy of the capability-token state per process.** A second copy splits
authority two ways: an authorized call is refused, and identity isolation declared in one copy
is not in effect in another. **Nothing for transport** — measured duplication-immune, and our
own symbol gate has always agreed. Mode configuration is a SHOULD.

**`§2.6`, "both surfaces"** — soften: a provider MUST expose dispatch, and MUST additionally
expose the typed per-method functions **where its build target supports them**; a caller MAY use
them **only when compiled against the generated declarations**; a descriptor-driven caller MUST
use dispatch.

**`§2.6`, descriptor verification** — permit substituting verification of an integrity seal for
re-parsing and re-hashing **where a statically bound provider's descriptor and roots are fixed
at build time and covered by the same seal as the executable image**, explicitly **not** for
dynamic schema text. And cap constrained-host descriptors against the current 8 MiB × 256.

**`§2.8`, two additions.** The applicable direct-subscriber set is **empty** for a publisher not
executing in the subscriber's address space, so the synchronous-publication requirement MUST NOT
be claimed for it. And — the genuine defect, on the native side — a direct subscriber callback
and a direct provider call **MUST NOT perform an unbounded wait** and MUST return within the
deployment's declared direct-call budget; a deployment whose calling thread is under an external
liveness bound MUST declare one. **The remedy for a module that cannot satisfy it is a different
realization, not an exception to the synchronous rule.**

> As written today, `§2.8` plus the absence of any duration bound is an unbounded-duration
> obligation on whatever thread published — which on a phone is frequently the one shared UI
> thread. **The specification currently describes the mechanism that kills the application as a
> feature.**

**`§5.1`** — allocate an optional advisory `detail` for transport errors with a closed cause
set. It MUST NOT authorize replay, and **MUST be omitted rather than guessed**, which is what
keeps a shared-renderer platform conforming.

**`§5.3`** — replace the flat limits with a **constrained-host profile** (32 levels, 262,144
bytes); an implementation MUST reject at its declared limits rather than terminate the envelope;
a recursive-descent implementation on a fixed stack MUST use explicit-stack traversal or derive
its depth limit from measured per-level frame cost. *(128 levels does not certainly overflow a
64 KiB stack — 48 B/level at `-O2`, 290 at `-O0`, two concurrent walkers overflow. Marginal and
implementation-dependent, which is worse for conformance than a certain crash.)*

**Non-normative note** — a realization whose module-facing ABI cannot defer MUST NOT present
that limit as a contract property, and MUST NOT cause a module to publish methods its native
twin does not.

---

## LOGOS-MODULE-RUNTIME

**`§3.4`** — `unloaded` establishes that Runtime holds no active realization. It does **not**
establish that the binding was released, the image unmapped, or process-local state discarded.

**`§3.4`** — platform suspension MUST NOT by itself change a module-instance state; on resume
Runtime MUST establish which contexts remain live before reporting any change, MUST presume loss
of every module sharing an envelope whose per-image liveness it cannot observe, and **MUST NOT
report survival it cannot observe**. A suspended realization maps to an existing state; calls
answer not-ready rather than stalling.

**`§6.4`** — one sentence making the direct-call budget a realization admission condition, so
the check has an owner.

**`§9`** — a memory-pressure advisory event carrying a level, no authority, no deadline;
ignoring it is conforming and Runtime MAY evict regardless.

**`§9.2`** — a third invocation-descriptor branch for the bridge: `kind` stays
`local-transport`, carrying **no path, no endpoint address and no TLS material** — an opaque
single-use channel reference that MUST NOT be derivable from the module name, enumerable from
the image, or reused, delivered through the protected realization handoff and never placed in
page-reachable storage.

**`§10.1`** — a realization profile MUST declare whether it provides a per-instance failure
signal. Where it does not — **including every `direct` realization** — Runtime **MUST NOT infer
module failure from a call timeout or from provider silence**, MUST NOT report a restart
capability, and its policy becomes a crash-loop quarantine evaluated at Runtime start. Eviction
by a profile's resource policy is an **authorized stop**, not a failure.

**`§10.2`** — a bounded per-phase deadline declared by the profile, total below the platform's
own termination budget; **strike envelope termination where the envelope contains Runtime**; a
deadline miss transitions to error, never to unloaded.

**`§11`** — a presentation surface is a realization resource, individually releasable, and its
loss MUST NOT by itself change the instance's state. A presentation profile that may withdraw a
surface MUST define a **budget** and a **deterministic eviction order**, and the order **MUST
NOT be acquisition order**. **This specification defines no budget value.**

---

## LOGOS-MODULE-TRANSPORT

**`§4.1`** — a callee MAY commit a Response in a later turn; a caller MUST NOT require it within
the delivering turn, nor treat that turn's completion as evidence of abandonment. Every Request
MUST have a **finite completion bound**, which MUST NOT be derived from method identity in a
contract. On expiry the awaiting side MUST send Cancel and report exactly one terminal Response
with a timeout error — **and MUST NOT close the connection for that reason alone**, because on a
bridge channel the connection is the entire module.

> Note the gap this fills: **the document contains the word "timeout" zero times**, while a
> timeout error code exists with nobody specified to emit it.

**`§6`** — where a callee's Responses, inbound Requests and Cancels are processed by one ordered
context, that context is the linearization point. A caller that already reported a terminal
Response MUST discard a later one for that id **silently**.

**`§8.2`** — close the vacuous-conformance door: **a peer-identity check satisfies this
requirement only if its predicate can fail** for a peer the deployment permits to exist. Where
every process able to reach the endpoint necessarily presents the expected identity, this
profile MUST NOT be selected.

**`§8.x` — `logos.local.in-process`**: no socket, no stream binding, no Hello; explicitly does
**not** claim protected route authorization; carries **no route ticket** and an implementation
MUST NOT synthesize one; claims no containment. Consumer identity is assigned by Runtime from
the realization record and MUST NOT be caller-supplied.

**`§8.6` — `logos.local.bridge-channel`**: inherits the message maps and the route-ticket
profile verbatim; **messages MUST NOT be length-prefixed** (the carrier's boundary is
authoritative); a transfer encoding is required and a receiver MUST reject a non-canonical one;
caps on chunk size, chunk count, concurrent reassemblies and a reassembly deadline, with any
breach closing the channel; **a mandatory accepted size with a profile ceiling, and a contract
needing more MUST be refused at admission**; each direction is one ordered sequence with at most
one send in flight and **no priority lane**; a bounded outbound queue **with one slot reserved
for a terminal protocol error**; and the peer-verification substitute — endpoint unforgeability
by construction plus Runtime-side origin, frame and world binding.

**Canonicalisation boundary** — deterministic encoding and payload commitments apply to the
**complete message octets after reassembly and after removal of the transfer encoding**.
Chunking, transfer encoding and batching are **carriage**, not part of any canonical form.

**`§8.4.1`** — exclude the TLS 1.2 leg explicitly; pin the encoding (label, **zero-length**
context, 32 bytes) with an implementer note that a null context is **not** an empty one; and
**forbid 0-RTT on a bound session**, since early-data enablement would let application bytes
precede a computable binding.

---

## LOGOS-MODULE-LOADER

**`§3`** — a fourth realization strategy, `hosted_interpreted`, with placements `webview` and a
**reserved, undefined** in-process-interpreter name. Module Loader MUST verify the exact image
bytes **before the image's execution context is created**. And `static_binding` gains
provenance fields marked **MUST-NOT-verify**.

**`§4`** — capability sub-fields for the interpreted strategy (per-image isolation, an image
termination signal, shared-memory threads, durable state, presentation evictability, live
configuration), plus a **top-level** `execution_while_hidden`, which is **absent on both
phones** and therefore not a divergence.

**`§5.1`** — the module-visible state path need not exist in the host's filesystem namespace;
for an interpreted realization the profile MUST provide an **origin-scoped persistent store**
surviving reload, image replacement and application restart, MUST **fail the realization rather
than supply a volatile path**, and MUST NOT map two instances onto one store. **A profile whose
durability barrier completes asynchronously MUST report a failed write-back at the next
barrier** — otherwise a conformance test asserting durability synchronously passes vacuously.

**`§6`** — "complete when it returns" constrains the **record, not the duration**; a Loader MUST
NOT return an active realization before initialization completes inside the created context.

**`§9`** — return a *failed* realization where cleanup cannot establish absence, rather than a
success it cannot establish.

**`§10`** — an `execution-context-evicted` outcome, retriable through release.

**`§12`** — the admission duty: a deployment that cannot realize any isolation profile MUST
treat admission as **admission to the trusted computing base**, MUST declare that trust basis,
and Runtime MUST NOT report a containment claim. Loader MUST NOT advertise a strategy the
platform cannot realize.

---

## LOGOS-MODULE-PACKAGE-MANAGER

**`§4`/`§10`** — `platform_constraint` gains a **realization** discriminator (native versus
interpreted) rather than overloading the operating-system field; the artifact kind splits into
native and interpreted module images **on the artifact, per variant**; an interpreted image
gains an entry point; and an artifact location becomes a **discriminated locator** — a path, a
container entry, or a platform-resolved name — because an artifact inside a package archive or
delivered by a first-party channel has no path.

**`§7.1`** — the variant selector takes a **supplied platform selector**, with the local
platform the default for installation only; preference follows **selector order** first.
**`§7.4`** — write the baseline platform profile in this document; none exists, and that is the
defect rather than a reason for a new file.

**`§5`/`§15`** — a **resolved-set record** with per-member target, peer-satisfaction marker,
available variants and a **`partial` flag**; a read-only `resolve_set`; a `sealed` package source
with present/deferred delivery; new `installation-unavailable` and `artifact-unavailable`
outcomes; and a **`list_capabilities`**, which the document lacks entirely.

> **A live defect**: an absent installed artifact is currently reported as an integrity failure —
> **a security alarm for a normal state**. `LOGOS-MODULE-LOADER` already has the honest code.

**`§9`/`§10`** — the artifact record carries a **build-derived, per-variant unloadability
verdict**; Package Manager MUST reject an author-supplied value; the verdict is covered by the
package signature; and a module MAY declare *intended* unloadability with a mismatch failing
packaging. **Declare intent, derive capability.**

---

## LOGOS-MODULE-CAPABILITY-AUTHORITY

**`§3.3`** — decouple placement from transport: a profile separating address spaces requires
local transport; one providing an in-process containment mechanism applies to direct; and a
direct decision omits placements unless its profile defines such a mechanism **and names its
enforcement component**. Replace the closed placement enum with a **registered isolation-profile
name**, so a future boundary is not a wire break.

**`§8`** — **quarantine**, for the case where no enforcement component can withdraw a permission
and the execution form cannot be stopped: deny every route touching the instance, mark it
terminal for the Runtime lifetime, establish no new realization, record the retained limitation,
and defer reclamation to the next start. **Quarantine MUST NOT be reported as a containment
claim.** *(Our container already implements exactly this and nobody named it.)*

**`§9.1`** — split "MUST implement" into **MUST accept when verifying** versus **MUST implement
one profile when producing**; register an **ES256** profile, because the mandatory algorithm is
unreachable in both platforms' protected key stores while P-256 is first-class; record a
**key-protection class in the protected trust input**, never in the signed claim; and narrow the
claim — where the key is software-resident in a shared address space, the container attests to
the **containing process and its admission decision**, not to the Runtime-controlled invocation
boundary.

**`§4.2.1`** (Runtime) — narrow it to the hosted case, and replace the in-process reading with a
**testable** rule: no authorization may depend on the confidentiality of material in that
address space — **replacing every such value with a constant MUST NOT change any outcome** — and
material whose compromise extends beyond the process MUST NOT be held there in recoverable form.

---

## LOGOS-MODULE-CONFIGURATION

**`§4`** — a **provenance-honesty** rule: record a protected-provisioning class only for a value
accepted through such a channel, and a deployment without one MUST NOT record that class.

**`§6`** — a bridge-hosted sibling to the native-ABI delivery sentence: the realization
mechanism MUST encode exactly one configuration input and install it into the image's bound
context **before that context is reachable by any inbound Request**, never in page-reachable
storage.

**`§8`** — the mode's application hook as a profile control frame, **not a schema method**;
deferral permitted and never required; and **an outcome is indeterminate only on deadline expiry
or context loss — a Response not yet written is not an indeterminate outcome.** Without that
narrowing the common case walks into an error state.

**`§8`/`LOADER §8`** — an apply MUST NOT be satisfied by re-realizing, and MUST NOT re-realize a
bridge-hosted instance in a way that **terminates another instance's context**.

---

## LOGOS-MODULE-SYSTEM-BCP

**`§5.1`** — scope the prohibition on substituting a pre-existing implementation to the **module
context and provider endpoint**, not the execution envelope of the process-local caller, which
pre-exists every static realization anyway. *(Without this the static strategy is dead text on
every platform, not just mobile.)*

**`§5.1.1` — Fixed-Set Deployment**: a deployment whose installed set is closed and supplied as
protected input; installation operations MAY be permanently unavailable and MUST be reported
with a distinct outcome; the read side stays mandatory over the closed set; and **no
unavailability may be reported as success, as an integrity failure, or as an empty result.**

**`§5.4`** — the first-party delivery channel's bounds, the delivery trigger, and the rule that
a deferred member has no realization descriptor until delivery completes.

---

## LOGOS-MODULE-COMMITMENT-MODEL

State explicitly that **the schema root does not commit to the module name**, and require the
ABI caller to obtain the expected name from the resolved record **before mapping**, rejecting a
mismatch **independently of descriptor validation**.

> The hole is the inverse of the one we expected: renaming is identity-preserving, and **the ABI
> prefix is bound to contract identity nowhere** — two documents differing only in the name share
> a root, while the only binding is a text comparison.

---

## LOGOS-MODULE-SECURITY-CONSIDERATIONS

**`§2`** — one sentence: the "not code confined by a browser sandbox" framing is inaccurate for
mobile, where on one platform the browser sandbox is the **only** containment available.

**`§10`** — an implementation MUST NOT claim tamper-evident local audit retention where the
retained state is reachable by module code in the same address space; a deployment needing audit
integrity MUST export to a consumer outside that process over a protected session.

**`§11.4`/`§11.5`** — record that the mandatory evidence algorithm is unreachable in both
platforms' protected key stores while P-256 is first-class, so **"hardware-back the evidence
key" is not re-derived as a mobile blocker**; and that one platform's process-creation refusal
is a **kernel** property, not a store rule.
