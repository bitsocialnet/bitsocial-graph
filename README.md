# Bitsocial Graph

> **Status:** thesis / project exploration — no code yet.
> **Date:** 2026-07-13
> **Last reviewed:** 2026-08-08
> **One-liner:** an open crawler and graph service that recursively maps the Bitsocial network — authors, communities, interactions — from public protocol data, with **liveness** as a first-class dimension of the graph.
> **Architecture decision:** [DECISION.md](DECISION.md) records the accepted direction for profiles, follows, privacy, availability, and graph snapshots. It is pre-BSIP and non-normative.

This is deliberately a docs-first repository. Read this thesis for the research and roadmap, then read [DECISION.md](DECISION.md) for the accepted architectural boundaries. There is no executable implementation yet.

---

## 1. Thesis

Bitsocial content is ephemeral by design: nothing lives on a blockchain, and a community that loses its owner, its peers, and its seeders becomes unreachable. This is a feature, not a bug — and it has a second-order consequence nobody has built for yet:

**The Bitsocial social graph is self-pruning.** Existence in the graph requires ongoing work (publishing, seeding, staying reachable), so the graph converges on *actually active* humans and communities. Facebook's graph is an archive that includes the dead and the departed; Bitsocial's graph is a living organism whose nodes must metabolize to persist.

Nobody can see this graph today. There is no follow primitive, no in-protocol community enumeration, and discovery runs on a hand-curated GitHub repo. **Bitsocial Graph** is the missing observation layer: a Googlebot-style crawler that recursively constructs the network graph from public data, tracks per-node liveness, and publishes the result as open data and APIs.

---

## 2. The premise, verified: ephemerality is real

Verified against the public [`pkcprotocol/pkc-js`](https://github.com/pkcprotocol/pkc-js) source and protocol docs (source paths below are relative to that repository):

| Mechanism | Behavior | Source |
|---|---|---|
| Community record | Owner-signed snapshot published to IPNS; owner republishes at least every **15 min** even without changes | `docs/protocol/data-permanence.md`, `src/runtime/node/community/local-community/ipns-publishing.ts:43-56` |
| IPNS record TTL | `publishInterval × 3` → **60 s** caching hint; record *validity* follows kubo default (~24 h) and expired records are rejected | `ipns-publishing.ts:313`, `src/util.ts:1240-1253` |
| HTTP router announcements | Expire after **24 h**; content not re-announced becomes unfindable via routers | `docs/protocol/data-permanence.md:64-67` |
| Pages (post listings, reply trees) | Regenerated with **new CIDs** on every update; old CIDs unpinned and discarded | `data-permanence.md:28` |
| Permanent data | Only `CommentIpfs` and `CommentEdit` are immutable — but "permanent" ≠ "available": *"IPFS data can become unavailable if no nodes pin it"* | `data-permanence.md:7-16, 88` |

So the claim holds, with two nuances worth keeping precise:

1. **Death is decay, not deletion.** With the owner offline and nobody seeding, a community degrades over minutes-to-hours (stale pages → expired router announcements → expired IPNS record → unreachable). While *any* copy of the immutable CIDs survives somewhere, the content is revivable.
2. **The owner key is the heartbeat.** Only the owner can sign community updates. Seeders can keep old content fetchable, but a community whose owner key is gone is frozen forever — it can never accept a new post. Liveness of the mutable pointer, not existence of the data, is what "alive" means.

This gives every node in the network a **native freshness signal** (the `updatedAt` heartbeat, ≤15 min for healthy communities) that Google had to invent for the web. The graph gets it for free.

---

## 3. The self-pruning graph vs. the alternatives

| | **Facebook** | **EFP (Ethereum Follow Protocol)** | **Bitsocial** |
|---|---|---|---|
| Persistence model | Database default — accounts persist until someone acts | Current state is mutable; the complete operation history is public onchain | Signed current state is live only while it remains available and valid |
| Dead/dormant nodes | Accumulate until the platform removes them | Follow lists and their histories outlive abandoned wallets | Stop counting as current after publication expires or becomes unreachable |
| Identity requirement | Platform account | Ethereum address; ENS is an optional complementary name/profile layer, not a requirement | Free Ed25519/profile keypair; `.bso`/`.eth` names optional |
| Edge cost | Free click | Wallet interaction and gas for an onchain write; multiple list operations can be batched | Signed P2P state; no per-edge chain fee |
| Enrollment | Platform controls account creation | Any Ethereum address can be followed, tagged, muted, or blocked without participating | An active profile node requires its own signed publication; inbound references to unpublished identities remain dangling |
| Safety state | Private to the platform, but platform-controlled | `block` and `mute` are standardized public list-record tags | Blocks, mutes, private groups, and safety notes stay local/private |
| Global control | Platform operator | Core contracts are pausable by a 3-of-4 team multisig; `brantly.eth` controls two listed signer keys, making the effective quorum that person plus either of two others | No global administrator in the P2P social protocol |
| Addressable users | Anyone the platform accepts | Ethereum accounts, including raw addresses without ENS | Anyone who can create and publish a keypair |

The EFP contrast is instructive because it makes the *opposite* persistence choice from Bitsocial. Facebook preserves dead **nodes** in a private database; EFP preserves the complete public history of **edges**. EFP does **not** require ENS, and its contracts support batching, but following still means wallet UX, gas, and an onchain record. A social graph should be cheap to write, cheap to stop serving, and should not turn private safety actions into permanent public data. Bitsocial's position: identity begins as a free keypair, public edges are signed current state, and activity must keep that state current.

**What EFP gets right and Bitsocial should reuse at the schema level:**

- a primary list plus optional additional lists;
- separation between an identity/list owner and a manager authorized to update it;
- versioned, typed data and batch publication updates, without copying the append-only public operation history;
- open-source indexers, a documented API, and integration tooling.

Those ideas do not require an NFT or onchain edge storage. In Bitsocial they map to a stable profile/root key, delegated device keys, a signed public follow list, and replaceable graph providers. Public relation tags should be treated conservatively; blocks, mutes, private groups, and private notes are never candidates for the public graph.

One honest flip side of self-pruning: history is only as durable as the communities that hold it. A beloved author's posts survive their absence as long as their communities stay alive — persistence proportional to how much the network values the content, rather than platform policy. That's the right default for a social network, but it means the graph service must decide its own memory policy explicitly (see §9).

---

## 4. What graph exists today (there are no follows)

Verified: **the protocol has no follow/friend/subscribe primitive.** The `pkc-js` API surface (`src/pkc/pkc.ts`) exposes communities, comments, votes, edits, moderation — no social edges. Client "subscriptions" are local app state. Follows are planned for master-plan Phase 3 ("Decentralize Twitter/X" — profile nodes, follows) but no BSIP specifies them yet.

Profile work is active but not settled. [`pkc-js` issue #31](https://github.com/pkcprotocol/pkc-js/issues/31) and [PR #227](https://github.com/pkcprotocol/pkc-js/pull/227) explore profiles using delegated community infrastructure. As of 2026-08-02, the bespoke `AuthorCommunityIpfs` design is paused and superseded in its author-specific parts pending [crossposts (#32)](https://github.com/pkcprotocol/pkc-js/issues/32); the current direction is a profile configured from general community and delegation primitives, not a distinct wire type. Bitsocial Graph therefore depends on profile capabilities, not on one unfinished profile schema.

What a permissionless crawler *can* extract, today, with zero special access:

| Edge / attribute | Source | Notes |
|---|---|---|
| author →posts-in→ community | Pages carry each comment's `author.address` (global, derived from the Ed25519 signing key, identical across communities) | The cross-community join key — `src/signer/util.ts:152-181` |
| author →replies-to→ author | Reply trees (`parentCid`, `postCid`, `depth`) | Interaction graph, thread co-participation |
| author →moderates→ community | `roles` map is public in every community record | High-signal edge: moderation is costly, hard-to-fake affiliation |
| author →history→ own past comments | `author.previousCommentCid` chain — explicitly usable to walk "a recent author comment history **in all communities**" | `src/schema/schema.ts:160` — see §5, this is the recursion |
| author →wallet/avatar→ external chains | `author.wallets` (signed address proofs), `author.avatar` (signed NFT ref) | Opt-in, signed links to crypto identity |
| author reputation per community | `postScore`, `replyScore`, `firstCommentTimestamp`, flairs in `CommentUpdate.author.community` | Community-scoped karma |
| community →links→ community | Link fields and community mentions in content; directory candidacy in `lists` | Same trick as web hyperlinks |
| community vital signs | `updatedAt` heartbeat, published `stats` (hourly→all-time active users/posts), challenge config, features | The liveness layer |

**Not available, by design:** individual votes (only aggregate up/down counts leave the owner's node — there is no vote graph to crawl), private subscription lists, and anything behind pseudonymity:

> **Pseudonymity islands.** When a community sets `features.pseudonymityMode`, the owner re-signs each publication with a fresh alias keypair — on-wire author addresses are unlinkable aliases (`src/runtime/node/community/local-community/publication-store.ts:83-155`). Verified against `lists/5chan-directories/5chan-directories-defaults.json`: **all 63 5chan directory codes specify a pseudonymity mode** (60 per-reply, 3 per-post). So essentially all current 5chan content is deliberately author-unlinkable, and the crawler couldn't defeat that even if it wanted to (the alias map lives only in the owner's private DB). The author-level graph therefore concentrates in Seedit-style communities now and Phase-3 profile surfaces later; 5chan contributes community-level nodes and liveness only. This is a constraint to *embrace* — it proves the protocol lets communities opt out of graphability, which is the ethical answer to §8 built in.

---

## 5. How the crawler works

The Googlebot analogy is right, with one substitution: on the web the recursive unit is the hyperlink; on Bitsocial it is **the author**.

```
        ┌────────────────────────────────────────────────────────┐
        │                      frontier                          │
        │   (community addresses + author addresses to visit)    │
        └───────┬────────────────────────────────────────────────┘
                │
   ┌────────────▼─────────────┐
   │ FETCH                    │  resolve IPNS → community record
   │ (reuses indexer's crawl) │  walk posts pages → reply trees
   └────────────┬─────────────┘
                │
   ┌────────────▼─────────────┐
   │ EXTRACT                  │  author addresses, displayNames, roles,
   │                          │  wallets, links, parent/child, stats
   └────────────┬─────────────┘
                │
   ┌────────────▼─────────────┐
   │ EXPAND (the recursion)   │  new frontier entries from:
   │                          │  a) author.previousCommentCid chains —
   │                          │     walk an author's history backwards
   │                          │     into communities we've never seen
   │                          │  b) links/mentions in content
   │                          │     (community addrs, .bso/.eth names)
   │                          │  c) name resolution (.bso/.eth → keys)
   │                          │  d) curated lists & future multisubs
   └────────────┬─────────────┘
                │
   ┌────────────▼─────────────┐
   │ LIVENESS PROBE           │  re-resolve IPNS on schedule;
   │ (the novel part)         │  updatedAt heartbeat → alive / stale /
   │                          │  dying / dead; decay curves per node
   └──────────────────────────┘
```

**Discovery vectors, ranked by cleanliness:**

1. **Seed lists** — the `bitsocialnet/lists` repo (5chan + Seedit directories) is today's entire discovery layer; multisubs (designed but *not yet implemented* in pkc-js — `README.md:283-305`) will add more.
2. **Author chains** — `author.previousCommentCid` is the killer primitive: every comment is a signed pointer into that author's past, across communities. Crawl a community → harvest authors → walk their chains → discover communities no list has ever mentioned. This is organic, permissionless, in-protocol recursive discovery.
3. **Content links** — posts/comments referencing other communities, same as web hyperlinks.
4. **Passive infrastructure telemetry** — HTTP routers/trackers (Forge-adjacent nodes like `routerofbitsocial.xyz`, `bsotracker.online` are already in pkc-js defaults) observe provide-announcements for IPNS names and pubsub topics as a side effect of operating. That's organic enumeration of everything alive — but it's also the vector with real privacy/centralization weight, so it should be a documented, opt-in, aggregate-only input if used at all.

**What already exists to build on** (all verified working code):

- **`bitsocial-indexer`** — a working crawler + SQLite/FTS5 archive that already walks pages/replies and stores `author_address` per comment. `SELECT DISTINCT author_address, community_address FROM comments` is a bootstrap author↔community bipartite graph *today*. The graph project is roughly "the indexer's crawl loop, turned author-centric and cross-community, plus liveness."
- **`bitsocial-seeder`** — proven daemon-bootstrap (`ensureDaemon` reuse-or-spawn of `bitsocial-cli`), list ingestion (`fetchCommunityListSource`), durable queues/scheduling. The graph crawler is architecturally a sibling: same PKC RPC daemon, different extraction.
- **`lists`** — the canonical seed set, plus `whitelist-challenge.json` as a trust signal.

---

## 6. What it unlocks

1. **Automated directories and multisubs.** The `lists` README already promises curation "will be automated by voting"; the pkc-js README describes a not-yet-implemented `search` multisub of "thousands of semi-curated communities." A graph service with liveness + activity + interaction-weight signals can *generate* those lists — crawler output becomes multisub input. This is the near-term, ecosystem-needed deliverable: discovery is the acknowledged unsolved gap in the protocol today.
2. **Cross-network search and profiles.** The indexer searches within communities it's told about; the graph layer adds "which communities exist, which are alive, who is where" — and author pages (for non-pseudonymous surfaces) assembled from chains.
3. **Follower/following counts and mutuals.** A profile can publish its own signed outbound follow list, but it cannot authoritatively state how many other profiles follow it. That inbound relation is distributed across the network and requires inverse indexing. Bitsocial Graph is therefore necessary infrastructure for familiar social UX even when profiles and follows themselves are pure P2P.
4. **The substrate for Phase 3 feeds.** "Swappable feeds and algorithms" (Forge master plan) presuppose a graph to compute over. Recommendation-as-a-marketplace needs a commodity graph layer underneath — this is it.
5. **A network observatory.** Live/dead dashboards, growth and decay curves, seeder coverage maps — the empirical answer to "how does the Bitsocial social graph evolve over time," measured rather than speculated. (Natural fit alongside `stats.bitsocial.net`.)
6. **Design input for the follows BSIP.** Building the crawler first reveals exactly what Phase-3 profiles should publish to be graphable. Current `pkc-js` work points toward profiles configured from delegated community primitives after crossposts land; the v3 experiment should follow that work without prematurely freezing its wire shape.

### Why counts need graph providers

Counts should be computed from verifiable edges, never accepted as self-reported profile scalars:

| UX value | Derivation | Trust boundary |
|---|---|---|
| `followingCount` | Count the profile's newest valid signed outbound edges | Can be derived locally once the complete current follow list is available |
| `followersCount` | Invert newest valid signed outbound edges published by other profiles | Requires a crawler, indexer, or graph provider with stated coverage |
| active/weighted followers | Apply liveness, challenge, age, interaction, or optional economic-signal rules | Provider policy; useful for ranking, never protocol consensus |

This prevents a user from spoofing a count by writing `followersCount: 1000000` into their own profile. It does not eliminate sybil inflation: many real keypairs can still publish signed follows. Providers should therefore expose the raw count for ordinary social UX and may additionally expose clearly labeled active or weighted counts. Every response should identify the provider, `asOf` time, ruleset/version, count type, coverage, and enough snapshot or source provenance to reproduce the result.

---

## 7. Fit with Bitsocial Forge

Forge's framing is "the blacksmith, not the landlord" — it forges tools for the network without owning it. A graph service fits, with one hard requirement: **the graph must be a commodity, not a moat.** The site's own competitor critique (Bluesky AppViews, Lens graphs, Farcaster indexers) is that discovery layers become de-facto owners of the network. The design answer:

- **Open-source crawler anyone can run**, empty by default (the indexer's exact stance).
- **Published current-state and aggregate snapshots as content-addressed datasets** — portable, verifiable, forkable observations with an explicit validity window. Anyone can mirror them or run a competing service from them, but the official publication series must not become a permanent raw history of author-level edges. See the snapshot content policy in §9 and [DECISION.md](DECISION.md#snapshot-content-policy).
- **APIs with exit** — same principle as Forge RPC's custody/exit story; a natural product line beside `rpc.bitsocialforge.com`.

Forge can run a convenient default provider, while other companies, communities, and users run competing providers or local instances. A pubsub-voted provider list could help clients discover and rank those services. Voting should select *which provider and ruleset a client trusts*; it must not vote a follower count into truth. Counts remain derived from signed edge records under disclosed currentness and coverage rules.

Master-plan mapping: Phase 3 needs the graph (follows, feeds); Phase 4 explicitly lists **discovery** among the services to pluralize. Bitsocial Graph is the first mover in that category, built to be one-of-many.

---

## 8. What about Bitsocial Chain? What belongs on a ledger

The obvious question: Bitsocial has its own chain — shouldn't the graph live there, as an immutable ledger of who-knows-whom? **No — and Bitsocial Chain's own design docs already commit to the reason.** But the chain does have three real roles in the graph, all as a *signal source*, never as the store.

**What the chain actually is** (verified in the public [`bitsocialnet/bitsocial-chain`](https://github.com/bitsocialnet/bitsocial-chain) repository): an Ethscriptions/Facet-inspired derivation design — intents posted as plain Ethereum L1 calldata to a keyless inbox address, state derived deterministically by anyone, no sequencer, no admin keys. The working POC implements exactly one primitive: the `.bso` name registry (`README.md:3-14`). BSO itself is a fixed-supply (210M), immutable, adminless ERC-20 already live on Ethereum mainnet (per the chain.bitsocial.net site copy). And the repo is emphatic about scope: *"Posts, comments, votes, moderation state, member lists, and feeds never appear in intents, derived state, or the read API — the intent schema has nowhere to put them, by construction"* (`DESIGN.md:19-25`). The phrase "social graph" appears exactly once in the entire repo — in the privacy section, **as the threat model**: the economic layer must not make it easy to "reconstruct a user's social graph and finances" (`DESIGN.md:29`). An onchain social graph wouldn't extend Bitsocial Chain; it would contradict its founding constraint.

**Why "an immutable-but-somewhat-mutable ledger" can't work — three structural reasons:**

1. **Mutable state, immutable history.** On a calldata-derived chain, you can make the *current state* mutable (the `.bso` registry already supports update/transfer/revoke), but every mutation is itself permanent public Ethereum history. An onchain follow graph with deletion still preserves, forever and publicly, the complete edge history of everyone — including inactive or deceased users. That is stricter than a private platform archive: Facebook at least *can* delete (and is sometimes legally forced to); Ethereum cannot, and no deletion key exists by design. "Mutable to some degree" buys you a mutable tip on an immutable archive.
2. **A ledger is structurally blind to liveness.** A chain can only observe transactions. Dead people don't transact — but neither do billions of living people who will use Bitsocial without ever touching a wallet. So onchain "alive" can only ever mean "paid recently" (name renewal, ENS-style rent), which measures *who pays*, not *who lives*. Meanwhile the P2P layer measures liveness natively and for free: publishing signed content, ≤15-minute community heartbeats. The mutability the graph actually needs is **decay** — and decay is precisely what a ledger cannot do and what the P2P layer cannot help doing. The living-social-fabric property comes from the medium, not from a schema choice.
3. **Cost physics.** The chain's own economics discussion cites ~$2.94 per user-op on Facet-style chains and concludes future primitives should derive "small economic state, never social content" (`ECONOMICS_DISCUSSION.md`, `DESIGN.md:107-117`). Putting ordinary edges directly on Bitsocial Chain would inherit that cost regime; even a cheaper chain would reintroduce wallet and gas UX. Edges need a chain-free path.

**The layering the ecosystem has already implicitly chosen — made explicit:**

| Layer | Holds | Persistence | On death |
|---|---|---|---|
| **Ledger** (Bitsocial Chain) | Property: BSO balances, `.bso` name ownership | Permanent | Correctly outlives you — like estates and deeds |
| **P2P** (pkc protocol) | Presence: content, gestures, edges | Decays without effort | Correctly dies with you — unless the network deliberately keeps your memory seeded |
| **Graph** (this project) | Observation: a decaying map of both | Mortal by policy (§9) | Mirrors the network |

There's a historical rhyme worth noticing: land registries and name registries are more or less the *only* social records pre-modern societies committed to permanent ledgers — deeds and parish registers. The living social fabric (who talks to whom, who matters to whom) lived in living memory and died with it. Bitsocial's stack reproduces that division of labor, and it matches how the real world actually works: your property survives your death; your presence does not.

**What the chain genuinely contributes to the graph:**

1. **Optional identity anchoring.** `.bso` names bind a human-readable, ETH-address-owned name to an Ed25519 key (`SPEC.md`) and can point to a rotated key. That makes them useful resolution and continuity signals, but a transferable name is not automatically a stable person identifier: the graph must distinguish profile identity from current name ownership so a name transfer cannot silently transfer followers. A paid registration (plus renewal, per the public site's "registered and renewed in BSO" — note the POC has no renewal yet, `POC_LIMITATIONS.md:14`) can be a weak sybil-costly or economic-liveness signal, never the primary identity or liveness rule.
2. **Costly-signal edges, when Phase 2 lands.** A tip or award is a social edge with skin in the game — onchain because it *moves value*, not because it's social. These become the highest-weight interaction edges the crawler can ingest. Caveat baked into the chain's privacy mandate: tipping must not force identity linkage (`DESIGN.md:27-41`), so these edges will be partial and opt-in — enrichment for the graph, never its backbone.
3. **Optional snapshot checkpoints — the only plausible graph→chain write.** A current-state or aggregate snapshot (§7) could be anchored by posting its content hash as a tiny intent: timestamping and integrity without placing edges in chain state. This should remain optional until a concrete verification use case justifies the cost. The checkpoint may outlive the snapshot, but it reveals only that a dataset was published, not its contents.

**Verdict:** the graph is a chain *consumer* and at most a chain *checkpointer*. This is EFP's mistake inverted — put value onchain and edges offchain, never the reverse. The chain holds what should outlive people; the P2P layer holds what shouldn't; the graph watches both and forgets on schedule.

---

## 9. Risks and open questions

- **Surveillance-adjacency.** Assembling cross-community author profiles is search-engine-normal but feels different on a pseudonymous network. Mitigations: index only public protocol data; pseudonymity mode is respected automatically (unlinkable by construction); honor tombstones the way the indexer does (serve-time redaction of removed/deleted); and consider proposing a **`noIndex` community feature — a robots.txt for communities — as a BSIP**, so opting out of graphability becomes protocol-legible rather than crawler-discretionary.
- **`noIndex` is preference, not privacy.** Cooperative graph services should honor it, but hostile crawlers can ignore any public flag. Pseudonymity mode and withholding/link-separating data are the actual privacy mechanisms; documentation must not imply that public data can be made unobservable by policy.
- **Sybil pressure.** Keypairs are free, so a raw follower count is useful UI but weak evidence of reputation. Providers may also publish clearly labeled active or weighted counts using interaction edges, challenge-passing, moderation roles, account age (`firstCommentTimestamp`), and optional wallet/NFT proofs. No weighting model is protocol truth.
- **Sparsity today.** ~70 curated communities, and the author layer is thin until Seedit grows and Phase 3 lands. Honest sequencing: v0 is a *community* graph and liveness observatory; the author graph is the growth curve, not the launch feature.
- **The current graph must itself be mortal.** If the graph service treats dead nodes as current forever, it becomes the permanent archive it critiques. Nodes decay after N days unreachable and resurrect on reappearance, with the parameter documented. This is a currentness/availability guarantee, not a deletion guarantee: anything public may be archived by third parties.
- **Snapshot content policy.** The durable official series may contain current community state, liveness measurements, coverage, and aggregates. It must not publish raw historical author-level edge dumps. Public current follow edges may appear only with clear validity/currentness semantics; superseded versions are not part of the official historical product.
- **Availability without permanent history.** Followers, RPCs, and seeders may cache and serve the latest valid signed profile/follow state until its expiry. They should not be expected to preserve superseded versions indefinitely. A grace window can absorb ordinary downtime without turning an append-only archive into the protocol model.
- **Centralization gravity.** Even open-source, the best-known graph endpoint accretes power. The snapshot-publishing discipline (§7) is the real mitigation; keep router telemetry (§5.4) disabled unless a documented privacy-preserving design justifies it.

---

## 10. Roadmap sketch

| Stage | Deliverable | Depends on |
|---|---|---|
| **v0 — Observatory** | Crawl seed lists; IPNS heartbeat monitor; live/stale/dead states; public dashboard + first published graph snapshot (communities only) | Nothing new — indexer/seeder patterns + pkc-js |
| **v1 — Community graph** | Recursive expansion via author chains + content links; community↔community edges; shared-author/shared-moderator affinity; **auto-generated directory rankings** offered back to `lists` | v0 |
| **v2 — Author graph** | Assembled author views on linkable (non-pseudonymous) surfaces; observed interaction graph; optional wallet/`.bso` enrichment from Bitsocial Chain (§8) | v1, Seedit growth |
| **v3 — Profile-state experiment** | Follow the evolving `pkc-js` profile-as-configured-community design; test signed current state, delegation, expiry/currentness, and cache/seed behavior without standardizing follows prematurely | v1 learnings; crossposts/profile work; no BSIP required for an experiment |
| **v4 — Follows and provider API** | Ingest standardized public follow lists; expose raw and labeled active/weighted follower counts, mutuals, tips/awards when Phase 2 ships, and feed/recommendation inputs | v3 evidence; follow/profile BSIP; at least two compatible providers |

---

## Contributing

Issues and pull requests are welcome, especially when they add source-backed research, challenge an assumption, specify a reproducible provider rule, or turn a roadmap stage into an implementation plan. Keep proposals non-normative here until the observatory and profile experiments produce enough evidence; protocol standards belong in the [Bitsocial Improvement Proposals](https://github.com/bitsocialnet/BSIPs) process.

## License

This repository is released under the [GNU General Public License v3.0 or later](LICENSE).

---

## Appendix: research trail (2026-07-13)

Claims above were verified against source, not READMEs alone:

- [`pkc-js`](https://github.com/pkcprotocol/pkc-js) — ephemerality: `docs/protocol/data-permanence.md`; IPNS publish/TTL: `src/runtime/node/community/local-community/ipns-publishing.ts:43-56, 313`; identity: `src/signer/util.ts:152-181`; cross-community author chains: `src/schema/schema.ts:160`; pseudonymity aliasing: `src/runtime/node/community/local-community/publication-store.ts:83-155`; no follow primitive: `src/pkc/pkc.ts` API surface; multisubs unimplemented: `README.md:283-305`; votes aggregate-only: `docs/protocol/comment-lifecycle.md:51`, `src/publications/vote/schema.ts:35-46`. Profile design state re-verified 2026-08-08 from [issue #31](https://github.com/pkcprotocol/pkc-js/issues/31) and [PR #227](https://github.com/pkcprotocol/pkc-js/pull/227): bespoke author-community parts are paused pending crossposts; delegation and read-only primitives survive.
- `bitsocial-indexer` — working crawler, `comments.author_address` in `server/src/db/schema.sql`, pages-walk in `server/src/crawler/crawler.ts:97-160`.
- `bitsocial-seeder` — CID-level seeding only (no author data), list ingestion `lib/utils.js:45-103`, daemon bootstrap `lib/daemon.js`.
- `lists` — curated discovery layer; pseudonymity defaults verified in `5chan-directories/5chan-directories-defaults.json` (63 codes: 60 per-reply, 3 per-post).
- `BSIPs` — six BSIPs (meta + comments, communities/pages, publishing/challenges, comment updates, votes); no follows/discovery BSIP yet; Application category reserves "profile and identity schemas" and "community list formats" (`bsip-1.md:68`).
- Forge master plan — `bitsocialforge.com/src/App.tsx:233-312` (Phase 3 follows/profile nodes, Phase 4 discovery services); non-custodial "blacksmith" framing `PRODUCT.md:29-30`.
- `bitsocial-chain` — Ethscriptions/Facet-inspired L1-calldata derivation; POC implements only the `.bso` registry (`README.md:3-14`); "social data stays P2P … by construction" (`DESIGN.md:19-25`, verified 2026-07-13); "social graph" appears only as a privacy threat (`DESIGN.md:29`); names owned by ETH addresses, resolving to Ed25519/libp2p keys, no renewal/expiry yet (`SPEC.md`, `POC_LIMITATIONS.md:14`); economics: burn as the Stage-2-native sink, ~$2.94/user-op Facet physics, future primitives = "small economic state, never social content" (`ECONOMICS_DISCUSSION.md`, `DESIGN.md:107-117`).
- Chain public site (`bitsocial-web/chain/src/sections`) — BSO: fixed-supply 210M, immutable adminless ERC-20 live on Ethereum; "Names and identity on the chain, registered and renewed in BSO" (renewal is site-promised but absent from the POC — open design question relevant to §8's economic-liveness signal).
- EFP (`efp.app`) — re-verified 2026-08-03 against [Introduction](https://docs.efp.app/intro), [FAQ](https://docs.efp.app/faq), [Deployments](https://docs.efp.app/production/deployments), [Infrastructure](https://docs.efp.app/production/infra), [Multisig](https://docs.efp.app/production/multisig), and [`ListRecordsV2`](https://github.com/ethereumfollowprotocol/ListRecordsV2/blob/ac61fa01fbfe778b92382ab6e888558a1feec6b5/src/EFPListRecordsV2.sol#L364-L382): Ethereum-address identity with ENS as an optional complement; primary/multiple list NFTs; ListRecords on Base, OP Mainnet, and Ethereum; batchable onchain list operations; public `block`/`mute` tags; an open-source but operationally substantial indexer/API stack; and a 3-of-4 core-team multisig with pause/configuration powers, where one listed person controls two signer keys.
