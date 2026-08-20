---
title: Public Folder
publisher: onyxblade
license: Apache-2.0
piuri: https://didcomm.org/public-folder/1.0
status: Draft
summary: Read and publish signed public folders through a relay. A reader asks a relay for an owner's folder content and receives the owner's signed root card plus a merkle proof chain, verifiable without any relationship to the owner or the relay; an owner pushes a new folder version to its relay with one signed card.
tags:
  - public-folder
  - content
  - relay
authors:
  - name: Estoc Research
    email: git@merely.ca
---

## Summary

A relay (typically a DIDComm mediator) holds, for each owner DID, a
**root card** — `{did, id, expires, root}` signed by the owner — and the
content-addressed objects of the folder tree the card's `root` names.
This protocol is how that folder is read and written over DIDComm:

- **Reading**: anyone sends `query { did, path? }` to the relay and gets
  back the card plus the proof chain for that path. The reader verifies
  the card's signature and hashes each hop; the relay is trusted only to
  be online.
- **Writing**: the owner sends `publish { card }`; the relay answers
  with the CIDs it is missing; the owner supplies them.

The folder encoding, card format, and relay invariants are specified in
the [public-folder repository](https://github.com/estoc-net/public-folder);
this document is the DIDComm wire protocol only.

## Motivation

An owner's agent is intermittently online, but questions about the owner
— profile, posts, where they moved — deserve answers at any hour. Rather
than granting a server keys or delegated signing, the owner pre-signs
the answers: the server can only replay them. DIDComm is the natural
read channel for DIDComm-native clients (no CORS, encrypted transport,
no extra endpoint discovery: the relay is reachable by its own DID), and
the natural write channel because the publish relationship can reuse an
existing mediation relationship.

## Roles

- **relay** — holds cards and objects; answers queries; accepts
  publishes.
- **reader** — asks about an owner's folder. Needs no relationship with
  anyone; MAY use an ephemeral DID.
- **owner** — publishes a folder version to its relay.

Note the addressing inversion: `query` is a message **to the relay's own
DID**. The owner's DID appears only as a body parameter. (A message
addressed to the owner's DID is encrypted to the owner's keys — a
keys-only relay could never open it, so "the relay answers mail meant
for the owner" is not a thing that can exist. The trust moves from "the
channel reaches the owner" to "the data carries the owner's
signature".) A reader therefore depends only on the relay's DID
document; it never needs to resolve the owner's DID at all.

## Requirements

Underlying protocols: DIDComm v2. Errors use
[report-problem 2.0](https://didcomm.org/report-problem/2.0).

Verification requires the reader to compute CIDs (sha2-256; raw codec
for files, dag-json for directory nodes) and verify a compact EdDSA JWS
against the owner's DID document.

## Basic walkthrough — reading

1. Reader sends `query { did, path: "profile.json" }` to the relay.
2. Relay looks up its stored card for `did`, collects the directory
   nodes along the path and the target file, and replies with `answer`:
   the card in the body, the proof chain as attachments whose `id` is
   each object's CID.
3. Reader verifies: card signature against `did`'s DID document → card's
   `root` names the first attachment → each directory node's entry names
   the next attachment → leaf bytes hash to the leaf CID. Freshness
   (`expires`) is the reader's policy call.

Queries are **anonymous and unauthorized by design**. The requester's
DID is a mailbox, not an identity — it exists so the answer can be
encrypted back, and a one-time DID is fine. A relay MUST NOT gate
`query` on authorization in any version of this protocol: the answer's
authority comes from the owner's signature, so who is asking cannot
matter. The relay's only defence is rate limiting.

A query is **exactly a path**. No filters, predicates, or pagination
will be added in any version; richer reads are pre-generated projection
files inside the tree. The mapping to HTTP intuition: `path` = GET,
`card_only` = HEAD, a `_redirects` rule = 3xx, no such path =
problem-report.

## Basic walkthrough — publishing

1. Owner sends `publish { card }`, optionally with object attachments.
2. Relay verifies the card (signature; local publish policy — in the
   relay-inside-mediator deployment, an existing mediation relationship)
   and replies `publish-result { missing: [cid, …] }`.
3. Owner re-sends `publish` with the same card and the missing objects
   as attachments; repeat until `missing` is empty. Unchanged subtrees
   are never re-sent — CIDs the relay already holds are never missing.
4. The relay serves the new card only once `missing` is empty.

`publish` is idempotent: re-sending the current card is not an error and
returns whatever is still missing.

The served version is simply **the most recent authenticated publish** —
the relay never orders or interprets card `id`s, which are opaque author
labels. Replaying a captured old publish envelope can therefore at worst
reinstate an older card until the owner's next activity — the same
staleness power the trust model already concedes to the relay, bounded
by the card's `expires`. A relay MAY deduplicate DIDComm message ids
within a bounded window as transport-level hardening.

Storage is a **lease**: `publish-result` MAY carry `retain_until`, the
relay's declaration of how long it commits to keeping the folder.
Because `publish` is idempotent, refreshing needs no extra message —
re-sending the current card renews the lease. A relay MAY also extend
the lease on any authenticated activity from the owner (local policy);
the republish is merely the guaranteed method. Note this is unrelated to
the card's `expires`, which is the owner's freshness declaration to
readers.

## Message reference

### query

Sent by *reader* to the relay's own DID.

```json
{
  "type": "https://didcomm.org/public-folder/1.0/query",
  "id": "1f4b…",
  "body": {
    "did": "did:peer:2:…",
    "path": "posts/2026/hello.md",
    "card_only": false
  }
}
```

- `did` (REQUIRED) — the owner being asked about.
- `path` (OPTIONAL) — slash-separated, relative to the tree root. Omitted:
  the answer carries the card and the root directory object.
- `card_only` (OPTIONAL, default `false`) — `true`: the answer carries
  the card and no attachments.

### answer

Sent by *relay*, `thid` = the query's `id`.

```json
{
  "type": "https://didcomm.org/public-folder/1.0/answer",
  "id": "9c2e…",
  "thid": "1f4b…",
  "body": {
    "card": "eyJhbGciOiJFZERTQSIsImtpZCI6…"
  },
  "attachments": [
    {
      "id": "baguqeera…",
      "media_type": "application/vnd.ipld.dag-json",
      "data": { "base64": "…" }
    },
    {
      "id": "bafkreib…",
      "media_type": "application/vnd.ipld.raw",
      "data": { "links": ["https://relay.example/objects/bafkreib…"] }
    }
  ]
}
```

- `body.card` — the owner's root card, compact JWS, always inline.
- `attachments` — the proof chain in root-to-leaf order: the directory
  nodes along `path`, then the target object. **The attachment `id` is
  the object's CID** — DIDComm attachments having an `id` slot makes
  content addressing native to the envelope.
- Each attachment carries either inline bytes (`data.base64`) or
  `data.links` pointing at the relay's trustless HTTP endpoint
  (`/objects/<cid>`). Large files SHOULD go by links: the message
  carries the trust skeleton, bytes travel over HTTP and are checked
  against the CID on arrival. When a whole answer would exceed transport
  limits, the relay degrades attachment by attachment from inline to
  links; the card is never elided.

### publish

Sent by *owner* to the relay's own DID.

```json
{
  "type": "https://didcomm.org/public-folder/1.0/publish",
  "id": "77aa…",
  "body": {
    "card": "eyJhbGciOiJFZERTQSIsImtpZCI6…"
  },
  "attachments": [
    { "id": "bafkreic…", "media_type": "application/vnd.ipld.raw", "data": { "base64": "…" } }
  ]
}
```

- `body.card` (REQUIRED) — the new root card.
- `attachments` (OPTIONAL) — objects, `id` = CID. The relay MUST hash
  every received object and discard any whose bytes do not match its
  claimed CID.

### publish-result

Sent by *relay*, `thid` = the publish's `id`.

```json
{
  "type": "https://didcomm.org/public-folder/1.0/publish-result",
  "id": "b0d1…",
  "thid": "77aa…",
  "body": {
    "missing": ["bafkreid…", "baguqeerb…"],
    "retain_until": "2027-08-20T00:00:00Z"
  }
}
```

- `missing` — CIDs reachable from the card's `root` that the relay does
  not yet hold. Empty array: the card is now the served version.
- `retain_until` (OPTIONAL) — RFC 3339 timestamp: how long the relay
  commits to storing the folder before it may garbage-collect.
  Republishing renews it. Relays that never collect omit the field.

### Problem reports

Standard report-problem 2.0 messages, `pthid` = the offending message's
`id`. Descriptor codes:

- `e.p.did.unknown` — the relay holds no card for the queried `did`.
- `e.p.path.not-found` — no such path under the current root.
- `e.p.card.invalid` — signature or structure of a published card does
  not verify.
- `e.p.unauthorized` — sender may not publish for this `did` under the
  relay's policy. (Never used for `query`.)

## Constraints

- One `path` per query in 1.0; batching is a possible future addition.
- The relay never interprets tree contents in this protocol; filename
  conventions (`_redirects`, `profile.json`) live entirely reader-side.
- Freshness verification of an answered card (`expires`) is reader
  policy; the relay serves the most recently published card it has, even
  past `expires`.

## Security and privacy

- The relay cannot forge content: every answer is verifiable against the
  owner's signature. Its remaining powers are withholding and staleness,
  bounded by `expires`.
- Readers are anonymous; `query` MAY be anoncrypted and sent from a
  one-time DID. Relays SHOULD rate-limit rather than authenticate.
- Publishing MUST be gated (it consumes storage); the reference policy
  is an existing mediation relationship with the relay.

## Prior art

DWN published records (owner-signed, anonymously readable), KERI
witnesses/OOBI, DNS/DNSSEC zone delegation, Nostr NIP-65 relay lists,
SIP redirect servers, DIDComm shorten-url/1.0 and user-profile/1.0 (the
profile query is this protocol's `query { did, "profile.json" }`),
IPFS trustless gateways (IPIP-402) for the proof-chain read shape.

## Implementations

| Name / Link | Implementation notes |
| --- | --- |
| (planned) [didcomm-mediator](https://github.com/estoc-net/didcomm-mediator) | relay role, alongside coordinate-mediation/3.0 |
| (planned) [@estoc/agent-core](https://github.com/estoc-net/estoc) | owner + reader roles; tree/card math in [@estoc/signed-dir](https://github.com/estoc-net/estoc/tree/main/packages/signed-dir) |
