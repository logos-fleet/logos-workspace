# The library phase proves libraries, not modules

The core is frozen while it is rewritten to the new module specification, and that rewrite
changes what a module is. Today a module is a Qt plugin that wraps a library and speaks the
old core's JSON dispatch. Under `spec-module-interface.md` a module is a plain C ABI: either
CDDL-first or C-first, called in-process in direct mode. Every Qt wrapper is therefore
replaced, not ported. Meanwhile the Nim libraries already expose generated C APIs over a
CBOR wire through nim-ffi. Those APIs are close to the body of a new-architecture module.

So until the core unfreezes, mobile work stops at the library boundary. We cross-compile
the libraries and exercise them on devices through their own C API, from Qt smoke apps,
with no wrapper and no core. We build no mobile variant of a wrapper, no Bare-module build
of one, and nothing that plays the core's role (transport glue between libraries, remote
clients).

## Considered options

- **Wrappers built for mobile, link-only.** Cheap, but it proves an artifact the new core
  will not load.
- **Running modules under a core.** Needs changes to frozen repositories: the core's mobile
  cross-compile chain and the Bare-module ABI.

## Consequences

- "Delivery runs on a phone" becomes a measured claim before the new core exists. What
  remains for the core phase is the module boundary, not the library.
- Anything that needs a core waits: blockchain (remote on mobile), chat over a separately
  loaded delivery, and wasm modules (the outbound door).
- Nothing is learned about the old core on mobile. That is deliberate, because it is being
  replaced.
