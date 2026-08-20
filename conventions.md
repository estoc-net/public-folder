# Filename conventions (non-normative)

The protocol does not know what any file means; the relay serves bytes.
Everything below is a client-side convention: a shared way to *read* a
folder that can grow — or be replaced — without touching the relay
protocol. Each convention file is an ordinary file in the tree, covered
by the root card like everything else.

## `_redirects`

A redirect table at the tree root, using a subset of the Netlify syntax
(the same file Cloudflare Pages executes natively and IPFS gateways
adopted as IPIP-0002 — the "content-addressed folder + redirect file"
precedent this protocol leans on):

```
/old-path        /new-path        301
/moved-post      /posts/2026/x    302
/*               https://example.org/:splat  303
```

- One rule per line: `source destination status`.
- Status codes 301, 302, 303, 307 only.
- `/*` and `:splat` are supported; nothing else from the full Netlify
  grammar is.

Moving away — to a new DID or a new home — is one rule:
`/* <new-address> 303` at the root of your final tree.

The relay MAY execute `_redirects` on the browse domain as a
convenience. Trust never depends on it: a reader holding the three
pieces can fetch `_redirects` and check any redirect themselves.

## `_headers`

Reserved, same lineage (Netlify / Cloudflare Pages): per-path
declarations of Content-Type, caching, CORS. Defined here so the name is
set aside; no relay behaviour is attached to it yet. Keeping headers in
one small file — rather than baking them into each file, `.asis`-style —
keeps every other object pure body bytes, which is what makes the merkle
tree dedupe.

## `profile.json`

The owner's public profile as a plain file. Field vocabulary borrowed
from DIDComm user-profile/1.0:

```json
{
  "displayName": "…",
  "displayPicture": "avatar.png",
  "description": "…"
}
```

Answering "who is this DID" is `query { did, path: "profile.json" }` —
the profile case needs no special protocol, it is just a path.

## Projection files

Because a query is only ever a path (never a filter — see spec §6), any
richer read is pre-computed by the owner at write time and placed in the
tree: `posts/index.json`, `feed.xml`, an HTML rendering, an AS2 JSON
projection. The query *language* is the shape of your content, not a
capability of the protocol — clients pick a path and interpret what they
fetch.
