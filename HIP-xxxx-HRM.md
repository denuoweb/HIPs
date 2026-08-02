# HIP-xxxx: Handshake Resource Manifests

```text
Number:  HIP-xxxx
Title:   Handshake Resource Manifests
Type:    Standards Track
Status:  Draft
Authors: Jaron Rosenau <@denuoweb>
Created: 2026-08-01
Related: HIP-0015, HIP-0016, Handshake P2P Rendezvous and
         Authenticated Service Relay (draft HIP)
Follow-ons: Handshake IP and Routing Resource Profile (planned HIP)
            Handshake Service and Transport Resource Profile (planned HIP)
            Handshake Link and Overlay Resource Profile (planned HIP)
```

## Abstract

This document specifies Handshake Resource Manifests (HRMs), an optional
protocol for binding a Handshake name to a signed, content-addressed description
of Internet resources, delegations, controller keys, and validity constraints.

An HRM separates authorization from transport. Handshake authenticates a small
commitment published by the current name owner. The committed manifest may be
retrieved from any untrusted storage or transport. Resource-specific profiles
then determine whether a claimed resource was originated legitimately, whether
a child resource is contained by a parent resource, and which actions a
delegation permits.

Version 1 uses ordinary Handshake `TXT` resource records and does not change
Handshake consensus, DNS resolution, covenant validation, or the interpretation
of any globally assigned number. It does not make an IP prefix routable, create
an Autonomous System Number (ASN), assign a port or protocol number, or replace
the policies of IANA, the Regional Internet Registries (RIRs), the IETF, or
IEEE. Globally coordinated resources require a proof profile rooted in an
authority already accepted for that resource. HNS-local overlay resources may
use profiles that explicitly permit origination by an HNS controller.

The initial protocol is deliberately limited to a reusable manifest,
commitment, signature, delegation, and verification format. IPv4/IPv6, ASN,
routing, service, and link-layer semantics are specified by separate profiles.
This allows independent implementations to agree on the security boundary
before any HRM data is used to generate RPKI objects, routing policy, service
configuration, or other operational output.

## Plain-language summary

Handshake currently proves who controls a name. It does not provide a standard
way for that owner to say:

```text
this key operates my network
this IPv6 prefix was legitimately delegated to me
this smaller prefix is delegated to this community
this service key may accept connections for this service
this authorization expires at this time
```

An HRM adds that common language.

The name owner places a small hash on Handshake. The larger document is stored
elsewhere and can be copied by anyone. A client retrieves the document, checks
that its hash matches Handshake, verifies its controller signature, and then
verifies the authority proof or parent delegation for the resource it needs.

The design has two important limits:

1. Owning an HNS name does not prove ownership of an unrelated public IP
   prefix, ASN, port, EtherType, or other externally coordinated resource.
2. A valid HRM is authorization evidence, not a command to a router, operating
   system, certificate implementation, or network operator.

Adapters may translate a successfully verified HRM into ordinary configuration,
but those adapters remain subject to local policy and the rules of the protocol
or registry they affect.

## Motivation

Internet resource control is recorded across many systems. IP prefixes and ASNs
are distributed hierarchically through IANA and the RIRs. Protocol parameters
are defined through IETF processes and recorded in IANA registries. Link-layer
identifiers may be coordinated by IEEE. Service keys, overlay identifiers, and
internal delegations are often kept in private databases.

These systems solve different problems and should not be treated as one policy
domain. They nevertheless repeat several operational functions:

- associate a resource with a controller;
- delegate a subset or a limited right;
- rotate operational keys;
- publish validity periods;
- revoke previous authority;
- provide an audit trail;
- expose current data to relying software.

Handshake can provide a common, independently verifiable control plane for
those functions without redefining the resources carried in packet headers.
The desired division is:

```text
resource policy and initial authority
        |
        v
recognized external proof or HNS-local origin rule
        |
        v
Handshake-anchored resource manifest
        |
        v
signed subdelegations and controller keys
        |
        v
opt-in adapters for RPKI, routing, services, overlays, and applications
```

This division is useful even if IANA, an RIR, IEEE, or another existing body
remains the source of the parent allocation. HRM can make later delegations
portable, auditable, and verifiable without requiring every resource holder to
operate a bespoke registry database or trusted API.

## Scope

This HIP defines:

- an on-chain HRM commitment carried by an existing HNS `TXT` record;
- a deterministic CBOR envelope and payload;
- controller signatures;
- resource entries and stable resource identifiers;
- external, HNS-local, and parent-delegated authority modes;
- parent-to-child delegations;
- current-state, expiry, transfer, and revocation behavior;
- a profile interface for resource-specific validation;
- a verification algorithm and minimum implementation limits.

This HIP does not define:

- allocation policy for any existing public registry;
- a complete IPv4, IPv6, ASN, RPKI, BGP, port, protocol, TLS, or IEEE profile;
- transient peer routes or relay reservations;
- a mandatory storage or retrieval network;
- automatic router or operating-system configuration;
- an HNS consensus change or new namestate data version;
- a new Handshake P2P message, service bit, port, or protocol number;
- economic policy for allocating HNS-local resources.

## Four-HIP program and dependency order

This document is the first of four intended, independently reviewed HIPs. The
three companion HIPs are not optional sections of HRM Core and do not receive a
HIP number through this document. Each must be proposed, implemented, tested,
reviewed, and assigned its own number separately.

```text
1. Handshake Resource Manifests (this HIP)
        common commitment, envelope, controller, delegation, and validation
        |
        +--> 2. Handshake IP and Routing Resource Profile
        |       IPv4/IPv6, ASN, RPKI proofs, prefix containment,
        |       routing rights, and read-only routing adapters
        |
        +--> 3. Handshake Service and Transport Resource Profile
        |       service identifiers, TCP/UDP/QUIC bindings, TLS and
        |       WireGuard keys, endpoint rights, and HNSR authorization
        |
        +--> 4. Handshake Link and Overlay Resource Profile
                virtual link identifiers, device/controller keys,
                overlay segments, and explicitly scoped link-layer use
```

The companion HIPs stack on the data model and verification algorithm defined
here. They MUST NOT weaken HRM Core's distinction between an HNS-local resource
and an externally coordinated public resource.

### Planned HIP 2: IP and routing

The second HIP should define canonical IPv4, IPv6, and ASN resource encodings;
RPKI-based external origin proofs; prefix containment; routing and
subdelegation rights; proof refresh and revocation; and an audit-only adapter
that compares HRM results with current RPKI and registry state.

It must not claim that an HNS name, auction, or transaction creates a routable
prefix or ASN. Any production route-generation behavior requires a later,
explicit deployment decision after multi-operator testing.

### Planned HIP 3: Services and transports

The third HIP should define HNS-local service identifiers, bindings to existing
TCP, UDP, QUIC, TLS, WireGuard, and similar transports, controller-key roles,
endpoint rights, and the relationship between long-lived HRM authority and
short-lived HNSR route records.

It must distinguish HNS-local service identifiers from IANA-assigned ports,
protocol numbers, TLS parameters, and other public wire values. Use of a public
wire value requires an existing assignment or coordination with its responsible
standards community.

### Planned HIP 4: Links and overlays

The fourth HIP should define identifiers and delegations for explicitly opt-in
virtual links and overlay networks, including virtual segments, device or
controller keys, membership rights, and containment between overlay authorities.

It must not reinterpret globally coordinated IEEE identifiers on ordinary
Ethernet. Any EtherType, OUI, or other public link-layer assignment must come
from the responsible registry or operate inside a formally assigned extension
space.

The ordering above is a review sequence, not a requirement that every
implementation support every profile. An implementation may support HRM Core
and any accepted subset of companion profiles.

## Requirements language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in BCP 14 when,
and only when, they appear in all capitals.

## Terminology

**HNS name owner**
: The controller of the current Handshake name covenant output.

**Subject**
: The Handshake name identified by the 32-byte name hash in an HRM.

**Manifest controller**
: The key that signs the HRM payload. The manifest controller may be operationally
  separate from the HNS name owner.

**Commitment**
: The HRM `TXT` record in current authenticated Handshake namestate. It contains
  the manifest sequence, envelope hash, and retrieval locators.

**Envelope**
: Deterministic CBOR containing the encoded payload and its signature set.

**Resource**
: An identifier or authority described by a resource profile. Examples may
  include an IPv6 prefix, an ASN, an HNS-local service identifier, or an overlay
  network identifier.

**Resource profile**
: A separate specification that defines the canonical identifier, origination
  proof, containment relation, rights, constraints, and adapter behavior for one
  resource family.

**Origin proof**
: Evidence accepted by a resource profile that starts an authority chain. It is
  either an external proof rooted in a configured trust anchor or an HNS-local
  origin explicitly permitted by the profile.

**Parent delegation**
: A current entry in a valid parent HRM authorizing a child subject and
  controller to exercise rights over a resource or valid subset.

**Relying implementation**
: Software that validates HRMs or uses validated output. Examples include a
  wallet, resolver, HNSR service, RPKI adapter, network controller, or auditor.

## Trust model

HRM has two independent authorization gates:

```text
current HNS name owner commits the envelope hash
                       AND
manifest controller signs the committed payload
```

The current authenticated HNS commitment selects the exact envelope. Only the
current name owner can replace or remove that commitment. The controller
signature proves continuity of the operational key and allows parent
delegations to bind authority to a key rather than silently following a name
sale.

Neither gate proves that a public resource originated legitimately. A resource
entry is valid only when its selected profile also validates one of:

- an accepted external origin proof;
- an HNS-local origin rule defined by that profile; or
- a complete parent-delegation chain ending in one of those origins.

A relying implementation chooses which profiles and external trust anchors it
accepts. Unsupported profiles and unrecognized proof types MUST fail closed.

## Protocol overview

Publishing an HRM consists of these steps:

1. Construct a payload for one HNS subject.
2. Encode it using deterministic CBOR.
3. Sign the domain-separated payload with the manifest controller key.
4. Construct and deterministically encode the envelope.
5. Hash the complete envelope with SHA-256.
6. Store the envelope at one or more retrievable locations.
7. Publish an `hrm1` `TXT` commitment containing the sequence, hash, and
   locators in the current HNS namestate.

Verification reverses those steps:

1. Authenticate current HNS namestate and select its current HRM commitment.
2. Retrieve an envelope from any advertised or locally discovered source.
3. Verify the envelope hash against the commitment.
4. Decode the envelope and payload using deterministic CBOR rules.
5. Match the subject, sequence, network, and validity interval.
6. Verify the controller signature.
7. Validate the requested resource using its selected profile.
8. Recursively validate any parent delegation to a recognized origin.
9. Apply local policy before producing operational output.

## HNS commitment record

### Existing namestate version

Version 1 commitments use the existing version `0` Handshake resource data
format and its `TXT` record. This permits an HRM commitment to coexist with
ordinary `NS`, `DS`, glue, synthesis, and other supported Handshake records.

This HIP does not allocate a new namestate data version under HIP-0015. A future
HIP may define a binary commitment if deployment experience shows that the
existing `TXT` representation is inadequate.

### Record form

An HRM commitment is one HNS `TXT` record whose first character-string is
exactly `hrm1` and whose remaining character-strings contain one field each.

The diagnostic JSON form is:

```json
{
  "type": "TXT",
  "txt": [
    "hrm1",
    "seq=7",
    "hash=sha256:47DEQpj8HBSa-_TImW-5JCeuQeRkm5NMpJWZG3hSuFU",
    "uri=https://registry.example/hrm/47DEQpj8HBSa-_TImW-5JCeuQeRkm5NMpJWZG3hSuFU"
  ]
}
```

The example digest is illustrative and does not commit to a valid example
envelope.

The fields are:

`seq`
: An unsigned 64-bit decimal integer without leading zeroes, except that zero
  is encoded as `0`.

`hash`
: Exactly `sha256:` followed by the unpadded base64url encoding of the 32-byte
  SHA-256 digest of the complete deterministic-CBOR envelope.

`uri`
: A retrieval URI. At least one `uri` field MUST be present for a self-contained
  publication. Additional `uri` fields MAY provide replicas. A relying
  implementation MAY use an envelope learned out of band after matching its
  hash.

All commitment character-strings MUST contain printable ASCII. An individual
string MUST NOT exceed the existing HNS `TXT` string limit, and the complete
Handshake resource data MUST remain within the existing 512-byte consensus
limit.

The order of `uri` fields has no security meaning. Unknown fields MUST be
ignored only when their key begins with `x-`. Any other unknown or duplicate
singleton field makes that commitment invalid.

### Commitment selection

A name SHOULD publish no more than one current `hrm1` record.

If authenticated current namestate contains multiple syntactically valid
`hrm1` records, a verifier MUST select the record with the greatest `seq`. If
two records have the same greatest sequence and different hashes, the HRM state
is ambiguous and verification MUST fail closed. Duplicate records with the same
sequence and hash are equivalent; their locator sets MAY be combined.

Sequence numbers are for cache invalidation and replay resistance. They are not
enforced by Handshake consensus. A publisher MUST increase `seq` for every new
envelope under a subject. A verifier that has previously accepted a greater
sequence at equal or greater chain work SHOULD reject a lower sequence unless
it has detected and accepted a Handshake reorganization according to local
finality policy.

### Authenticated retrieval

An ordinary unauthenticated DNS response containing an HRM commitment is not
sufficient. The relying implementation MUST authenticate the record using one
of:

- locally validated Handshake full-node state;
- a Handshake light-client name proof anchored in accepted chain headers; or
- DNSSEC validation anchored in a locally accepted Handshake root trust path.

The retrieval URI and retrieval server are untrusted. Integrity comes from the
on-chain envelope hash, not from HTTPS, a gateway, a relay, or a content host.

## Deterministic CBOR

The envelope and payload MUST use deterministic CBOR as specified by RFC 8949,
Section 4.2, with these additional restrictions:

- maps MUST use the integer keys assigned by this document or a profile;
- map keys MUST be unique;
- indefinite-length items MUST NOT be used;
- floating-point values MUST NOT be used;
- text strings MUST be valid UTF-8;
- profile identifiers MUST contain only lowercase ASCII letters, digits,
  hyphen, period, and slash;
- decoders MUST reject trailing data after the top-level CBOR item;
- an encoder MUST produce the preferred serialization of every integer and
  length.

An implementation MUST re-encode a decoded object and compare it byte-for-byte
with the received encoding before treating it as canonical. This prevents two
encodings of the same logical object from producing different security
interpretations.

## Envelope format

The envelope is the following CBOR map:

| Key | Name | Type | Required | Meaning |
| ---: | --- | --- | :---: | --- |
| `0` | `payload` | byte string | yes | Complete deterministic-CBOR payload |
| `1` | `signatures` | array | yes | One or more signature objects |

A version 1 envelope MUST contain no other top-level keys.

Each signature object is a CBOR map:

| Key | Name | Type | Required | Meaning |
| ---: | --- | --- | :---: | --- |
| `0` | `algorithm` | unsigned integer | yes | Signature algorithm identifier |
| `1` | `public_key` | byte string | yes | Encoded public key |
| `2` | `signature` | byte string | yes | Signature bytes |

Version 1 assigns algorithm `1` to deterministic secp256k1 ECDSA. For algorithm
`1`, `public_key` MUST be a valid compressed 33-byte secp256k1 public key and
`signature` MUST use strict DER encoding with low-S normalization. Unknown
algorithms MUST NOT be accepted as satisfying a required controller signature.

The signature digest is:

```text
BLAKE2b-256(
    ASCII("HNS-HRM-v1") || 0x00
    || network_magic_u32le
    || payload
)
```

`network_magic_u32le` is the four-byte little-endian encoding of the active
Handshake network magic. This prevents a signature created for mainnet,
testnet, regtest, or another configured network from being replayed as valid on
a different network.

The envelope MUST contain a valid signature made by the controller key declared
in the payload. Extra signatures MAY be present for transition experiments but
have no version 1 authority unless a resource profile assigns them meaning.

## Payload format

The payload is a CBOR map:

| Key | Name | Type | Required | Meaning |
| ---: | --- | --- | :---: | --- |
| `0` | `version` | unsigned integer | yes | Must equal `1` |
| `1` | `subject` | byte string | yes | 32-byte Handshake name hash |
| `2` | `sequence` | unsigned integer | yes | Must equal commitment `seq` |
| `3` | `issued_at` | unsigned integer | yes | Unix time in seconds |
| `4` | `expires_at` | unsigned integer | yes | Unix time in seconds |
| `5` | `controller` | map | yes | Algorithm and public key |
| `6` | `resources` | array | yes | Resource entry objects |
| `7` | `delegations` | array | yes | Delegation objects |
| `8` | `extensions` | map | no | Profile-defined, non-critical data |

`subject` MUST equal the consensus Handshake name hash for the name from which
the commitment was obtained.

`issued_at` MUST be less than `expires_at`. A verifier MUST reject the payload
before `issued_at` or at and after `expires_at`, subject to a locally configured
clock-skew allowance. Profiles MAY impose shorter maximum lifetimes.

`controller` is a CBOR map with key `0` containing the signature algorithm and
key `1` containing the public key. Version 1 requires algorithm `1` and a valid
compressed 33-byte secp256k1 public key.

`resources` and `delegations` are complete current snapshots. Removing an entry
and publishing a greater sequence revokes that entry after the new Handshake
state reaches the verifier's required finality.

## Resource entries

A resource entry is a CBOR map:

| Key | Name | Type | Required | Meaning |
| ---: | --- | --- | :---: | --- |
| `0` | `profile` | text string | yes | Resource-profile identifier |
| `1` | `resource_id` | byte string | yes | 32-byte stable identifier |
| `2` | `identifier` | byte string | yes | Profile-defined canonical resource |
| `3` | `authority` | map | yes | Origin or parent authority object |
| `4` | `not_before` | unsigned integer | yes | Earliest valid Unix time |
| `5` | `expires_at` | unsigned integer | yes | Exclusive expiry Unix time |
| `6` | `attributes` | map | no | Profile-defined attributes |

The profile MUST define:

- canonical encoding of `identifier`;
- calculation and validation of `resource_id`;
- whether HNS-local origination is permitted;
- accepted external authority proof types;
- the subset or containment relation;
- recognized rights and constraints;
- maximum validity periods;
- whether multiple simultaneous authorities are meaningful;
- behavior expected from operational adapters.

The resource validity interval MUST be contained by the payload validity
interval. An implementation MUST NOT assign semantics to an unknown profile.

### Authority objects

The `authority` map has key `0`, `kind`, with one of these values:

| Kind | Name | Meaning |
| ---: | --- | --- |
| `0` | HNS-local origin | The profile permits controller origination |
| `1` | External origin | An external proof establishes origin authority |
| `2` | Parent delegation | A current HRM delegates the resource |

#### HNS-local origin

An HNS-local authority object is `{0: 0}`.

It is valid only if the selected resource profile explicitly permits HNS-local
origination and derives a collision-resistant namespace from the subject name
hash and controller key. A public IPv4 prefix, IPv6 prefix, ASN, IANA protocol
number, IANA port, or IEEE EtherType MUST NOT be originated this way.

#### External origin

An external authority object contains:

| Key | Name | Type | Required |
| ---: | --- | --- | :---: |
| `0` | `kind` | unsigned integer equal to `1` | yes |
| `1` | `proof_profile` | text string | yes |
| `2` | `proof_hash` | 32-byte string | yes |
| `3` | `proof_uris` | array of text strings | yes |

`proof_hash` is the SHA-256 digest of the exact external proof object. The proof
may be retrieved from any source. The resource profile defines its format,
trust anchors, revocation behavior, and how it binds the resource, subject,
controller key, and validity interval.

Merely displaying matching RDAP, WHOIS, registry, or web-page text MUST NOT be
treated as a cryptographic origin proof unless a future profile defines and
secures that exact trust model.

#### Parent delegation

A parent-delegation authority object contains:

| Key | Name | Type | Required |
| ---: | --- | --- | :---: |
| `0` | `kind` | unsigned integer equal to `2` | yes |
| `1` | `parent_subject` | 32-byte string | yes |
| `2` | `parent_resource_id` | 32-byte string | yes |
| `3` | `delegation_id` | 32-byte string | yes |

The verifier MUST retrieve and validate the current HRM for `parent_subject`,
validate `parent_resource_id`, and locate the current `delegation_id`. The
parent delegation must exactly authorize the child subject, child controller,
child resource, requested right, and current time.

## Delegations

A delegation object is a CBOR map:

| Key | Name | Type | Required | Meaning |
| ---: | --- | --- | :---: | --- |
| `0` | `delegation_id` | byte string | yes | 32-byte identifier |
| `1` | `parent_resource_id` | byte string | yes | Controlled parent resource |
| `2` | `child_profile` | text string | yes | Child resource profile |
| `3` | `child_resource_id` | byte string | yes | Child resource identifier |
| `4` | `child_identifier` | byte string | yes | Canonical child resource |
| `5` | `child_subject` | byte string | yes | 32-byte child HNS name hash |
| `6` | `child_controller` | map | yes | Required child algorithm and key |
| `7` | `rights` | array of text strings | yes | Profile-defined rights |
| `8` | `not_before` | unsigned integer | yes | Earliest valid Unix time |
| `9` | `expires_at` | unsigned integer | yes | Exclusive expiry Unix time |
| `10` | `may_subdelegate` | boolean | yes | Child may delegate further |
| `11` | `constraints` | map | no | Profile-defined restrictions |

The delegation interval MUST be contained by both the payload interval and the
parent resource interval. The child resource MUST be equal to or a valid subset
of the parent resource according to the parent profile. Cross-profile
delegation is invalid unless the parent profile explicitly defines the mapping.

`child_controller` uses the same algorithm/key map as the payload controller.
Binding the delegation to both `child_subject` and `child_controller` prevents a
sale or compromise of only one identity component from silently transferring
the delegated resource.

The profile defines the canonical calculation of `delegation_id`. It SHOULD be
the SHA-256 digest of a domain-separated deterministic encoding of every
security-relevant delegation field except `delegation_id` itself.

Removing the delegation from a later parent manifest revokes it. A child copy
of an older delegation never overrides current parent state.

## Name ownership, transfer, and key rotation

An HRM binds a current HNS commitment and a manifest controller key. They serve
different purposes:

- the HNS owner chooses which envelope is current;
- the controller signs the resource state;
- a parent delegation may require that exact controller key.

Ordinary HNS `UPDATE`, `RENEW`, and `TRANSFER` processing does not invalidate an
otherwise current HRM when the exact commitment remains in current namestate.
This matches ordinary DNS-resource continuity and the HNSR root-key model. A new
name owner can remove the commitment, but cannot forge a replacement under the
existing controller key.

A name transfer alone does not give the new owner an externally originated or
parent-delegated resource. Those authority chains remain bound to the manifest
controller. The new owner can leave that controller's exact committed manifest
available or withdraw it, but cannot change its signed contents.

An HNS-local profile MAY define a resource as following current control of the
HNS name. A profile requiring transfer-sensitive invalidation MUST define the
extra binding and its validation rule explicitly; HRM Core does not infer
external resource ownership from a name transfer.

Controller-key rotation has the same rule: HNS-local resources may select a new
controller through a new owner-committed envelope, but external and
parent-delegated resources require a refreshed proof or parent delegation that
binds the new controller.

If the Handshake name is revoked, expired, or has no current owner, its HRM is
invalid.

## Verification algorithm

To authorize action `A` over resource `R` for subject `S`, a version 1 verifier
MUST perform the following steps.

1. Obtain authenticated current Handshake namestate for `S` under a locally
   accepted chain-finality policy.
2. Confirm the name has a current owner and is not revoked or expired.
3. Parse all `hrm1` commitment records and select the current commitment.
4. Retrieve an envelope whose SHA-256 digest matches the commitment.
5. Enforce deterministic CBOR and all envelope structural limits.
6. Decode the payload and confirm:
   1. `version` is `1`;
   2. `subject` equals `S`;
   3. `sequence` equals commitment `seq`;
   4. the signature is evaluated with the active network magic;
   5. the payload is currently within its validity interval.
7. Verify the required controller signature over the exact payload bytes.
8. Locate `R`, reject duplicate resource identifiers, and validate its profile,
   canonical identifier, resource identifier, attributes, and validity.
9. Validate its authority:
   1. for HNS-local origin, confirm the profile permits origination and derives
      the resource namespace correctly;
   2. for external origin, retrieve and verify the exact proof under configured
      trust anchors and current revocation state;
   3. for parent delegation, recursively validate the current parent resource
      and exact current delegation.
10. Confirm every link authorizes action `A`, every child is contained by its
    parent, every time interval is contained by its parent, and every required
    constraint is satisfied.
11. Apply local operational policy before returning authorization.

A verifier MUST detect subject/resource cycles. It MUST impose a recursion limit
of no more than 32 parent links and SHOULD permit operators to configure a lower
limit. A cycle, exceeded limit, unavailable required parent, ambiguous state,
unknown critical profile, failed proof, or expired link MUST fail closed.

## Current state and revocation

HRM is a current-state protocol. Handshake history provides an audit trail but
does not keep removed authority active.

To revoke a resource or delegation, the publisher increments the sequence,
removes the entry from the complete snapshot, signs the new payload, publishes
the new envelope, and updates the commitment. The revocation takes effect for a
verifier when that verifier accepts the new Handshake state under local finality
policy.

Short emergency expiry periods can reduce exposure when a publisher cannot
immediately update the chain. Resource profiles SHOULD define maximum validity
periods appropriate to their operational risk.

A verifier MAY cache a successful result only until the earliest of:

- payload expiry;
- resource expiry;
- delegation expiry;
- external-proof expiry or revocation refresh;
- locally configured cache limit;
- observation of a new commitment or relevant Handshake reorganization.

## Availability and retrieval

Handshake commits to HRM integrity but does not guarantee delivery of the
envelope or external proofs.

Publishers SHOULD provide multiple independent locators. Content-addressed
storage, HTTPS, HNSR, peer-to-peer storage, local operator mirrors, or other
transports MAY be used. A mirror does not need publisher permission because the
committed hash authenticates the bytes.

Failure to retrieve a current required object means the authorization is
unavailable and MUST NOT be treated as valid. An implementation MUST NOT fall
back to an older envelope merely because the current object cannot be fetched.

Transient HNSR route records, relay reservations, live socket addresses, and
similar presence data SHOULD NOT be placed in HRM. HRM establishes longer-lived
controller and resource authority; a rendezvous protocol may use that authority
to authenticate its own short-lived records.

## Resource profiles

Profiles are separate specifications so a defect or policy dispute in one
resource family does not redefine the HRM core.

A profile specification MUST provide:

1. A stable lowercase profile identifier.
2. Canonical resource encoding.
3. Resource-ID calculation.
4. Origin modes and accepted proof profiles.
5. Parent-child containment rules.
6. Rights, constraints, and subdelegation rules.
7. Time and expiry requirements.
8. Conflict and multiple-origin behavior.
9. At least one positive and negative deterministic test vector.
10. Security, privacy, and operational-adapter considerations.

The planned companion HIPs are expected to define profiles including:

`hns.overlay.service/v1`
: HNS-local, collision-resistant service identifiers and controller keys.

`hns.overlay.network/v1`
: HNS-local identifiers for explicitly opt-in virtual networks.

`rpki.ip-prefix/v1`
: IPv4/IPv6 resources rooted in a cryptographically validated RPKI proof, with
  canonical prefix containment and subdelegation.

`rpki.asn/v1`
: ASN authority rooted in an accepted RPKI proof.

`hnsr.service/v1`
: Longer-lived authorization keys and rights consumed by HNSR named-service
  rendezvous. Short-lived HNSR routes remain in HNSR.

Names above are illustrative and are not allocated by this HIP. The three
companion documents identified in the four-HIP program must assign their exact
profile identifiers and semantics.

## Operational adapters

An adapter consumes a verified HRM result and produces output for another
system. Examples might include:

- a proposed RPKI certificate or routing authorization;
- a BGP import-policy input;
- a network-controller configuration fragment;
- an HNSR named-service authorization;
- a WireGuard or TLS trust configuration;
- an overlay network membership decision;
- an audit report comparing HRM and an existing registry.

Version 1 adapters MUST be opt-in and SHOULD default to audit or dry-run mode.
An adapter MUST NOT treat an HNS commitment alone as authority over a public
resource. It MUST report the complete validated proof chain and the local policy
decision that caused operational output.

No adapter should directly change public routing, filtering, certificate trust,
or another security-sensitive system without an additional operator policy and
an explicit deployment specification.

## Security considerations

### False public-resource claims

Any HNS owner can publish arbitrary bytes. Therefore an HRM without a valid
profile-specific authority chain is only a claim. Public-resource profiles MUST
validate a recognized external origin or complete parent delegation.

### Name squatting

Similarity between an HNS name and an organization, registry entry, domain, or
resource label does not establish authority. Validation is based on the
resource proof chain, not human-readable resemblance.

### Name transfer

An unchanged commitment may survive a name transfer, but externally originated
and parent-delegated authority remains bound to the manifest controller. The new
name owner may withdraw the manifest and cause denial of service; it cannot
alter the signed state or satisfy a controller-bound authority chain. HNS-local
profiles must state explicitly whether their resources follow the name.

### Controller compromise

A compromised controller can sign malicious manifests only if the HNS owner
also commits them, but an entity may operate both keys. Publishers SHOULD keep
the HNS owner key and manifest controller key in separate security domains.
High-risk profiles SHOULD use short expiry and explicit recovery procedures.

### HNS owner compromise

A compromised name owner can withdraw or replace the commitment and cause
denial of service. It cannot satisfy a controller-bound parent delegation or
external proof without the required controller authority. Operational systems
must nevertheless plan for the resulting unavailability.

### Replay and rollback

Sequence, current authenticated namestate, validity intervals, and local
chain-finality policy limit replay. Implementations must distinguish a genuine
Handshake reorganization from an envelope replay at equal or lower chain work.

### Locator equivocation

Locators may return different bytes, track clients, refuse service, or serve
malware. Hash validation prevents undetected content substitution but not
denial of service or metadata collection.

### External proof revocation

An unchanged HRM may outlive an external allocation or proof. Every external
profile MUST define current revocation and refresh behavior. A cached HRM result
must not outlive the proof state on which it depends.

### Delegation cycles and exhaustion

Attackers may construct cycles, deep graphs, large manifests, repeated
locators, or expensive proof chains. Implementations MUST bound object size,
retrieval count, redirects, total bytes, signature operations, recursion depth,
and validation time.

Suggested default limits for an initial implementation are:

| Item | Default maximum |
| --- | ---: |
| Envelope bytes | 1 MiB |
| Resources per manifest | 1,024 |
| Delegations per manifest | 4,096 |
| Locators attempted per object | 4 |
| Parent depth | 16 |
| Total fetched objects per decision | 64 |
| Total fetched bytes per decision | 8 MiB |

Implementations MAY impose smaller limits. A profile requiring larger values
must justify them explicitly.

### Parser differentials

Security depends on independent implementations hashing and interpreting the
same bytes identically. Deterministic CBOR restrictions, byte-for-byte
re-encoding, strict profile canonicalization, and shared negative test vectors
are REQUIRED before a profile is used for operational authorization.

### Unsafe automation

A cryptographically valid statement can still violate an operator's business,
safety, routing, or abuse policy. HRM validation and operational acceptance are
separate decisions. Adapters must preserve that boundary.

## Privacy considerations

HRMs are public or publicly retrievable metadata. They may reveal network
structure, customer relationships, service inventory, controller rotation,
organizational hierarchy, and planned expiry times.

Publishers SHOULD disclose only information needed for verification. Private
addresses, personal device identifiers, internal topology, transient routes,
and individual customer data SHOULD NOT be published unless the profile and
participants explicitly accept that disclosure.

Retrieval can reveal which resource a client is interested in. Mirrors,
content-addressed caches, privacy relays, or locally synchronized datasets may
reduce that leakage.

## Backwards compatibility

This proposal uses existing valid HNS version `0` resource data and `TXT`
records. Legacy HNS nodes, miners, resolvers, and wallets require no consensus
change and may ignore `hrm1` records.

HRM-aware software is an optional consumer. Names may continue using ordinary
DNS records. The 512-byte HNS resource limit remains unchanged.

An implementation MUST NOT reinterpret unrelated TXT records as HRM data. Only
a record beginning with the exact `hrm1` marker is in scope.

## Relationship to HIP-0015 and HIP-0016

HIP-0015 describes Handshake name updates as authenticated update chains and
allows future application-defined namestate versions. HRM uses the same general
idea of an HNS name anchoring changing application state, but version 1 remains
inside the existing DNS resource format for compatibility.

HIP-0016 demonstrates how an off-chain authenticated data structure may commit
its root through HNS while light clients verify selected data. A future HRM
scaling proposal may replace the single-envelope snapshot with a provable map
or accumulator. That is not required for HRM version 1.

## Relationship to HNSR

The draft Handshake P2P Rendezvous and Authenticated Service Relay protocol
defines short-lived discovery and relay routes. HRM and HNSR are independent:

```text
HRM
    who controls a resource or long-lived service authority

HNSR
    where an authorized endpoint is reachable right now
```

A future `hnsr.service/v1` profile may allow HNSR to consume HRM controller and
delegation proofs. HRM does not require HNSR for retrieval, and HNSR does not
require HRM for unnamed node rendezvous.

## Implementation plan

The first reference implementation should be a small library and CLI, not a
consensus patch.

Recommended components are:

1. A Rust crate that encodes, signs, decodes, and validates HRM Core objects.
2. A command-line tool with `create`, `sign`, `publish`, `fetch`, `inspect`, and
   `verify` operations.
3. An HNS adapter that reads and writes the version `0` commitment record.
4. Deterministic positive and negative vectors shared with a JavaScript
   implementation.
5. A first HNS-local overlay-service profile.
6. A read-only experimental IP-prefix profile that compares HRM results with
   existing RPKI state and never changes routing.

`hsd` and `hnsd` need no consensus modification for the core experiment. Wallet
support may improve publication ergonomics but is not required for validation.

## Deployment gates

HRM should advance in independently reviewed stages.

### Stage 0: Core vectors

- finalize CBOR encodings;
- publish controller-signature and commitment vectors;
- test malformed and non-canonical inputs across implementations.

### Stage 1: HNS-local pilot

- use only identifiers whose profiles are explicitly scoped to opt-in HNS
  overlays;
- test owner transfer, controller rotation, revocation, expiry, reorganization,
  data loss, and parent delegation;
- run on regtest and testnet before mainnet publication.

### Stage 2: Multi-operator retrieval

- replicate envelopes over independent transports;
- measure availability, latency, cache behavior, and denial-of-service cost;
- perform independent security review.

### Stage 3: External-resource audit

- use legitimately allocated test resources;
- validate but do not originate routing or registry changes;
- compare HRM delegation results with RPKI and registry state;
- document conflicts and revocation latency.

### Stage 4: Profile and adapter proposals

- submit each operational profile separately;
- require multi-operator evidence and negative test vectors;
- coordinate any wire-level assignment with the responsible standards and
  registry community;
- keep production-changing adapters opt-in until their own threat models and
  rollback procedures are accepted.

No permanent IANA, IEEE, HNS P2P, or other wire assignment is requested by this
HIP.

## Test requirements

Before this proposal can move beyond Draft, the repository should contain
deterministic vectors covering at least:

- canonical payload and envelope encoding;
- valid and invalid deterministic secp256k1 controller signatures;
- commitment-hash match and mismatch;
- HNS subject and active-network match and mismatch;
- commitment/payload sequence match and mismatch;
- selection among multiple commitments;
- equal-sequence conflicting commitments;
- payload, resource, and delegation expiry;
- name transfer with commitment retention, removal, and replacement;
- controller-key mismatch;
- valid one-level and multi-level delegation;
- invalid containment, rights escalation, and forbidden subdelegation;
- missing and revoked parent delegation;
- cycle and recursion-limit rejection;
- valid and invalid external proof stubs;
- corrupt, unavailable, and equivocating retrieval sources;
- accepted and rejected Handshake reorganization cases.

External-resource profiles require their own additional vectors.

## Rationale

### Why use an on-chain hash instead of placing the manifest on chain?

Handshake resource data is limited to 512 bytes. Internet resource proofs,
delegation graphs, certificates, and metadata can be substantially larger.
Committing only a hash keeps consensus state bounded and allows untrusted
replication.

### Why use `TXT` instead of a new namestate version?

`TXT` works under current consensus and can coexist with ordinary name
resolution. A new version would require ecosystem coordination before the core
security model has deployment evidence.

### Why require a controller signature when HNS already commits the hash?

The second key separates durable name custody from operational resource control
and gives parent delegations a stable cryptographic target. It also prevents a
name transfer alone from silently acquiring delegated infrastructure authority.
Version 1 uses the same key and signature family as the draft HNSR protocol so
an HRM service profile can reuse an existing HNSR root key.

### Why not accept any resource claimed by an HNS owner?

Many public resources already have globally coordinated meanings. Treating an
HNS auction as ownership of an existing IP prefix, ASN, port, protocol number,
or EtherType would create collisions that browsers and application gateways
cannot repair.

### Why separate resource profiles?

An IPv6 prefix has containment semantics; an ASN usually does not. A port has
different policy and operational consequences from a virtual service ID. A
single generic validator cannot safely guess these rules.

### Why make manifests complete snapshots?

Current snapshots make removal an unambiguous revocation and avoid requiring a
client to replay an unbounded off-chain log. Handshake history remains
available for auditing.

## Alternatives considered

### Independently allocate public wire values through HNS

Rejected for HRM Core. Existing routers, switches, kernels, and protocol
implementations require coordinated meanings. HNS-local profiles may allocate
values only inside explicit overlays or formally delegated extension spaces.

### Store unverified claims in plain TXT records

Rejected as an authorization protocol. Plain text is useful for discovery but
does not define canonical encoding, controller continuity, authority proofs,
delegation containment, expiry, or revocation.

### Put every resource and delegation directly on Handshake

Rejected for version 1 because of the 512-byte limit, chain growth, privacy,
proof size, and the burden placed on unrelated full nodes.

### Require a single HRM storage network

Rejected. Content addressing makes storage replaceable. Mandating one delivery
network would recreate the availability dependency HRM is intended to avoid.

### Make HRM immediately authoritative for router configuration

Rejected. Verification formats, external trust roots, operational policy,
rollback, and multi-operator experience must be established first.

## Open questions

The following items should be resolved during Draft review:

- whether a later version should register Ed25519 or another optional controller
  algorithm in addition to mandatory secp256k1;
- whether one retrieval URI should be required or out-of-band retrieval should
  be valid without an advertised URI;
- whether a future compact binary commitment should replace or supplement TXT;
- exact default clock-skew and Handshake-finality guidance;
- registry and review process for resource-profile identifiers;
- whether controller rotation should gain a cross-signature mechanism in Core;
- which HNS-local profile is small enough to serve as the first complete pilot;
- whether future selective disclosure should use an authenticated map based on
  HIP-0016, another accumulator, or independent signed objects.

## References

1. RFC 2119, *Key words for use in RFCs to Indicate Requirement Levels*.
2. RFC 8174, *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*.
3. RFC 8949, *Concise Binary Object Representation (CBOR)*.
4. RFC 4648, *The Base16, Base32, and Base64 Data Encodings*.
5. RFC 7020, *The Internet Numbers Registry System*.
6. RFC 6480, *An Infrastructure to Support Secure Internet Routing*.
7. RFC 9323, *A Profile for RPKI Signed Checklists (RSCs)*.
8. RFC 8126, *Guidelines for Writing an IANA Considerations Section in RFCs*.
9. HIP-0015, *Update Chains*.
10. HIP-0016, *Escher Update Chains for decentralized subdomains*.
11. Draft HIP, *Handshake P2P Rendezvous and Authenticated Service Relay*.
12. Handshake developer documentation, *Resource Records*.
13. IANA, *Governance of the IANA Functions*.
14. IANA, *Number-related Registries*.
15. IANA, *Protocol Parameter Assignments*.
