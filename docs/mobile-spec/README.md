# Mobile under the new module architecture

Four artifacts, answering one question: **can the mobile builds live under the next module
architecture, and what work does that imply?**

| | |
|---|---|
| [01 — Specification proposals](01-spec-proposals.md) | Where the specifications are incomplete for mobile, as text a reviewer could apply, organised by document and section. |
| [02 — Spike results](02-spike-results.md) | What each experiment showed: device, version, date, exact error text. Negatives carry the same weight as positives. |
| [03 — What mobile cannot do](03-what-mobile-cannot-do.md) | The genuine impossibilities, each with the evidence that makes it one. The highest standard of proof in this set. |
| [04 — Alternative approaches](04-alternative-approaches.md) | The hosting shapes that could carry a conforming module on a phone, per platform, with the text each needs — and the recommended combination. |

Specifications are cited by document name and section against the dated snapshot in
[`../spec/`](../spec/README.md). Evidence is graded in
[`../evidence-ledger.md`](../evidence-ledger.md).

---

## The answer, in short

**Yes — and the architecture is substantially less constrained on mobile than we believed when
we started.**

Of roughly twenty-three load-bearing claims this work began from, **nine were defective, and
every single one made mobile look more constrained than it is.** That bias is the most important
finding here, because it is the bias that produces a bad specification proposal: one asking
authors to carve out limitations that do not exist, and quietly foreclosing options nobody
decided to give up.

What turned out to be possible after all:

- **Runtime-delivered native code works at the OS layer on both platforms.** On Android a staged
  `.so` loaded *and executed*; on iOS a team-signed dylib staged into the app's data container
  loaded *and executed*, because **location is not the gate — signature trust is**.
- **One store permits it through a first-party channel**, documented by the platform vendor. The
  two stores forbid different things: one forbids the *act*, the other the *source*. They must
  never again be written as one constraint.
- **Remote modules have no blocker** — the channel-binding requirement recorded as one is
  reachable through platform TLS on both phones.
- **A `web` variant can do real networking** — measured, over TLS, from a Worker. The real line
  is **listening versus dialing**, not "networking".
- **An in-process interpreter runs a shipped module image at ~0.13 MB per instance** — roughly
  700× cheaper than a webview page — and **can re-enter**: a blocking outbound call completed
  inside a live guest frame, with a nested re-entry answered at frame depth 2. The constraint
  that a hosted image cannot block is a **webview property, not a wasm property**.
- **One platform can host a module out-of-process after all** — an isolated process reached a
  local endpoint over a descriptor-delivered socket and **executed a module delivered after
  install**, giving it a per-module failure domain and a real identity boundary the other
  platform cannot have. It had been recorded as unavailable because the only defined transport
  profile is unusable there — **a profile gap, not a platform one**.

What is genuinely closed is in [artifact 03](03-what-mobile-cannot-do.md), and it is a short
list.

## The design rule this produced

Two platforms that diverge on nearly every axis, and **zero platform-conditional normative
statements**. Divergence belongs in **capability advertisement**, which the specifications
already have and which already carries the right rule; where divergence would otherwise touch a
contract, the contract is pinned to the weaker platform and the stronger platform's capability
becomes implementation freedom.

The evidence that this sits at the right layer: two measurements that *inverted* a platform's
behaviour mid-effort changed **no normative text at all** — only what a deployment advertises.

## How to read the evidence

Every claim carries a grade, and the grades mean what they say:

- **Implemented** — shipping code plus a check that fails if the claim is false.
- **Measured** — a command, a device, a number, a date. True once, on one build.
- **Asserted** — written down with nothing behind it. Repetition across a dozen files is still
  assertion.

Only the first may be taken for granted. This distinction is not pedantry: *"iOS never unloads a
`dlopen`'d framework"* was asserted in code comments, in a decision-record rationale and in a
memory note, was repeated as fact by five independent analyses, and is **false**. The real pin is
a thread-local variable, and roughly half our modules could be made unloadable.

**The rubric catches what it is applied to; it does not apply itself.** Three of the corrections
recorded here were to claims made *by this effort, after the ledger existed*. What caught them
was someone asking "says who?".
