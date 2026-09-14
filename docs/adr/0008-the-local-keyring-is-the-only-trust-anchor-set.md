# The local `did:jwk` keyring is the only trust-anchor set; COSE is a later load-time integrity check on the same identity, not a second anchor; a repository's `trustedSigners` is advisory display data

A Store shell runs `setSignaturePolicy("require")`, and under `require` a
package is installable only when an **active anchor validates its signer**. The
anchor set is the local keyring and nothing else: `did:jwk` entries in
per-name JSON files under the app's own config root, put there by an explicit
act (`lgx keyring add` on a desktop, `--trust-signer <name>=<did>` on a phone,
`package_manager.addTrustedKey` in either). A signer DID carried in a package,
advertised by a catalog entry, listed in a repository's `trustedSigners`, or
arriving beside a downloaded key is a **self-assertion** and establishes no
anchor.

Three questions were open and are answered here.

**Format: `did:jwk` + detached Ed25519 over `manifest.json`, and COSE is not a
competing answer.** Upstream `logos-co/logos-liblogos#68` asks to "require user
approval on first run and check signature (e.g. COSE) for each execution". That
is a **different checkpoint**, not a different format for this one:

| | install-time authorisation | load-time integrity |
|---|---|---|
| question | may this publisher's code land on this device at all? | are the bytes about to be loaded still the ones that were authorised? |
| decided by | the local keyring | the on-disk package against its own manifest |
| exists today | yes — `PackageManagerLib::installPluginFile`'s gate | no |
| upstream issue | (this fork's signing work) | logos-liblogos#68 |

Adopting COSE now would change the **encoding** of a signature that already
verifies, and would answer nothing about **who anchors** — the question that
actually blocks a Store shell. It would cost a CBOR/COSE dependency across
`logos-package`, `logos-package-manager`, the module, every signed fixture and
every already-published `.lgx`, and buy no new authorisation semantics. So:
`manifest.sig` (`{version, algorithm, did, signature, signer{name,url},
linkedDids}`) stays the wire format for install-time trust. If a `manifest.cose`
envelope is later added for the load-time check, it carries **the same Ed25519
key and the same `did:jwk` identity**, re-verifying what install already
verified. One identity, one anchor set, two checkpoints.

**Who anchors on a phone: the user, eventually through the signer prompt — and
until that ships, nobody, and Downloaded modules stay unavailable.** There is no
`lgx` CLI on a device, and the three candidate answers are not equal:

- *Ship a vendor anchor in the Store build* — **rejected.** It converts the
  keyring from a user-owned anchor set into a vendor-owned one, silently
  authorising every first-party-signed package on every device; and the only
  existing candidate key, `mobile/catalog/keys/logos-catalog-test.jwk`, is a
  committed fixture whose private scalar `d` is in the repository. No shipped
  default, ever.
- *An explicit trust action in the signer prompt* — **accepted as the
  destination.** It is the only option that keeps the anchor set user-owned and
  populated by an explicit act, which is the invariant the whole gate rests on.
  It is not in milestone 1: the prompt today is Install / Cancel, and a
  tap-to-trust button on a phone with no out-of-band way to check a DID is
  trust-on-first-use with the verification removed.
- *Downloaded modules stay unavailable to a user who has anchored nothing* —
  **accepted as the milestone-1 posture, and it is not a bug.** The catalog
  browses, the package downloads, the signature verifies, the signer prompt's
  contents are computed, and `require` refuses. That refusal is the feature.

So on milestone-1 builds the only route into the keyring is
`--trust-signer <name>=<did>` on the Shell's command line, which is a
development and acceptance affordance and deliberately not a QML one. Because
that is the only route, **a refusal has to name the DID it refused** — it is the
one actionable thing in the message and a device has no keyring CLI to ask
afterwards.

**`trustedSigners`: kept, and defined as advisory display data.** It stays in
the `logos-repo.json` schema, stays parsed into `Repository::trustedSignerDids`,
and stays echoed by `listRepositoriesJson()`. It is **consulted by nothing** and
must never be: a repository that can vouch for its own packages is a repository
with no gate in front of it. Its one honest job is to supply the **candidate**
for a future "this repository publishes as X — add X to your keyring?"
affordance: a candidate, shown to a person, who acts. Never a decision and never
an input to one. It is not dropped because removing it would break a published
catalog schema for a field that has a real (non-authorising) use.

## Consequences

- `SignaturePolicy::REQUIRE` is now demonstrated to refuse, at all three layers
  rather than assumed at two: `logos-package-manager`
  (`RequirePolicyStillRefusesAnUnanchoredSignerAndNamesTheDid`),
  `logos-package-manager-module`
  (`under_require_a_signature_from_an_unknown_publisher_is_not_installable`),
  and now `logos-basecamp`
  (`underRequireASignerTheKeyringDoesNotVouchForIsRefused` and
  `aSignerNoAnchorValidatesIsRefusedAndTheDidIsStillNamed`).
- `InstallGate::refusedSigner()` / `StoreAppManager.refusedSigner` carry the
  refused signer's name and DID out of the refusal. `signerPrompt()` stays
  empty for a refusal: identity survives, the Install button does not. A
  package with no signature reports no signer, because there is nothing to
  anchor.
- A Store shell in a user's hands can browse the catalog and cannot install a
  Downloaded module. Any acceptance run that installs one must anchor inside
  the run and must also assert the refusal with an empty keyring; the fixture
  key must not become an anchor anywhere a user's device would honour it.
- `logos-package-downloader` gains regression guards that a repository's
  `trustedSigners` neither satisfies a signer pin nor adds a trust verdict to a
  resolved entry, including the case where it vouches for exactly the DID that
  signed the package.
- Desktop is unchanged: the library default stays `WARN` with
  `UnknownSignerPolicy::Lenient`, nothing under `logos-basecamp/app/` sets a
  policy or a keyring, and `~/.config/logos/trusted-keys/` need not exist. At
  `WARN` the anchor set changes the diagnostic; at `REQUIRE` it changes the
  decision. That asymmetry is the point of having two levels.
- Still open, and deliberately: the user-facing trust action in the signer
  prompt (the device keyring story's last mile), and the load-time
  per-execution integrity check of logos-liblogos#68. Neither changes the
  anchor set decided here.
