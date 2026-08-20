# public-folder

A public folder you sign once at the root, and anyone can serve on your
behalf.

Your agent lives on a phone and is offline most of the day, but the world
still has questions for you: what's your profile, where are your posts,
which DID did you move to. The usual fix is to hand a server your keys or
an account. This protocol's fix is smaller: you put the answers in a
folder, hash the folder into a merkle tree, sign one tiny **root card**,
and give all of it to a **relay** — typically the DIDComm mediator you
already depend on. The relay can then answer anyone, any time, yet it
cannot forge a byte: every answer traces back to your signature. The only
misbehaviour left to it is withholding, or serving stale — the same trust
you already extend to DNS, and bounded the same way, with a version and an
expiry on the card.

## The trust contract is three pieces

A reader who holds all three can trust the entire tree without
understanding a single file in it:

1. **The folder** — plain files, any layout you like.
2. **The tree** — an IPLD-notation merkle tree: files are raw CIDs,
   directories are dag-json nodes, the root is one CID.
3. **The root card** — `{did, id, expires, root}`, signed as a compact
   JWS with a key from your DID document.

CIDs are used strictly as a hash *format*, not as infrastructure: there is
no DHT, no IPNS, no pinning network. Addressing is always **DID + path**.

## Reading takes three forms

- **DIDComm query** — send `query {did, path?}` to the relay's own DID and
  get back the card plus the proof chain for that path, verify it
  yourself. Anonymous by design, and *never* gated by authorization: the
  answer's authority comes from the owner's signature, not from who asked.
  See [didcomm/public-folder/1.0](didcomm/public-folder/1.0/readme.md).
- **Trustless HTTP** — `GET /objects/<cid>`, `GET /card/<did>`; machine
  formats only, never HTML.
- **Browse domain** — a separate apex where each DID gets an isolated
  subdomain, for the rare case a browser should render the folder
  directly.

## The relay speaks four verbs

Verify a card and accept a push; fill in objects by CID; serve objects
as-is; follow `_redirects`. That is the whole protocol — the relay never
interprets the tree. What files *mean* (`_redirects`, `profile.json`,
feeds, post indexes) is a matter of client-side filename
[conventions](conventions.md) that can grow without touching the relay.

## Repo map

- [spec.md](spec.md) — the normative core: tree encoding, root card,
  relay behaviour, HTTP read, browse domain.
- [didcomm/public-folder/1.0](didcomm/public-folder/1.0/readme.md) — the
  DIDComm protocol (didcomm.org format): `query`/`answer` for reading,
  `publish`/`publish-result` for writing.
- [conventions.md](conventions.md) — non-normative filename conventions
  the protocol deliberately does not know about.

## Implementations

- Tree and card math: [`@estoc/signed-dir`](https://github.com/estoc-net/estoc/tree/main/packages/signed-dir)
  — pure functions, zero IO, runs in Node, workerd, and the browser.
- Relay: planned as a module of
  [didcomm-mediator](https://github.com/estoc-net/didcomm-mediator)
  (publish rights = mediation relationship; the mediator stays keys-only).

## Status

Draft. The design is settled; the relay and the DIDComm protocol are not
yet implemented. Field-level details marked *provisional* in the spec may
still move.

## License

Apache-2.0
