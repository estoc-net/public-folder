# public-folder specification

Status: **Draft** — version 1.0.

The key words MUST, MUST NOT, SHOULD, MAY are to be interpreted as in RFC
2119.

## 1. Terminology

- **Owner** — the DID controller whose folder is being served.
- **Folder** — a set of plain files the owner wants publicly readable.
- **Tree** — the merkle encoding of the folder (§2). "Root" is its top
  CID.
- **Root card** (or just **card**) — the owner's one signature over the
  tree (§3). A card naming no tree is a **takedown card** (§3.1).
- **Relay** — a server that holds cards and objects and answers for
  owners (§4). Typically the owner's DIDComm mediator.
- **Reader** — anyone fetching the folder; needs no relationship with the
  owner or the relay.
- **Object** — a block addressed by CID: either a file's bytes or an
  encoded directory node.

## 2. Directory tree

The tree uses IPLD notation. CIDs are CIDv1 with a sha2-256 multihash;
their text form is lowercase base32 (the CIDv1 default).

### 2.1 File objects

A file object is the file's bytes, unwrapped and unframed. Its CID uses
the **raw** codec (0x55): the CID names the sha-256 of exactly the bytes
a reader will receive — aligned with `crypto.subtle.digest`, R2
checksums, and SRI, with no git-style `blob <len>\0` header.

### 2.2 Directory objects

A directory object is a **dag-json** (0x0129) document:

```json
{
  "entries": [
    { "hash": { "/": "bafkrei…" }, "name": "a.txt", "size": 17, "type": "file" },
    { "hash": { "/": "baguqee…" }, "name": "posts", "size": 5120, "type": "dir" }
  ]
}
```

- `entries` MUST be sorted by `name` in UTF-8 byte order.
- `type` is `"file"` or `"dir"`; `hash` is a dag-json link to the entry's
  object.
- `size` is the file's byte length, or for a directory the recursive sum
  of its files' byte lengths.
- dag-json's canonical form (sorted map keys, canonical link encoding) is
  what gets hashed; the directory's CID uses the dag-json codec.

The **root** is the CID of the top directory object. This is the value
the card's `root` field carries.

### 2.3 Constraints

- Empty directories do not exist (as in git): a directory is present only
  by virtue of its entries.
- Within one directory, duplicate names are invalid; a name cannot be
  both a file and a directory.
- Path segments `.` and `..` are invalid.

### 2.4 Properties

Verifying a single file is O(depth): walk the path from the root, one
directory object per hop, and check the leaf bytes against the leaf CID —
the same proof-chain shape as an IPFS trustless gateway (IPIP-402), which
is deliberate: **CAR** (`application/vnd.ipld.car`) is the packing format
when a card plus its proof chain travel as one stream. Unchanged subtrees
keep their CIDs across versions, so pushes are incremental and storage
dedupes for free.

CIDs are adopted as a *format* only. There is no DHT, no IPNS, no
pinning, no UnixFS chunking, and no reliance on public gateways.
Addressing is always DID + path; the CID is an integrity check and an
object-exchange name, not a discovery mechanism.

## 3. Root card

### 3.1 Payload

```json
{
  "did": "did:peer:2:…",
  "id": "0198c5b2-7c3a-7d21-9f6a-2f4e8a1b0c3d",
  "expires": "2026-09-18T00:00:00Z",
  "root": "baguqee…"
}
```

`did`, `id`, and `expires` are REQUIRED; `root` is OPTIONAL.

- `did` — the owner. Signing the root alone would allow replaying one
  owner's tree under another's name; the card binds them.
- `id` — an opaque version label chosen by the owner. The protocol
  attaches no meaning to its contents and never orders ids; readers and
  relays use it for equality only (naming a card, noticing it changed).
  Estoc vaults happen to mint UUIDv7s, whose ids sort by time — that is
  the author's own convention, invisible to the protocol.
- `expires` — RFC 3339 timestamp; the DNS-TTL equivalent. A card past
  `expires` is *stale*, which bounds how long a withholding relay can
  keep serving yesterday's truth.
- `root` — the tree's root CID (§2.2). A card without `root` is a
  **takedown card**: the owner's signed statement that this DID
  currently publishes nothing. Taking a folder down is therefore just
  another publish, not a separate verb — which keeps the §4.4 invariant
  whole: every state the relay serves, the gone state included, traces
  to an owner signature. An unsigned "delete" message would have made
  "gone" the one state a reader could not tell from relay withholding.
  `expires` keeps its meaning: it bounds how long the relay can keep
  asserting the owner's takedown as fresh.

A `prev` field (hash-chaining to the previous card, KERI-style) is
deliberately absent from 1.0; it is an additive field and may appear in a
later version if auditable card history grows a real consumer. It is
also the designated home for everything `id` deliberately does not do:
detecting rollback across observations and ordering cards between
relays.

### 3.2 Signature

The card is a compact JWS: `alg: "EdDSA"` (Ed25519), `kid` set to a
verification-method id in the owner's DID document (`<did>#<key>`). The
`did` in the payload and the DID in `kid` MUST match.

### 3.3 Verification and freshness

Signature verification proves exactly *who signed what*, nothing more.
Whether a card is fresh enough — `expires` in the future — is **reader
policy**, evaluated by whoever holds the card.

Which card a relay serves is decided by the publish channel, not by any
card field: publishing is authenticated (§4.2), and **the most recent
authenticated publish wins**. The relay does not interpret `id` — the
same restraint it applies to the tree. A captured old publish envelope
replayed to the relay can therefore, at worst, reinstate an older card
until the owner next publishes or renews — which is exactly the
withholding/staleness power the trust model already concedes to the
relay, bounded by `expires` and self-healing on the owner's next
activity. A relay MAY harden against replay at the transport layer by
deduplicating DIDComm message ids within a bounded window — a defence
that reads no card field. Detecting rollback across observations, and
ordering cards at all, is the job of the deferred `prev` chain (§3.1),
not of `id`.

## 4. The relay

### 4.1 Four verbs

The relay protocol is exactly four verbs:

1. **Verify a card and accept a push** — check the JWS against the
   owner's DID document, apply local publish policy.
2. **Fill in objects by CID** — after accepting a card, request whatever
   objects it is missing; hash each received object to check it is the
   CID it claims. (This is structurally a git push: the relay is a bare
   remote for the folder.)
3. **Serve objects as-is** — by CID, with no interpretation.
4. **Follow `_redirects`** — the one convenience: on the browse domain
   the relay MAY apply the tree's `_redirects` file (see
   [conventions](conventions.md)). Trust never depends on this; a reader
   with the three pieces can check any redirect themselves.

The relay MUST NOT interpret the tree beyond verb 4. New filename
conventions are client-side and require no relay change.

### 4.2 Publishing

Publishing rides DIDComm: the owner sends `publish { card }` to the
relay's own DID, the relay answers with the CIDs it is missing, the owner
supplies them; the card MUST NOT become the served version until every
object under its root is present, and completion is announced with a
`published` receipt the owner can keep. Wire details are in
[didcomm/public-folder/1.0](didcomm/public-folder/1.0/readme.md).

Who may publish is local policy, out of scope for the wire protocol. The
intended deployment is relay-inside-mediator, where the policy is one
line: *a mediation relationship grants publish rights* — the
DIDComm-authenticated sender is an existing mediation client, and the
card's `did` is one of that client's DIDs.

Publishing a takedown card (§3.1) is the same exchange collapsed:
nothing can be missing under an absent root, so a well-formed takedown
is answered with `published` immediately and the previous version stops
being served.

A relay MAY refuse a publish at any round of the exchange (over-size,
quota, other policy), reported as a problem-report. Refusal is cheap by
construction: directory nodes carry recursive sizes (§2.2), so the root
node — the first object pushed — already states the publication's total
size, and a size limit is enforced before any content bytes travel.

### 4.3 Storage and retention

The relay is a cache of a projection, not an archive: the owner's vault
remains the source of truth, so garbage-collecting an owner's data loses
nothing that a single republish cannot restore (unchanged subtrees keep
their CIDs, so even the bytes rarely travel again).

Retention is therefore a **lease**, and its unit is a completed
**publication**: the current card plus every object reachable from its
root, as one indivisible closure. When a publish completes, the relay
MAY declare how long it commits to keeping that closure (`retain_until`
in the `published` receipt); re-publishing the current card — `publish`
is idempotent — renews the lease. A promise is only ever made about a
completed publication: an in-flight publish gets `missing` lists, not
retention promises.

While a lease lives, the closure MUST stay whole: the relay cannot keep
the lease yet quietly reclaim some large file inside it — a tree with
holes is worse than an absent one, because a reader cannot tell loss
from selective withholding. Past the lease, the relay MAY drop the card
and reclaim storage.

The object store beneath is derived state, never a promise of its own:
objects are content-addressed and shared — across versions, and across
owners — so an object lives exactly as long as some live lease's
closure references it (refcounting, not policy). Two consequences:

- Publishing a new card ends the old version's protection immediately —
  the relay serves only the current version and is not an archive.
  Keeping newly-unreferenced objects anyway is a useful cache courtesy
  (the next publish's `missing` list shrinks), never an obligation.
- Objects staged for a publish that never completed carry no promise at
  all, though a relay SHOULD hold them long enough for a multi-message
  publish to finish.

The lease is the relay's declaration, distinct in every way from the
card's `expires`: `expires` is the *owner* telling *readers* when
content goes stale, signed into the card; the lease is the *relay*
telling the *owner* when storage may be reclaimed, unsigned and
invisible to readers. Lease lengths SHOULD be far longer than typical
`expires` values, so a folder spends a long time merely stale before it
disappears.

Two freedoms are left to local policy. A relay MAY extend the lease on
any authenticated activity from the owner (in the relay-inside-mediator
deployment, an ordinary pickup connection is signed activity — normally
active owners then never refresh explicitly; a stored `retain_until` is
then merely a lower bound). And once a lease has lapsed — never inside
one — a relay MAY retain cards longer than objects (cards are a few
hundred bytes; `card_only` queries and moved-away notices survive after
heavy blobs are reclaimed).

A relay that no longer wants to store a publication has one graceful
exit: **stop renewing** — shorten or omit `retain_until` on the next
`published` and let the lease lapse. There is deliberately no message
for breaking a live lease: the lease is a commitment, and removal under
force majeure (a legal takedown, a dying disk) is a broken promise the
protocol does not dress up — readers simply see the folder gone, which
is the honest signal.

The **owner's** exit is symmetric and immediate: publish a takedown
card (§3.1). As with any publish, the previous version's protection
ends at once — the old closure's objects lose their references and MAY
be reclaimed, which is exactly what a withdrawing owner wants. The
takedown card is itself a completed publication whose closure is just
the card, leased and renewed like any other (the card-only retention
case in miniature). When the owner stops renewing even that, the relay
drops the card and the DID converges to unknown — being forgotten needs
no verb of its own.

### 4.4 Invariants

- The relay never holds owner keys and never signs anything on an
  owner's behalf. Everything it serves *about* an owner traces to the
  owner's signed card.
- Reading the published tree requires no authentication, ever.
- The served version is the most recent authenticated publish; the
  relay never orders or interprets card ids.
- On its machine hostname the relay speaks machine formats only and
  never renders `text/html` (at most `Content-Disposition: attachment`);
  HTML rendering happens only on the isolated browse domain (§5.2).

## 5. HTTP read

### 5.1 Trustless endpoints (provisional)

On the relay's machine hostname:

- `GET /objects/<cid>` — the object's bytes.
  `application/vnd.ipld.raw` for files, `application/vnd.ipld.dag-json`
  for directory nodes. Immutable; cache forever.
- `GET /card/<did>` — the owner's current card as a compact JWS,
  `application/jose`. The DID is percent-encoded.
- A relay SHOULD also offer proof-chain reads in CAR form
  (`?format=car`, IPIP-402 semantics): given a root and a path, one
  response carrying the card, the directory nodes along the path, and
  the target file. The exact CAR subset is an open question (§7).

DID dereference to a tree root is a **303**, per the W3C DID Resolution
HTTP binding — the relay knows DID → bucket from the publish
relationship and needs to read no file to redirect. When the current
card is a takedown (§3.1) there is no root to redirect to: dereference
answers **410 Gone**, while `/card/<did>` still serves the signed
takedown so the gone state stays verifiable.

### 5.2 Browse domain

Rendering for browsers lives on a **separate apex domain** shared with
nothing else, one subdomain per DID — the IPFS subdomain-gateway lesson
(a shared path gateway makes every site share cookies, storage, and
service workers).

- Subdomain label: `base32(sha256(did))`, lowercase, no padding — 52
  characters, safely under the 63-character DNS label limit that raw
  DIDs exceed. The relay records label → DID at publish time.
- Path-form URLs 301-canonicalize to the subdomain form.
- A DID whose current card is a takedown answers **410 Gone** across
  its subdomain.
- The apex SHOULD be submitted to the Public Suffix List so each
  subdomain is its own site.

## 6. What this protocol never does

- **No authorization on reads, ever.** The query side is semantically
  public; the answer's authority is the owner's signature, so who is
  asking cannot matter. Private sharing is ordinary DIDComm messaging,
  not a mode of this protocol.
- **No query language.** A query is a path — no filters, predicates, or
  pagination, in any version. Rich queries are the owner's job at write
  time: pre-generate projection files (`posts/index.json`, `feed.xml`)
  into the tree, signed like everything else.
- **No per-file signatures** — the root card is the one signature.
- **No relay-side rules or delegation.** The relay only replays
  pre-signed answers (type A). Executing owner rules (type B) and
  delegated signing via UCAN/ZCAP (type C) are explicitly out of scope.
- **No IPFS infrastructure** (§2.4) and no fork of DWN — DWN's
  published-record shape, protocol-definition idea, anonymous reads,
  and replay-based sync are acknowledged prior art, borrowed as ideas
  only.

## 7. Open questions

- Batch paths in one query, and the degradation rule when a proof chain
  exceeds message-size limits (today: attachments switch from inline
  bytes to links).
- The exact CAR subset a relay serves (full IPIP-402 vs. minimal
  card+proof CAR).
- When to add `prev` to the card (§3.1) — now also the designated home
  for rollback detection and cross-relay card ordering, which `id`
  deliberately does not provide.
- How an owner's DID document advertises the relay (existing
  `serviceEndpoint` vs. a dedicated `#content` service).
- Rate limiting for anonymous reads — shared with the mediator's
  anonymous-forward problem, solved there.
- Concrete lease and `expires` magnitudes (§4.3) — the intended shape is
  expires on the order of weeks, leases on the order of a year, but real
  numbers await deployment experience.
