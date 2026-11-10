# A Zapstore-like catalog on Logos — and the "Nostr of Logos"

*Research notes, 2026-11-09. Based on the local checkouts under `~/src/logos-*` (specs in `logos-co/logos-lips/docs/`, docs in `logos-co/logos-docs/docs/`) and on the Zapstore repos as of 2026-11-07.*

## TL;DR

1. **Zapstore no longer works the way the brief assumes.** In v2 (Nov 2026), phones don't search relays remotely. They pull a signed snapshot+diff bundle (`tar.zst`) from one catalog relay, check every hash and event signature, and search the device's own copy. The search uses vectors and a small model that ship inside the bundle. Nostr events are still the data format, but the relay rejects "vague" REQ filters. So Zapstore's real design is **signed events as the data model, a curator-signed bundle as the distribution unit, and search on the device**. That suits Logos well: if you download the whole catalog, no query reveals what you are interested in.
2. **No single Logos component is a Nostr relay.** Each one does part of the job:
   - **Delivery** is the live gossip layer, with Mix-private publishing and RLN against spam. Its Store keeps about **48 h** by default (store-to-store sync covers 1 h), messages are capped at **150 KiB**, and you can query only by content topic, time or hash.
   - **Storage** holds immutable, content-addressed blobs. The marketplace was removed in Jan 2026, blocks expire after 30 days by default, and a file stays available only while someone keeps it.
   - **Bedrock channels** are permanent, append-only, hash-chained logs signed with Ed25519, paid per byte. Blocks come about every 30 s and finality takes about an hour. Throughput is tiny (the PoW parameters assume about 2 transactions per block).
   - **LEZ** runs programs that can hold names, registries and votes, and programs can be deployed on the public testnet.
3. **No relay is needed: every client rebuilds the index itself.** Each publisher keeps a hash-linked log of signed events, stores it in Storage, and records the log's head on a Bedrock channel. A directory (an LEZ program or a shared channel) lists publishers. Each client reads the directory, fetches and verifies the logs, builds its own index and search, stays current through Delivery, and re-seeds what it fetched. Signed events plus Delivery, Storage and channel heads make up the "Nostr of Logos". The same engine can run the catalog, forum (LP-0026), microblog and social-media prizes. A **Keeper** (a relay-like caching service) is optional, an untrusted accelerator rather than an authority.
4. **The catalog doesn't need to wait for that layer.** Logos already has a federated catalog: `logos-repo.json` + `index.json`, deterministic `.lgx` files with an Ed25519 `did:jwk` signature, packages fetched from Storage first, and trust from the user's own keyring. The cheapest path is to (a) host the index in Storage instead of GitHub, (b) publish its head as a signed pointer over Delivery and anchor it on a channel, and (c) then rewrite the index as signed events based on NIP-82, so it can join the general event layer later.

---

## 1. What Zapstore actually is (Nov 2026)

| Piece | Mechanism |
|---|---|
| App | kind **32267** (addressable, `d` = reverse-DNS app id, `f` platform tags, markdown `content`) |
| Release | kind **30063** (addressable, `d` = `app@version`, `c` channel, `e` → assets) |
| Asset | kind **3063** (NIP-94 descendant: `x` sha256, `size`, `version_code`, `apk_certificate_hash`, `url`, Blossom fallback `GET /<sha256>`) |
| Identity proof ("C1") | kind **30509**. The APK signing certificate signs `"Verifying at <t> until <exp> that I control the following Nostr public key: <hex>"`, which links the signing key to an npub. Supports delegates and revocation. |
| Curation | kind **30267** "stacks" (NIP-51 sets of 32267 references) |
| Catalog relays | kind **10067** (NIP-65-like list of catalog relays) |
| Relay gating | blacklist → whitelist → **Vertex** web-of-trust score; check that the forge account is at least 30 days old; a "buffer zone" holds a listing until the relay has checked the C1 proof against the real APK; refuses certificates whose keys are known to be leaked |
| Distribution | Every 5 min the relay writes a snapshot `N.tar.zst` and a diff `N-1→N`. A kind 30078 manifest, signed by the catalog key, lists one index file, and that index lists the hash of every other member. The client pins the catalog key on first use. |
| Search | NIP-50 on the relay. In v2 it also runs on the device, from embedding vectors and an ONNX query encoder shipped in the bundle. |
| Trust UI | Global PageRank followers in the bundle, a personal Vertex DVM query, and a fallback that scans kind 3 follow lists |
| Social | NIP-22 comments, NIP-57 zaps, NIP-56 reports, all scoped to an app by the `i` tag |

NIP-82 (PR #2526, opened Oct 2026) is still open, so these kinds are not yet a ratified standard.

**What Zapstore's evolution teaches us:**

- Open relays and remote queries were not enough. Zapstore had to add (a) a **publisher gate**, (b) **proof that the binary's signing key belongs to the listing key**, (c) **signed, verifiable bulk distribution**, and (d) **search on the device**.
- Treat the transport as untrusted. Everything is verified end to end, from the bundle signature down to each event signature, file hash and APK certificate.

## 2. The comparables, reduced to four primitives

Every catalog needs four things: a way to **enumerate** entries, a **metadata** store, a **blob** store and a **trust** model.

| System | Enumerate | Metadata | Blobs | Trust | Takeaway for Logos |
|---|---|---|---|---|---|
| Zapstore | catalog relay → signed bundle | Nostr events | Blossom / CDN | WoT + C1 + relay gate | the model to beat |
| F-Droid index-v2 | signed entry → index + diffs | one JSON per repo | HTTP mirrors | key pinned by fingerprint | almost identical to today's Logos catalogue |
| Myco | your relays + mesh neighbours | Nostr | each phone runs Blossom | follows-only recommendations | local relay per device; offline mesh |
| PearBrowser | per-relay `catalog.json` / Hyperbee | Hyperdrive manifest | Hyperdrive | operator seeds | append-only signed B-tree as the index |
| Holochain App Store | DHT anchor | DevHub DHT | DHT chunks | agent signatures | **Launcher paused**; publishers must stay online about 30 min, which warns against DHT-only designs |
| Dappnode | Directory contract | ENS → APM repo contract | IPFS | contract whitelist | registry on chain, blobs elsewhere; reads need a node |
| Qortal | `SEARCH_QDN_RESOURCES` | (name, service, id) on chain + hash | QDN | registered names | names gate who may publish |
| DappStop | chain records | ENS / Ceramic | IPFS | wallet | hackathon project only |
| NorviTech armiger-ultra | — | — | — | — | **Not a catalog.** An 8M-parameter classifier for BitTorrent DHT metadata, used by indexers such as Bitmagnet. Its own card has withdrawn benchmark claims. The valid idea behind it is "crawl a public log and index and rank it locally". |

## 3. Logos building blocks, judged as catalog and social substrate

| Need | Delivery | Storage | Bedrock channel | LEZ |
|---|---|---|---|---|
| Live fan-out | ✅ gossip / filter, Mix for senders | — | ~30 s blocks | — |
| History | ⚠️ 48 h store, sync covers 1 h only | ✅ immutable CIDs; durable only while someone hosts them (30-day default TTL) | ✅ permanent | ✅ account state |
| Queries | topic / time / hash only | by CID only | by channel id | accounts + events (indexer being deprecated) |
| Size | 150 KiB per message | unbounded (64 KiB blocks) | ~1.8 MB per inscription, but paid per byte | 100 KB per account |
| Authorship | none (put a signature in the payload) | none | Ed25519, threshold keys, rotation | program logic |
| Spam control | RLN: each user holds a membership and gets a per-epoch message quota | — | fees | fees |
| Privacy | Mix for lightpush and store queries; filter subscriptions are linkable | private downloads over Mix | public | private accounts possible |
| Maturity | store / filter specs draft; mix, SDS, store-sync raw | testnet v0.3 | testnet v0.3 | testnet; programs deployable |

Prior art inside Logos to reuse rather than reinvent:

- **OpChan** (`messaging/application/raw/opchan.md`): a forum on Waku made of typed, Ed25519-signed JSON messages (post, comment, vote, moderation), with session-key delegation. Effectively Nostr without relays.
- **Community history** (`storage/raw/community-history.md`): an immutable archive plus a signed pointer that gets re-announced over messaging. This is the "mutable head" pattern.
- **Remote log** (`messaging/application/draft/3/remote-log.md`): content-addressed log plus a name system for the head.
- **Accountlog** (`identity/raw/accountlog.md`): an Ed25519 account as an append-only log that adds and removes keys per context. This gives key rotation, which Nostr lacks.
- **Private name resolution RFP** (`rfp/RFPs/RFP-XXX-private-name-resolution.md`): self-certifying, sequence-numbered, expiring records over a "Logos DHT". This is a mutable pointer with lookup privacy.
- **Logos SQL**: a replicated SQLite on a channel, a ready-made "index that anyone can replay".
- **Planning entries**: `RD-catalog-program` (on-chain listing + Basecamp feature, depends on a name service), `LP-XXX-catalog-governance` (community curation, kept separate from IFT for legal reasons), LP-0023 (LEZ Program Registry), LP-0026 (forum), `LP-XXX-social-media` ("sign with any key, distribute over any protocol"), all in `flywheels.logos.co/data/flywheels.yaml`.

---

## 4. The "Nostr of Logos": a proposed abstraction

Working name: **Logos Events**. The name doesn't matter; the layering does.

```
                 ┌──────────────────────────────────────────────────────────┐
 apps            │ catalog · forum · microblog · publication · reviews ...  │
                 └──────────────────────────────────────────────────────────┘
 L1 semantics       kinds + tags (reuse NIP-01 ranges: regular / replaceable / addressable / ephemeral)
 L0 event           {id=H(body), author, created_at, kind, tags, content, sig}  — self-authenticating
 ─────────────────────────────────────────────────────────────────────────────────────────────
 transports         HOT   Delivery gossip on bucketed content topics (Mix-published, RLN-limited)
 (all untrusted)    WARM  Delivery Store (≈48 h) for catch-up
                    COLD  Logos Storage: each publisher's hash-linked log segments (CIDs);
                          every client that fetches a segment re-seeds it
                    HEAD  Bedrock channel: publisher inscribes {seq, head_cid}; a directory
                          (LEZ program or shared channel) enumerates publishers
 index              built by EVERY CLIENT locally from the above (no relay)
                    optional Keepers = untrusted caches / convenience snapshots
 identity           accountlog (rotating keys) → name service (LEZ) → C1-style proofs binding
                    package-signing did:jwk / app keys to the identity
```

### 4.1 Event envelope

- Copy NIP-01 exactly where you can: `id = sha256(canonical [0, author, created_at, kind, tags, content])`, single-letter indexed tags, the same kind ranges, and the same rules for replaceable and addressable events. Most of the Nostr NIPs then carry over unchanged (lists, labels, comments, reports, communities, long-form).
- **Open decision: signature scheme.**
  - **(a) Plain Nostr secp256k1 BIP340.** Events are byte-for-byte Nostr, so you get free interop: any node can re-serve events as a Nostr relay over websockets, and Nostr clients and tools just work. The cost is a new request kind in `logos.signer/1`, which today only signs BIP340 with the BIP86 tweak.
  - **(b) Ed25519 with `author` = an accountlog identity.** This matches channels, `did:jwk` package signing and the identity specs, and allows key rotation. The cost is losing Nostr interop unless you add a bridge.
  - **Lean: (a), with an optional binding event in the C1 style that links the Nostr key to the Logos accountlog identity and package `did:jwk`.** Interop gives the forum and social prizes an existing client ecosystem.
- **Don't use Delivery's own message hash or timestamp as the event identity.** The Waku hash covers the shard, and store nodes reject timestamps more than ±20 s off. Republishing an old event therefore means wrapping the same signed bytes in a fresh Waku envelope, which is fine because the identity is in the payload.

### 4.2 Hot path on Delivery

- **Topic layout:** `/logos-events/1/<bucket>/proto`. One shard per app/version is chosen by autosharding, and the topics spec warns against having too many topics. So:
  - Bucket by **kind family** (for example `apps`, `releases`, `social`), and
  - optionally by **author-hash prefix** (`a-3f`) for outbox-style following. Subscribing to a prefix bucket gives you k-anonymity: you receive everyone in the bucket and filter locally, so the filter node can't learn exactly whom you follow.
- **Publish through Mix and lightpush** so a publisher's IP isn't linked to their key.
- **RLN** is the spam floor, and it adds something Nostr can't have: an RLN proof with an application-level external nullifier such as `H(app_id ‖ epoch)` allows **one anonymous review per membership per app per epoch**, giving Sybil-resistant anonymous ratings. This is new design work at the app layer, not something that exists today.
- Messages are capped at 150 KiB, so events carry metadata only. Icons, screenshots and `.lgx` files go in Storage as CIDs.

### 4.3 Every client rebuilds the index (no relay)

Nostr needs relays because its events have no durable home and no order. On Logos, three layers give events both, so a client can rebuild everything from untrusted sources and nothing has to hold the index.

**1. Publisher log.** Each publisher's events form a hash-linked log: each event carries `seq` and `prev` (the id of the previous event), as in accountlog. Log segments, from one event up to a batch, are written to **Storage**.

**2. Head.** The publisher inscribes `{seq, head_cid}` on its **own Bedrock channel**. That channel record is permanent, ordered and reorg-safe.
- Channels support **threshold key changes and rotating writers**, so an organisation gets an M-of-N release key with no extra machinery.
- The publisher also announces the new head over Delivery for live delivery.

**3. Directory.** A directory enumerates publishers. Options:
- an **LEZ program** mapping `app_id → publisher channel`, which also handles namespace and anti-squatting (see Design C);
- a shared registry channel;
- for social apps, simply the authors a user follows, or authors seen on a community topic.

**4. Client.** The client:
- reads the directory;
- for each publisher, reads the head from the channel and fetches log segments from Storage, walking `prev` and checking every hash and signature;
- builds its **own local index and search**;
- subscribes on Delivery for live updates and uses Store to fill short gaps;
- **re-seeds what it fetched**. Every client is a replica, which is how logs stay available despite the 30-day TTL and without a storage market.

Properties this gives you, without a relay:
- **Censorship by omission becomes detectable.** The client knows the head `seq` from the chain, so any missing event is a visible gap rather than a silent omission. This is a guarantee Nostr relays can't provide.
- **Curation is local policy.** Stacks, labels and reports are just more publisher logs that the client chooses to apply.
- **Downloading is private.** The client fetches whole logs by CID, optionally over Mix, so there are no queries revealing interest.

Where it gets hard, and how to handle it:

| Issue | Mitigation |
|---|---|
| **Cold start:** a new client fetches N logs instead of one bundle | Anyone may publish a **convenience snapshot** (a merged index with its CID). The client verifies it against each publisher's channel head, so it's a cache, not a trusted source. Basecamp can also ship a seed snapshot, as Zapstore does. |
| **Head inscriptions cost money and chain throughput is small** | Fine for a catalog (one inscription per release). For high-volume social data, batch heads (one inscription per N posts or per day), use Delivery and Store for the latest posts, and fill gaps from other clients. Store-sync's range-based set reconciliation is already specified for this. |
| **A dormant publisher whose log nobody has fetched** | The log is unavailable until the publisher comes back online. That's acceptable for a catalog, where the apps people install are the ones that get seeded. |
| **Full-corpus search on a device** (all forum threads) | Fine for catalogs and communities you join. For "search everything on the network", someone has to aggregate, and that is the only real case for a Keeper. |

**Keeper (optional):** any node that has done the rebuild above can also re-serve its index, as a Nostr-style relay, a query endpoint over Mix, or a published snapshot. It is untrusted by design, because clients can check any answer against the channel heads. It is an accelerator, not part of the protocol.

### 4.4 Choosing what to anchor

- Inscriptions are paid per byte and chain throughput is tiny, so anchor **heads, not events**. A head is about 100 bytes.
- For a catalog, one inscription per release is the right granularity.
- For social data, batch heads as described in §4.3.

### 4.5 Why the same engine runs forum, social and publication

| App | Kinds (reuse NIPs) | Transport profile |
|---|---|---|
| App catalog | 32267 / 30063 / 3063 / 30267 / 30509 (NIP-82 draft) | LEZ directory → publisher channel heads → logs in Storage; client-built index; Delivery for freshness |
| Forum (LP-0026) | NIP-7D threads (11), NIP-22 comments (1111), NIP-72 communities (34550, 4550 approvals), NIP-32 labels for moderation | community bucket topic; members' logs discovered from the topic and rebuilt by clients; moderators' label sets applied by the client |
| Microblog / social | kind 1, kind 0 profile, kind 3 follows, NIP-65-style outbox via author-prefix buckets | rebuild the logs of authors you follow; batched heads; RLN for anti-spam |
| Publication | NIP-23 long-form (30023), body or media in Storage | addressable events + Storage CIDs; optional anchor for timestamping |
| Private groups | — | **don't use public events.** Use Delivery reliable channels (SDS) + chat module encryption, and keep Logos Events for the public layer |

---

## 5. Concrete catalog designs (from cheapest to most ambitious)

### Design A: "Signed feed": today's catalogue, de-GitHubbed (weeks)

- Keep `logos-repo.json`, `index.json`, `.lgx` and `did:jwk` as they are.
- Put `index.json` (and icons, read straight from the `.lgx` `assets/`) in **Storage**.
- The catalog operator publishes a signed **head pointer** `{seq, index_cid, prev_cid}`:
  - over a Delivery topic for live updates, re-announced periodically so it survives the 48 h Store window;
  - inscribed on a **Bedrock channel** as the durable, ordered record.
- `logos-repo.json` swaps `indexUrl` for `{channel_id, topic, signer}`. The client pins the signer, as F-Droid and Zapstore do.
- **Search runs on the device** over the whole index. Download the entire catalog and no query leaks anything.
- **Pros:** reuses almost everything, and Basecamp's "suggest an installable package" for unmet intents keeps working.
- **Cons:** the curator is the publisher of record, and you still need someone pinning blobs.

### Design B: Publisher-signed events + curator bundles (the Zapstore v2 shape on Logos)

- Publishers sign NIP-82-style events: App, Release, Asset (`x` = `.lgx` `rootHash`/sha256, `storage` = CID, `platform` variants, `requested_permissions`), and a **C1-style binding** between the `.lgx` `did:jwk` signer and the publisher's event key.
- They publish over Delivery and optionally anchor on their own channel.
- **Curators** are optional. A client can do all of the following itself (§4.3). A curator is a client that publishes its result: it gathers the events, verify them (C1 binding, rootHash, for Logos: dependency resolution against LIDL interfaces, size and platform checks), then publish signed snapshot+diff bundles to Storage and anchor them on their channel.
- **Users subscribe to one or more curators.** "Official Logos", "community-governed" and "my friend's list" are all just bundles.
- **Curation is subjective and layered:** NIP-51-style stacks, NIP-32 labels ("audited", "malware"), NIP-56 reports.
- This cleanly handles the legal split the flywheels yaml calls for: IFT can run a technical curator without "deciding" inclusion, while `LP-XXX-catalog-governance` votes produce another curator's policy.

### Design C: On-chain registry for names, governance only (LEZ)

- An LEZ program (SPEL) holds a PDA for each **app id** recording `{owner identity, publisher event key, latest release anchor}`. Only the owner can update it. This matches LP-0023's "only the original author can update" rule.
- It solves **namespace squatting and impersonation**, which events alone can't do. Pair it with `rfp/appendix/anti-squatting-mechanisms.md`.
- Governance votes (`LP-XXX-catalog-governance`, LP-0019) write an `included` flag that a curator turns into its policy.
- **Keep metadata off chain.** Store only names, pointers and votes, because LEZ state is public and paid for.
- This is also the natural **directory** for the client-rebuild model in §4.3: `app_id → publisher channel`.

### Design D: Logos SQL catalog (curious alternative)

- The catalog *is* a Logos SQL database synced over a channel: rows for apps, releases and assets, with writers rotating among curator keys.
- Every client can replay it into local SQLite, which gives SQL queries locally, full replication and reorg handling for free.
- The cost is a per-byte inscription for every row change. That's acceptable for a few thousand apps updated weekly, but not for social data.
- Worth a prototype as the "zero-infrastructure" variant: no Store, only the chain plus Storage for blobs.

### Design E: DHT-first (Holochain / GNS style), mostly *not* recommended alone

- Each publisher is a self-certifying name (the private name resolution RFP) whose records include `catalog-head = <cid>`.
- **The trouble is enumeration.** A privacy-preserving DHT is *designed* to resist enumeration, and Holochain's paused Launcher shows how fragile DHT-only availability is.
- **Use the DHT for what it's good at:** per-publisher and per-app mutable pointers and private lookups, with enumeration coming from curator bundles or channel anchors.
- The armiger-style "crawl and rank locally" idea shows up instead as on-device ranking over downloaded bundles, which is what Zapstore v2 already does.

## 6. Ideas specific to Logos (things Zapstore can't easily do)

1. **Rebuild attestations as events.** `.lgx` builds are deterministic and come from Nix. Independent rebuilders publish an "I rebuilt commit X → rootHash Y" event, and clients can require **k-of-n matching rebuilds**. This is stronger than a single Sigstore attestation, which nothing checks on the client today.
2. **Install = seed.** Opt-in: installed `.lgx` files and catalog snapshots stay advertised in Storage, so popular apps stay available without a marketplace. The UI can show availability (`getProviders` count) as a health signal.
3. **Private discovery.** Bulk bundle downloads over Mix, private Storage fetches, k-anonymous bucket subscriptions, and whole-log fetches instead of queries. Neither Nostr nor Zapstore offers this end to end.
4. **Anonymous but rate-limited reviews and ratings** through RLN nullifiers scoped to an app (§4.2).
5. **Private tips** in place of zaps: an LEZ private-account transfer plus a public "tip receipt" event that shows only amount buckets or nothing. Note that Zapstore doesn't verify zap receipts either.
6. **Threshold publishers.** An org's release key is a Bedrock channel config or an `lez-multisig`, and the anchors prove that M of N signed.
7. **Capability transparency.** `requested_permissions` (LIP 311) plus the LIDL interface files sit in `assets/`, so curators can index and display what a module asks for before install, and labels can flag over-asking modules.
8. **Proof that nothing was omitted.** Each publisher's head `seq` is on chain, so a client can see a missing event as a gap. This is a censorship-resistance property relays don't offer.

## 7. Gaps and risks to resolve

- **Signer interface:** no Ed25519 and no untweaked BIP340 signing of arbitrary bytes for apps (`logos-signer-interface/signer.lidl`). Either choice in §4.1 needs a new request kind.
- **Delivery module API:** apps can't set `meta` or choose pubsub topics. This only matters if someone builds an optional Keeper query endpoint.
- **Persistence:** logs stay available because clients re-seed what they fetch. A publisher whose log nobody has fetched depends on its own node staying online, and blocks have a 30-day default TTL that re-seeding clients should configure.
- **Chain economics:** per-byte storage gas and about 2 transactions per block. Budget head inscriptions: one per release for the catalog, batched for social.
- **Specs used here are raw or draft:** store-sync, mix, SDS, lightpush v3, OpChan, accountlog, LIP 311/312.
- **Unverified:** Zapstore's private `product/specs` (search ranking, publisher policy, bundle diff schema); Myco's event kinds; whether the CLI's "catalog source logos" also fetches the *index* from Storage, or only packages.

## 8. Suggested next steps

1. **Spec Design A** (a head pointer over Delivery, anchored on a channel) as a small change to `catalog-format.md`, and prototype it with the existing `logos-package-downloader`.
2. **Write a "Logos Events" LIP draft** covering the envelope, kind ranges (reusing NIP-01), topic bucketing, the publisher log format (`seq`/`prev`, segments in Storage), the head inscription and directory, the verifiable convenience-snapshot format, and the signature decision.
3. **Map NIP-82 to `.lgx`**: App, Release and Asset events with `rootHash`, CID, variants, LIDL and permissions, plus the C1-style `did:jwk` ↔ event-key binding.
4. **Frame the client-rebuild library (publisher log + head + local index) as one RFP or λ-Prize.** It's the shared dependency of the catalog, forum (LP-0026), microblog and social-media entries, so fund it once rather than per app.
5. **Prototype Design D** (Logos SQL catalog) as a cheap comparison point on a local devnet.
