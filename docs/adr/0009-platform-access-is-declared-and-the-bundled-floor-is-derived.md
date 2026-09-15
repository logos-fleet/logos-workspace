# Platform access is declared, not inferred: a `platform: true` module has no `web` variant and is always Bundled, and each shell's floor is derived from the catalog it offers

ADR 0007 decided that everything first-party that opens sockets is always in the
Bundled set and that Downloaded modules reach the network through those services
over the Web bridge. It stated that as a property of a known list — libp2p,
delivery, the chat backend — and nothing checks it: `eth_rpc_module` publishes a
`web` variant today while its network layer is `reqwest::blocking` with
`rustls-tls` and `socks`, and of the core modules with `web` outputs only
`keystore_module` has a check that drives the image, so "it links" stands in for
"it works in a Web container". A module therefore **declares** `platform: true`
in `metadata.json` when it owns access the webview cannot provide — raw TCP/UDP,
background execution, secure enclave — and the builder gives it no `web` variant,
by name at eval, in the shape `mkLogosModule` already uses to refuse a Go core or
an unported `nix.external_libraries`. The declaration is not inferred from what
the crate links: `reqwest` with the `js` feature is legitimate in wasm, so the
dependency list cannot carry intent, and a wrong inference either drops a working
variant or ships one that dies on the phone. Everything that is not a Platform
module is built on top of one and may be Downloaded.

The mirror of ADR 0007's "a Bundled module may depend only on Bundled modules" is
that a Downloaded module may depend on a Platform module only where that module is
actually bundled, and that is **derived, never listed**: at shell build time, for
every module the catalog will offer as Downloaded on that shell, each dependency
marked `platform: true` must be in the Bundled closure `nix/bundled-set.nix`
already resolves, or the catalog lists that module unavailable naming what is
missing. A hand-kept floor would be a third home for the truth, after
`metadata.json` and the shell's bundle set, and it drifts silently into the one
failure ADR 0007's honest-availability rule exists to prevent — a module offered
for install that dies at load. Guideline 4.7 makes the host liable for what it
lists, so a module never offered is better review material than one that installs
and fails.

## Consequences

- The layering becomes load-bearing on the outbound door for core `web` images
  (#162): if Platform modules are Bundled and everything else is built on top,
  the normal Downloaded module calls a Bundled one, and a leaf — a Downloaded
  module that talks to nobody — is the rare case. Until that door lands the
  catalog can host only leaf core modules plus `ui_qml` views, which have one.
- A product shell that omits a Platform module gets a catalog that does not offer
  what depends on it, rather than a broken install: omit `delivery_module` and
  `chat_module` is listed unavailable, with that as the reason.
- A shell's build now depends on the catalog's dependency graph, not only on the
  modules it bundles. A third-party module appearing in the catalog can change
  what a shell must bundle to keep offering it, which is a reason to pin a
  product shell to a curated catalog snapshot rather than to a live index.
- `platform: true` is a claim the author makes and the build trusts. It is
  checkable only in the negative — a `web` variant that fails in the container —
  so a module that owns platform access and forgets to declare it is caught by
  its own `web-variant` check or not at all, which is an argument for those
  checks existing on every module that publishes a `web` output.
- The set of Platform modules is a product surface, not an implementation
  detail: it is the floor every Store shell carries, and growing it grows every
  shell's binary.

## Considered options

- **Infer platform access from the linked crates or `nix.external_libraries`.**
  Rejected: the same crate is legitimate on both targets depending on its
  features, so the inference is wrong in both directions, and its failures are
  silent.
- **Keep ADR 0007's named list and check membership.** Rejected: it does not
  extend to third-party modules, which are the ones the rule most needs to bind,
  and the list lives nowhere the builder can read.
- **Refuse a `web` variant for any module that declares dependencies.** Rejected:
  that is the wrong predicate — having dependencies is what a Downloaded module
  built on top of a Platform module does, and the rule would have to be
  un-written the day the outbound door lands. The temporary form of that refusal
  belongs on the protocol pin's capability, not on the module.
