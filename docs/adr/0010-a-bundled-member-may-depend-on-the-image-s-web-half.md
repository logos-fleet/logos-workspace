# A Bundled member may depend on the app image's `web` half, and the two halves are resolved together

ADR 0007's first consistency rule is "a Bundled module may depend only on
Bundled modules", because "core auto-loads dependencies and a build-time set
cannot reach a runtime one". The premise held when the only thing an app image
carried at build time was the Bundled set. It no longer does: a Store shell also
ships `web` modules **inside the image** — `nix/mobile-web-assets.nix` lays them
out under `web-modules/<name>/` exactly as `lgpm install` would, and the core
discovers them at startup. Those modules are as fixed at build time as a
Bundled member is; nothing about them is "a runtime one".

Resolving the two halves separately made that invisible, and the phone's
keystore is where it bit. `keystore_module` reaches a device as a `web` variant
with an `idbfs` vault (logos-workspace#147), and two native modules name it in
`dependencies` — `wallet_backend_module` and `railgun_module`. The Bundled-set
closure reads the catalog index and only the catalog index, so it refused both
by name for a module that was running on the device three feet away.

So ADR 0007's rule is restated as: **a Bundled module may depend only on modules
this image carries**, Bundled *or* `web`. `nix/bundled-set.nix`'s `resolveSet`
takes the `web` module names this build ships; a dependency found there is
satisfied and is *not* a member — nothing about it is fetched, verified or
embedded by the Bundled-set stage, because the web half is a different stage of
the same image. The names the closure leaned on are recorded in
`bundled-set.json` as `webSatisfied`, and the web-assets stage then ships them
whether or not `LOGOS_SHELL_WEB_MODULES` names them. One resolution, one image.

What makes the dependency real at run time is ADR 0005. A page cannot be
registered on the host's provider registry, so the Web container registers a
`WebModuleGlue` instead: an ordinary `LogosProviderObject` that relays to the
page. Every consumer — another module included — reaches a `web` module exactly
as it reaches a subprocess one, and nothing above the container learns that the
module is JavaScript. A native Bundled member calling the `web` keystore is an
ordinary module-to-module call, not a new mechanism.

## Consequences

- **The keystore stays one module.** The alternative that looked obvious — give
  `keystore_module` a Bare mobile build and a catalog entry — puts two vaults in
  one app under one name: a filesystem one and the `idbfs` one #147 proved a key
  survives a page reload in. Which of them holds the user's keys is a question
  nobody had an answer to, and this decision means nobody has to invent one.
- **A shell's two halves are now coupled at eval.** Dropping a `web` module from
  a build's assets breaks the closure of any Bundled member that depends on it,
  by name, at evaluation — which is the intended coupling and the reason the
  refusal text now prints what the web half ships beside what the catalog holds.
- **The `web` member's own dependencies are not checked here.** They are not in
  the catalog index, and the core resolves them on the device from the installed
  layout. ADR 0007's guarantee is unchanged for everything the Bundled-set stage
  places in the image — a Bundled member is still never filled in later, and a
  `web` module in the assets is not filled in later either — but the half the
  *core* resolves is not walked by the *builder*.
- **A name in both halves is a Bundled member.** The loader brings that one up;
  a rule that preferred the page would silently move a dependency.
- **`--bundle` still cannot name a `web` module.** `--bundle` names members of
  the Bundled set, and a `web` module has no variant to embed.
- **The closure sees `web` MODULES, not the builder's `web` fixtures.**
  `web_counter` / `web_counter_b` are `logos-module-builder`'s instrumented
  fixtures; no catalog member can legitimately depend on one. Keeping them out
  also keeps `packages.<ios-system>.bundled-set` evaluable on a Mac, which is the
  attribute `ws build --target ios-sim-arm64` uses from one.
- **MEASURED: native → `web` works on a handset (#196).** This was recorded as
  unverified when the decision was taken — the proven directions were `web` →
  native and `web` → `web`, and the argument for this one was architectural.
  Driven on the venue's physical iPad Air (4th generation), iOS 26.5.2, with
  `railgun_module` (native, Bare, in-process) calling the image's `web`
  `keystore_module` (a page, 1.09 MB wasm, `idbfs`) through the module's own
  `web_dependency_probe()`:

  - **It answers.** Three ordinary crossings — `caller_identity`,
    `list_accounts`, `caller_identity` — all returned: 25 ms, 1 ms, 1 ms. The
    first carries the capability handshake; a crossing costs nothing after that.
  - **The page names the caller correctly.** `kind: "module"`, `identity:
    "railgun_module"`. The `web` → `web` defect of #129 (a page answering its own
    name to every caller) does not recur in this direction. The contrast is in
    the same run: the Shell's own `--call keystore_module.caller_identity()`
    answers `kind: "host"`, so the page really is distinguishing callers.
  - **Nothing blocks.** `railgun_module` declares `concurrency: "single"`, and
    the dispatch still ran on a thread other than the one the image was loaded
    on (`dispatchLeftTheLoadThread: true`) — `BareModuleGlue`'s worker, not the
    Qt main thread the page answers on.
  - **The ordering takes care of itself.** The core loads the closure
    topologically and the Web container's `awaitLoad` waits for the page to
    serve, so the `web` dependency was up before the native member existed:
    `Module loaded: keystore_module` at 07:52:17.758, `railgun_module` at
    07:52:17.789. A Bundled member calling "too early" does not arise on this
    path.

  Not measured on Android: `railgun_module` has no working `aarch64-android`
  Bare build (`wasmer`'s `wasmi` bindgen, see #196's follow-up), which is a
  property of that crate rather than of this decision.

## Considered options

- **Give the phone a native Bare keystore and stop shipping the `web` one
  there.** One keystore again, and the `web` wallet reaches it the way it
  already reaches the Bundled `eth_rpc_module`. Rejected: it discards the
  `idbfs` store and everything #147 proved about it, and it needs the vault
  either migrated or declared fresh — a user-visible loss traded for a build-time
  convenience.
- **Declare `keystore_module` an `optional_dependency` on mobile only.**
  `platforms` overlays may vary `dependencies` and `optional_dependencies`
  (`logos-module-builder/lib/resolvePlatforms.nix`), and an optional dependency
  is not bundled, so the closure would resolve. Rejected on two counts: it is
  not true — the wallet cannot sign without a keystore, so the dependency is not
  optional — and it converts a refusal the build can name into an
  `object_unavailable` at the first call on a device. It also solves nothing for
  a dependency that is genuinely absent, which is the *other* half of #183
  (`fee_module`).
- **Publish `web` variants as rows in the mobile catalog.** Would let the
  existing closure resolve them with no new argument. Rejected: a catalog row
  carries a signed `.lgx` with per-target variants and a Merkle root the Bundled
  stage verifies and embeds; a `web` module in the image is none of those things,
  and making it look like one would put the Bundled stage in charge of bytes it
  does not lay down.

## Status

Accepted. Implemented in `logos-basecamp` `nix/bundled-set.nix` (`resolveSet`,
`webSatisfied` in `bundled-set.json`) and `flake.nix` (`mobileWebModulesFor`,
`mobileWebAssetsFor`'s `alsoShip`), with `railgun_module` as the first catalog
member whose closure it closes. See logos-workspace#183, and #196 for the
device measurement the decision was taken without.
