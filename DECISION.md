# Bitsocial Graph Architecture Decision

> **Status:** accepted direction; pre-BSIP and non-normative
> **Decision date:** 2026-08-03
> **Last reviewed:** 2026-08-08
> **Related research:** [README.md](README.md)

## Decision

Bitsocial will not place ordinary social-graph edges in Bitsocial Chain state.

Profiles and public follows should be signed, current-state publications on the P2P network. Bitsocial Chain may contribute optional identity anchors and economic signals, and may eventually checkpoint a graph snapshot hash, but it is not the graph's storage layer.

The implementation sequence is crawler first, follow standard second:

1. build a community/liveness observatory;
2. build community discovery and directory outputs;
3. experiment with signed, expiring profile state;
4. use the evidence to write a profile/follow BSIP;
5. support multiple independently operated graph providers.

The short doctrine is:

> Put ownership and value onchain. Keep social relationships in signed, expiring P2P state.

## Why

An onchain follow graph has four structural costs that do not fit Bitsocial:

1. **Permanent history.** Removing an edge from current chain state does not erase the transactions that created and removed it.
2. **Wallet and gas requirements.** A follow is a casual gesture and should not require an economic account or chain write.
3. **Public safety history.** Onchain blocks, mutes, private groups, and annotations expose sensitive relationships permanently.
4. **Global control and indexing gravity.** Contracts, canonical indexers, and privileged emergency controls tend to become shared choke points.

Bitsocial already has a better medium for current social state: signed records addressed by public keys, mutable pointers, P2P distribution, and native liveness signals. The chain is designed for small ownership/economic state, not posts, feeds, member lists, or social edges.

## Graph boundaries

Bitsocial Graph should distinguish two products that have different consent and privacy properties:

- **Declared graph:** profiles and public follows deliberately published by users.
- **Observed graph:** public interactions already visible in the protocol, such as posting in a community, replying, or moderating.

Providers must not present an observed interaction as if the author declared a social relationship. Pseudonymity mode remains unlinkable by construction and must not be worked around.

### Active existence and inbound references

EFP permits any Ethereum address to be followed, tagged, muted, or blocked without the address ever participating. Bitsocial should make a narrower claim:

- an **active profile node** exists only when that profile publishes valid signed state;
- another user may reference an unpublished profile identifier, but the provider treats it as a dangling reference;
- a dangling reference does not create an active account, profile, or implied endorsement;
- public authors can still appear in the observed graph because publishing content is itself active publication.

This is active existence, not consent by the target to every inbound public follow.

## Profile and follow state

The first profile experiment should reuse Bitsocial's signing, IPNS, pagination, and seeding primitives without freezing a separate profile wire type prematurely. Current [`pkc-js` issue #31](https://github.com/pkcprotocol/pkc-js/issues/31) work points toward a profile configured from general delegated-community primitives after [crossposts (#32)](https://github.com/pkcprotocol/pkc-js/issues/32), while the bespoke `AuthorCommunityIpfs` parts in [PR #227](https://github.com/pkcprotocol/pkc-js/pull/227) are paused. Bitsocial Graph should integrate the eventual capability, not make the unfinished representation part of this decision.

A non-normative minimum shape is:

```text
ProfileState
  version
  profileId
  sequence
  updatedAt
  expiresAt
  public profile fields
  follow-list root or page references
  authorized manager/device keys
  signature

FollowEdge
  targetProfileId
```

Initial semantics:

- The highest valid sequence published by the profile is current.
- An old signed snapshot remains authentic, but it is not current.
- An expired or unreachable profile stops counting as live after a documented grace period.
- Unfollow is represented by absence from the newest current list, not by a permanent public event.
- The first BSIP should not require public follow timestamps or an append-only operation log.
- Names such as `.bso` are aliases/resolution aids, not automatically the identity stored in an edge. Name transfer must not silently transfer another person's followers.

The exact stable profile identifier, key-rotation rules, and page encoding remain open BSIP questions.

## Ownership and delegation

Bitsocial should reuse EFP's useful separation of concerns without copying the list NFT:

- a stable owner/root authority controls the profile;
- delegated manager or device keys may update public profile and follow state;
- revoking a device key does not require replacing the public identity;
- `.bso` may optionally help anchor resolution, recovery, or key rotation;
- a social list is not a transferable financial asset.

Delegation must work for users who never create an Ethereum wallet. Any chain anchor is optional enrichment, not a prerequisite for a profile or follow.

## Privacy and safety policy

Public state may contain deliberately public follows. It must not contain:

- blocks;
- mutes;
- private groups or client-side folders;
- private notes or safety classifications;
- hidden recommendation feedback;
- mappings that defeat community pseudonymity modes.

Public relation tags require separate justification. The initial follow model should prefer the smallest public edge rather than standardizing labels that can become harassment, reputation, or surveillance metadata.

A `noIndex` flag can communicate preference to cooperative providers, like `robots.txt`, but it is not a privacy mechanism. Anything publicly published can be copied or archived. Documentation must describe self-pruning as currentness and availability semantics, never guaranteed deletion.

## Availability and expiry

EFP's strongest operational advantage is that any indexer can reconstruct an onchain list without the owner remaining online. Bitsocial should address that trade honestly:

- profile owners republish current signed state;
- followers, RPCs, and seeders may cache and serve the latest valid state until expiry;
- providers apply a documented grace window for ordinary downtime;
- caches are not expected to preserve superseded profile or follow versions indefinitely;
- a newer valid publication resurrects an expired profile;
- expiry parameters are protocol-visible and provider behavior is documented.

This provides durable-enough current state without making an append-only archive the protocol model. It cannot prevent third parties from independently archiving public data.

## Graph providers

Bitsocial Graph is an observation and discovery category, not a single authoritative service.

The ecosystem should provide:

- an open-source crawler that starts empty by default;
- deterministic currentness and edge-interpretation rules;
- a portable current-state snapshot format;
- a documented query API;
- enough provenance to explain which public records produced an edge;
- client support for changing provider or running locally.

Ranking, recommendations, and sybil heuristics belong to providers and clients, not consensus. Raw node or follower counts must not become protocol truth.

### Counts and inverse indexing

A profile can prove only its own current outbound follow state. It cannot authoritatively publish its inbound follower count, because that count depends on current signed lists distributed across other profiles.

- `followingCount` is the number of valid current outbound edges and can be derived locally when the full current list is available.
- `followersCount` is an inverse index over other profiles' valid current outbound edges and requires a crawler, indexer, or graph provider.
- a profile-provided scalar such as `followersCount` is never authoritative and should be ignored;
- providers may expose raw counts plus clearly labeled active or weighted counts, but the weighting rules remain provider policy;
- every count response identifies its provider, `asOf` time, ruleset/version, count type, coverage, and snapshot or source provenance.

Signed edges prevent a user from inventing a scalar count, but they do not prevent sybil rings from publishing many authentic follows. Clients must not present a provider's coverage or weighting choice as protocol consensus.

Forge may operate a convenient default provider, while companies, communities, and users operate alternatives or local instances. A pubsub-voted provider registry may help clients discover or rank providers. Such voting chooses a provider and ruleset; it does not vote a count into truth.

## Snapshot content policy

Official Bitsocial Graph publications may include:

- current community nodes and liveness states;
- current directory/discovery data;
- coverage and seeder measurements;
- aggregate growth, decay, and activity metrics;
- public current-state edges with explicit validity semantics, if needed for portability.

The official durable series must not include raw historical author-level edge dumps. Superseded public follow lists may exist in third-party archives, but preserving them is not an official product goal.

Content addressing does not guarantee permanence by itself, but publishing datasets for mirroring increases the chance that they persist. Dataset scope therefore matters more than claims that an IPFS object will eventually disappear.

An optional chain checkpoint may contain only a snapshot hash and minimal integrity metadata. It should not be built until a concrete verification use case justifies it, and it must never contain raw edges.

## Bitsocial Chain's permitted roles

Bitsocial Chain may contribute:

1. **Optional identity resolution:** `.bso` names can resolve to profile/public keys and support an explicit rotation model.
2. **Optional economic signals:** public tips or awards may enrich a graph when users deliberately expose attribution; private or opaque payments must remain possible.
3. **Optional integrity checkpoints:** a hash of a permitted current-state or aggregate snapshot.

The chain must not store ordinary follows, blocks, mutes, feeds, private lists, recommendation feedback, or raw graph snapshots.

## EFP lessons

### Adopt conceptually

- primary and optional additional lists;
- owner/manager separation;
- versioned and typed data;
- batch updates where useful;
- open-source indexers and stable integration surfaces.

### Reject for Bitsocial

- wallet or chain access as a prerequisite;
- list ownership represented by a transferable NFT;
- permanent operation history for ordinary social edges;
- public block/mute tags;
- a canonical hosted index as the practical source of truth;
- global pausing or configuration powers over the social protocol.

The comparison was verified on 2026-08-03 against EFP's [Introduction](https://docs.efp.app/intro), [FAQ](https://docs.efp.app/faq), [Deployments](https://docs.efp.app/production/deployments), [Infrastructure](https://docs.efp.app/production/infra), [Multisig](https://docs.efp.app/production/multisig), and [`ListRecordsV2`](https://github.com/ethereumfollowprotocol/ListRecordsV2/blob/ac61fa01fbfe778b92382ab6e888558a1feec6b5/src/EFPListRecordsV2.sol#L364-L382).

## Delivery sequence

### v0 — Observatory

Crawl known community lists and measure reachability, heartbeat freshness, activity, directory coverage, and seeder coverage. Publish community-level current state and aggregates only.

### v1 — Community discovery

Derive community relationships from public links and observed activity. Produce explainable candidate directory rankings and offer them back to `lists`.

### v2 — Author graph

Expose observed author relationships only on linkable, non-pseudonymous surfaces. Keep declared follows and observed interactions visibly distinct.

### v3 — Profile-state experiment

Follow the evolving `pkc-js` profile-as-configured-community work and test profile identity, manager delegation, sequence/currentness rules, expiry, caching, seeding, and resurrection without prematurely standardizing follows.

### v4 — Profile/follow BSIP and plural providers

Write the standard from empirical evidence. Before treating it as infrastructure, have at least two independent implementations derive compatible current edges and follower counts under the same declared ruleset.

## Success criteria

- Creating a profile and following someone requires no wallet or chain fee.
- Blocks and mutes never enter public protocol state.
- Expired state stops appearing as current according to documented rules.
- Unfollow does not require publishing a permanent public tombstone.
- A user can change graph provider or self-host without changing identity.
- Independent providers can explain and reproduce declared current edges.
- Following counts derive from the profile's signed current outbound edges; follower counts identify provider, time, ruleset, type, and coverage.
- Raw, active, and weighted counts are visibly distinct, and no profile can self-report an authoritative follower scalar.
- No team or multisig can globally pause the social protocol.
- Pseudonymity modes remain outside the linkable author graph.

## Open questions before a BSIP

1. Which general community, delegation, and crosspost fields will identify a configured profile after `pkc-js` issue #31 resumes?
2. What is the stable `profileId`, and how do owner, publishing, device, and recovery keys rotate?
3. What expiry and grace periods balance availability with meaningful currentness?
4. Should any public relation tags exist in the first version?
5. How are large follow lists paginated and updated without republishing unnecessary data?
6. What snapshot fields are necessary for provider portability without creating an official surveillance archive?
7. When does an optional `.bso` anchor follow a person, a root key, or a transferable name?
8. What evidence is sufficient for two providers to demonstrate compatible current-state derivation?
9. What provider response and snapshot formats make follower counts independently reproducible without requiring a permanent raw edge archive?

## Non-goals

- Designing an onchain social graph.
- Guaranteeing deletion of data that was publicly published.
- Making one crawler, API, ranking model, or discovery provider canonical.
- Defining the final follow BSIP before the observatory and profile experiments produce evidence.
- Treating observed interaction edges as declared friendships or follows.
