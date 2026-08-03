# HIP-xxxx: HNSA Profile for Handshake P2P Rendezvous

```text
Number:   HIP-xxxx
Title:    HNSA Profile for Handshake P2P Rendezvous
Type:     Standards Track
Status:   Draft
Authors:  Jaron Rosenau <@denuoweb>
Created:  2026-08-03
Requires: Named Service Authority for Handshake (draft HIP)
          Handshake P2P Rendezvous and Authenticated Service Relay (draft HIP)
```

## Abstract

This document defines the named-service adapter between Handshake Named
Service Authority (HNSA) and Handshake P2P Rendezvous and Authenticated Service
Relay (HNSR).

HNSA authenticates a stable named service and delegates short-lived endpoint
keys. HNSR discovers those endpoints and carries opaque application streams
through relays. This adapter defines a versioned named route record that binds
the two protocols without changing the existing unnamed `HNS_NODE_V1` route
format or making a relay an identity authority.

The complete route chain is:

```text
authenticated current HNS name state
        -> hsa1 root key and epoch
        -> HNSA ServiceAuthorizationV1
        -> HNSA EndpointDelegationV1
        -> HNSR NamedRouteRecordV2
        -> one or more HNSR RelayTicketV1 objects
        -> endpoint-authenticated application session
```

The adapter introduces no consensus rule, permanent packet assignment, new
bootstrap network, browser permission, or mandatory public gateway.

## Goals

- use one transport-independent named-service authority chain;
- keep unnamed Handshake node reachability wire compatible;
- keep route lookup stable across endpoint, relay, and provider rotation;
- let relays forward opaque traffic without becoming service authorities;
- permit bounded route storage even when the storing peer does not resolve the
  HNS name;
- require clients to validate current authenticated HNS state before use;
- support read-only mobile and browser clients that do not mine, relay, store
  routes, or publish endpoints; and
- give each application profile control over capabilities, constraints,
  framing, lifetimes, and browser policy.

## Non-goals

This document does not define an application protocol, pool-statistics schema,
HTTP gateway, TLS policy, miner control API, wallet workflow, or public profile
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

Named routes use:

```text
route record version = 2
authority type       = 1
```

Unnamed `HNS_NODE_V1` routes continue to use route-record version `1`,
authority type `0`, and the endpoint-key self-authorization defined by HNSR.
Implementations MUST NOT reinterpret either format as the other.

The `profile_id` carried by an HNSR reservation, relay ticket, route, and
circuit is the same `profile_id` carried by the HNSA service authorization.
Profile ID zero is invalid. Profile ID `1` remains reserved for unnamed
`HNS_NODE_V1` and MUST NOT be used by a named route. Application profiles MUST
use private values during development until an assignment is accepted.

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

The inputs are the exact HNSA `ServiceIdentity` fields. The route key therefore
does not change when a service authorization, endpoint key, relay, reservation,
address, or hosting provider changes. It is a lookup key, not an authentication
key, and reveals only a deterministic hash of an identity a requester already
knows.

For the deterministic regtest identity used by the Rust implementation:

```text
network_magic = 0xae3895cf
name_hash      = 0f repeated 32 times
service_name   = pool-stats
profile_id     = 0xff00
route_key      = 7e1a513c71518f69164fdcc754202a769
                 e8cbd2dd980da3fd231b9b0de90e60b
```

The implementation fixture also pins the complete canonical route record and
every signature. Independent implementations MUST reproduce that complete
vector before interoperating.

## Named route record

```text
NamedRouteRecordV2 {
    version:                    u8
    authority_type:             u8
    route_key:                  u8[32]
    profile_id:                 u16
    record_sequence:            u64
    issued_at:                  u64
    expires_at:                 u64
    authorization_length:       u16
    service_authorization:      u8[authorization_length]
    endpoint_delegation_length: u16
    endpoint_delegation:        u8[endpoint_delegation_length]
    ticket_count:               u8
    tickets:                    RelayTicketV1[ticket_count]
    endpoint_signature_length:  u8
    endpoint_signature:         u8[endpoint_signature_length]
}
```

`service_authorization` is the complete canonical HNSA
`ServiceAuthorizationV1`, including its root signature.
`endpoint_delegation` is the complete canonical HNSA
`EndpointDelegationV1`, including its service signature.

The endpoint-signature digest is:

```text
BLAKE2b-256(
    "HNSR-HNSA-ROUTE-RECORD-V2\0"
    || all preceding canonical NamedRouteRecordV2 fields
)
```

The signature MUST be canonical strict-DER, low-S secp256k1 and MUST verify
under the endpoint key in the embedded HNSA delegation.

## Canonical and resource limits

- `version` MUST equal `2` and `authority_type` MUST equal `1`;
- `record_sequence` MUST be nonzero;
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
- each embedded HNSA object retains its HNSA size bound;
- signatures MUST be 1 through 80 bytes; and
- noncanonical lengths, unsupported versions, unknown authority types,
  malformed keys or signatures, and trailing bytes MUST be rejected.

An application profile MUST define a maximum route lifetime no greater than
7,200 seconds, allowed service-authorization flags, allowed and required
endpoint capabilities, the expected detached constraints hash, inner-session
authentication, framing, and resource limits.

## Client validation

A requester MUST complete all of the following before sending application
data:

1. Decode the route and enforce its canonical and allocation bounds.
2. Derive the HNSA service identity expected by the user or application. The
   embedded object MUST NOT choose a different expected identity.
3. Recompute and match the named route key.
4. Obtain sufficiently current authenticated HNS name state under local
   finality policy and select exactly one canonical `hsa1` record.
5. Validate the complete HNSA service authorization against that root key,
   epoch, identity, current height, profile, and allowed flags.
6. Validate the complete HNSA endpoint delegation against that authorization,
   current time, capabilities, and detached constraints.
7. Require all profile-defined capability bits.
8. Validate route sequence, time interval, profile limits, and delegation
   lifetime containment.
9. Validate every relay ticket, including network, profile, endpoint key,
   reservation, address, limits, time interval, relay signature, and endpoint
   confirmation. Duplicate tickets are rejected.
10. Validate the endpoint signature over the complete route.
11. Establish the profile-defined endpoint-authenticated inner session before
    accepting application bytes.

Failure at any step MUST fail closed. A requester MUST NOT substitute an
unauthenticated endpoint, directory result, DNS answer, relay identity, or
conventional-web endpoint under the same HNSA identity.

The client MUST repeat the HNSA authorization check when its accepted HNS name
state changes or advances beyond a cached authorization interval.

## Relay and rendezvous behavior

A relay authenticates reservations and signs tickets using the HNSR rules. It
MUST enforce an explicit allowlist of supported profile IDs before allocating
reservation or circuit state. Support for `HNS_NODE_V1` does not imply support
for any named profile, and support for one named profile does not enable
another.

Relays forward opaque bytes and do not need the HNSA root key. A relay ticket
is reachability evidence, not authority over the named service.

A rendezvous node MUST enforce canonical parsing, route-key derivation,
endpoint/delegation/ticket signatures, expiry, size, per-key, global, and
per-source admission limits before retaining a record. It MAY additionally
validate the HNSA root signature against local authenticated HNS state.
Allowing bounded storage without that optional root check prevents HNS name
lookups from becoming a publication bottleneck; it does not weaken requester
validation because every requester performs the complete current-state check.

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

## Browser and read-only observer behavior

A mobile browser, browser extension, or monitoring client MAY discover and
verify a named service without advertising an HNSR role, accepting inbound
circuits, mining, publishing a route, or storing records for other peers.

The security origin is the HNSA tuple:

```text
(network_magic, name_hash, canonical_service_name, profile_id)
```

Relay addresses, public gateways, endpoint keys, and route sequences MUST NOT
change or merge that origin.

An application profile MAY expose a signed read-only representation through a
direct or conventional web endpoint for clients that cannot open an HNSR
circuit. Such a representation MUST carry enough canonical signed objects for
the client to validate its application payload and HNSA chain. The serving URL,
TLS connection, extension package, or downloaded script MUST NOT be treated as
the HNSA trust root. If a client only parses structure and does not perform
cryptographic and current-chain validation, its UI MUST label the data
unverified.

HNSA/HNSR authorization does not grant local-network, VPN, device, persistent
background, wallet, or mining permission. Those remain explicit browser or
operating-system choices.

## Privacy and abuse considerations

Keyed lookup exposes the route key and requester-to-rendezvous relationship.
Relays observe connection timing, byte counts, and both outer peers, but cannot
authenticate or modify a correctly protected inner session. Application
profiles SHOULD minimize public fields, use short lifetimes, support several
rendezvous paths, and avoid stable identifiers not required by validation.

Storage and signature verification are denial-of-service surfaces. Nodes MUST
bound bytes before allocation, candidate count before expensive verification,
records per key and source, total records, verification rate, and response
size. HNSR work MUST be scheduled below direct Handshake consensus and block
propagation traffic.

## Compatibility and transition

This adapter does not alter the unnamed route format currently implemented for
HNSR. Existing unnamed nodes can continue publishing and consuming version-1
records while implementations add version-2 named routes.

The earlier HNSR draft described an `hnsr1` TXT root and HNSR-specific named
authorization domains. Those objects are not HNSA objects and are not accepted
by this adapter. New named deployments use `hsa1` and the exact HNSA encodings.
Implementations may recognize the earlier experiment in a separately selected
test configuration, but MUST NOT silently convert it, use it as fallback, or
share browser identity with HNSA routes.

No permanent mainnet assignment is requested while HNSA, this adapter, and
application profiles remain Draft.

## Deployment gates

1. Publish exact positive and negative vectors for route key, record encoding,
   signatures, malformed lengths, wrong networks, wrong identities, expiry,
   capability failures, and equal-sequence conflicts.
2. Demonstrate deterministic Rust and JavaScript decoding and verification.
3. Exercise multi-relay publication, endpoint rotation, relay failure, route
   expiry, HNSA epoch rotation, and chain reorganization on regtest.
4. Demonstrate a read-only mobile client and browser extension that validate
   the same signed application snapshot without taking a network role.
5. Measure lookup latency, signature-verification cost, storage churn, direct
   block-propagation impact, and failure behavior under load.
6. Complete independent security and browser-origin review before requesting
   public profile or wire assignments.

## References

1. RFC 2119, *Key words for use in RFCs to Indicate Requirement Levels*.
2. RFC 8174, *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*.
3. Draft HIP, *Named Service Authority for Handshake*.
4. Draft HIP, *Handshake P2P Rendezvous and Authenticated Service Relay*.
