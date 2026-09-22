# Spike results

What each experiment showed, as measurements — device, version, date, exact text — so the next
reader inherits evidence rather than belief.

**Negative results are recorded with the same weight as positive ones**, and several here
overturned claims this project had been building on.

---

## Native code arriving after install

### iOS — the gate is signature trust, not the app-bundle boundary

iPad Air (4th generation), Xcode 27 / iPhoneOS 27.0 SDK, 2026-09-18. Images staged into the
app's `Documents` at runtime, then `dlopen(RTLD_NOW|RTLD_LOCAL)`.

| image | signing | `dlopen` | `dlerror()` |
|---|---|---|---|
| unsigned | none | **NULL** | *"mapped file has no cdhash, completely unsigned? Code has to be at least ad-hoc signed."* |
| ad-hoc | `codesign -f -s -` | **NULL** | *"code signature invalid … (errno=1)"* |
| **team-signed** | development identity | **non-NULL** | `(null)` |
| **team-signed, freshly compiled** | development identity | **non-NULL** | `(null)` — **and its constructor ran in-process** |
| in-bundle control | — | non-NULL | `(null)` |

Two tiers: dyld refuses a wholly unsigned image; AMFI refuses an ad-hoc one; a trusted-identity
signature is accepted **anywhere in the sandbox**, including the writable data container. And
it is not merely mapping — the staged image's `__attribute__((constructor))` executed.

**Caveat, stated plainly**: the device is in Developer Mode with a development provisioning
profile. What is proven is that **dyld and AMFI enforce no app-bundle boundary**. Whether an
ordinary device trusts such a signature is **open** — the remaining spike.

*Unexplained, with the explanation labelled as inference*: a byte-identical copy of the app's
own embedded framework loads at its bundle path and is refused in `Documents`. Plausibly
`installd` registering that cdhash against the bundle path; untested. The freshly-compiled rows
are the evidence that matters, since their cdhashes were never registered anywhere.

### Android — it loads and it runs

Xiaomi 25028RN03Y, Android 15 / API 35, targetSdk 36, 2026-09-18.

A real Bare module (`libcapability_module_bare.so`, with its full eight-library closure) and a
marker library both loaded from `/data/user/0/<pkg>/files/` and from `code_cache/`, `dlerror()`
`(null)`, and the marker function **returned its value** — the staged code executed.
`/proc/<pid>/maps` showed `r-xp` file-backed mappings; both files carried the ordinary
`app_data_file` label; no AVC denial.

**So `dlopen` from the app's writable data directory is not a W^X violation.** Android 10
removed `execve()` of a file in the app home, **not** `mmap(PROT_EXEC)` of one — the linker path
was never closed.

**Three traps cleared**, two of which would each have produced a confident wrong answer:
`adb shell` and `run-as` are *not* the app domain, so the probe ran inside a normally-installed
APK whose SELinux context was **verified to match the real Shell's**; bionic dedups by SONAME,
so the in-APK control ran **last**; and an inconclusive first run (an unshipped transitive
dependency failing *all* cases including the control) was reported rather than buried.

---

## The Web container

### iOS isolates; Android does not; and the cost inverts

iPhone 15 Pro Max, iOS 26.6.1, WebKit 8624.5.1.10.3, 2026-09-21. A 172 KB standalone WKWebView
app built by plain `clang` — no Xcode project, no nix, ~5 s a build cycle.

| pages | WebContent (one pid each) | GPU | Networking | host | total |
|---|---|---|---|---|---|
| 1 | 89.3 | 85.3 | 5.9 | 8.8 | 189.3 MB |
| 2 | 90.0 + 89.5 | 88.1 | 5.9 | 9.0 | 282.5 MB |
| 3 | 91.0 + 90.1 + 89.2 | 90.5 | 5.9 | 9.5 | 376.2 MB |
| 3, after killing B | 91.2 + 90.3 (**B's pid gone**) | 84.9 | 5.9 | 9.5 | 281.8 MB |

**~93 MB per extra page** (~65 MB engine baseline). `GPU` and `Networking` are **one process
each for the whole app** — so the ~85 MB GPU cost is one-time, and a budget that multiplies it
per module is wrong.

Killing one page's renderer removed **only** its pid; the others kept running with their
animation counters advancing. The host app's own footprint never exceeded 9.5 MB while 3×24 MB
of heap was live, independently confirming nothing is shared.

**Recovery is clean**: `webViewWebContentProcessDidTerminate:` fires for exactly the dead
webview **18 ms** after the kill; afterwards `evaluateJavaScript` returns `WKErrorDomain` code 5
while live webviews answer normally. So **a terminated page is distinguishable from a quiet
module with no timeout heuristic.**

*Caveat*: the renderer was killed deliberately via SPI. **A real jetsam eviction was never
observed**; that it takes the same delegate path is inference.

**Android, same page weight**: one shared renderer, **~37 MB marginal**, and reaping it took
**both pages at once**, with `onRenderProcessGone` firing for both in the same millisecond.

*This corrected our own published figure*: the previously recorded ~8 MB marginal page had been
measured with much lighter pages. Like-for-like: **37 MB Android shared / 114 MB Android
per-process / 93 MB iOS**.

### Per-app-process hosting on Android isolates — but only the foreground module executes

Three activities in `:mod1/:mod2/:mod3`, each calling `setDataDirectorySuffix`, produced three
app processes, three renderer pids, three isolated uids. Crashing one removed only its renderer;
siblings kept advancing; **modules outlived the core process**; and the core observed nothing —
there is no cross-process callback.

**The catch**: without a foreground service the ROM **freezes** a backgrounded module process
(`cgroup.freeze=1`); with one, the renderer still throttles a hidden page to ~1 Hz and then
stops it. So the shape gives **one live module and N−1 suspended**, at ~114 MB each.

**Storage follows the data-directory suffix, not the process** — a different suffix reads empty,
the same suffix in another process reads the same database, and a live collision is a **fatal**
`RuntimeException`.

### Secure context, SharedArrayBuffer and threads

| origin | secure? | `crossOriginIsolated` | `SharedArrayBuffer` |
|---|---|---|---|
| iOS `logos://` + COOP/COEP | true | **false** | undefined |
| iOS `http://127.0.0.1` + COOP/COEP | true | **TRUE** | **function** |
| Android shipped origin + COOP/COEP | true | **false** | undefined |
| Android Chrome control, same device | true | **TRUE** | **function** |

**The platforms fail differently, and that is the finding.** iOS *honours* the isolation headers
— just not on a custom-scheme origin. Android's WebView **ignores them everywhere**: they
demonstrably reach the renderer (`fetch(location.href)` shows them) and a cross-origin fetch
that should have been blocked returned 200.

**So adding headers to our shipped origin buys nothing on either platform. Moving to a loopback
origin buys threads on iOS only** — measured: a shared `WebAssembly.Memory` survived
`postMessage`, and two Workers completed a real `Atomics.wait`/`notify` handshake, the waiter
woken after 353 ms. The same probe on Android throws `DataCloneError`.

*Trap for any threaded build*: `new WebAssembly.Memory({shared:true})` **succeeds even when not
isolated**, on every origin tested — so a threaded image constructs its memory happily and dies
much later at the worker handoff, a long way from the cause.

### Networking from a `web` variant

From a dedicated Worker on the shipped Android origin, **no COOP/COEP**: `wss://` binary echo
in **319 ms**, local `ws://` in **38 ms**, `typeof WebSocket` = `function`. External `fetch`
reached an arbitrary https host (200, 190,468 bytes, 404 ms) subject to CORS.

**So "everything that opens a socket must be Bundled" is false.** The real line is **listening
versus dialing**.

*Still asserted, not measured*: that emscripten's own C APIs (`emscripten/websocket.h`,
`-sFETCH`) work from our container. Documented, never run here.

### A deferred Response works

Plain `node` against the shipped browser SDK. A handler answered **only** from the completion
callback of a real outbound bridge call — frame #7, after the entire outbound round trip,
correlated by `id`. Mutating it to answer synchronously moves the Result to frame #4 and fails
the assertion, so the harness discriminates. Two Requests also completed **out of order**.

**But a wasm image cannot defer**: all **40** methods of the shipped keystore payload answered
on the stack. That is our own ABI, not the boundary.

---

## Process model and protocol state

**iOS cannot create a process.** `posix_spawn(self)` → EPERM, no child created (the child's own
marker file absent); `fork()` → −1 EPERM. The sandbox, not a store rule.

**Two protocol copies**: the capability-token half is fatal and now reproducible in under a
second — two distinct `TokenManager` addresses, an authorized call refused, with an
anti-vacuity control proving the refusal is caused by the split. **`StoreRegistry` splits the
same way and fails OPEN** — isolation declared in one image is silently not in effect in
another. **Transport did not reproduce**: three hosts bound the *same* URL with no error.

**Unloadability is fixed by implementation language.** A thread-local pins a Mach-O image;
packaging is irrelevant. C++ reaches zero TLVs by fixing one SDK line; light Rust by
`-C prefer-dynamic` plus the same fix; single-threaded Nim by `--threads:off
-d:noSignalHandler`. **Async Rust and threaded Nim never will** — tokio, tracing, rayon and the
Nim GC own thread-local state that cannot be cleared.

### An Android isolated process can host a module

Xiaomi, Android 15 / API 35, 2026-09-22. Every probe run twice — once against a same-UID
service, once against an isolated one — **and the control passed every probe the isolated
process failed**, so each negative is a property of isolation rather than a broken fixture.

- **No address-based socket is reachable.** Filesystem: `connect()` → **`errno=2 ENOENT`**, with
  `stat` and `access` also ENOENT on both socket and directory, and the node `chmod 0777` — so
  the app data directory is **not in the isolated process's mount namespace**. Abstract
  namespace: `errno=13`, `avc: denied { connectto } scontext=u:r:isolated_app:s0`.
- **A Binder-delivered socketpair works**, both directions.
- **Peer credentials are vacuous across it** — both ends report the *creating* process
  (`uid=10252`) when the real peer is `uid=99314`, because credentials are stamped at creation.
  **Ancillary credentials are not**: the kernel attached `pid=5812 uid=99314`, reading `10252`
  against the control. **The predicate can fail.**
- **Attestation is one-directional** — the isolated side cannot enable it
  (`setsockopt → errno=13`, mandatory access control denying `setopt`).
- **Module loading**: a file staged into `files/` fails every route, with `execute` denied on the
  app-data label. An **anonymous memory file works** — mapped directly by descriptor, the
  module's exported function **executed and returned its value**. It must never be re-opened by
  path.

*Not tested*: sealing the anonymous file before hand-off (it almost certainly should be), other
vendors' policy, and **whether a real Module Host survives an isolated process** — this proves
the transport and the loader, not the runtime.

### An in-process wasm interpreter runs a shipped module image, and can re-enter

iPad Air 4, iOS 26.5.2, 2026-09-22. An interpreter already vendored in this tree, as an arm64
staticlib in a 5.3 MB standalone probe app — **not** the product shell. Image: the shipped,
**unmodified** core web image.

| | run 1 | run 2 |
|---|---|---|
| validate + translate | 17.2 ms | 11.8 ms |
| instantiate | 0.1 ms | 0.1 ms |
| constructors + entry | 0.8 ms | 0.6 ms |
| **total cold** | **19.2 ms** | **12.8 ms** |

**Re-instantiating an already-compiled module: 0.4 ms** — validation is ~95% of the cost and is
paid **once per image, not per instance**.

**Per call**, 200 iterations after 20 warm-ups: a real method answered by the module's own code,
full frame in and Result out — **mean 0.23 ms, best 0.17 ms**; dispatch floor **0.13 ms**. No
process, thread or event-loop hop. *No local bridge round-trip figure exists, so no ratio is
claimed.*

**Memory**: first image **+0.9–1.1 MB**; each additional live instance **+0.13 MB**, with five
more alive at once and **all five still answering** afterwards. The image's declared 20.1 MB of
linear memory **never becomes resident**. Against the webview's ~93 MB per page that is roughly
**700×**.

**Re-entrancy — the decisive result.** Inside a host import the guest called *while its own
dispatch frame was still on the stack*: a **real blocking outbound round trip** completed
(0.32 ms warm), then a **nested re-entry into the same instance** which the guest answered from
**frame depth 2**, then a return into the still-live outer frame, which produced its own Result
normally. Separately, an import the **shipped** image already calls during init does the same
thing — blocks on an outbound, calls back into the guest, and returns a value the guest consumes.

**Import surface generalises**: 18 imports for a C++-cored image (all implemented, none stubbed),
31 for a Rust-cored one — **the same host-implementable kinds**, the extra thirteen being
filesystem syscalls. **No JS-only import in either**, so no JS engine is required. The surface is
a property of the **core-image host** and does not grow with the module's language.

**Two limits.** The Rust-cored image does not load, for a reason **orthogonal to imports** —
legacy exception-handling opcodes the interpreter cannot parse, originating in Rust's unwinder on
this target; the likely fix is named and **untested**. And **view-backend images are structurally
out of scope**: they embed arbitrary JS and need a JS engine.

*Not attempted*: a rebuild with the proposed flag; a real inter-module outbound rather than a
socket round trip (**stack mechanics measured, equivalence inferred**); shell integration, so no
claim about its cost; **one device, iOS only**; persistence; and **no teardown measurement**, so
nothing here speaks to reclaiming memory.

---

## Platform APIs, settled from primary sources

**RFC 9266 channel binding is reachable through platform TLS on both phones** —
`sec_protocol_metadata_create_secret_with_context` (iOS 12+, read from the shipped SDK header)
and `android.net.ssl.SSLSockets.exportKeyingMaterial` (API 31+). Secure Transport never had an
exporter, established by grepping the shipped header rather than by failing to find a mention.

**Ed25519 is unreachable in the Secure Enclave and in StrongBox**; P-256 is first-class. So
**transport identity is hardware-bindable everywhere while call-evidence identity is
hardware-bindable nowhere on iOS.**

---

## Open experiments

1. **Does an ordinary iOS device accept a staged team-signed module?** Needs a device outside
   Developer Mode with a distribution profile. Decides whether the staged-load result is a
   product option or a lab result. *(Off-store only regardless — store policy forbids the act.)*
2. **Can an Android isolated process reach a local-transport endpoint?** If an fd-passed
   `socketpair` works and the UID differs, `hosted_dynamic` becomes advertisable on Android.
3. **Does emscripten's WebSocket binding link and dial from our container?** Converts the
   networking-floor finding from documented to measured.
4. **Does a real jetsam eviction take the same delegate path?** The iOS termination signal leans
   on this and it is inference.
5. **Is `logos://` categorically outside cross-origin isolation, or only as configured?** Only
   that scheme was tried.
