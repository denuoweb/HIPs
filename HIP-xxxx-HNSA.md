# HIP-xxxx: Named Service Authority for Handshake

```text
Number:  HIP-xxxx
Title:   Named Service Authority for Handshake
Type:    Standards Track
Status:  Draft
Authors: Jaron Rosenau <@denuoweb>
Created: 2026-08-01
Related: Handshake P2P Rendezvous and Authenticated Service Relay
         (draft HIP)
```

## Abstract

This document specifies Handshake Named Service Authority (HNSA), an optional
protocol that allows the owner of a Handshake name to authorize separate keys
for named application services without transferring the name or exposing its
wallet key to an online service.

A name opts in by publishing one service-authority root key and epoch in its
existing HNS `TXT` resource data. The root key signs bounded service
authorizations. A service key may then authorize short-lived endpoint keys used
by a transport or discovery profile such as HNSR, direct QUIC, HTTPS, messaging,
or another HNS-aware application.

The authority chain is:

```text
current authenticated HNS name state
        |
        v
service-authority root key and epoch
        |
        v
root-signed named service authorization
        |
        v
service-signed endpoint delegation
        |
        v
profile-specific endpoint or route record
```

HNSA defines identity and delegation, not transport. It does not define relay
routing, DNS resolution, HTTP framing, browser UI, IP-address allocation,
routing policy, or a general registry for non-service resources. Those concerns
remain in the service profile or protocol that consumes the authorization.

Version 1 uses current Handshake resource records and requires no consensus
change, hard fork, new namestate version, permanent P2P message assignment, or
new public Internet number.

## Plain-language summary

An HNS name is normally controlled by a wallet key that should remain private
and mostly offline. A website, chat service, mobile endpoint, or hosting
provider needs a different key that can be used by online software.

HNSA gives the name owner a standard way to say:

```text
this root key may authorize services for my HNS name
this service key operates my web service
these endpoint keys may serve that service for a limited time
```

A client retrieves those signed objects from any discovery mechanism and
verifies the complete chain against current HNS name state. A relay, hosting
provider, directory, or endpoint cannot substitute its own service key without
an authorization from the HNS name.

## User stories

### Mobile or home hosting

As the owner of `alice/`, Alice authorizes a `web` service key and one or more
endpoint keys for her phone, home server, and optional VPS. An HNS-aware browser
can reach any currently available endpoint while treating them as the same
named service.

### Provider delegation

Alice authorizes a hosting provider to operate `web` without giving the provider
her HNS wallet key or control of `alice/`. She can replace the provider by
publishing a new service authorization under her service-authority root.

### Key rotation after compromise

Alice replaces a compromised service key while keeping the HNS name and browser
identity unchanged. Clients reject endpoint records that do not lead to a
currently valid service authorization.

### Multiple services under one name

Alice authorizes `web`, `chat`, and `files` with independent keys, profiles, and
lifetimes. Compromise or migration of one service does not require rotating the
keys for every other service or transferring the HNS name.

### Stable browser identity

A user opens an HNS service whose direct address, relay, or provider has
changed. The browser preserves the same origin and permissions because identity
is based on the HNS name, service name, and profile rather than the selected
network path.

## Motivation

Handshake authenticates root names and DNS resource data. It does not currently
provide an implementation-independent authorization format for application
services operated beneath those names.

Without a separate service-authority layer, an application tends to choose one
of these designs:

- use the name wallet key directly in online service software;
- invent a new TXT key format for every application;
- trust the endpoint returned by a directory or relay;
- equate an IP address or hosting account with service identity;
- place provider-specific credentials in name data;
- define a complete authorization chain inside each transport proposal.

These approaches either expose durable name custody, duplicate security rules,
or bind identity to infrastructure that changes more frequently than the HNS
name.

HNSA introduces two deliberate delegation boundaries:

```text
name custody                   service operation
HNS wallet -> root key         root key -> service key

service operation              endpoint reachability
service key -> endpoint key    endpoint key -> route or transport record
```

The name wallet only needs to update HNS when the root key or authority epoch
changes. The root key may remain offline and sign relatively long-lived service
authorizations. Service keys and endpoint keys can rotate on shorter schedules
without exposing the name wallet.

## Scope

This HIP defines:

- a canonical HNS `TXT` record for the service-authority root key and epoch;
- canonical service names;
- the stable identity tuple for a named service;
- `ServiceAuthorizationV1`;
- `EndpointDelegationV1`;
- signature domains and canonical encodings;
- expiry, sequence, replacement, and emergency revocation behavior;
- validation rules and implementation limits;
- the interface between HNSA and service profiles;
- browser-origin and permission requirements for profile specifications.

This HIP does not define:

- an endpoint discovery or storage network;
- HNSR rendezvous, relay tickets, circuits, or transport frames;
- DNS, HTTP, TLS, QUIC, WireGuard, or messaging wire behavior;
- an application-specific endpoint record;
- a generic Internet resource manifest;
- IP prefixes, ASNs, RPKI, BGP, ports, protocol numbers, or EtherTypes;
- automatic operating-system or network configuration;
- allocation policy or economic rules for services;
- unnamed Handshake peer authority.

## Requirements language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in BCP 14 when,
and only when, they appear in all capitals.

## Terminology

**HNS name owner**
: The controller of the current Handshake name covenant output.

**Root key**
: The service-authority key committed in current authenticated HNS name data.
  It signs service authorizations and should normally remain offline.

**Authority epoch**
: An integer in current HNS name data. Changing it invalidates all service
  authorizations issued under a previous epoch, even if the root key is reused.

**Named service**
: An application service identified by an HNS name hash, canonical service
  name, and profile ID.

**Service key**
: A profile-specific operational key authorized by the root key for exactly one
  named service.

**Endpoint key**
: A short-lived or device-specific key authorized by a service key. A service
  profile defines how the endpoint key authenticates its route or transport.

**Service profile**
: A separate specification defining the meaning of a profile ID, endpoint
  records, capabilities, constraints, discovery, transport, and browser
  behavior.

**Authorization ID**
: The BLAKE2b-256 hash of a complete canonical service authorization, including
  its root signature.

**Delegation ID**
: The BLAKE2b-256 hash of a complete canonical endpoint delegation, including
  its service signature.

## Design principles

### The HNS name remains the root of identity

A client starts with current authenticated Handshake name state. A service
authorization learned from a relay, web server, peer, QR code, cache, or other
untrusted source has no authority unless it validates under the root key and
epoch currently published by that name.

### Name custody is separate from service operation

The name wallet key does not sign ordinary service or endpoint messages. An HNS
update selects a service-authority root key; that key delegates online work to
service keys.

### Service identity is independent of network path

An IP address, relay, transport connection, hosting provider, or endpoint key is
not the named service identity. Those details may change while the HNS name,
service name, and profile remain stable.

### Profiles define transport semantics

HNSA verifies who may operate a named service. It does not decide how a client
finds or reaches the endpoint. HNSR, direct web, messaging, and future profiles
may carry the same authorization objects over different transports.

### Authorization is bounded

Root-key changes and epoch increments provide an on-chain emergency boundary.
Service and endpoint authorizations also have finite validity periods so stale
objects eventually stop working without a further HNS transaction.

## Service identity

A named service is identified by this tuple:

```text
Handshake network magic
HNS name hash
canonical service name
service profile ID
```

The tuple is stable across root-key rotation, service-key rotation, endpoint
rotation, provider migration, and transport failover.

Version 1 service names:

- are 1 through 63 ASCII bytes;
- contain only lowercase `a-z`, digits, and hyphen;
- MUST NOT begin or end with a hyphen;
- MUST NOT contain period, slash, underscore, whitespace, or percent escapes;
- are compared byte-for-byte without locale processing.

Examples include `web`, `chat`, `files`, `node`, and `p2p-site`.

The service name is an application namespace beneath the HNS root. It is not a
second-level DNS registration and is not independently owned on chain.

## HNS root-key record

### Record form

A name opts into HNSA by publishing exactly one canonical `TXT` record:

```text
hsa1 k=<base32-compressed-secp256k1-public-key> e=<epoch>
```

Diagnostic Handshake resource JSON:

```json
{
  "type": "TXT",
  "txt": [
    "hsa1 k=ak3m... e=4"
  ]
}
```

The abbreviated key above is illustrative and is not a test vector.

Requirements:

- the complete record is one printable ASCII character-string;
- fields are separated by exactly one ASCII space;
- fields appear in the order shown;
- `hsa1` is lowercase and exact;
- `k` is an unpadded lowercase base32 encoding of a valid compressed 33-byte
  secp256k1 public key;
- `e` is an unsigned 32-bit decimal integer without leading zeroes, except zero
  is encoded as `0`;
- unknown, missing, duplicated, or reordered fields make the record invalid;
- more than one syntactically valid `hsa1` record is ambiguous and MUST fail
  closed.

The record uses existing version `0` Handshake resource data and may coexist
with supported `NS`, `DS`, glue, synthesis, and unrelated `TXT` records.

### Authentication

An ordinary unauthenticated DNS response is insufficient. A relying client MUST
authenticate the root-key record using one of:

- locally validated Handshake full-node state;
- a Handshake light-client name proof anchored in accepted chain headers; or
- DNSSEC validation anchored in a locally accepted Handshake root trust path.

### Update, transfer, and revocation

An HNS `UPDATE` that changes or removes the `hsa1` record replaces or revokes the
service-authority root after the client accepts the new safe name state.

Incrementing the epoch invalidates every service authorization containing a
lower epoch. This permits emergency revocation while retaining the same root
key.

A name transfer that leaves the resource data unchanged leaves the existing
root key and epoch in effect until the new owner updates them. This preserves
service continuity while giving the new owner the ability to replace or remove
the authority record.

## Cryptographic primitives

Version 1 uses:

- BLAKE2b-256;
- compressed 33-byte secp256k1 public keys;
- deterministic secp256k1 ECDSA signatures;
- strict DER signature encoding;
- low-S normalization;
- four-byte little-endian Handshake network magic in every signature domain.

A verifier MUST reject invalid public keys, noncanonical DER, high-S
signatures, wrong network magic, unsupported versions, and signatures over any
noncanonical encoding.

All unsigned integer fields use little-endian encoding. Every variable-length
field is preceded by the length shown in its structure. No field may contain
trailing bytes or an alternative encoding.

## Service authorization

The root key authorizes one named service with:

```text
ServiceAuthorizationV1 {
    version:                u8
    network_magic:          u32
    name_hash:              u8[32]
    authority_epoch:        u32
    service_name_length:    u8
    service_name:           u8[service_name_length]
    profile_id:             u16
    service_key:            u8[33]
    flags:                  u16
    serial:                 u64
    valid_from_height:      u32
    valid_until_height:     u32
    max_endpoint_lifetime:  u32
    root_signature_length:  u8
    root_signature:         u8[root_signature_length]
}
```

The root-signature digest is:

```text
BLAKE2b-256(
    "HNS-SERVICE-AUTH-V1\0"
    || network_magic_u32le
    || all remaining unsigned canonical fields
)
```

The `network_magic` field appears in the object and signature input exactly
once. The phrase `remaining unsigned canonical fields` starts with `name_hash`
and ends with `max_endpoint_lifetime`.

Rules:

- `version` MUST equal `1`;
- `network_magic` MUST match the active Handshake network;
- `name_hash` MUST match the name containing the current `hsa1` record;
- `authority_epoch` MUST equal the current `hsa1` epoch;
- `service_name` MUST be canonical;
- `profile_id` MUST be recognized by the relying client;
- `service_key` MUST be a valid compressed secp256k1 public key;
- `flags` MUST contain only bits defined by the selected profile;
- `serial` MUST increase when the root key replaces an authorization for the
  same service identity;
- `valid_until_height` MUST be greater than `valid_from_height`;
- an authorization MUST be rejected outside that block-height interval;
- `max_endpoint_lifetime` MUST be 300 through 604,800 seconds;
- the root signature MUST verify under the current HNSA root key;
- unknown trailing data MUST be rejected.

The profile MAY impose a shorter authorization span or endpoint lifetime.

The authorization ID is:

```text
authorization_id = BLAKE2b-256(
    "HNS-SERVICE-AUTH-ID-V1\0"
    || complete_canonical_service_authorization
)
```

This includes the canonical root signature.

## Endpoint delegation

A service key authorizes an endpoint key with:

```text
EndpointDelegationV1 {
    version:                 u8
    network_magic:           u32
    authorization_id:        u8[32]
    endpoint_key:            u8[33]
    endpoint_sequence:       u64
    issued_at:               u64
    expires_at:              u64
    capabilities:            u32
    constraints_hash:        u8[32]
    service_signature_length:u8
    service_signature:       u8[service_signature_length]
}
```

The service-signature digest is:

```text
BLAKE2b-256(
    "HNS-ENDPOINT-DELEGATION-V1\0"
    || network_magic_u32le
    || all remaining unsigned canonical fields
)
```

Rules:

- `version` MUST equal `1`;
- `network_magic` MUST match the active Handshake network;
- `authorization_id` MUST identify the validated service authorization;
- `endpoint_key` MUST be a valid compressed secp256k1 public key;
- `endpoint_sequence` MUST increase for replacement of the same logical
  endpoint under a profile-defined endpoint identifier;
- `issued_at` MUST be less than `expires_at`;
- lifetime MUST NOT exceed the service authorization's
  `max_endpoint_lifetime`;
- the delegation MUST NOT remain valid after the service authorization's
  block-height expiry;
- `capabilities` MUST contain only bits defined by the selected profile;
- `constraints_hash` is all zeroes when the profile defines no detached
  constraints;
- a nonzero constraints hash MUST commit to the exact canonical profile object;
- the service signature MUST verify under the authorized service key;
- unknown trailing data MUST be rejected.

The delegation ID is:

```text
delegation_id = BLAKE2b-256(
    "HNS-ENDPOINT-DELEGATION-ID-V1\0"
    || complete_canonical_endpoint_delegation
)
```

A service may authorize several endpoints concurrently for redundancy,
geographic distribution, device migration, or transport choice.

## Profile-specific endpoint records

HNSA stops at the endpoint key. A service profile defines the next object used
for discovery or transport.

A profile-specific endpoint or route record MUST bind at least:

- active network magic;
- service authorization ID;
- endpoint delegation ID;
- endpoint sequence;
- endpoint or route expiry;
- profile ID;
- profile-specific locator, route, or session data.

It MUST be signed by the endpoint key and MUST NOT remain valid beyond the
endpoint delegation.

Examples include:

- an HNSR signed route with one or more active relay tickets;
- a direct QUIC endpoint with an authenticated transport key;
- an HTTPS endpoint with profile-defined TLS or DANE binding;
- a messaging endpoint with a delivery key and discovery locator.

Those objects and their wire behavior are outside this HIP.

## Validation algorithm

To validate an endpoint record for named service `S`, a client MUST:

1. Obtain authenticated current HNS state for the root name under local chain
   finality policy.
2. Parse the `hsa1` record and reject missing or ambiguous authority.
3. Parse the service authorization using bounded canonical decoding.
4. Confirm network magic, HNS name hash, authority epoch, service name, profile,
   flags, height interval, and endpoint-lifetime limit.
5. Verify the root signature under the current `hsa1` key.
6. Calculate and match the service authorization ID.
7. Parse the endpoint delegation using bounded canonical decoding.
8. Confirm its authorization ID, capabilities, constraints, sequence, time
   interval, and profile rules.
9. Verify the service signature under the authorized service key.
10. Calculate and match the endpoint delegation ID.
11. Validate the profile-specific endpoint record and its endpoint signature.
12. Apply local browser or application policy before connecting.

A failure at any step MUST fail closed for HNSA authorization. A client MUST NOT
silently replace a failed HNSA chain with an endpoint learned from an
unauthenticated directory, relay, DNS response, or legacy fallback.

## Replacement and freshness

### Root authority

The current authenticated HNS `hsa1` record is authoritative. A key change or
epoch increment invalidates older service authorizations once the new HNS state
is accepted under local finality policy.

### Service authorization replacement

Service authorizations are finite. When several otherwise valid
authorizations for the same service identity are available, a client MUST
select the greatest `serial`. Equal serials with different canonical bytes are
ambiguous and MUST fail closed.

A client cannot know about an unavailable higher serial merely from an older
object. Profiles MUST therefore define discovery replication and maximum
service-authorization lifetimes appropriate to their risk. Immediate global
revocation uses an HNS root-key change or epoch increment.

### Endpoint delegation replacement

Endpoint delegations are short-lived and may overlap for failover. Profiles
must define the logical endpoint identifier used when comparing
`endpoint_sequence` and must bound replay through short route or endpoint
record expiry.

### Caching

A client MAY cache a validated chain only until the earliest of:

- observation of changed HNS name data or a relevant chain reorganization;
- service authorization height expiry;
- endpoint delegation time expiry;
- endpoint or route record expiry;
- profile-specific cache limit.

## Service profiles

A service profile assigns a `profile_id` and MUST specify:

1. User-visible purpose and at least one concrete user story.
2. Meaning of the service authorization `flags`.
3. Meaning of endpoint `capabilities` and detached constraints.
4. Endpoint or route record encoding and signature domain.
5. Discovery and replication behavior.
6. Direct, relayed, and fallback connection policy.
7. Maximum authorization, delegation, and endpoint-record lifetimes.
8. Browser origin, cookies, storage, permissions, and mixed-content behavior
   when the profile is web-facing.
9. Resource, parser, and network limits.
10. Positive and negative deterministic test vectors.
11. Privacy, abuse, and denial-of-service considerations.

Profile IDs are not assigned by this HIP. A profile proposal SHOULD use private
experimental values until its specification, implementations, and test vectors
are accepted.

## Browser behavior

This HIP does not require a URI scheme or browser interface. A web-facing
profile must nevertheless preserve these properties.

### Stable origin

Browser identity and storage MUST be scoped to at least:

```text
Handshake network
HNS name hash
canonical service name
profile ID
```

They MUST NOT be scoped only to an IP address, relay, endpoint key, or hosting
provider. Transport failover must not create a new origin or share storage with
another named service.

### Identity indication

A browser may indicate that the endpoint is authorized by current HNS state.
That indication must not claim that the operator, content, or service is honest
or safe. HNSA authenticates control, not reputation.

### Failure behavior

If the authority chain expires, is ambiguous, changes unexpectedly, or fails a
signature check, the browser must stop before sending application data. Any
legacy or conventional-web fallback must be separately identified and require
explicit profile and user policy.

### Network permissions

A profile that requests VPN, overlay, local-network, device, or persistent
background access must use an explicit browser or operating-system permission.
An HNSA authorization alone does not grant those capabilities.

## Relationship to HNSR

The draft Handshake P2P Rendezvous and Authenticated Service Relay protocol
contains a named-service authorization chain developed for HNSR routes. HNSA
extracts that concept into a transport-independent layer.

The relationship is:

```text
HNSA
    authorizes the named service key and endpoint key

HNSR
    discovers a currently online endpoint and relays its traffic
```

Unnamed HNSR node rendezvous remains independent and does not require an HNS
name or HNSA.

The current HNSR draft uses an `hnsr1` root record and HNSR-specific service and
endpoint signature domains. Adoption of this HIP would require a separate HNSR
revision that either consumes `hsa1` authority directly or defines an explicit
compatibility transition. This HIP does not silently change the existing HNSR
draft or claim wire compatibility before that revision.

## Backwards compatibility

The `hsa1` record is valid existing HNS `TXT` data. Legacy full nodes, miners,
resolvers, wallets, and applications may ignore it.

HNSA-aware clients are optional consumers. Names without a valid `hsa1` record
continue operating under existing Handshake and DNS rules.

The proposal does not reinterpret `hnsr1`, unrelated TXT records, DNSSEC keys,
TLS certificates, or wallet addresses.

## Security considerations

### Wallet-key exposure

Implementations MUST NOT use the HNS name wallet key as a root, service, or
endpoint key merely for convenience. The separation of custody and operation is
a primary security property of this protocol.

### Root-key compromise

A compromised root key can authorize malicious services until the HNS owner
changes the key or increments the epoch. The root key should remain offline or
in hardware-backed storage and should not sign endpoint or transport messages.

### Service-key compromise

A compromised service key can authorize endpoints only for its named service,
profile, validity interval, flags, and endpoint-lifetime limit. It cannot modify
the HNS name or authorize a different service identity.

### Endpoint-key compromise

A compromised endpoint key can impersonate one endpoint until its delegation
and profile records expire. Endpoint lifetimes should reflect how securely the
device can protect its key.

### Replay and stale authorization

Network magic, HNS state, epochs, serials, sequences, finite validity, and
profile record expiry limit replay. Clients must validate current HNS state and
must not accept an older object merely because a current object is unavailable.

### Name transfer

An unchanged HNSA root survives name transfer until the new owner updates the
record. This supports service continuity, but participants must recognize that
the new owner can revoke or replace the entire service authority.

### Ambiguous authority

Multiple valid `hsa1` records, equal service serials with different bytes,
noncanonical names, unknown profiles, or conflicting required constraints must
fail closed.

### Untrusted discovery

Discovery peers, relays, directories, and content hosts may omit, replay, or
reorder authorization objects. They cannot forge the signature chain, but they
can deny service or attempt to keep clients on older still-valid state.

### Downgrade

A failed HNSA connection must not silently become an unauthenticated connection
to the same presentation name. Profiles must define explicit downgrade and
fallback behavior.

### Parser and resource exhaustion

Implementations MUST bound all lengths before allocation and signature work.
Recommended initial maximums are:

| Item | Maximum |
| --- | ---: |
| Service name | 63 bytes |
| Signature | 80 bytes |
| Service authorization | 256 bytes |
| Endpoint delegation | 256 bytes |
| Concurrent service candidates | 16 |
| Concurrent endpoint candidates | 32 |
| Detached constraint object | 64 KiB |

A profile may impose smaller limits. A larger limit requires explicit
justification and tests in that profile.

## Privacy considerations

Service names, service keys, provider changes, endpoint keys, capabilities, and
validity intervals may reveal how a name owner's infrastructure is organized.

Profiles should avoid publishing personal device labels, private addresses,
internal topology, or long-lived correlatable endpoint keys when they are not
required for verification. Short-lived endpoint keys and privacy-preserving
discovery may reduce correlation.

Clients should consider that fetching a service authorization or endpoint
record can reveal which HNS service they intend to use.

## Reference implementation plan

The first implementation should extract the authority objects and validators
from the HNSR research implementation into a transport-independent library.

Recommended deliverables are:

1. Shared canonical encoders and decoders for JavaScript and Rust.
2. Root, service, and endpoint signing and validation APIs.
3. Wallet support for creating and updating the `hsa1` record.
4. A command-line inspection and verification tool.
5. Deterministic positive and negative vectors.
6. An HNSR named-service adapter using HNSA objects.
7. A mobile-browser diagnostic that displays the validated authority chain and
   preserves origin across direct and relayed endpoints.

No `hsd` consensus change is required. HNSR integration, wallet ergonomics, and
browser support remain separate implementation changes.

## Deployment gates

### Stage 0: Canonical vectors

- finalize all binary encodings and signature domains;
- verify byte-identical JavaScript and Rust implementations;
- publish malformed, ambiguous, replayed, and cross-network negative vectors.

### Stage 1: Regtest authority chain

- publish `hsa1` in authenticated name state;
- authorize multiple independent named services;
- rotate service and endpoint keys;
- increment the epoch and verify immediate invalidation;
- transfer the name with and without changing the root record.

### Stage 2: HNSR integration

- carry HNSA service authorization and endpoint delegation with named HNSR
  routes;
- demonstrate direct and relayed endpoints under one stable service identity;
- test route expiry, relay failover, stale authorization, and no legacy
  fallback contact.

### Stage 3: Independent clients and operators

- run multi-operator testnet trials;
- validate the same objects in independent JavaScript and Rust clients;
- measure authorization freshness, retrieval availability, mobile lifecycle,
  and abuse limits;
- complete browser-origin and permission review for any web profile.

No permanent wire assignment or mainnet-default behavior is requested by this
HIP.

## Test requirements

Before this proposal can move beyond Draft, deterministic tests must cover:

- canonical `hsa1` parsing and ambiguity rejection;
- valid and invalid root signatures;
- wrong name hash, epoch, profile, network magic, and height interval;
- valid service replacement and equal-serial conflict;
- valid and invalid endpoint signatures;
- endpoint lifetime beyond the service maximum;
- invalid capabilities and detached constraint hashes;
- root-key rotation and epoch-only revocation;
- service-key and endpoint-key rotation;
- name transfer with retained and replaced root records;
- concurrent endpoints and failover;
- expired and replayed endpoint records;
- malformed lengths, DER signatures, public keys, and trailing bytes;
- stable browser origin across endpoint, relay, and provider changes;
- rejection without unauthenticated fallback.

## Rationale

### Why store a root key in HNS instead of every endpoint?

Endpoint state changes frequently and may exceed the 512-byte HNS resource-data
limit. A root key creates a bounded delegation hierarchy while keeping name
state small and durable.

### Why not use the HNS wallet key directly?

Wallet keys control valuable names and coins and should not be exposed to
online service processes. A separate root key permits service authorization
without weakening name custody.

### Why have both service and endpoint keys?

A service may run on several devices, providers, or relays. The service key
represents the operator role, while short-lived endpoint keys limit the impact
of one device compromise and allow independent rotation.

### Why include an on-chain epoch?

Service authorizations may be replicated through untrusted discovery systems.
An epoch increment gives the HNS owner one current, authenticated mechanism for
invalidating every authorization under an old epoch without relying on those
systems to distribute a revocation list.

### Why use block height for service authorization and time for endpoints?

The service root is verified against Handshake chain state, so block height
provides a consistent validity boundary. Endpoint records are short-lived
network objects whose transports already depend on wall-clock expiry.

### Why keep transport behavior out of this HIP?

Rendezvous, direct web, messaging, and other transports have different routing,
privacy, availability, and abuse models. Combining them with the authority core
would make one proposal responsible for unrelated deployment risks.

## Alternatives considered

### Generic Internet resource manifests

Not included. IP prefixes, ASNs, routing policy, link-layer identifiers, and
other public resources have separate authority systems and no immediate role in
the named-service chain defined here.

### Application-specific TXT keys

Not selected as the general model. A separate key record for every application
duplicates rotation, transfer, expiry, and validation rules and consumes scarce
HNS resource data.

### One online key for all services

Not selected. Independent service keys contain compromise and allow providers
or devices to be changed without replacing unrelated services.

### On-chain endpoint records

Not selected. Mobile and relayed endpoints change too frequently for name
transactions and may require multiple concurrent records.

### A mandatory manifest server

Not selected. Authorization objects are signed and can be carried by any
profile-defined discovery system. Requiring one server would create a new
availability and metadata dependency.

## Open questions

The following items remain for Draft review and implementation evidence:

- exact private profile-ID range for regtest and acknowledged testnet use;
- whether service authorizations need a core maximum block-height span;
- whether service-specific revocation needs an optional on-chain commitment in
  addition to bounded lifetime and global epoch revocation;
- whether a future version should support threshold or hardware-backed root
  keys;
- exact transition from the HNSR draft's `hnsr1` record to `hsa1`;
- whether direct web and HNSR should share one web profile or use separate
  profile IDs with explicit origin relationships;
- default browser presentation for HNS-authorized service identity.

## References

1. RFC 2119, *Key words for use in RFCs to Indicate Requirement Levels*.
2. RFC 8174, *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*.
3. RFC 6979, *Deterministic Usage of the Digital Signature Algorithm*.
4. SEC 1, *Elliptic Curve Cryptography*.
5. Handshake developer documentation, *Resource Records*.
6. Draft HIP, *Handshake P2P Rendezvous and Authenticated Service Relay*.
7. Handshake `hsd` authenticated name-proof and resource validation behavior.
