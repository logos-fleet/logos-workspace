# Evidence ledger: what mobile actually proved

Input to the mobile/specifications map. Its job is to separate **"mobile cannot do this"**
from **"we assumed mobile cannot do this"**, because only the first belongs in a document
sent to specification authors.

## Why this exists

A claim stated confidently in a code comment, repeated in an ADR and echoed in a README
reads exactly like evidence. Two examples from this workspace, both of which survived years
of being cited:

- *"iOS never unloads a `dlopen`'d framework."* **False.** The pin is a thread-local
  variable; filetype, `.framework` packaging and `CFBundle` are all irrelevant, and roughly
  half our module images could be built unloadable. Measured 2026-09-18.
- *"dyld refuses an unsigned image on a device, so native code cannot be staged at runtime."*
  **Untested.** The spike it rests on proved a *signed, bundle-resident* framework loads.
  Nobody ever tried the thing that is supposedly forbidden.

Both were repeated back as fact by five independent analyses, because the brief they were
given asserted them. Hence this file.

## Grades

| grade | means |
|---|---|
| **IMPLEMENTED** | shipping code, plus a check that fails if the claim is false. The build or a test defends it. |
| **MEASURED** | a concrete observation — command, device, number, date. True once, on one build, not re-run. |
| **ASSERTED** | written in an ADR, README or comment with nothing behind it. Repetition is not evidence. |
| **INHERITED** | general platform knowledge with no local evidence at all. |

Four rules decide the hard cases:

1. **Proving the positive never proves the prohibition.** "A signed framework loaded fine"
   is not evidence that an unsigned one is refused.
2. **A simulator result cannot evidence a device claim.** Our simulator builds set
   `CODE_SIGNING_ALLOWED NO` — there is no AMFI there to observe.
3. **A comment citing an ADR is not evidence.** Follow the chain; if it ends in an
   assertion, the whole chain is asserted.
4. **A test that would pass even if the claim were false proves nothing.** Ask what
   observation would differ if the claim were untrue.

## Platform claims

| claim | grade | note |
|---|---|---|
| App Store 2.5.2 forbids feature-adding downloads | MEASURED | verbatim quote, source fetched 2026-09-09, four rejection precedents. True as worded — it constrains *feature-adding* downloads, not all downloads. |
| AMFI/dyld refuses any unsigned Mach-O on a device | **ASSERTED** | origin is one ADR clause; four files echo it, one citing the ADR back as evidence. Three sub-claims never separated: refused *because unsigned*, *because outside the sealed bundle*, or *not at all*. |
| Writing into the app bundle breaks the code seal | **INHERITED** | appears nowhere in this tree. It entered via a briefing document only. |
| No runtime native install, ever | **ASSERTED** | the policy half is measured; the *technical* half is not. The smoke run would pass identically if runtime install worked. |
| No fork/exec a store will accept | **INHERITED** | no guideline number anywhere; the one primary-source policy document never mentions fork, exec or spawn. Conflates a sandbox restriction with a store rule — which have opposite consequences off-store. |
| No JIT; a JIT dies at the CALL | **MEASURED** | iPad Air 4, dated, staged: engine ok → compile ok → instantiate ok → call → signal 9. An interpreter did the same work in 817 ms. |
| …"because W^X is enforced" | **INHERITED** | an explanation invented for a measured symptom. No `mprotect`/`MAP_JIT` error was ever observed. |
| Android: `dlopen` from the app data dir is a W^X violation since API 29 | **MEASURED FALSE** | A real Bare module loaded *and executed* from the app's own `files/` on a Xiaomi (Android 15, API 35, targetSdk 36), in the `untrusted_app` domain verified to match the Shell's, with `r-xp` file-backed mappings in `/proc/maps` and no AVC denial. Android 10 removed `execve()` of a file in the app home (`execute_no_trans`), **not** `mmap(PROT_EXEC)` (`execute`) — the linker path was never closed. One device; not "Android". |
| Google Play forbids downloading a `.so` | **MIS-STATED, and the conclusion is false** | The rule is source-scoped — "from a source other than **Google Play**" (quoted 2026-09-18; stable since 2021-10-15). A code comment dropped the qualifier. **Google documents `dlopen()` of a Play-delivered `.so`**: a Play Feature Delivery module's path is recovered via `ClassLoader.findLibrary()` and passed to `dlopen`. So post-install native modules are legal on Play when Play is the transport — 50 modules/device, 500 MB each, membership fixed at app release, no third-party publication. **iOS forbids the act; Play forbids the source** — they are not one constraint. |
| Android needs an explicit `DT_NEEDED` on the protocol library | **IMPLEMENTED + MEASURED** | named handset, the obvious alternative (`RTLD_GLOBAL` re-open) actually attempted and its failure recorded verbatim, plus a gate that regresses. **This is the standard the rest should meet.** |
| Exactly one protocol instance per process — **token state only** | **MEASURED**, and reproducible in <1 s | Two `TokenManager::instance()` addresses; an authorized call refused with `ModuleProxy: rejecting unauthorized call`, with an anti-vacuity control. **`StoreRegistry` splits the same way and FAILS OPEN** — isolation declared in one image is silently not in effect in another, leaving that identity holding the ambient ring. Darwin/PE; ELF collapses duplicates and is untested. |
| …stretched to the transport node | **MEASURED FALSE** | Three `QRemoteObjectHost` nodes bound the **same** URL in one process with no error; a client still connected. The factories are stateless, and `PluginRegistry` keeps its table on `QCoreApplication` properties — one per *process* regardless of code copies, i.e. duplication-immune by construction. `LogosModeConfig` is per-image but degrades rather than failing. **Our own symbol gate has always agreed**: it gates `TokenManager\|StoreRegistry\|LogosAPI\|LogosAPIClient` and no transport type. |
| A Bare module leaves `lp_*` undefined, resolved upward | **IMPLEMENTED** | gate captures `nm -u` and fails with "has no undefined `lp_*` — it is not resolving upward". |

| RFC 9266 `tls-exporter` is unreachable on mobile | **MEASURED FALSE** (API existence, verified in shipped headers) | `sec_protocol_metadata_create_secret_with_context` — iOS 12+, read from the shipped iOS 27.0 SDK header; `android.net.ssl.SSLSockets.exportKeyingMaterial` — API 31+, Conscrypt below. Secure Transport never had one (header grep). **No bundled TLS stack required.** Recorded here because this project published it as a hard blocker before checking. |
| Ed25519 call-evidence keys can be hardware-backed on a phone | **MEASURED FALSE** | Secure Enclave is P-256 only; Curve 25519 is TEE-KeyMint-2+ on Android, **not** StrongBox. ECDSA P-256 client certs *are* first-class in TLS 1.3, so **transport identity is hardware-bindable everywhere while call-evidence identity is hardware-bindable nowhere on iOS.** |

## Web-container claims

| claim | grade | note |
|---|---|---|
| The Web container is the only store-legal home for runtime-installed code on iOS | **ASSERTED, and contradicted by its own cited source** | the research it cites concludes the engine does **not** matter (the WebKit carve-out was removed June 2017), ranks downloaded QML *above* WKWebView for safety, and notes 2.5.2 is engine-agnostic. Nothing of ours has been through App Review. |
| A wasm module cannot block | **IMPLEMENTED (enforced in the negative)** | the blocking shape does not link: `wasm-ld: error: undefined symbol: lp_client_create`. Deliberate, with the reason written at the time. One Worker, one event loop, no ASYNCIFY. |
| No SharedArrayBuffer on Android WebView | **MEASURED TRUE — but the recorded reason was wrong** | Xiaomi, WebView 152.0.7977.64, 2026-09-18. Our origin was never the problem: `https://appassets.androidplatform.net` **is** a real secure context. **This WebView ignores COOP/COEP outright** — the headers demonstrably reach the renderer (`fetch(location.href).headers` shows `require-corp`), `crossOriginIsolated` stays false, and a cross-origin fetch that `require-corp` should block returns 200. A **same-device Chrome control does grant SAB**, localising the failure to the WebView embedding rather than the engine. So adding COOP/COEP to our page would buy nothing. One WebView build; "Android WebView in general" is still inference. |
| "Single-threaded only" because SharedArrayBuffer is unavailable | **TRUE on Android, WRONG REASON; and it is not a mobile constraint** | Shared memory can be *created* on Android WebView but not *shared*: `postMessage` throws `DataCloneError: SharedArrayBuffer transfer requires self.crossOriginIsolated`, identically for the `WebAssembly.Memory` wrapper and the bare buffer — **one gate, not two** — so emscripten's `-pthread` cannot start. A same-device Chrome control passed all three steps (`Atomics.wait` → `ok` after 707 ms, three contexts sharing bytes), proving the harness can produce a positive and that `crossOriginIsolated` is the only variable. **No origin or header work can change it**, and **one flag is the whole switch** — re-probe on a WebView bump rather than calling it permanent. On iOS `crossOriginIsolated` IS reachable from loopback, so this is an **Android** constraint, not a mobile one. |
| A `web` variant cannot open a WebSocket | **MEASURED FALSE** | From a dedicated Worker on the shipped Android origin with no COOP/COEP: `wss://echo.websocket.org` binary echo in 319 ms, local `ws://` in 38 ms, `typeof WebSocket` = `function`. External `fetch` also reaches an arbitrary https host subject to CORS (api.github.com, 200, 190 KB). Plain browser APIs — emscripten's `-lwebsocket.js` binding is still untested. |
| iOS reaches `crossOriginIsolated` only via an in-app loopback listener | **MEASURED**, but the framing was wrong | device and simulator. **SharedArrayBuffer is available on iOS** — WebKit enabled it in Safari 15.2 ("Enabled SharedArrayBuffer support when COOP/COEP headers are used", STP 133, 2021-09-30). What denies it is **our own `logos://` origin**, not the platform: identical COOP/COEP headers gave `crossOriginIsolated=false` from `WKURLSchemeHandler` and `true` from `http://127.0.0.1` on the same device. This was repeatedly restated in this project as "no SharedArrayBuffer on iOS", which is false. |
| `file://` is dead on both platforms | **MEASURED** | verbatim console output, two physical devices. A real negative, properly recorded. |
| Qt's network layer refuses a custom scheme | **MEASURED**, mechanism named | QNAM's wasm backend serves `http`/`https`/relative only. **Nothing in CI watches this** — a Qt bump could quietly make the loopback listener unnecessary. |
| WebGL required, no software fallback | **ASSERTED** | vendor documentation plus three *positive* observations. Nobody here has seen Qt-wasm without WebGL. |
| Artifact size band (26.3 MB / 6.8 MB brotli) | **IMPLEMENTED** | the probe exits 1 outside the band. The one number a build defends. |
| Cold start 2.6–3.0 s; renderer 185–240 MB | **MEASURED but STALE** | from a different binary than ships today; the shipping code's own figure is 290 MB. |
| Only one QML runtime alive at a time | **SUPERSEDED BY THE CODE** | a *budget*, not a limit: 3 by default, derived from device RAM, overridable per run, least-recently-**visible** eviction. The count was 1 once and that was fixed as a defect. Every installed module's Wasm host stays alive; only the UI page is budgeted. |
| The bridge is a custom scheme with fetch/long-poll | **STALE** | the shipped bridge percent-encodes frames into a chunked **GET query string**, because neither webview hands a request body to its interceptor. |
| The bridge relays and attributes but never authorizes — *attributes* | **IMPLEMENTED** | a real negative test: posting with another launch's token asserts 403. |
| …*never authorizes* | **ASSERTED** | the test would pass identically if the bridge authorized on its own. |
| Bundled closure breaks on a dependency with no native variant | **FALSE AS WORDED** | since ADR 0010 a dependency shipped as the image's `web` half is satisfied; the closure refuses only names absent from **both**. |
| `platform: true` modules have no `web` variant | **ASSERTED** | the ADR concedes its own counterexample; the pinned builder publishes `web` unconditionally. |
| …and are always Bundled | **IMPLEMENTED** | the flag is read twice and the build fails when the two readings disagree. |

## Proven impossible on mobile

The short list that may be stated as fact. Everything here rests on a measurement on named
hardware, a build that refuses, or quoted current policy text.

1. **A wasm module cannot block on an outbound call.** The blocking shape does not link:
   `wasm-ld: error: undefined symbol: lp_client_create`. Deliberate, with the reason recorded
   at the time. Structural cause: one Worker, one event loop, no ASYNCIFY — a call that
   blocked for its reply would deadlock the loop that must deliver it.
2. **`file://` cannot load a wasm image on either platform.** iPhone 16 Pro simulator,
   physical iPad Air 4, physical Samsung SM-G990B, console output preserved. iOS: `fetch`
   resolves status 0, no MIME, both paths abort. Android: `URL scheme "file" is not supported`.
3. **Qt's network layer cannot fetch over a custom URL scheme.** `Loader.source = "qtapp://…"`
   → `Network error`, because QNAM's wasm backend serves `http`/`https`/relative only.
   Decisive because the same scheme serves the 26.3 MB image in 7–13 ms — the refusal is Qt's
   client, not the scheme. **Write this as "true of Qt 6.11", not "true of mobile"**: nothing
   watches it, and a Qt bump could change it.
4. **`WKScriptMessageHandler` traps under Qt's iOS entry point.** Root cause traced into Qt's
   own source (the iOS event dispatcher lowers `RLIMIT_STACK` to the separate main stack's
   size), two named workarounds tried and rejected, and independently corroborated by a second
   instance of the same cause — `evaluateJavaScript` with a completion handler also SIGTRAPs.
   *Honesty note:* the spike's measurement table is headed "(simulator, Debug host)" and the
   trap section never names its target, so device confirmation is inferred from the mechanism
   rather than recorded.
5. **Neither mobile webview hands a request body to its interceptor.** `WKURLSchemeHandler`
   sees `HTTPBody` and `HTTPBodyStream` both nil for a `fetch` POST; Android's
   `WebResourceRequest` has no body accessor at all. Hence frames are percent-encoded and
   chunked into a GET query string.
6. **`SharedArrayBuffer` is unavailable under a custom scheme on iOS.** COOP/COEP on
   `WKURLSchemeHandler` responses do not produce `crossOriginIsolated`; the identical headers
   from `http://127.0.0.1` do. Simulator and physical iPad Air 4. The paired positive is what
   makes the negative credible — same page, same headers, two transports, two answers.
7. **Native code cannot be downloaded and executed on either store.** Quoted current text:
   DPLA 3.3.1(B) — "an Application may not download or install executable code"; Play's Device
   and Network Abuse policy — "An app may not download executable code (such as dex, JAR, .so
   files) from a source other than Google Play", whose carve-out is explicitly for "code that
   runs in a virtual machine or an interpreter … (such as JavaScript in a webview)".
   **This constrains *native* code only**, and it is the one policy claim resting on quoted
   current text rather than interpretation.

### A distinction item 7 forces

"Downloading native code is forbidden" is proven. "Therefore wasm-in-a-webview is the only
legal home for downloaded code" does **not** follow — the interpreter carve-out is
engine-agnostic, so downloaded QML sits in the same exemption. The two were conflated, and
only the first is evidenced.

It also bounds what a staged-load experiment can buy: even if the loader permits it, 3.3.1(B)
still forbids it for store distribution. Such an experiment settles the **OS layer** and the
**off-store channels** — enterprise, sideload, alternative marketplaces — not store legality.

## Assumed impossible — the open questions

Ordered by how much changes if the assumption turns out false. A claim is here because we
cannot point at a measurement, a primary source, or a build that refuses.

1. **"The Web container is the only store-legal home for runtime-installed code on iOS."**
   *If false: the Bundled/Downloaded split, the whole wasm UI stack and the 26 MB runtime
   answer a constraint that does not exist.* Contradicted by the document it cites — the
   interpreter carve-out is engine-agnostic, so downloaded QML sits in the same exemption.
   What **is** proven is narrower: downloading *native* code is forbidden on both stores.
2. **"Only one QML runtime may be alive at a time."** *If false: the eviction machinery and
   the cold start paid on every app switch are unnecessary on capable devices.* Already
   measured false on Android — three live runtimes on two phones, all sharing one Chromium
   renderer at ~8 MB per page after the first (290 → 298 → 300 MB on a Xiaomi). **iOS is
   unmeasured and is the half that decides**, since each `WKWebView` is its own WebContent
   process.
3. **"One webview per Downloaded module gives per-module isolation and crash blast radius."**
   *If false: the isolation story the container is built on does not hold.* Measured false on
   Android: one renderer for all pages, and when it was reaped **both pages died at once**.
   Never measured on iOS, where per-process isolation may be real.
4. **"Raw TCP/UDP and libp2p cannot exist in a Web variant, so everything opening sockets must
   be Bundled."** *If false: the Platform-module floor — which grows every Store shell and
   which every Downloaded module builds on — is larger than it needs to be.* The load-bearing
   premise of ADR 0007's networking rule and ADR 0009's `platform: true` mechanism, and it is
   general knowledge with no local test, no primary source and no build that refuses. Browser
   transports are named as the alternative but nothing records what was tried.
5. **"`platform: true` modules have no web variant."** *If false — and it is currently simply
   unenforced — a Platform module can ship a `web` variant that dies on the phone.* The rule
   lives only in prose; the pinned builder publishes `web` unconditionally and the ADR concedes
   its own live counterexample.
6. **"`SharedArrayBuffer` is NEVER available on Android WebView."** *If false: threads become
   available and "single-threaded only" stops being a platform fact and becomes a build
   choice.* One device, one WebView build, plus someone else's unfixed upstream ticket. A
   universal resting on a single negative observation.
7. **"WebGL is required, with no software fallback."** *If false in the direction that matters
   — a device in the install base without WebGL2 — every Downloaded module is dark there with
   no diagnostic.* Vendor documentation plus absence of evidence; every local observation is
   positive, which says nothing about what Qt does without it.

## Claims this effort got wrong about itself

Corrections made *after* the ledger existed, recorded because the pattern matters more than
the items:

- **"SharedArrayBuffer is unavailable on iOS."** Restated repeatedly in this effort as a
  platform limit. It is available — WebKit enabled it in Safari 15.2 given COOP/COEP — and
  what denies it is **our own `logos://` origin**. Our own measurement showed the identical
  headers yielding `crossOriginIsolated=false` from the custom scheme and `true` from
  loopback, on the same device.
- **"RFC 9266's `tls-exporter` is a mobile blocker requiring a bundled TLS stack."** Published
  in a resolution as the single hard blocker for remote modules. It is reachable through
  platform TLS on both phones: `sec_protocol_metadata_create_secret` (iOS 12+, verified in the
  shipped SDK header) and `android.net.ssl.SSLSockets.exportKeyingMaterial` (API 31+).
- **"A `web` variant can do WebSocket networking today."** True on documented emscripten APIs,
  but published with a confidence the evidence did not carry — nothing had been run in our
  container. Downgraded to asserted-with-a-good-source pending measurement.

All three were made by the author of this ledger, after writing it. The rubric catches what
it is applied to; it does not apply itself. What actually caught these was someone asking
"says who?" — twice a human, once a subagent instructed to grade its own claims.

## The bias worth noticing

Every defective claim in this ledger errs in the same direction: it makes mobile look **more
constrained than it is**. That is the bias that produces a bad specification proposal — one
that asks authors to carve out limitations we do not actually have, and that forecloses
product options nobody decided to give up.

## Maintaining this

A claim moves up a grade only when someone produces the artefact that justifies it: a
measurement with a device and a date, or a check that fails when the claim is false. Nothing
moves up because it has been repeated, and nothing moves up because it is probably true.
