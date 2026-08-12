# HIP-xxxx: HRM/HNSA Profile for Handshake P2P Rendezvous

```text
Number:   HIP-xxxx
Title:    HRM/HNSA Profile for Handshake P2P Rendezvous
Type:     Standards Track
Status:   Draft
Authors:  Jaron Rosenau <@denuoweb>
Created:  2026-08-03
Requires: Handshake Resource Manifests (draft HIP)
          Named Service Authority Profile for Handshake Resource Manifests
          (draft HIP)
          Handshake P2P Rendezvous and Authenticated Service Relay (draft HIP)
```

## Abstract

This document defines the named-service adapter between Handshake Resource
Manifests (HRM), Handshake Named Service Authority (HNSA), and Handshake P2P
Rendezvous and Authenticated Service Relay (HNSR).

HRM and HNSA authenticate a stable named service, delegate that service to an
operational key, and allow the service key to authorize short-lived endpoint
keys. HNSR discovers those endpoints and carries opaque application streams
through relays. This adapter defines a versioned named route record that binds
the protocols without changing the existing unnamed `HNS_NODE_V1` route or
making a relay, rendezvous node, manifest host, or DNS server an identity
authority.

The complete route chain is:

```text
authenticated current HNS name state
        -> hrm1 commitment
        -> controller-signed current HRM envelope
        -> hns.named-service/v1 resource
        -> current HRM delegation to a service key
        -> HNSA EndpointDelegationV1
        -> HNSR NamedRouteRecordV3
        -> one or more HNSR RelayTicketV1 objects
        -> endpoint-authenticated application session
```

The adapter introduces no consensus rule, permanent packet assignment, new
bootstrap network, browser permission, or mandatory public gateway.

## Goals

- consume the HRM/HNSA authority chain without introducing another manifest or
  on-chain root-key record;
- keep unnamed Handshake node reachability wire compatible;
- keep route lookup stable across HRM, service-key, endpoint, relay, and
  provider rotation;
- avoid embedding a potentially large complete HRM envelope in every route;
- let relays forward opaque traffic without becoming service authorities;
- permit bounded route storage when a storing peer does not resolve the HNS
  name or retrieve the HRM;
- require clients to validate current authenticated HNS and HRM state before
  using a route;
- support read-only mobile and browser clients that do not mine, relay, store
  routes, or publish endpoints; and
- give each application profile control over capabilities, constraints,
  framing, lifetimes, and browser policy.

## Non-goals

This document does not define HRM Core, HNSA resource semantics, an application
protocol, username syntax, payment schema, pool-statistics schema, HTTP
gateway, TLS policy, miner control API, wallet workflow, or public profile
number. It does not make route availability proof of service honesty, uptime,
or reputation.

## Requirements language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in BCP 14
when, and only when, they appear in all capitals.

## Versioning and assignments

This adapter reuses the HNSR envelope, rendezvous opcodes, relay reservation,
relay ticket, circuit, flow-control, and error encodings. It adds no packet
opcode.

HRM-backed named routes use experimental values:

```text
route record version = 3
authority type       = 2
```

Version `2`, authority type `1`, identifies the superseded experimental
`hsa1`-backed named route. The new values prevent those bytes from being
reinterpreted as an HRM-backed chain.

Unnamed `HNS_NODE_V1` routes continue to use route-record version `1`,
authority type `0`, and the endpoint-key self-authorization defined by HNSR.
Implementations MUST NOT reinterpret any of the three formats as another.

The `profile_id` carried by an HNSR reservation, relay ticket, route, and
circuit is the HNSA application profile ID in the named-service identifier.
Profile ID zero is invalid. Profile ID `1` remains reserved for unnamed
`HNS_NODE_V1` and MUST NOT be used by a named route. Application profiles MUST
use documented private values during development until an assignment is
accepted.

## Stable named route key

The rendezvous key for a named service is:

```text
BLAKE2b-256(
    "HNSR-NAMED-ROUTE-V1\0"
    || network_magic_u32le
    || name_hash
    || service_name_length_u8
    || canonical_service_name
    || profile_id_u16le
)
```

The inputs are the exact HNSA named-service identity fields. The route key does
not change when an HRM sequence, controller, service delegation, endpoint key,
relay, reservation, address, or hosting provider changes. It is a lookup key,
not an authentication key, and reveals only a deterministic hash of an
identity a requester already knows.

For the deterministic regtest identity used by the earlier implementation:

```text
network_magic = 0xae3895cf
name_hash      = 0f repeated 32 times
service_name   = pool-stats
profile_id     = 0xff00
route_key      = 7e1a513c71518f69164fdcc754202a769
                 e8cbd2dd980da3fd231b9b0de90e60b
```

The route-key calculation is unchanged by the HRM migration. New fixtures MUST
pin the complete version-3 route, HRM resource and delegation IDs, endpoint
delegation, tickets, and signatures.

## Named route record

```text
NamedRouteRecordV3 {
    version:                    u8
    authority_type:             u8
    route_key:                  u8[32]
    profile_id:                 u16
    record_sequence:            u64
    issued_at:                  u64
    expires_at:                 u64
    service_resource_id:        u8[32]
    service_delegation_id:      u8[32]
    service_generation:         u64
    service_controller_key:     u8[33]
    endpoint_delegation_length: u16
    endpoint_delegation:        u8[endpoint_delegation_length]
    ticket_count:               u8
    tickets:                    RelayTicketV1[ticket_count]
    endpoint_signature_length:  u8
    endpoint_signature:         u8[endpoint_signature_length]
}
```

All integers use little-endian encoding. `endpoint_delegation` is the complete
canonical HNSA `EndpointDelegationV1`, including its service signature.

The route deliberately carries HRM-derived IDs, generation, and the service
controller key instead of a complete HRM envelope. The compact fields allow a
storage node to check internal bindings and signatures. They are not an HRM
proof: a requester MUST retrieve and validate the current HRM and match every
field against it.

Let `canonical_route_body` be the exact bytes from `version` through the final
canonical relay ticket, excluding `endpoint_signature_length` and
`endpoint_signature`. The endpoint-signature digest is:

```text
BLAKE2b-256(
    "HNSR-HRM-HNSA-ROUTE-RECORD-V3\0"
    || canonical_route_body
)
```

The signature MUST be canonical strict-DER, low-S secp256k1 and verify under
the endpoint key in the embedded HNSA endpoint delegation.

## Canonical and resource limits

- `version` MUST equal `3` and `authority_type` MUST equal `2`;
- `record_sequence` and `service_generation` MUST be nonzero;
- `service_controller_key` MUST be a valid compressed secp256k1 key;
- the service resource ID, delegation ID, generation, and controller key MUST
  match the embedded endpoint delegation and its verifying key where
  applicable;
- `issued_at` MUST be less than `expires_at`;
- a route lifetime MUST be at most 7,200 seconds and MAY be reduced by the
  application profile;
- the route MUST NOT begin before the endpoint delegation;
- the route MUST NOT expire after the endpoint delegation;
- `ticket_count` MUST be 1 through 8;
- duplicate canonical tickets are invalid;
- every ticket MUST bind the route network, profile, and endpoint key;
- every ticket MUST be active for the complete route lifetime;
- the complete encoded route MUST be at most 8,192 bytes;
- the endpoint delegation MUST satisfy HNSA's 320-byte bound;
- signatures MUST be 1 through 80 bytes; and
- noncanonical lengths, unsupported versions, unknown authority types,
  malformed keys or signatures, and trailing bytes MUST be rejected.

An application profile MUST define a maximum route lifetime no greater than
7,200 seconds, allowed service-resource flags, allowed and required endpoint
capabilities, the expected detached constraints hash, inner-session
authentication, framing, and resource limits.

## Client validation

A requester MUST complete all of the following before sending application
data:

1. Decode the route and enforce its canonical and allocation bounds.
2. Derive the HNS name, canonical service name, and application profile ID
   expected by the user or application. The route MUST NOT choose a different
   expected identity.
3. Recompute and match the named route key.
4. Obtain sufficiently current authenticated HNS namestate under local
   finality policy and select its one current canonical `hrm1` commitment.
5. Retrieve and validate the committed current HRM envelope under HRM Core,
   including its controller signature, subject, network, sequence, validity,
   resources, delegations, and current-snapshot rules.
6. Construct the expected HNSA named-service identifier and resource ID, then
   select and validate exactly one current `hns.named-service/v1` resource.
7. Select and validate exactly one current HNSA service-controller delegation,
   including its ID, generation, controller key, rights, constraints, and
   interval.
8. Match the route's profile ID, service resource ID, service delegation ID,
   service generation, and service controller key to that verified HRM state.
9. Validate the complete endpoint delegation against the current service
   delegation, current time, capabilities, detached constraints, and service
   controller signature.
10. Validate route sequence, time interval, profile limits, and endpoint-
    delegation lifetime containment.
11. Validate every relay ticket, including network, profile, endpoint key,
    reservation, address, limits, time interval, relay signature, and endpoint
    confirmation. Duplicate tickets are rejected.
12. Validate the endpoint signature over the complete canonical route body.
13. Establish the profile-defined endpoint-authenticated inner session before
    accepting application bytes.

Failure at any step MUST fail closed. A requester MUST NOT substitute an
`hsa1` authorization, stale HRM, unauthenticated endpoint, directory result,
DNS answer, relay identity, or conventional-web endpoint under the same
HRM/HNSA identity.

The requester MUST repeat HRM/HNSA validation when its accepted HNS namestate
changes, the accepted HRM validity interval ends, or another profile-defined
cache limit expires.

## Relay and rendezvous behavior

A relay authenticates reservations and signs tickets using the HNSR rules. It
MUST enforce an explicit allowlist of supported profile IDs before allocating
reservation or circuit state. Support for `HNS_NODE_V1` does not imply support
for any named profile, and support for one named profile does not enable
another.

Relays forward opaque bytes and do not need the HRM. A relay ticket is
reachability evidence, not authority over the named service.

A rendezvous node that does not retrieve current HRM state MUST still enforce:

- canonical bounded parsing;
- route, endpoint-delegation, and ticket time limits;
- consistency of all duplicated IDs, generations, profile IDs, and keys;
- the endpoint-delegation signature under the claimed service controller key;
- relay-ticket signatures and endpoint confirmations;
- the route signature under the delegated endpoint key; and
- per-key, global, per-source, byte, and verification-rate admission limits.

Those checks establish internal consistency, not name authority. A rendezvous
node MAY additionally retrieve and validate the current HRM/HNSA chain. Every
requester performs that current-state validation regardless of storage-node
policy.

Named routes MUST NOT be returned by the unnamed `SAMPLEROUTES` operation.
They are returned only for an explicit keyed lookup. A rendezvous response is
untrusted input and does not attest that a service is authorized or online.

## Replacement and conflict handling

`record_sequence` is a route-publication counter scoped to:

```text
(route_key, endpoint_key)
```

It is independent of the HNSA endpoint sequence so an endpoint can refresh
routes and relay tickets without issuing a new delegation. Publishers MUST
persistently reserve a new nonzero sequence before signing. Crash gaps are
safe; reuse is not.

A storage node MUST replace a record only with a greater sequence for the same
route and endpoint key. Equal sequences with different canonical bytes are a
conflict and MUST fail closed. Several currently authorized endpoint keys MAY
coexist under one route key for redundancy.

An HRM sequence or service-generation change invalidates routes whose bound
resource, delegation, generation, or controller no longer matches the current
manifest, even if their local route expiry has not yet passed.

## Browser and read-only observer behavior

A mobile browser, browser extension, wallet, or monitoring client MAY discover
and verify a named service without advertising an HNSR role, accepting inbound
circuits, mining, publishing a route, or storing records for other peers.

The security origin is the HNSA tuple:

```text
(network_magic, name_hash, canonical_service_name, application_profile_id)
```

HRM retrieval locations, relay addresses, public gateways, controller keys,
endpoint keys, and route sequences MUST NOT change or merge that origin.

An application profile MAY expose a signed read-only representation through a
direct or conventional web endpoint for clients that cannot open an HNSR
circuit. The representation MUST carry or identify enough canonical signed
objects for the client to validate its application payload and current
HRM/HNSA chain. The serving URL, TLS connection, extension package, or
downloaded script MUST NOT be treated as the trust root. A client that only
parses structure and does not perform cryptographic and current-chain
validation MUST label the result unverified.

HRM/HNSA/HNSR authorization does not grant local-network, VPN, device,
persistent-background, wallet-signing, mining, or value-transfer permission.
Those remain explicit application or operating-system decisions.

## Availability, privacy, and abuse considerations

An HNSR route may be available while the current HRM envelope is unavailable.
In that case the requester cannot complete authorization and MUST fail closed.
Publishers SHOULD replicate HRM envelopes through independent content-addressed
or authenticated retrieval paths; HNSR itself MAY be one such untrusted
transport.

Keyed lookup exposes the route key and requester-to-rendezvous relationship.
Relays observe connection timing, byte counts, and both outer peers, but cannot
authenticate or modify a correctly protected inner session. Application
profiles SHOULD minimize public fields, use short lifetimes, support several
rendezvous paths, and avoid stable identifiers not required by validation.

Storage and signature verification are denial-of-service surfaces. Nodes MUST
bound bytes before allocation, candidate count before expensive verification,
records per key and source, total records, verification rate, HRM retrieval,
and response size. HNSR work MUST be scheduled below direct Handshake consensus
and block-propagation traffic.

## Compatibility and transition

This adapter does not alter the unnamed route format currently implemented for
HNSR. Existing unnamed nodes can continue publishing and consuming version-1,
authority-type-0 records while implementations add version-3,
authority-type-2 named routes.

The earlier named-route experiment used version `2`, authority type `1`, an
`hsa1` TXT root, and an embedded fixed `ServiceAuthorizationV1`. Those objects
are not HRM/HNSA objects and MUST NOT be accepted by this adapter, converted
implicitly, used as fallback, or share application/browser identity with an
HRM-backed route. An implementation MAY retain them only behind an explicitly
selected experimental compatibility mode.

An even earlier HNSR draft described an `hnsr1` TXT root and HNSR-specific
named authorization domains. Those objects are also outside this adapter and
receive no implicit conversion or fallback.

No permanent mainnet assignment is requested while HRM, HNSA, this adapter,
and application profiles remain Draft.

## Deployment gates

1. Publish exact positive and negative vectors for the HRM resource and
   delegation, route key, version-3 record, signatures, malformed lengths,
   wrong networks, wrong identities, expiry, capability failures, and
   equal-sequence conflicts.
2. Demonstrate deterministic Rust and JavaScript HRM/HNSA decoding and
   verification before enabling named routes.
3. Exercise multi-relay publication, HRM replacement, service-controller and
   endpoint rotation, relay failure, route expiry, and chain reorganization on
   regtest.
4. Demonstrate a read-only mobile client and browser extension that validate
   the same HRM-backed signed application snapshot without taking a network
   role.
5. Measure HRM retrieval, lookup latency, signature-verification cost, storage
   churn, block-propagation impact, and failure behavior under load.
6. Complete independent security and browser-origin review before requesting
   public profile or wire assignments.

## References

1. RFC 2119, *Key words for use in RFCs to Indicate Requirement Levels*.
2. RFC 8174, *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*.
3. Draft HIP, *Handshake Resource Manifests*.
4. Draft HIP, *Named Service Authority Profile for Handshake Resource
   Manifests*.
5. Draft HIP, *Handshake P2P Rendezvous and Authenticated Service Relay*.
