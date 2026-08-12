# HIP-xxxx: Named Service Authority Profile for Handshake Resource Manifests

```text
Number:   HIP-xxxx
Title:    Named Service Authority Profile for Handshake Resource Manifests
Type:     Standards Track
Status:   Draft
Authors:  Jaron Rosenau <@denuoweb>
Created:  2026-08-01
Requires: Handshake Resource Manifests (draft HIP)
Related:  HIP-0002, Handshake P2P Rendezvous and Authenticated Service Relay
          (draft HIP)
          HRM/HNSA Profile for Handshake P2P Rendezvous
          (draft HIP)
```

## Abstract

This document specifies Handshake Named Service Authority (HNSA), the first
application-resource profile for Handshake Resource Manifests (HRM).

HNSA allows the owner of a Handshake name to create stable named application
services and delegate each service to a separate operational key without
transferring the name or exposing its wallet key to online software. The
current authenticated HNS name selects an HRM commitment. The committed,
controller-signed HRM contains HNSA named-service resources and HRM
delegations to service keys. A service key may then authorize short-lived
endpoint keys used by HNSR, HTTPS, QUIC, messaging, payment, or another
separately specified application profile.

The authority chain is:

```text
current authenticated HNS name state
        |
        v
HRM hrm1 commitment
        |
        v
controller-signed current HRM envelope
        |
        v
hns.named-service/v1 resource
        |
        v
HRM delegation to the service key
        |
        v
service-signed endpoint delegation
        |
        v
profile-specific endpoint, route, or signed application record
```

HNSA does not define a second manifest format or a parallel on-chain authority
record. In particular, this version defines no `hsa1` record and no
root-signed `ServiceAuthorizationV1` object. The HRM envelope, resource, and
delegation encodings are the sole durable authority format.

## Plain-language summary

An HNS name is normally controlled by a wallet key that should remain private
and mostly offline. Application services need operational keys that can rotate,
move between devices, or be delegated to users and providers.

HRM gives the name owner a signed, content-addressed manifest and a general
delegation model. HNSA defines how one HRM resource means:

```text
this stable application service exists beneath my HNS name
this service key currently operates it
this endpoint key may serve it for a limited time
```

For example, a future payment profile may define the user-facing identifier
`jaron@denuoweb` as the HNSA service tuple:

```text
HNS name      = denuoweb
service name  = jaron
profile       = payment profile
```

The current HRM for `denuoweb` may delegate that service to Jaron's key.
Jaron can then publish profile-defined HNS, BTC, XMR, invoice, or dynamic
payment endpoints without holding the `denuoweb` name wallet key. The payment
syntax and payload are defined by that future payment profile, not by HNSA
Core.

## User stories

### Mobile or home hosting

As the owner of `alice/`, Alice creates a `web` named-service resource and
delegates it to a service key. That service key authorizes endpoint keys for her
phone, home server, and optional VPS. An HNS-aware browser can reach any
currently valid endpoint while treating them as one service.

### Provider delegation

Alice delegates the `web` service to a hosting provider without giving the
provider her HNS wallet key, HRM controller key, or control of unrelated
resources. She replaces the provider by publishing a greater HRM sequence with
a replacement service delegation.

### Application username

A registry or community operating `example/` uses an accepted application
profile in which `jaron@example` maps to service name `jaron`. The current
HRM delegates that exact service/profile tuple to Jaron's key. Jaron controls
its profile-specific endpoints but cannot modify `example/`, another
username, or another profile.

### External wallet destinations

A payment profile authorizes a named service to return destinations for
multiple currencies. HNSA proves which service key may speak for the named
payment identity. The payment profile defines assets, networks, address
formats, invoices, replacement rules, and whether destinations are static
signed records or dynamic endpoint responses.

### Key rotation after compromise

Alice replaces a compromised service key in the complete current HRM snapshot.
Clients reject endpoint delegations that bind the removed service delegation,
even if an untrusted directory continues serving them.

### Multiple services under one name

Alice creates `web`, `chat`, and `files` resources with independent
application profile IDs and service keys. A profile may instead interpret a
service name as a username or another application-local label. Compromise or
migration of one service does not authorize another tuple.

### Stable browser identity

A user opens an HNS service whose direct address, relay, or provider has
changed. The browser preserves the same origin and permissions because identity
is based on the HNS name, service name, and application profile rather than the
selected network path.

## Goals

HNSA version 1 is intended to:

- be a strict HRM resource profile rather than a competing manifest system;
- separate HNS name custody, HRM control, service operation, and endpoint
  reachability;
- give each named service a stable identity across key and transport rotation;
- permit the HRM controller to delegate a service to a user, provider, or
  device key;
- reuse HRM current-snapshot, transfer, expiry, revocation, and parent-
  delegation behavior;
- keep rapidly changing endpoint and route records outside the HRM envelope;
- allow application profiles to define usernames, payments, web, chat, files,
  and future payloads without changing HRM Core; and
- support deterministic independent implementations.

## Non-goals

This document does not define:

- HRM Core, its commitment, envelope, or generic delegation encoding;
- a universal username syntax;
- a payment, wallet-address, invoice, or currency schema;
- endpoint discovery or storage;
- HNSR routing, relay tickets, or circuits;
- DNS, HTTP, TLS, QUIC, messaging, or payment wire behavior;
- globally assigned IP prefixes, ASNs, ports, protocol numbers, or link-layer
  identifiers; or
- automatic browser, wallet, operating-system, or network permission.

Those semantics belong to HRM Core, another HRM resource profile, or the
application/transport profile consuming HNSA.

## Requirements language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in BCP 14
when, and only when, they appear in all capitals.

## Terminology

**HRM subject**
: The HNS name hash in the current verified HRM payload.

**HRM controller**
: The operational key declared by and signing the current HRM payload. For
  HNS-local named services, it originates the resource and signs the HRM
  delegation to the service controller.

**Named service**
: An HNS-local HRM resource identified by a Handshake network, HNS name hash,
  canonical service name, and application profile ID.

**Application profile**
: A separate specification assigning meaning to an application profile ID and
  defining user-facing naming, service rights, endpoint records, capabilities,
  constraints, discovery, transport, and application behavior.

**Service controller**
: The key in the current HRM service delegation. It may sign bounded endpoint
  delegations for exactly one named service.

**Endpoint key**
: A short-lived or device-specific key authorized by the service controller.
  An application or transport profile defines how it signs or authenticates
  endpoint records and sessions.

**Service resource ID**
: The stable HRM resource ID calculated from the named-service identity.

**Service delegation ID**
: The HRM delegation ID for the current delegation of a named service to its
  service controller.

## Dependency on HRM Core

A relying implementation MUST validate the complete current HRM before applying
this profile. That validation includes:

1. authenticated current HNS namestate;
2. current `hrm1` commitment selection;
3. exact envelope-hash matching;
4. deterministic-CBOR validation;
5. subject, network, sequence, and validity checks;
6. HRM controller signature verification;
7. resource origin or parent-delegation verification; and
8. current-snapshot and local finality policy.

HNSA MUST NOT duplicate, bypass, or weaken those checks.

One HRM may contain HNSA resources alongside resources from unrelated profiles.
Adding HNSA does not prevent the same manifest and delegation graph from
supporting later routing, overlay, payment, identity, or other resource
profiles.

## Named-service identity

A named service is identified by:

```text
Handshake network magic
HNS name hash
canonical service name
application profile ID
```

This tuple is stable across HRM controller rotation, service-controller
rotation, endpoint rotation, provider migration, and transport failover.

Service names:

- are 1 through 63 ASCII bytes;
- contain only lowercase `a-z`, digits, and hyphen;
- MUST NOT begin or end with a hyphen;
- MUST NOT contain a period, slash, underscore, whitespace, `@`, or percent
  escape; and
- are compared byte-for-byte without locale processing.

The service name is an application-local label beneath the HNS root. It is not
a DNS registration and is not independently owned on chain. An application
profile MAY map a user-facing local part such as `jaron@example` to canonical
service name `jaron`, but it MUST define that mapping and its collision rules.

Application profile ID zero is invalid. Draft profiles MUST use an explicitly
documented private experimental value until an assignment is accepted.

## Canonical named-service identifier

The `identifier` byte string of an `hns.named-service/v1` HRM resource is
the deterministic-CBOR encoding of:

| Key | Name | Type | Required |
| ---: | --- | --- | :---: |
| `0` | `network_magic` | unsigned integer, at most `u32` | yes |
| `1` | `name_hash` | 32-byte byte string | yes |
| `2` | `service_name` | canonical text string | yes |
| `3` | `application_profile_id` | unsigned integer, at most `u16` | yes |

No other key is permitted in version 1.

The network and name hash MUST match the active network and HRM subject. The
application profile ID MUST be nonzero and recognized by the relying
implementation.

The service resource ID is:

```text
SHA-256(
    ASCII("HNS-HRM-NAMED-SERVICE-ID-V1") || 0x00
    || canonical_identifier
)
```

Two canonical identifiers are equal only when all four fields are equal.

## Named-service resource

An HNSA service uses the HRM resource profile identifier:

```text
hns.named-service/v1
```

Its authority MUST be HNS-local origin unless a future profile explicitly
defines a compatible parent-delegation mapping. Its validity interval MUST be
contained by the HRM payload interval.

The resource `attributes` map is:

| Key | Name | Type | Required | Meaning |
| ---: | --- | --- | :---: | --- |
| `0` | `profile_flags` | unsigned integer, at most `u16` | yes | Application-profile flags |
| `1` | `profile_constraints_hash` | 32-byte byte string | yes | Hash of detached profile constraints, or zero |
| `2` | `presentation` | map | no | Non-authoritative profile-defined display data |

The application profile defines allowed `profile_flags`, the detached
constraints encoding and hash domain, and any non-authoritative presentation
fields. Presentation data MUST NOT change the service identity or grant rights.

A current HRM MUST contain at most one canonical service resource with a given
service resource ID. Duplicate or conflicting entries are invalid.

## Delegation to the service controller

The HRM controller delegates operation of a named service using an ordinary HRM
delegation object.

This is a profile-defined operational delegation over the same resource, not a
claim that a second HNS subject originated a child resource. HNSA therefore
permits `child_subject` to equal the parent HRM subject and
`child_resource_id` to equal `parent_resource_id`. The delegated right is
consumed directly by HNSA endpoint validation; it does not require a separate
child HRM. This same-subject, same-resource mapping is valid only for the exact
rights and constraints below and MUST NOT be generalized to another HRM
profile without that profile defining its own mapping.

For version 1:

- `parent_resource_id` MUST equal the service resource ID;
- `child_profile` MUST equal `hns.named-service/v1`;
- `child_resource_id` MUST equal the service resource ID and
  `child_identifier` MUST equal the service resource's canonical `identifier`
  byte string;
- `child_subject` MUST equal the HRM subject;
- `child_controller` MUST use HRM algorithm 1 and a valid compressed
  secp256k1 service key;
- `rights` MUST be the canonical two-element array
  `["delegate-endpoint", "operate"]`;
- `may_subdelegate` MUST be false;
- the delegation interval MUST be contained by the resource and HRM payload
  intervals; and
- `constraints` MUST use the map below.

The constraints map is:

| Key | Name | Type | Required | Meaning |
| ---: | --- | --- | :---: | --- |
| `0` | `service_generation` | nonzero unsigned integer, at most `u64` | yes | Replacement and replay generation |
| `1` | `max_endpoint_lifetime` | unsigned integer, at most `u32` | yes | Maximum seconds |
| `2` | `allowed_endpoint_capabilities` | unsigned integer, at most `u32` | yes | Profile-defined bit mask |
| `3` | `endpoint_constraints_hash` | 32-byte byte string | yes | Expected detached endpoint constraints, or zero |

`max_endpoint_lifetime` MUST be 300 through 604,800 seconds. The application
profile MAY impose a lower maximum.

The service generation MUST increase whenever a service controller is replaced,
withdrawn and later restored, or its authority is intentionally reset. An
unrelated HRM change does not require incrementing it.

Exactly one current delegation may contain `operate` for a service resource.
More than one is ambiguous and MUST fail validation. Concurrent endpoint
redundancy is expressed through several endpoint delegations beneath the one
service controller.

Let `service_delegation_body` be the deterministic-CBOR map containing the
ordinary HRM delegation fields with integer keys `1` through `11`, omitting
only key `0` (`delegation_id`). The service delegation ID is:

```text
SHA-256(
    ASCII("HNS-HRM-NAMED-SERVICE-DELEGATION-ID-V1") || 0x00
    || service_delegation_body
)
```

That digest MUST be stored as field `0` of the delegation. A verifier MUST
re-encode fields `1` through `11`, recompute the digest, and reject a mismatch.
The ID therefore commits to the service key, generation, rights, constraints,
subject, resource, and validity interval without a self-reference.

## Endpoint delegation

A current service controller may sign one or more short-lived endpoint
delegations:

```text
EndpointDelegationV1 {
    version:                       u8
    network_magic:                 u32
    service_resource_id:           u8[32]
    service_delegation_id:         u8[32]
    service_generation:            u64
    endpoint_key:                  u8[33]
    endpoint_sequence:             u64
    issued_at:                     u64
    expires_at:                    u64
    capabilities:                  u32
    constraints_hash:              u8[32]
    service_signature_length:      u8
    service_signature:             u8[service_signature_length]
}
```

All integers in this object use little-endian encoding. Let
`canonical_endpoint_body` be the exact fixed-width bytes from `version` through
`constraints_hash`, excluding `service_signature_length` and
`service_signature`. The service-signature digest is:

```text
BLAKE2b-256(
    "HNS-HRM-HNSA-ENDPOINT-DELEGATION-V1\0"
    || canonical_endpoint_body
)
```

Rules:

- `version` MUST equal 1;
- `network_magic` MUST match the active Handshake network;
- both IDs and the generation MUST match the current verified HRM service
  resource and delegation;
- `endpoint_key` MUST be a valid compressed secp256k1 public key;
- `endpoint_sequence` MUST be nonzero and increase for replacement of the
  same profile-defined logical endpoint;
- `issued_at` MUST be less than `expires_at`;
- the lifetime MUST NOT exceed `max_endpoint_lifetime`;
- the interval MUST be contained by the current service resource and service
  delegation intervals;
- `capabilities` MUST contain no bit outside
  `allowed_endpoint_capabilities`;
- required capabilities are defined by the application profile;
- `constraints_hash` MUST match the current service-delegation constraint;
- the signature MUST use canonical strict-DER, low-S secp256k1 and verify under
  the current service-controller key; and
- malformed lengths, unsupported versions, and trailing bytes MUST be rejected.

The endpoint-delegation ID is:

```text
SHA-256(
    ASCII("HNS-HRM-HNSA-ENDPOINT-DELEGATION-ID-V1") || 0x00
    || complete_canonical_endpoint_delegation
)
```

A service may authorize several endpoints concurrently for redundancy,
geographic distribution, device migration, or transport choice.

## Profile-specific records and application payloads

HNSA stops after authorizing the endpoint key. An application or transport
profile defines the next object.

A profile-specific record MUST bind at least:

- active network magic;
- service resource ID;
- service delegation ID and generation;
- endpoint-delegation ID and sequence;
- application profile ID;
- record sequence and validity interval;
- profile-specific payload, locator, or route data; and
- an endpoint-key signature over all preceding fields.

A profile may define:

- static signed application data stored with the HRM or in content-addressed
  storage;
- dynamic request/response behavior through an authorized endpoint;
- direct and relayed endpoint records;
- user-facing identifiers derived from the service name;
- payment assets, networks, addresses, invoices, and expiry;
- web origins and transport authentication;
- messaging delivery keys; or
- other bounded application semantics.

A valid HNSA chain authorizes the service and endpoint keys. It does not make
an otherwise malformed or semantically invalid application payload valid. The
application profile remains responsible for payload validation.

## Validation algorithm

To validate a profile-specific endpoint record for expected named service
`S`, a client MUST:

1. Derive the expected HNS name, canonical service name, and application
   profile ID from trusted application input.
2. Obtain and validate the complete current HRM under HRM Core.
3. Construct the canonical HNSA identifier and service resource ID.
4. Select exactly one current `hns.named-service/v1` resource with that ID.
5. Validate its identifier, HNS-local authority, attributes, and interval.
6. Select exactly one current HRM delegation with `operate` for that service.
7. Validate its child resource, subject, controller, rights, generation,
   endpoint limits, capabilities, constraints, and interval.
8. Calculate and match the service delegation ID.
9. Decode the endpoint delegation using bounded canonical parsing.
10. Match its service resource, current service delegation, generation,
    capabilities, constraints, and interval.
11. Verify its service-controller signature.
12. Calculate and match the endpoint-delegation ID.
13. Validate the profile-specific record and endpoint signature.
14. Apply application and local operational policy before using the result.

Untrusted objects MUST NOT select a different HNS name, service name, or
application profile from the identity requested by the user or application.

Failure at any step fails HNSA authorization. A client MUST NOT silently
replace a failed HRM/HNSA chain with a legacy `hsa1` object, an
unauthenticated directory result, a plain DNS record, or a conventional endpoint
under the same presentation identity.

## Replacement, revocation, and caching

### HRM and resource replacement

HRM Core's current complete snapshot is authoritative. A greater accepted HRM
sequence replaces the prior resources and delegations. Removing the service
resource revokes the service. Removing or replacing its delegation revokes that
service controller.

### Service-controller replacement

A replacement delegation MUST use a greater `service_generation`. Equal
generations with different controller, rights, constraints, or canonical bytes
are conflicting and invalid.

An endpoint delegation binds both the service delegation ID and generation.
Consequently an endpoint issued under a removed service controller cannot
become current under its replacement.

### Endpoint replacement

Endpoint delegations are short-lived and may overlap. The application profile
defines the logical endpoint identifier used when comparing
`endpoint_sequence`.

### Caching

A client may cache a validated chain only until the earliest of:

- observation of a changed HRM commitment or relevant HNS reorganization;
- HRM payload expiry;
- named-service resource expiry;
- service delegation expiry;
- endpoint-delegation expiry;
- endpoint or application-record expiry; or
- an application-profile cache limit.

A current object being unavailable does not authorize fallback to an older
manifest, delegation, endpoint, or application record.

## Application profiles

An application profile using HNSA MUST specify:

1. A profile ID and versioning policy.
2. User-facing purpose and concrete user stories.
3. Mapping from user input to canonical HNS name and service name.
4. Meaning of service flags, capabilities, and detached constraints.
5. Service and endpoint replacement scope.
6. Application-record encoding and signature domain.
7. Discovery and replication behavior.
8. Direct, relayed, and fallback policy.
9. Maximum service, endpoint, and record lifetimes.
10. Application payload validation and resource limits.
11. Browser origin and permission behavior when applicable.
12. Positive and negative deterministic test vectors.
13. Privacy, abuse, and denial-of-service considerations.

A payment profile must additionally define asset and network identifiers,
address and invoice validation, static versus dynamic destinations, conflict
handling, destination expiry, and transaction-intent binding.

A username-bearing profile must define normalization, allowed characters,
display form, collision handling, and whether the local part maps directly to
the HNSA service name.

## Browser behavior

### Stable origin

A web-facing profile MUST scope browser identity and storage to at least:

```text
Handshake network
HNS name hash
canonical service name
application profile ID
```

It MUST NOT scope origin only to an IP address, relay, endpoint key, retrieval
URI, or hosting provider.

### Identity indication

A browser may indicate that an endpoint is authorized by current HNS and HRM
state. That indication must not claim that the operator, content, or service is
honest or safe.

### Failure behavior

If the HRM or HNSA chain expires, becomes ambiguous, changes unexpectedly, or
fails validation, the browser must stop using that authority. Conventional-web
or legacy resolution is a separate identity unless an application profile
explicitly defines and secures a transition.

### Permissions

HNSA authorization does not grant local-network, VPN, device, wallet,
persistent-background, mining, or value-transfer permission. Those remain
application and operating-system decisions.

## Relationship to HIP-0002 and wallet TXT conventions

HIP-0002 HTTP paths such as `/.well-known/wallets/<asset>` and prefixed HNS
`TXT` wallet records are application-specific publication and discovery
conventions. They are not HRM resources or HNSA delegations by themselves.

A future payment profile MAY define an adapter that reads or emits either
convention. It MUST specify authenticated namestate requirements, asset and
network identifiers, address syntax, precedence, conflicts, expiry, and
fallback behavior. An HTTP response synthesized from a `TXT` record does not
gain a stronger authority chain merely because it is exposed through a
well-known URL.

Such an adapter MUST NOT silently merge a legacy domain-wide wallet record with
a user-scoped HNSA identity such as `jaron@denuoweb`. The payment profile must
define the exact mapping and present it as a separately selected compatibility
mode unless the record is cryptographically bound to the expected HNSA
resource and delegation.

## Relationship to DNS delegation

DNS and DNSSEC already support child names such as `jaron.denuoweb`, including
NS delegation and TXT or HTTPS records. HNSA does not replace that mechanism.

An application may display `jaron@denuoweb` while resolving
`jaron.denuoweb`; that is a DNS-based application convention, not HNSA.

HNSA instead permits an application profile to treat `jaron` as a service
label within the HRM for `denuoweb` and delegate it directly to Jaron's
service key. This does not create a DNS owner name or require Jaron to operate
an authoritative nameserver. Profiles must not silently treat these two models
as interchangeable.

## Relationship to HNSR

HNSA establishes the durable named-service resource and current service
controller. HNSR may discover short-lived endpoints and relay opaque
application streams.

The companion HRM/HNSA HNSR profile defines route records that bind:

- the stable HNSA service resource ID;
- the current HRM service delegation ID and generation;
- an HNSA endpoint delegation; and
- current HNSR relay tickets.

An HNSR relay or rendezvous storage node is not required to retrieve or
validate an HRM, but a client consuming an HRM/HNSA named route MUST validate
the current chain. HRM need not be retrieved over HNSR, and unnamed HNSR node
rendezvous does not require HRM or HNSA.

## Compatibility and transition

The `hrm1` commitment and deterministic-CBOR objects are ordinary current HNS
TXT data plus off-chain content. Nodes, miners, resolvers, wallets, and
applications that do not implement HRM or HNSA may ignore them.

The earlier experimental HNSA draft used:

- an on-chain `hsa1` root-key record;
- a fixed binary `ServiceAuthorizationV1`; and
- an endpoint delegation bound to that authorization ID.

Those objects are not HRM objects and are superseded by this profile. They MUST
NOT be accepted as this version, converted implicitly, used as fallback, or
share application/browser identity with an HRM-backed HNSA service.

Because no permanent assignment or final HIP was issued for the earlier
experiment, implementations SHOULD use a new experimental record or authority
version for HRM-backed HNSA and retain the earlier parser only in explicitly
selected compatibility tests.

## Security considerations

### Wallet-key exposure

The HNS name wallet key selects the current HRM commitment through an ordinary
name update. It MUST NOT be reused as the HRM controller, service controller, or
endpoint key merely for convenience.

### HRM-controller compromise

A compromised HRM controller can sign malicious current manifests only while
the HNS owner continues committing their hash. The HNS owner can replace the
commitment. Parent-delegated or externally originated resources retain the
additional HRM authority requirements.

### Service-controller compromise

A compromised service controller can authorize endpoints only for its exact
named-service resource, current delegation, generation, rights, constraints,
and interval. It cannot modify the HRM or another service.

### Endpoint-key compromise

A compromised endpoint key can impersonate its endpoint until the earliest
applicable delegation or record expiry.

### External wallet destinations

An HRM/HNSA chain proves which current named-service key authorized a payment
record. It does not by itself prove control of an address on BTC, XMR, HNS, or
another external network, nor that paying it is safe. A payment profile MUST
define whether asset-specific control proof is required and MUST bind the
asset, network, destination, memo or tag requirements, expiry, and transaction
intent strongly enough to prevent cross-network and substitution errors.

### Replay and rollback

The current HNS commitment, HRM sequence, complete-snapshot semantics, service
generation, delegation IDs, endpoint sequence, and bounded validity intervals
limit replay. Implementations must preserve the rollback protections required
by HRM Core and the consuming profile.

### Name transfer

Name transfer follows HRM Core. An unchanged current commitment preserves its
exact controller-signed manifest. The new name owner may withdraw or replace
that commitment but cannot alter its signed contents.

### Untrusted retrieval

HRM hosts, directories, relays, and endpoints may omit, replay, reorder, or
equivocate. They cannot forge a current hash/signature chain, but they can deny
availability. Unavailability does not make an older object current.

### Parser and resource exhaustion

HRM Core bounds envelope and delegation processing. HNSA additionally requires:

| Item | Maximum |
| --- | ---: |
| Service name | 63 bytes |
| Endpoint-delegation signature | 80 bytes |
| Endpoint delegation | 320 bytes |
| Concurrent service candidates per identity | 2 before ambiguity rejection |
| Concurrent endpoint candidates | 32 |
| Detached constraints object | 64 KiB |

Application profiles may impose smaller bounds. Larger bounds require explicit
justification and tests.

## Privacy considerations

A complete HRM may reveal service names, delegated users or providers,
controller changes, validity intervals, and relationships between resources.
Generic manifests make selective retrieval and disclosure an important future
HRM concern.

Application profiles should avoid personal device labels, private addresses,
internal topology, and long-lived correlatable endpoint keys when not required.
A username or payment profile must document the public correlation created by
its naming and discovery model.

## Reference implementation plan

Implementation should proceed in this dependency order:

1. Deterministic HRM Core encoders, decoders, signatures, commitment selection,
   storage retrieval, and current-state validation.
2. The exact `hns.named-service/v1` resource and HRM service-delegation
   validator.
3. Endpoint-delegation encoders, signers, validators, and vectors bound to HRM
   IDs and generations.
4. At least one application profile.
5. The HRM/HNSA HNSR adapter.
6. Wallet tooling for creating, signing, publishing, replacing, and inspecting
   HRMs.
7. Mobile and browser consumers that preserve the exact verified identity.

Existing `hsa1`-based Rust and JavaScript implementations conform to the
superseded experiment, not this draft, until their authority source and object
bindings are migrated to HRM.

## Deployment gates

### Stage 0: HRM Core

- finalize HRM deterministic CBOR and signature vectors;
- implement current HNS commitment selection and envelope validation;
- test transfer, replacement, rollback, expiry, and unavailable retrieval.

### Stage 1: Named-service profile

- publish exact resource-ID and service-delegation vectors;
- verify byte-identical Rust and JavaScript implementations;
- test service creation, controller replacement, removal, generation rollback,
  ambiguity, and profile mismatch.

### Stage 2: Endpoint authority

- publish endpoint-delegation signature and ID vectors;
- test concurrent endpoints, capability constraints, expiry, replacement, and
  removed-controller rejection.

### Stage 3: Application and transport profiles

- implement separately reviewed web, chat, payment, or other profiles;
- demonstrate profile-specific payload validation and identity mapping;
- integrate direct and relayed transports without changing service identity.

### Stage 4: Independent clients and operators

- run multi-operator regtest and testnet trials;
- measure retrieval, validation, storage, and denial-of-service behavior;
- complete security and browser-origin review before permanent assignments.

## Test requirements

Deterministic positive and negative vectors MUST cover:

- every HRM Core requirement used by HNSA;
- canonical named-service identifier and resource ID;
- wrong network, subject, service name, and application profile;
- invalid resource origin, flags, or constraints;
- valid service delegation and controller signature through the HRM envelope;
- missing, duplicate, or conflicting service delegations;
- service generation replacement and rollback;
- endpoint delegation encoding, signature, and ID;
- wrong service resource, delegation ID, generation, key, capabilities, or
  constraints;
- endpoint expiry and sequence replacement;
- manifest replacement, removal, transfer, and reorganization;
- legacy `hsa1` and fixed service-authorization rejection;
- application-profile identity and payload failures; and
- no unauthenticated or cross-model fallback.

## Rationale

### Why make HNSA an HRM profile?

Named services need the same commitment, controller separation, complete
snapshots, delegation, transfer, expiry, and revocation behavior as other
resources. A second authority format would duplicate those rules and prevent
services from participating in a larger resource graph.

### Why keep endpoint delegations outside HRM?

Service controllers may operate many mobile, residential, or relayed endpoints
whose keys and locators change more frequently than an HNS update and complete
manifest publication. The HRM delegates the durable service role; the service
key signs bounded transient endpoint authority.

### Why use a complete manifest snapshot?

It makes removal an explicit current-state revocation and lets one HNS
commitment select the coherent set of resources and delegations. Endpoint
presence remains separately short-lived.

### Why distinguish application profiles from HRM resource profiles?

`hns.named-service/v1` defines the common resource and controller chain.
Application profiles define what `web`, `chat`, `jaron`, or another
service label means and what records or sessions are valid. This permits shared
authority without pretending that every application has identical semantics.

### Why not use DNS delegation alone?

DNS delegation already works for DNS child names and remains appropriate when
that is the desired model. HNSA provides a manifest-native application
delegation model for services that should not require a child DNS zone or bind
identity to one DNS transport.

## Open questions

The following remain for Draft review:

- permanent registry and assignment policy for application profile IDs;
- whether the service-controller delegation should permit explicitly bounded
  concurrent controllers;
- whether future threshold controllers should be added through HRM Core;
- whether selective disclosure should use HIP-0016, an authenticated map, or
  independent committed submanifests;
- exact payment and username profile separation;
- migration tooling for experimental `hsa1` records; and
- whether direct Web and HNSR should share one application profile or use
  distinct profile IDs with an explicit origin relationship.

## References

1. RFC 2119, *Key words for use in RFCs*.
2. RFC 8174, *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*.
3. RFC 8949, *Concise Binary Object Representation (CBOR)*.
4. HIP-0002, *Well Known directory for wallets address*.
5. Draft HIP, *Handshake Resource Manifests*.
6. Draft HIP, *Handshake P2P Rendezvous and Authenticated Service Relay*.
7. Draft HIP, *HRM/HNSA Profile for Handshake P2P Rendezvous*.
8. Handshake developer documentation, *Resource Records*.
