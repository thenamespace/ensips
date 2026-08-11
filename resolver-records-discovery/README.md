# Resolver Record Discovery

Working folder for a draft ENSIP: a standard resolver method that enumerates which
records a name has set, so clients can fetch exactly the records that exist instead
of probing a hard-coded guess list of common keys.

- [ENSIP.md](ENSIP.md): the draft spec (pre-submission).

## The idea in one paragraph

Resolvers answer point queries (`text(node, key)`, `addr(node, coinType)`) but offer
no way to ask "which keys does this name have?". This draft adds one method,
`getRecordsData(bytes name, bytes cursor)`, returning the coin types, text keys, and
data keys with non-empty values, plus contenthash/pubkey/ABI presence flags. It is
advisory only: clients still fetch values through the standard verified methods. The
method rides the paths every client already speaks (ENSIP-10 wildcard resolution,
ERC-3668 CCIP Read), so one code path covers onchain, offchain, and L2 names, and
the same gateway that serves a name's values can serve its inventory.

## Current design state

- Signature: `getRecordsData(bytes name, bytes cursor)`, selector `0xe60504ba`
  (also the ERC-165 interface id). The request is plain arguments; the response is
  a struct.
- Pagination: opaque cursor. Empty cursor requests the first page; a non-empty
  `nextCursor` in the response means truncated, pass it back verbatim. Clients
  bound the number of pages they fetch.
- `complete` is a source-fidelity flag only (an indexer with coverage gaps says
  false), not a truncation signal. Truncation is `nextCursor`'s job.
- Response covers: `coins`, `texts`, `dataKeys` (ENSIP-24 profile), `contenthash`,
  `pubkey`, `abiContentTypes` (bitmask), `nextCursor`, `complete`.
- ENSIP-10 interaction: the name appears in both the outer `resolve(name, data)`
  argument and the inner calldata. Clients must pass identical copies; resolvers
  answer for the inner copy.

## Decisions made so far

- Advisory semantics: a lying source can hide records, never forge them, because
  values are always re-fetched through the standard verified paths.
- Dropped the single-field `RecordsRequest` struct: it added ABI nesting and bought
  nothing. Extensibility comes from new interfaces with new selectors, not structs.
- Added cursor pagination: a size cap alone would abandon exactly the names that
  need discovery most (rich record sets outside any guess list).
- Included `pubkey` and ENSIP-24 `dataKeys` rather than excluding low-adoption
  record kinds, since the earlier exclusion rationale was inconsistent (ABI was
  included despite equally low adoption).
- Positioning: new ENSIP, not an extension of ENSIP-16. Different consumer
  (resolution clients vs indexers), different query model (per-name pull vs
  where-does-data-live), no shared mechanics, and the ENSIP-16 lineage carries the
  baggage of the stalled GraphQL approach.

## Prior art (researched August 2026)

- **ENSIP-16**: rewritten in November 2025 from the original GraphQL-gateway
  design to a minimal "Metadata Event Discovery" spec (`metadata(name)` returns
  `rpcURLs, chainId, baseRegistry`). Complementary, not competing: it tells
  indexers where data lives, this draft tells clients what records exist.
- **Blockful, "ENSIP-16 v2: Metadata API Improvements"** (pikonha, Aug 2024):
  closest prior proposal, same goal via extending the ENSIP-16 GraphQL schema to
  return all keys and values. Stalled after DoS objections from nick.eth, and its
  GraphQL substrate has since been deleted from the live spec.
  https://discuss.ens.domains/t/ensip-16-v2-metadata-api-improvements/19500
- **JustaName, "ENS Offchain Subname Publication Standard"** (justghadi.eth,
  Apr 2025): standardized events endpoint so indexers can track offchain subnames.
  Push/index model for indexers, complementary to this pull/query model.
  https://discuss.ens.domains/t/temp-check-ens-offchain-subname-publication-standard/20564
- **ENSIP-24 (Arbitrary Data Resolution)**: its optional `ISupportedDataKeys` is
  the closest precedent for key enumeration as a resolver interface, scoped to the
  `data()` profile only.
- **ENSIP-22 (Contract Features)**: `supportsFeature()` capability detection,
  a candidate mechanism for the detection story.
- Nobody has proposed this specific mechanism (discovery as a resolver method
  through ENSIP-10 + CCIP Read). Two teams independently hitting the same pain is
  demand evidence worth citing.

## Verified facts

- Selectors computed with `cast sig`: `getRecordsData(bytes,bytes)` = `0xe60504ba`;
  candidate rename `records(bytes,bytes)` = `0x63cac68e`.
- ENSv2 deploys entirely on Ethereum mainnet; the Namechain L2 was scrapped in
  February 2026. v2 resolvers emit per-record events with full key strings
  (deployed `PermissionedResolver`: `NamedTextResource(resource, name, keyHash,
  key)`; unmerged `SharedResolver`: `TextUpdated(uint256(node), keyHash, key,
  value)`), so v2-native onchain records are indexable in principle. The
  motivation's ENSv2 bullet was rewritten accordingly: the honest claims are that
  every indexer must be rebuilt for the new event vocabulary, the v1/v2 migration
  leaves the guess list as the only universal fallback, and offchain trees under
  v2 wildcards still have no events.

## Open items

- Rewrite the Motivation's ENSIP-16 paragraph: it still describes the
  pre-November-2025 GraphQL version of that spec.
- Possible rename of the method to `records(bytes,bytes)` (selector `0x63cac68e`)
  to match the resolver-profile noun convention (`addr`, `text`, `data`).
- Soften the cross-page duplicate rule: stateless cursors can yield overlapping
  pages when records change mid-enumeration, so clients must tolerate duplicates.
- Detection through the UniversalResolver path is hand-wavy (clients do not call
  `supportsInterface` on the end resolver; a failed CCIP call is ambiguous).
  ENSIP-22 `supportsFeature()` is one candidate fix.
- Open questions in the draft: reverse-registration hints in the response, and a
  UniversalResolver convenience method combining discovery with a value multicall.
