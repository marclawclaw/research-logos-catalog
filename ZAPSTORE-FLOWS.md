# Zapstore over Nostr: sequence diagrams (as of Nov 2026)

Source: the zapstore/{zapstore,zsp,relay,hotbox} repos at their 2026-11-07 HEAD, and NIP-82 draft PR #2526. Zapstore's private `product/specs` were not available, so search ranking and the bundle diff schema are not shown in detail.

## 1. Publish a release (developer → relay)

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer
    participant ZSP as zsp CLI
    participant HB as hotbox (NIP-46 bunker)
    participant BL as Blossom (cdn.zapstore.dev)
    participant R as Catalog relay (relay.zapstore.dev)

    Dev->>ZSP: zsp publish (zapstore.yaml, APK)
    ZSP->>ZSP: parse APK: version_code, signing cert, lineage
    ZSP->>R: REQ {kinds:[30509], #d:[sha256(cert)]}
    R-->>ZSP: C1 identity proof (cert signed "I control npub X")
    Note over ZSP: abort unless proof is active and names signer as owner or delegate.<br/>Release guard: version_code must exceed every published one.
    ZSP->>HB: sign kind 24242 (Blossom upload auth)
    HB-->>ZSP: signed auth event
    ZSP->>BL: HEAD /<sha256>
    BL-->>ZSP: 404 (not stored yet)
    ZSP->>BL: PUT /upload (APK, icon, screenshots) + auth
    BL-->>ZSP: blob descriptor (url, sha256, size)
    ZSP->>HB: sign 32267 App, 3063 Asset(s), 30063 Release, 30509 proof if new
    HB-->>ZSP: signed events
    ZSP->>R: EVENT 32267 / 3063 / 30063 / 30509
    Note over R: Publisher gate: blacklist, then whitelist, then Vertex WoT score<br/>(new pubkeys only). New app id: forge account at least 30 days old.
    R-->>ZSP: OK (accepted into the buffer zone, not served yet)
    R->>BL: fetch APK
    R->>R: read the real signer cert and lineage, verify the C1 proof against it,<br/>reject known leaked/test keys
    R->>R: promote the listing to the served table
```

## 2. Catalog snapshotting (relay, every 5 min)

```mermaid
sequenceDiagram
    autonumber
    participant R as Catalog relay
    participant CK as Catalog key (relay-held)
    participant FS as Bundle files (HTTP)

    loop every 5 minutes
        R->>R: resolve each app's release chain (head = highest version_code)
        R->>R: apply bans / deletions (kind 5), add notices
        R->>FS: snapshots/N.tar.zst (events.jsonl + notices)
        R->>R: build diff N-1→N, index (hashes of every member), per-app artifacts<br/>(search vectors, ONNX encoder, top followers by PageRank, avatars)
        R->>CK: sign kind 30078 manifest {d: from-to-v, file: index sha256}
        CK-->>R: signed manifest
        R->>FS: bundles/(N-1)-N.tar.zst
    end
```

## 3. Phone sync, search, trust check, install

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant P as Zapstore app (iolite, local SQLite)
    participant R as Catalog relay
    participant V as Vertex DVM
    participant BL as Blossom / asset url
    participant A as Android package installer

    Note over P: First run: an embedded seed bundle, catalog pubkey pinned from its manifest
    P->>R: GET /v1/bundle?from=<last epoch>
    R-->>P: tar.zst (30078 manifest + index + diff.jsonl + artifacts)
    P->>P: verify manifest sig against the pinned catalog key, epoch continuity,<br/>every member hash, every event sig, then import
    U->>P: search "notes app"
    P->>P: on-device ranking: exact name/app id, prefix, one-edit, facts + vector dot product
    P-->>U: results (no query leaves the phone)
    U->>P: open app page
    P->>V: kind 5312 "who ranks highest among the people following publisher X, as I see the graph"
    V-->>P: ranked follower list
    Note over P: fallback: scan follow lists (kind 3) of people I follow for a p-tag on X
    U->>P: Install
    P->>BL: GET APK (url, or Blossom GET /<sha256>; optionally over Tor)
    BL-->>P: APK bytes (hashed while downloading)
    P->>P: sha256 == x tag? APK signer cert == apk_certificate_hash?<br/>C1 proof valid → "Claimed by the developer"
    P->>A: install / update (signer must match the installed copy)
```

## 4. Zap (NIP-57 tip) to a publisher

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant P as Zapstore app
    participant LN as Publisher's LNURL server (from lud16 in kind 0)
    participant W as User's wallet (NWC, NIP-47)
    participant NR as Nostr relays (incl. catalog relay)

    P->>LN: GET /.well-known/lnurlp/<name>
    LN-->>P: callback, min/max, allowsNostr, nostrPubkey
    P->>P: sign kind 9734 zap request {p: publisher, i/a: app, amount, relays:[...]}
    P->>LN: GET callback?amount=..&nostr=<9734>
    LN-->>P: bolt11 invoice (description hash commits to the 9734)
    P->>NR: kind 23194 NWC pay_invoice (encrypted to the wallet service)
    NR->>W: forward request
    W->>W: pay over Lightning
    W->>NR: kind 23195 result (preimage)
    NR-->>P: payment result
    LN->>LN: sees the invoice settled
    LN->>NR: kind 9735 zap receipt (signed by nostrPubkey, embeds the 9734 + bolt11)
    NR-->>P: 9735 appears on the app page (zap total, top zappers)
    Note over NR: the catalog relay accepts 9735 scoped by an i-tag,<br/>but does not check that the invoice was actually paid
```
