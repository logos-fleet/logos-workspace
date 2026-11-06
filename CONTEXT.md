# Logos Workspace

The aggregation point for the Logos platform repos. This context covers how the platform is assembled, built and delivered across desktop and mobile targets; the individual repos own their own domain language.

## Language

### Basecamp

**Basecamp**:
The Logos desktop application: a shell that hosts UI apps and manages installed modules.
_Avoid_: Logos App, the app

**Shell**:
Basecamp's own user interface (sidebar, App Manager, Settings). Built as one plugin that knows nothing about the Logos runtime.
_Avoid_: main UI, main_ui, the frontend

**Shell host**:
The side of Basecamp that owns the window and the runtime and hands the Shell its data. There is one per delivery target; the Shell does not change between them.
_Avoid_: host app, container

**UI app**:
A separately delivered application (e.g. Package Manager) that Basecamp loads into its workspace. Distinct from the Shell.
_Avoid_: UI plugin, plugin app

### Mock tiers

**Fixture**:
A static data set that stands in for a live Logos backend so the Shell can render without one.
_Avoid_: mock data, stub data

**Shell preview**:
The Shell running against a Fixture with no Logos code present at all. No UI apps load. The designated first thing to bring up on a new platform.
_Avoid_: UI-only preview, mock UI

**Mock backend**:
Basecamp running with the full Logos plugin machinery but with module calls answered from a Fixture. UI apps load for real. Not the mobile starting point.
_Avoid_: mocked basecamp (ambiguous between the two tiers)

### Mobile

**Mobile bring-up milestone**:
Shell preview running on an Android device and an iOS device from one workspace command. Milestone 1 of the mobile track; precedes porting the Logos runtime to mobile and precedes Status integration.

### Mobile module hosting

**Bundled module**:
A module compiled into the mobile app at build time and shipped with it through the store. The set is fixed per app release. First-party modules on mobile are Bundled.
_Avoid_: built-in module, native module (says how it is built, not how it is delivered)

**Downloaded module**:
A module the user installs at runtime from the catalog, after the app was shipped. On iOS it may only run inside a Web container.
_Avoid_: third-party module (ownership is not the distinction; delivery is)

**Native container**:
The in-process container that runs Bundled modules on supervised threads in the app's address space.
_Avoid_: in-proc loader

**Web container**:
The container that runs Downloaded modules inside a platform webview (WKWebView on iOS). The only store-legal home for runtime-downloaded code on iOS.
_Avoid_: wasm container (wasm is one way to fill it, not the container itself)


**Web variant**:
The `web` LGX variant of a module: the artifact the Web container runs. Required for a module to be listed on iOS.
_Avoid_: wasm build, browser build

**Wasm host**:
The Web container's counterpart of `logos_host` / `ui-host`: one module artifact (plugin shell, Rust core, QtCore) compiled to WebAssembly and run in a Web Worker, remoted to the QML runtime and to the core over the container's transports.
_Avoid_: wasm module (ambiguous with the artifact), worker

**Web bridge**:
The "web" transport: the plain protocol's messages carried over a webview's postMessage channel between a Wasm host and the core. Relays and attributes frames; never authorizes.
_Avoid_: JS bridge, native bridge, RPC shim

**Bare module**:
The protocol-free module artifact: impl plus core, lp_* symbols left undefined for the host image to supply. The build shape shared by the iOS embedded framework and the Wasm host.
_Avoid_: cdylib (names an interface kind, not the artifact), framework build

**Store shell**:
A Basecamp (or product) build distributed through the App Store or Google Play. Runs Bundled native modules and installs `web` variants only.
_Avoid_: mobile build (a sideloaded Android build is mobile but not a Store shell), release build

**Bundled set**:
The modules a Store shell embeds at build time, pulled as signed native variants from the catalog. Closed under dependencies.
_Avoid_: built-in modules, first-party set (ownership is not the criterion)

### Mobile phases

**Library phase**:
The mobile work done while the core is frozen. It proves the libraries behind modules on devices, through the libraries' own C API, with no module wrapper and no core.
_Avoid_: phase 1 (the numbering changes), mobile POC

**Core phase**:
The mobile work that begins once the new core unfreezes: the core and selected modules running on a device, then wasm modules.

**Frozen repo**:
A repository the new core is replacing. During the library phase nothing is delivered to it.
_Avoid_: core (too narrow: the freeze also covers the protocol, the loader, the host and the package manager)

**Caller**:
Whoever builds a library and decides how: target, linkage, sealing. On mobile in the library phase the caller is logos-nix. The library itself states only what it is.
_Avoid_: consumer, integrator

**Smoke app**:
A minimal app that loads one or more libraries on a device and asserts that they do real work. It is a proof vehicle, never a product.
_Avoid_: harness, test app, demo

**Remote module**:
A module that runs in full on a liblogos the phone does not host, reached through the core. It needs no special mode in its library.
_Avoid_: client mode, light module, remote core (the core still runs on the phone)
