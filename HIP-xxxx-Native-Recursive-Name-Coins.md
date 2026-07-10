# HIP-xxxx : Native Recursive Name Coins

```text
Number:  HIP-xxxx
Title:   Native Recursive Name Coins
Type:    Standards Track
Status:  Draft
Authors: Jaron Rosenau <@denuoweb>
Created: 2026-07-10
```

## Abstract

This proposal extends Handshake ownership below the root by allowing an active name to register an immediate child as a first-class Handshake name coin.

A name coin consists of an owner-controlled covenant output and the corresponding authenticated name state. After a child is registered, the child has its own owner, resource data, renewal state, transfer lifecycle, and descendant namespace. An active parent cannot modify, revoke, transfer, or replace the child. The child remains valid independently of any later transfer, expiration, revocation, or re-registration of its ancestors.

For example, the owner of `acme` may register `alice.acme`. After confirmation, `alice.acme` is controlled only by its own name coin unless it later expires or is revoked under its own lifecycle.

## Motivation

Handshake provides consensus-enforced ownership of root labels, while ordinary subdomains remain controlled by the parent name's DNS operator.

HIP-0015 defines Update Chains. HIP-0016 applies an Update Chain to a per-TLD Escher tree containing unrevocable subdomains. In that design, child ownership remains in a secondary registry committed through updates to the parent name.

This proposal takes a different approach. Each sovereign child receives native Handshake state in the authenticated name tree. A child can transact without spending the parent, renew independently, store an ordinary Handshake resource, survive the parent lifecycle, and recursively register sovereign descendants.

This proposal is an alternative to the HIP-0016 state model. It does not modify HIP-0015 or HIP-0016.

## Conventions and Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

- **Root name:** A current Handshake name identified by the SHA3-256 hash of one root label.
- **Parent:** The immediate ancestor that authorizes registration of a child.
- **Sovereign child:** A hierarchical name represented by its own native name coin and authenticated name state.
- **Active name:** A registered name with a current owner that is valid under the existing expiration, revocation, and closed-state rules.
- **Epoch:** The value carried in item 1 of an owner covenant. For a root name it is the existing auction start height. For a sovereign child it is the child's generation.
- **Generation:** A monotonically increasing `uint32` identifying one lifecycle of a sovereign child.
- **Child log:** An append-only rolling commitment to every sovereign-child registration authorized by a name.
- **Immediate label:** The single leftmost label added to a parent to form its child.

## Design Overview

Registration is atomic:

1. A `DELEGATE` output spends and recreates the active parent name coin.
2. A paired `REGISTER_CHILD` output creates the child name coin.
3. The parent name state appends a commitment to the complete child output.
4. The child receives independent owner and renewal state at the containing block height.

After registration, no parent or ancestor participates in the child's ordinary lifecycle. The parent is needed again only if that exact child becomes inactive and is re-registered with a new generation.

Each registration creates permanent authenticated metadata for the child. This tombstone is required to preserve the parent relationship, reject stale lifecycle transactions, and determine the next generation after expiration or revocation.

## Canonical Labels and Name Length

Consensus operates on lowercase ASCII label bytes without a trailing dot.

A root label MUST satisfy the existing Handshake root-name rules.

A child label MUST satisfy the same byte-level syntax as an existing Handshake name:

- Its length MUST be between 1 and 63 octets.
- Each octet MUST be an ASCII digit (`0x30`–`0x39`), lowercase letter (`0x61`–`0x7a`), hyphen (`0x2d`), or underscore (`0x5f`).
- A hyphen or underscore MUST NOT be the first or last octet.
- Uppercase and non-ASCII octets are invalid.

The existing root-only blacklist is not applied to child labels. For example, `test.acme` is a valid child even though `test` is not a valid Handshake root label.

Unicode is not a consensus input. User-facing software MAY apply an IDNA profile, but consensus receives and validates only the resulting lowercase ASCII label bytes. Selection of an IDNA profile is outside this proposal.

The complete DNS wire-format name, including every label-length octet and the terminal zero octet, MUST NOT exceed 255 octets.

For a root label:

```text
root_wire_length = 1 + length(root_label) + 1
```

For an immediate child:

```text
child_wire_length = parent_wire_length + 1 + length(child_label)
```

The wire length of a root state is computed from its stored root label. The wire length of a hierarchical state is its authenticated `wire_length` field:

```text
wire_length(name_state) =
    1 + length(name_state.name) + 1, if not hierarchical
    name_state.wire_length,          otherwise
```

A registration for which `child_wire_length > 255` is invalid.

## Name Hash Derivation

### Root names

Existing root hashes are unchanged:

```text
root_hash = SHA3-256(root_label)
```

### Sovereign children

The exact child domain-separation tag is the 12 ASCII octets:

```text
HNS-CHILD-V1
```

Its hexadecimal encoding is:

```text
484e532d4348494c442d5631
```

An immediate child hash is:

```text
child_hash = SHA3-256(
    ASCII("HNS-CHILD-V1") ||
    parent_hash ||
    uint8(length(child_label)) ||
    child_label
)
```

No terminator is appended to the label. `parent_hash` is exactly 32 octets.

For `www.alice.acme`, implementations first hash `acme`, then derive `alice.acme`, then derive `www.alice.acme`.

The recursive, domain-separated construction prevents hierarchical names from sharing the root-label hash domain and permits a resolver to derive every candidate sovereign boundary directly from the queried labels.

## Authenticated Name State

The authenticated name state is extended with the following logical fields:

```text
hierarchical
parent_hash
generation
wire_length
child_log_hash
child_log_count
```

The existing `name` field stores the immediate label. For a root name, it remains the root label. For a sovereign child, it stores only the newly added child label.

### Hierarchical identity

For a sovereign child:

```text
hierarchical = true
parent_hash  = immediate parent's name hash
generation   = current or most recent generation
wire_length  = complete DNS wire-format length
```

For a root name, `hierarchical` is false and those three fields are absent.

The state key remains the 32-octet name hash.

### Child-log state

A name that has never registered a sovereign child has the implicit child-log state:

```text
child_log_hash  = ZERO32
child_log_count = 0
```

Once a child has been registered, the latest hash and count are retained in the parent name state.

### Epoch

The epoch of a name is:

```text
epoch(name_state) =
    generation, if hierarchical
    height,     otherwise
```

All owner lifecycle covenants for a sovereign child use `generation` in covenant item 1. Existing root names continue to use their auction start height without modification.

### Active and inactive child states

A sovereign child is treated as closed for owner lifecycle validation immediately after `REGISTER_CHILD`.

An expired, revoked, unregistered, or ownerless child MUST fail all ordinary owner lifecycle transitions. Revoked children become eligible for re-registration only after the existing revocation maturity has elapsed.

An inactive ancestor does not make an active descendant inactive.

### Persistent tombstones

After the first successful `REGISTER_CHILD`, the child's hierarchical identity and latest generation MUST remain in authenticated state even if the child expires or is revoked.

For every root or hierarchical name, `child_log_hash` and `child_log_count` survive update, renewal, transfer, finalization, revocation, expiration, and re-registration. A child's own log is therefore inherited by every later generation of that child.

The hierarchical identity, generation, and child-log fields MUST NOT be reset by ordinary name expiration. Without the tombstone, an old generation could be replayed and consensus could not determine the next generation.

### Canonical name-state encoding

The current serialized `NameState` field word is a little-endian `uint16`. This proposal assigns:

```text
bit 10: HIERARCHICAL
bit 11: HAS_CHILD_LOG
```

After all existing optional fields have been serialized, the following bytes are appended in order.

If `HIERARCHICAL` is set:

```text
parent_hash    32 octets
generation      4 octets, uint32 little-endian
wire_length     2 octets, uint16 little-endian
```

If `HAS_CHILD_LOG` is set:

```text
child_log_hash 32 octets
child_log_count 4 octets, uint32 little-endian
```

`HAS_CHILD_LOG` MUST be set if and only if `child_log_count > 0`. When it is unset, the hash and count are implicitly `ZERO32` and zero.

A state with neither bit set has byte-for-byte the legacy encoding. Implementations MAY use any internal representation, but the value committed to the authenticated name tree MUST use this encoding.

## Child-Log Commitment

The child log is a rolling hash chain rather than a Merkle Mountain Range. This permits consensus to verify one append using only the current parent state and the paired child output.

Definitions:

```text
ZERO32 = 32 zero octets
u32le(x) = x serialized as uint32 little-endian
```

The exact domain-separation tags are:

```text
HNS-CHILD-ENTRY-V1
HNS-CHILD-LOG-V1
```

Their hexadecimal encodings are:

```text
484e532d4348494c442d454e5452592d5631
484e532d4348494c442d4c4f472d5631
```

`serialize_output(child_output)` is the canonical non-witness serialization of that complete Handshake transaction output: value, address, and covenant, exactly as the output appears inside a transaction.

`serialize_outpoint(parent_prevout)` is the 36-octet transaction-input outpoint encoding: the 32-octet transaction hash followed by the output index as `uint32` little-endian.

For a paired child output:

```text
output_hash = BLAKE2b-256(
    serialize_output(child_output)
)

entry_hash = BLAKE2b-256(
    ASCII("HNS-CHILD-ENTRY-V1") ||
    parent_hash ||
    serialize_outpoint(parent_input.prevout) ||
    u32le(child_output_index) ||
    output_hash
)

new_child_log_count = old_child_log_count + 1

new_child_log_hash = BLAKE2b-256(
    ASCII("HNS-CHILD-LOG-V1") ||
    old_child_log_hash ||
    u32le(new_child_log_count) ||
    entry_hash
)
```

A name with `child_log_count == 0xffffffff` cannot register another child.

The commitment proves that a supplied ordered history is complete when the history recomputes to the authenticated terminal hash and count. It does not permit enumeration or inclusion proofs without the underlying history.

## New Covenant Types

This proposal assigns:

```text
DELEGATE       = 12
REGISTER_CHILD = 13
```

All integer covenant items are fixed-width little-endian values.

### `DELEGATE`

`DELEGATE` is a linked owner covenant. It spends the active parent coin at input index `i` and recreates the parent coin at output index `i`.

Covenant layout:

| Item | Size | Meaning |
|---:|---:|---|
| 0 | 32 | `parent_hash` |
| 1 | 4 | `parent_epoch` |
| 2 | 32 | `new_child_log_hash` |
| 3 | 4 | `new_child_log_count` |
| 4 | 4 | `child_output_index` |

A `DELEGATE` transition is valid only if all of the following hold:

1. The parent input exists at the same index as the `DELEGATE` output.
2. The input prevout equals the current owner outpoint in the parent name state.
3. The parent is active.
4. Items 0 and 1 equal the parent state key and `epoch(parent_state)`.
5. The parent input covenant is an active owner covenant: `REGISTER`, `REGISTER_CHILD`, `UPDATE`, `RENEW`, `FINALIZE`, `DELEGATE`, or `TRANSFER`.
6. The output address and locked value equal the parent input coin's address and value.
7. `child_output_index` identifies an output in the same transaction.
8. The child output index is not linked to an input; it MUST be greater than or equal to the transaction's input count.
9. The referenced output has covenant type `REGISTER_CHILD`.
10. The referenced `REGISTER_CHILD` identifies the same `parent_hash`.
11. Every `DELEGATE` references exactly one child output, and every `REGISTER_CHILD` is referenced by exactly one `DELEGATE`.
12. `new_child_log_count` equals the authenticated old count plus one.
13. `new_child_log_hash` equals the child-log computation specified above.

The parent name-state transition:

- Sets the owner outpoint to the `DELEGATE` output.
- Clears a pending transfer, matching the existing ability of `UPDATE` to cancel a transfer.
- Sets `child_log_hash` and `child_log_count` to the committed new values.
- Preserves all other parent state, including resource data, registration height, renewal height, renewal count, hierarchical identity, and existing descendant history.

A `DELEGATE` output may subsequently transition wherever an `UPDATE` output may transition, including to `UPDATE`, `RENEW`, `TRANSFER`, `REVOKE`, or another `DELEGATE`.

### `REGISTER_CHILD`

`REGISTER_CHILD` is an unlinked creation covenant.

Covenant layout:

| Item | Size | Meaning |
|---:|---:|---|
| 0 | 32 | `child_hash` |
| 1 | 4 | `child_generation` |
| 2 | 32 | `parent_hash` |
| 3 | 1–63 | `child_label` |
| 4 | 0–512 | `resource_data` |

The output address becomes the initial child owner.

A `REGISTER_CHILD` output is valid only if all of the following hold:

1. It is referenced exactly once by a valid `DELEGATE` in the same transaction.
2. Its output index is greater than or equal to the transaction's input count.
3. `child_label` is one canonical immediate child label.
4. The parent state's wire length and the child label produce a valid complete wire length no greater than 255.
5. `child_hash` equals the recursive derivation from `parent_hash` and `child_label`.
6. The output value is at least `1 HNS` (`1,000,000` base units).
7. `resource_data` is no more than 512 octets.
8. If no state exists at `child_hash`, `child_generation` is 1.
9. If state already exists at `child_hash`, it is a hierarchical tombstone with the same `parent_hash` and `child_label`, it is eligible for re-registration under the existing expiration or revocation-maturity rules, and `child_generation` equals the stored generation plus one.
10. A non-hierarchical state at `child_hash` makes the registration invalid.
11. A generation of zero is invalid. A child at generation `0xffffffff` cannot be re-registered.

On confirmation, the child state is initialized for a new lifecycle as follows:

```text
name           = child_label
name_hash      = child_hash
height         = containing block height
renewal        = containing block height
owner          = child output outpoint
value          = child output value
highest        = 0
data           = resource_data
transfer       = 0
revoked        = 0
claimed        = 0
renewals       = 0
registered     = true
expired        = false
weak           = false
hierarchical   = true
parent_hash    = parent_hash
generation     = child_generation
wire_length    = computed child wire length
```

The child's existing `child_log_hash` and `child_log_count`, if any, are preserved across re-registration. For a first registration they are the implicit empty values.

`resource_data` is copied exactly, including when it is empty. The child enters the active closed state immediately. No auction is performed because the active parent already controls allocation within its immediate namespace.

## Sovereign-Child Lifecycle

A sovereign child uses the existing owner lifecycle covenants:

```text
UPDATE
RENEW
TRANSFER
FINALIZE
REVOKE
```

It may also use `DELEGATE` to register an immediate sovereign child.

The existing covenant byte layouts remain unchanged. Contextual validation changes as follows:

- Item 1 MUST equal `epoch(name_state)`.
- For a sovereign child, item 1 is therefore the generation rather than a block height.
- `REGISTER_CHILD` and `DELEGATE` are treated as owner name covenants for linked-input transition and coin-selection rules.
- An active `REGISTER_CHILD` output may transition to `UPDATE`, `RENEW`, `TRANSFER`, `REVOKE`, or `DELEGATE`.
- A `DELEGATE` output has the same subsequent transitions.
- A `TRANSFER` output may transition to `DELEGATE`, which cancels the transfer in the same manner as `UPDATE`.
- In `FINALIZE`, item 2 is the immediate label. For a sovereign child, non-contextual validation applies the child lexical rules without the root-only blacklist. Contextual validation MUST require that the label equals the stored child label and that hashing it with the stored `parent_hash` yields the name-state key. Root-name `FINALIZE` validation is unchanged.
- Existing transfer lockup, renewal timing, renewal-block commitment, resource-size, signature, value-preservation, and revocation rules continue to apply.

If a sovereign child expires or reaches revocation maturity, the then-current active owner of its immediate parent may register the same child hash with the next generation. Until that point, the parent cannot replace or reclaim it.

Descendants survive the expiration, revocation, transfer, and re-registration of every ancestor.

## Independence From Ancestors

After child registration:

- An ancestor cannot update or revoke the child.
- Transferring an ancestor does not transfer the child.
- Expiring or revoking an ancestor does not expire or revoke the child.
- Re-registering an ancestor does not remove the child.
- A later owner of an ancestor acquires that namespace subject to all active sovereign descendants.
- The child renews and transacts without spending any ancestor coin.
- An active child may register descendants even if a more distant ancestor is inactive.

The only parent-controlled transition after registration is re-registration following the child's own expiration or mature revocation.

## Resolution

A resolver implementing this proposal treats sovereign name coins as an overlay that takes precedence over ordinary DNS data supplied by ancestors.

For a canonical query, the resolver derives every suffix hash from right to left.

For:

```text
www.alice.acme.
```

the candidate sovereign names are:

```text
acme
alice.acme
www.alice.acme
```

The resolver MUST inspect authenticated state for every candidate. It MUST NOT stop because an intermediate ancestor is absent or inactive; a deeper sovereign descendant may remain active.

The deepest active candidate is the authoritative sovereign boundary.

Once a boundary is selected:

- Its resource data controls resolution at and below that boundary.
- No ancestor resource or ordinary DNS data may override it.
- Failure, emptiness, or malformed resource data at that boundary MUST NOT cause fallback to an ancestor.
- Empty resource data represents an existing sovereign boundary with no referral and SHOULD produce an authenticated NODATA response rather than be treated as nonexistence.
- Malformed resource data SHOULD produce `SERVFAIL`.
- Ordinary DNS resolution may continue below a valid referral contained in the selected boundary's resource.
- An active deeper sovereign descendant takes precedence over ordinary DNS data beneath a shallower sovereign boundary.

Dynamic ICANN fallback or root-name fallback is considered only after every Handshake candidate has been checked and no active sovereign boundary exists.

A validating light client selecting a shallower boundary MUST verify:

1. Inclusion and active state for the selected boundary; and
2. Inclusion showing inactivity, or non-inclusion, for every deeper candidate suffix.

A proof for only the selected ancestor is insufficient because an omitted deeper child would otherwise be able to be shadowed.

## Block Limits, Covenant Size, and State Growth

Each child registration performs two authenticated name-state changes:

- One for `DELEGATE`.
- One for `REGISTER_CHILD`.

For block accounting:

- Each `DELEGATE` counts as one name update.
- Each `REGISTER_CHILD` counts as one name update and one name renewal.
- Existing counters for later child lifecycle operations are unchanged.

Both new covenant types MUST be explicitly included in the name-operation counting functions. The current update limit is 600 per block, so a block containing no other name updates can contain at most 300 child registrations.

A single parent spend can authorize exactly one child. Multiple registrations by the same parent require chained transactions, which may appear in the same block in dependency order.

At maximum field sizes:

```text
DELEGATE serialized covenant size       = 83 octets
REGISTER_CHILD Covenant.getSize()       = 650 octets
REGISTER_CHILD serialized covenant size = 652 octets
```

The post-activation known-covenant sanity rules MUST permit the 650-octet `REGISTER_CHILD` payload. Legacy generic unknown-covenant limits need not be changed.

Every registration locks at least `1 HNS` under the existing name-coin lifecycle; that value cannot later be recovered as an ordinary payment. Expiration or revocation can therefore leave a permanently unusable output while a later generation locks a new output. This requirement does not eliminate state-growth risk. Every first child registration creates a permanent authenticated tombstone, and every generation consumes UTXO history, block space, proof bandwidth, and renewal activity.

No auction, burn, per-parent child limit, or separate registration tax is introduced by this proposal.

## Parent Encumbrance Disclosure

Because sovereign descendants survive parent transfer and expiration, wallets, auction interfaces, and marketplaces implementing this proposal MUST disclose that a name may be encumbered by sovereign descendants.

A complete child-log disclosure is the ordered sequence of registration records sufficient to recompute the authenticated `child_log_hash` and `child_log_count`. Each record includes the parent input prevout, child output index, and complete serialized child output.

A buyer can:

1. Recompute the terminal child-log commitment.
2. Reject an incomplete or reordered history.
3. Derive every historically issued child hash from the supplied outputs.
4. Inspect current authenticated state for those children.

The commitment does not make history self-retrieving. A seller, archival service, or indexed full node must supply the sequence. Full nodes MAY maintain non-consensus parent-to-child and descendant indexes for discovery and user interfaces.

## Security Considerations

### Permanent namespace allocation

A parent owner can create a sovereign child that the parent cannot later revoke while it remains active. A compromised or malicious parent key can therefore create durable namespace encumbrances.

### State exhaustion

The minimum locked value and block limits bound throughput and impose cost, but they do not make persistent state free of denial-of-service risk. The economic value of the `1 HNS` threshold SHOULD be reviewed before activation.

### Parent shadowing

Resolvers must check every candidate suffix and must not fall back after selecting an active child. Any implementation that checks only the root or falls back on child failure restores effective control to an ancestor.

### Replay across registrations

The generation is the lifecycle epoch for a sovereign child. It invalidates pre-signed transactions from an older registration and prevents an expired owner from acting after re-registration.

### Historical-data availability

The rolling commitment detects omission only when a purported history is supplied. It does not guarantee that delegation history remains available. Archival and indexing services are therefore operationally important for parent valuation and discovery.

### Hash security

Name identity relies on SHA3-256 collision resistance. Child-log integrity relies on BLAKE2b-256 collision and second-preimage resistance. A collision between an existing root state and a proposed child state causes child registration to fail.

### Reorganizations

A `DELEGATE` and its paired `REGISTER_CHILD` are in one transaction and must be applied or reverted atomically. Undo data must restore parent ownership, transfer state, child-log state, and the complete child lifecycle or tombstone state.

### Deep names

The 255-octet DNS wire limit bounds recursive depth, but resolution and proof work still grow linearly with the number of labels.

## Rationale

### Native state instead of an Escher tree

HIP-0016 keeps subdomain ownership in a per-parent secondary tree committed through Update Chains. This proposal instead accepts greater global authenticated state in exchange for:

- Independent child liveness.
- Native Handshake ownership proofs.
- Independent transactions and renewal.
- Ordinary Handshake resource data.
- Survival across ancestor lifecycles.
- Recursive delegation without a secondary-tree transaction format.

### Rolling commitment instead of an MMR

An MMR root and leaf count are not sufficient to verify a one-leaf append. Consensus would also need authenticated peaks or an append witness.

The rolling commitment makes every append constant-size and directly verifiable from the current parent state and paired output. Its tradeoff is that it supports complete-history verification, not logarithmic inclusion proofs.

### Generation instead of registration height

The containing block height is unknown when a transaction is constructed and signed. Using it as the child covenant epoch would make the paired parent commitment dependent on an unknown future value or require height-locked transaction construction.

A generation is known before signing, remains stable throughout one child lifecycle, and prevents stale transaction replay after re-registration. The containing height remains available separately for renewal and expiration.

### No child auction

The active parent already owns allocation authority over its immediate namespace. An auction would add delay and cost without resolving a competing root-level claim.

### Independent expiration

A child that depends on parent renewal remains ultimately controlled by the parent. Sovereign children therefore retain independent renewal state and survive ancestor expiration.

### Child-label blacklist

The current blacklist protects special root names and ICANN fallback behavior. Applying it at every depth would unnecessarily prohibit ordinary labels such as `test` and `localhost` beneath privately controlled parents. Child labels therefore inherit the lexical rules but not the root-only blacklist.

## Backward Compatibility and Activation

This proposal changes covenant interpretation, authenticated name-state encoding, name lifecycle validation, operation counting, and resolver behavior. It requires a hard fork.

The new meanings of covenant types 12 and 13 apply only to outputs created in blocks at or after the activation height.

An output of type 12 or 13 created before activation remains a legacy unknown-covenant output and MUST retain its pre-activation spend semantics. Implementations must use the coin's creation height when interpreting such an input. This rule prevents pre-existing unknown outputs from being retroactively converted into name coins.

Existing root name states require no eager migration. They have the implicit empty child log and retain their legacy byte encoding until a child is registered.

Before activation:

- Mainnet history MUST be scanned for existing uses of covenant types 12 and 13.
- Independent consensus implementations MUST agree on all serialization and transition vectors.
- Full-node, wallet, miner, mempool, RPC, proof, and resolver implementations MUST be available.
- Reorganization and database-migration behavior MUST be tested.
- Empty-resource, malformed-resource, ancestor-inactive, and descendant-survival resolution cases MUST be tested.
- Operation-count limits and maximum covenant sizes MUST be tested at exact boundaries.
- Wallets, exchanges, registrars, marketplaces, resolvers, and archival/index services MUST have an upgrade path.

The placeholder `HIP-xxxx` remains until a HIP number is assigned by a repository maintainer.

## Test Vectors

All hexadecimal digests are displayed in direct byte order, not reversed transaction-ID display order.

### Name hashes

```text
root label ASCII:
  acme
  61636d65

SHA3-256("acme"):
  51d5d86f52c390f55c86ee644e76fc12
  c0fcef6a5e2d442e9177d94bf8cb98dc

child label:
  alice
  616c696365

hash("alice.acme"):
  7cb0cc70575350eec510f40f5c6cc7e8
  5956ffb431516ea58662da1482778e5f

child label:
  www
  777777

hash("www.alice.acme"):
  ce94161fb4b7d4145a213772117bc65a
  21472d6a357cc20b24da4300dd486803
```

Wire lengths, including the terminal zero octet:

```text
acme.             6
alice.acme.      12
www.alice.acme.  16
```

### Child-log primitive

This vector uses the `acme` parent hash above and a synthetic one-octet serialized child output `00`. The synthetic output is not a valid transaction output; it isolates the hash primitive.

```text
old_child_log_hash:
  00000000000000000000000000000000
  00000000000000000000000000000000

old_child_log_count:
  0

parent input outpoint:
  hash  = 32 zero octets
  index = 00000000

child_output_index:
  01000000

serialize_output(child_output):
  00

output_hash:
  03170a2e7597b7b7e3d84c05391d139a
  62b157e78786d8c082f29dcf4c111314

entry_hash:
  ae9944e9a9a2186b7a3b95eab70e2fc5
  eda5cb8e5237d1079e81b0c58b818d49

new_child_log_count:
  1

new_child_log_hash:
  8d310c032269f42fd8771451a71bd646
  2dde0d4a8e888b5a254a3456d09ab7d7
```

## References

- [HIP-0015: Update Chains](https://github.com/handshake-org/HIPs/blob/master/HIP-0015.md)
- [HIP-0016: Escher — Decentralized Subdomains](https://github.com/handshake-org/HIPs/blob/master/HIP-0016.md)
- [Handshake Improvement Proposals](https://github.com/handshake-org/HIPs)
- [Handshake covenant rules](https://github.com/handshake-org/hsd/blob/698e252ebc7b5c1dd0a9587e342fdd153d020ae4/lib/covenants/rules.js)
- [Handshake NameState](https://github.com/handshake-org/hsd/blob/698e252ebc7b5c1dd0a9587e342fdd153d020ae4/lib/covenants/namestate.js)
- [Handshake covenant primitive](https://github.com/handshake-org/hsd/blob/698e252ebc7b5c1dd0a9587e342fdd153d020ae4/lib/primitives/covenant.js)
- [RFC 1035: Domain Names — Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)
- [RFC 2119: Key words for use in RFCs](https://www.rfc-editor.org/rfc/rfc2119)
- [RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words](https://www.rfc-editor.org/rfc/rfc8174)
