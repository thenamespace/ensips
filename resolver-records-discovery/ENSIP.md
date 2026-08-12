# ENSIP-X: Resolver Record Discovery (resolver-records-discovery)

| | |
|---|---|
| **Author** | @artii.eth, happysingh.eth |
| **Status** | Draft (pre-submission) |
| **Type** | Standards Track |
| **Created** | 2026-08-11 |

## Abstract

A standard resolver method, `getRecordsData`, that enumerates which records a name
has set: the coin types with an address record, the text keys with a value, the
data keys with a value, and whether a contenthash, pubkey, or ABI record exists.
Clients use the response to fetch
exactly the records that exist instead of probing a hard-coded list of common keys.
The method is defined so it can itself be answered via CCIP Read (ERC-3668), which
makes record discovery work uniformly for onchain, offchain, and L2 names: the same
gateway and data source that serve a name's record values can serve its record
inventory.

## Motivation

A resolver answers point queries (`text(node, key)`, `addr(node, coinType)`) but
offers no way to ask "which keys does this name have?". Clients cope by multicalling
a guess list: a dozen popular text keys and a few coin types. Everything outside the
guess list is invisible. A name with `text("com.farcaster")` or an address for an
uncommon chain simply does not render in wallets and profile UIs, even though the
data is set and resolvable.

Parameterized text records cannot be covered by a guess list at all. ENSIP-25
defines `agent-registration[<registry>][<agentId>]`; a client starting from an ENS
name does not know which registry addresses or agent IDs to query, so its agent
registrations cannot be discovered through resolver point queries alone.

For names with onchain resolvers on L1, indexers partially repair this: the ENS
subgraph reconstructs key sets from `TextChanged` and `AddressChanged` events. Even
there the repair is incomplete, since early PublicResolver versions emitted only the
indexed key hash, which is not recoverable to the key string. But indexing events is
a repair, not a solution, and it does not reach:

- **CCIP Read (offchain) names**: wildcard resolvers serving entire subname trees
  from a database. No events, no onchain state, nothing to index.
- **L2 names**: records live in a contract on another chain; L1 clients see only the
  verifying resolver. Each L2 needs its own indexer, and clients need to know which
  one to ask.
- **ENSv2 names**: ENSv2 deploys entirely on Ethereum mainnet (the planned
  Namechain L2 was scrapped in February 2026), and its resolvers emit per-record
  events carrying full key strings, so v2-native onchain records are indexable in
  principle. But the repair starts from zero: the v2 event vocabulary is new, so
  every indexer must be rebuilt before any v2-native name renders, clients face a
  long migration in which v1 and v2 resolvers coexist and the guess list is the
  only universal fallback, and offchain subname trees served under v2 wildcard
  resolvers fall back into the first bullet, with no events to index at all.

ENSIP-16 (offchain metadata) gestures at this by letting a resolver advertise a
GraphQL gateway, but it defines a parallel query stack that resolution clients do
not already speak, has no onchain-name story, and has seen minimal adoption. What
wallets actually have today is exactly one universal capability: calling resolver
methods, wildcard-wrapped per ENSIP-10, with ERC-3668 reverts handled transparently.
Record discovery should ride that capability, not a new one.

## Specification

The key words MUST, MUST NOT, SHOULD, and MAY are to be interpreted as described in
RFC 2119.

### Interface

```solidity
interface IRecordsData {
    struct RecordsDataResponse {
        // ENSIP-9 coin types for which addr(node, coinType) returns a non-empty value.
        uint256[] coins;
        // Text keys for which text(node, key) returns a non-empty string.
        string[] texts;
        // Keys for which the ENSIP-24 data(node, key) profile returns a non-empty
        // value. Sources that do not implement ENSIP-24 return an empty array.
        string[] dataKeys;
        // True when contenthash(node) returns a non-empty value.
        bool contenthash;
        // True when pubkey(node) returns a non-zero value.
        bool pubkey;
        // Bitmask of ABI content types present, per the ABI record's contentTypes
        // semantics; 0 means no ABI record.
        uint256 abiContentTypes;
        // Opaque pagination cursor. Non-empty when this response was truncated;
        // the client passes it back verbatim to fetch the next page. Empty on
        // the final (or only) page.
        bytes nextCursor;
        // True when the source, across all pages, enumerates every record it
        // knows about for this name. False signals a best-effort answer from a
        // source that only sees part of the picture (e.g. an indexer with
        // coverage gaps), telling the client it MAY additionally probe keys of
        // interest.
        bool complete;
    }

    /// @param name DNS-encoded name, as in ENSIP-10.
    /// @param cursor Empty for the first page; a prior response's nextCursor
    ///        to continue enumeration.
    function getRecordsData(bytes calldata name, bytes calldata cursor)
        external
        view
        returns (RecordsDataResponse memory);
}
```

Interface selector: `getRecordsData(bytes,bytes)` = `0xe60504ba`, which is also the
ERC-165 interface id of `IRecordsData`.

### Resolver behaviour

- A resolver implementing this ENSIP MUST return true from
  `supportsInterface(0x1e6eb67a)`.
- The resolver MUST accept the DNS-encoded name and MUST answer for any name it is
  authoritative for under ENSIP-10 wildcard rules, deriving the node itself when its
  storage is node-keyed.
- The method MAY revert with `OffchainLookup` per ERC-3668. Gateways SHOULD compute
  the response from the same data source that serves the name's record values, so
  the inventory and the values cannot drift apart.
- Every listed coin type, text key, and data key SHOULD resolve to a non-empty
  value at the time of the response. Entries MUST NOT repeat within a response or
  across the pages of one enumeration. Order carries no meaning and clients MUST
  NOT rely on it.
- Implementations SHOULD cap response sizes (a suggested cap is 256 entries per
  array) and MUST return a non-empty `nextCursor` when truncating. The cursor is
  opaque: the source defines its encoding (an offset, a database cursor, a last-key
  marker), clients pass it back unmodified, and a source MAY reject a stale or
  invalid cursor, which clients handle by restarting from an empty cursor or
  falling back.
- `complete` describes source fidelity, not pagination: a paginated but fully
  enumerable answer keeps `complete = true` on every page. `complete = false` is
  reserved for sources that genuinely cannot see every record (an indexer with
  coverage gaps, a subgraph missing pre-2019 keys).
- For a name the resolver is authoritative for but which has no records, the
  resolver MUST return an empty response with an empty `nextCursor` and
  `complete = true`.

### Client behaviour

- Clients MUST treat the response as advisory: a hint about which queries to make,
  never as a substitute for the records themselves. Record values MUST still be
  fetched through the standard methods (`addr`, `text`, `contenthash`, `ABI`),
  which keeps every existing verification property (L2 state proofs, gateway
  signatures, onchain reads) exactly as it is today.
- Clients SHOULD call the method through the normal resolution path (the
  UniversalResolver or equivalent), so ENSIP-10 wrapping and ERC-3668 handling are
  applied uniformly.
- Clients SHOULD follow `nextCursor` until it is empty, and MUST bound the number
  of pages they will fetch (a malicious source could otherwise string a client
  through unlimited pages).
- When `complete` is false, or the resolver does not implement the interface,
  clients SHOULD fall back to today's behaviour (probing a list of common keys).

### Interaction with ENSIP-10

For wildcard resolution, clients call
`resolve(name, abi.encodeCall(IRecordsData.getRecordsData, (name, "")))`
on the resolver found for `name`. The name appears in both layers deliberately: the
outer copy drives resolver discovery, the inner copy makes the method self-contained
for resolvers and gateways that are called directly. Clients MUST place the same
DNS-encoded name in both layers; resolvers answer for the name inside the
`getRecordsData` calldata and MAY ignore the outer copy entirely.

## Rationale

**Why the request carries a DNS-encoded name, not a node.** Offchain and wildcard
resolvers generally cannot reverse a namehash; their gateways key data by name.
ENSv2 shared resolvers and every CCIP gateway fall in this class. A node-keyed
resolver can derive the node from the name in one pass. Passing the name serves
both, and matches the ENSIP-10 convention.

**Why keys are enumerated but values are not.** Values can be large and already have
standard, individually verifiable query methods. Enumeration plus a targeted
multicall keeps responses bounded and leaves the trust model of value resolution
untouched. A discovery response that included values would turn an advisory hint
into an unverified data channel.

**Why `abiContentTypes` is a bitmask rather than a bool.** The ABI record is keyed
by content type; a bool would force a second discovery round-trip to learn which
content types to request. The bitmask mirrors the record's own semantics at no cost.

**Why pagination is an opaque cursor.** A size cap alone would abandon exactly the
names that need discovery most: a name with hundreds of records is the one whose
keys fall furthest outside any guess list, and "truncate and fall back to probing"
hands it back the original problem. The cursor is opaque bytes rather than a
numeric offset so that every source shape can paginate in whatever terms its
storage understands (an offset, a database cursor, a last-key marker) without the
spec dictating storage layout.

**Why a `complete` flag exists, separately from pagination.** Truncation is
mechanical and `nextCursor` handles it. `complete` answers a different question:
does this source even know the whole picture? An indexer that only tracks some
record types, or a subgraph with pre-2019 resolver gaps, can honestly enumerate
everything it sees and still not see everything. The flag lets such sources say
so, and tells clients when probing is still worthwhile.

**Why the response is a struct but the request is not.** The request is plain
arguments because a single-field wrapper struct adds ABI nesting and buys nothing.
The response is a struct because eight named return values are unreadable at every
call site and in every gateway implementation. Extensibility comes from neither:
adding record kinds later means a new interface with a new selector and ERC-165
id, which clients feature-detect and prefer, and both interfaces can coexist on
one resolver during migration.

**Why this beats an offchain-only convention.** The elegant property of making
discovery a resolver method is that it works for every deployment shape with one
client code path. An offchain resolver answers from its database. An L2 resolver's
gateway answers from the L2 state it already proves values from. An ENSv2 shared
resolver answers for any name it serves, wildcard-inherited subnames included. And
notably, even a plain onchain PublicResolver can adopt this without storing key
sets onchain: it can answer `getRecordsData` with an `OffchainLookup` to a
subgraph-backed gateway, because advisory data does not need the storage-proof
rigor that values do.

## Backwards Compatibility

Purely additive. Existing resolvers are unaffected; clients detect support via
ERC-165 (or a failed call) and fall back to current behaviour. No changes to the
registry, the UniversalResolver, or any existing resolver method.

## Security Considerations

- **A lying source can hide records, not forge them.** Since clients re-fetch every
  value through the standard verified paths, the worst a malicious gateway can do is
  omit keys (a censorship/UX degradation, equivalent to today's status quo of not
  knowing the keys) or list keys that resolve empty (wasted queries). It cannot
  cause a client to display a record value the resolver would not itself serve.
- **Response-size abuse.** A name could set thousands of text keys to bloat
  discovery responses. The per-page cap bounds any single response, and the
  client-side page bound (Client behaviour) bounds the enumeration as a whole;
  clients SHOULD also enforce their own read limits.
- **Privacy.** Records are already public, but enumeration removes the practical
  obscurity of unusual keys. Users and tooling should not treat obscure text keys
  as private storage; they never were.
- **Freshness.** Gateway-served inventories can lag the underlying store. The
  advisory semantics absorb this: a stale hint yields a missing or empty-valued
  query, both of which clients already handle.

## Reference implementation sketch

Offchain (CCIP Read) resolver:

```solidity
function getRecordsData(bytes calldata name, bytes calldata cursor)
    external
    view
    returns (RecordsDataResponse memory)
{
    revert OffchainLookup(
        address(this),
        _gatewayUrls(),
        abi.encode(name, cursor),
        this.getRecordsDataCallback.selector,
        abi.encode(name, cursor)
    );
}
```

The gateway resolves the name in its store and returns the ABI-encoded
`RecordsDataResponse` (signed per the resolver's existing ERC-3668 verification
scheme). An indexer-backed gateway for onchain names is identical, with the store
being the subgraph or equivalent. An onchain resolver that prefers to answer
directly can maintain key sets in storage on write (an EnumerableSet per node),
trading write gas for gatewayless discovery; the spec deliberately allows either.

## Open questions for the draft discussion

1. Should the response include reverse-registration hints (`name` record presence)
   for forward-resolved names, or is that out of scope?
2. Should the UniversalResolver grow a convenience method that combines discovery
   and a value multicall in one round trip?

## Copyright

Copyright and related rights waived via CC0.
