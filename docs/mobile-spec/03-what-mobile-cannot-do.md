# What mobile cannot do

The genuine impossibilities, each with the evidence that makes it one.

**The standard here is the highest in this set.** A claim belongs in this document only if an
experiment established it or a primary source states it. Anything resting on a decision record,
a code comment, or received wisdom belongs in the open-questions section of the spike results
instead — and several claims that *felt* like they belonged here were moved there, or deleted,
because they turned out to be false.

That is not hypothetical caution. Of roughly twenty-three load-bearing claims this effort
started from, **nine were defective, and every one of them made mobile look more constrained
than it is.** The list below is what survived.

---

## 1. iOS cannot create a process

`posix_spawn(self)` returns **EPERM** with no child created — the child's own marker file never
appears, so this is not "a child started and died" — and `fork()` returns −1 with EPERM.

Measured on a physical iPad Air (4th generation), signed device build, 2026-09-18.

**This is the sandbox, not a store rule.** It holds on every distribution channel: App Store,
enterprise, ad-hoc, sideload, alternative marketplace. The long-repeated formulation "iOS has
no fork/exec *a store will accept*" reached the right conclusion for the wrong reason, and was
never sourced to any guideline.

**Consequences**: out-of-process module hosting, `process`/`sandbox` placement, and any
per-module failure domain are permanently unavailable on iOS. A module's crash is the
application's crash. "Stop the execution form" for a Bundled native module means killing the
application.

---

## 2. A module in the Web container cannot block on its own outbound call

Structural, on both platforms, and for a reason that is platform-independent.

A bridge-hosted module runs on a single turn-based scheduler. The Response to an outbound
Request can only be observed by a *later turn* of the very scheduler a blocking wait would be
occupying — so blocking prevents delivery of the thing awaited. **Deadlock, not slowness.**

Corroborated at the link step: the blocking shape does not compile, deliberately, with the
reason recorded at the time — `wasm-ld: error: undefined symbol: lp_client_create`.

**Threads do not rescue it even where they exist.** The channel is bound to one image execution
context, and a worker thread has no channel. So the constraint is capability-independent as
well as platform-independent — and **justifying it by SharedArrayBuffer is wrong**, because
that story differs by platform while this one does not.

**Note what this does *not* say.** A module may answer on a *later turn*; deferral works and is
measured. What it cannot do is *block*. A contract need not be reshaped per boundary.

---

## 3. A `web` variant cannot listen, accept inbound connections, or use UDP

No web API accepts a connection. The browser libp2p transports return
`MultiaddrNotSupported` from `listen_on`, and a hand-rolled transport would do the same. The
only inbound path is a **relayed** address, never a listener.

All UDP is likewise absent — QUIC transport, mDNS discovery, UDP DHT dialing — and NAT
traversal with it: the hole-punching specification defines only TCP simultaneous-connect and
QUIC packet exchange, and mentions browsers nowhere.

**The real line is listening versus dialing, not "networking".** Dialing works — a `wss` binary
echo round-tripped in 319 ms from a dedicated Worker on the shipped origin, and an external
`fetch` reached an arbitrary host subject to CORS. See the spike results for what this
*removes* from the native floor.

---

## 4. WebRTC is unreachable from where our modules run

`RTCPeerConnection` is `[Exposed=Window]`. Our image runs in a Worker. `RTCDataChannel` is
transferable *into* a Worker but not creatable there, so it would require page JavaScript to
perform signalling and hand across a pipe.

Emscripten provides no WebRTC C API at all, so the gap is not bridgeable on our build target
either.

This is **architectural, not a target choice** — it would remain true if every other build
decision changed.

---

## 5. Wasm threads are unavailable on Android WebView

Shared memory can be *created* there but not *shared*: `postMessage` throws
`DataCloneError: SharedArrayBuffer transfer requires self.crossOriginIsolated`, identically for
the `WebAssembly.Memory` wrapper and the bare buffer — **one gate, not two**. Emscripten's
thread model posts the memory to each worker, so it cannot start at all.

Measured on a Xiaomi, Android 15 / API 35, WebView 152.0.7977.64, 2026-09-22.

**The negative is trustworthy because the control passes**: the same harness in Chrome on the
same device completed a genuine two-worker handshake — one waiter woken after 707 ms, three
contexts observing each other's writes. The only variable is `crossOriginIsolated`.

**Two qualifications that must travel with this claim.**

- It is an **Android** constraint, not a mobile one. On iOS `crossOriginIsolated` *is*
  reachable from an in-app loopback origin, and there the same probe passes.
- **One flag is the whole switch.** That WebView ignores the isolation headers — they
  demonstrably reach the renderer and are disregarded — so no work on our origin can change it.
  But if a future WebView build honours them, threads appear with no change to our design.
  **Re-probe on a WebView bump rather than treating this as permanent.**

---

## 6. Native code cannot be downloaded and executed under either store's rules

Quoted from current policy text rather than interpreted.

- iOS forbids the **act**: an application "may not download or install executable code".
- Google Play forbids the **source**: an app "may not download executable code (such as dex,
  JAR, `.so` files) **from a source other than Google Play**", with a carve-out explicitly for
  "code that runs in a virtual machine or an interpreter".

**These are different prohibitions and must never again be written as one.** Play provides a
first-party channel that satisfies its own rule; iOS has no equivalent. And the carve-out
covers *interpreted* code, so it does not distinguish WebAssembly from any other interpreted
form.

**This constrains policy, not the operating systems.** Both platforms load and execute
runtime-delivered native code at the OS layer — see the spike results. Conflating the two is
what produced the false claim that the platforms forbid it.

---

## What is NOT in this document, and why

Each of the following was believed to be a hard limit at the start of this work, and each was
measured false:

- *"iOS never unloads a `dlopen`'d framework."* The pin is a **thread-local**; packaging is
  irrelevant.
- *"Android cannot `dlopen` from the app's data directory."* It loads **and executes** — Android
  removed `execve()` of a file in the app home, not `mmap(PROT_EXEC)` of one.
- *"dyld refuses a runtime-staged image."* **Location is not the gate; signature trust is.**
- *"A `web` variant cannot do networking."* It dials, over TLS, measured.
- *"There is no SharedArrayBuffer on iOS."* Available since Safari 15.2; our own origin denied
  it.
- *"RFC 9266 channel binding is unreachable on mobile."* Public API since iOS 12 and API 31.
- *"One runtime copy per process is required."* True of the capability-token state; **false of
  transport**, which is duplication-immune by construction.
