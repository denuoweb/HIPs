# HIP-xxxx : Handshake P2P Rendezvous and Authenticated Service Relay

```text
Number:  HIP-xxxx
Title:   Handshake P2P Rendezvous and Authenticated Service Relay
Type:    Standards Track
Status:  Draft
Authors: Jaron Rosenau <@denuoweb>
Created: 2026-07-19
Updated: 2026-07-21
Related:  Handshake P2P Transport for Oblivious DNS Relay (draft HIP, optional)
```

## Abstract

This document specifies an optional Handshake peer-to-peer extension that
allows phones, laptops, home computers, and other devices which can establish
outbound Handshake peer connections, but cannot accept directly addressed
inbound sockets, to publish and receive incoming logical service streams
through ordinary relay-capable Handshake peers.

The protocol does not introduce a separate rendezvous network, third-party
resolver, HTTP directory, tunnel provider, relay account, or dedicated relay
operator class. Service authorization is rooted in authenticated Handshake name
state when a human-readable HNS service is used, while service registration,
relay reservation, keyed discovery, circuit establishment, and circuit
transport all occur over the existing Handshake P2P network. Any suitable
publicly reachable Handshake node may opt into the rendezvous and relay roles.

An endpoint establishes an ordinary outbound Handshake peer connection and
reserves bounded relay capacity over that connection. It publishes a
short-lived, cryptographically authenticated route record to a deterministic
set of rendezvous-capable Handshake peers. A requester looks up that record over
the same P2P network, connects to a selected relay as an ordinary Handshake
peer, and asks the relay to open a logical circuit to the endpoint's existing
outbound connection.

Application traffic is protected end to end between requester and endpoint.
The relay forwards an opaque, flow-controlled byte stream and may deny, delay,
rate-limit, or observe traffic metadata, but it cannot authenticate as the
endpoint or modify protected application plaintext without detection.

The first mandatory service profile carries complete Handshake peer sessions,
allowing outbound-only full nodes to receive logical inbound peers while they
are online. This document also defines an initial HNS web-service profile so an
HNS-aware client can reach a site such as `hnsr://denuoweb/p2p-site/` when its
endpoint is a phone or other device without an open public port. Additional
profiles may define messaging, content distribution, developer previews,
personal APIs, and other explicitly scoped services.

A conforming implementation does not expose arbitrary local ports and is not a
general-purpose TCP proxy. Each supported profile terminates at an explicitly
registered service handler. This extension is opt-in, requires no consensus
change, does not alter Handshake name or block validation, and creates no new
public listener separate from the ordinary Handshake P2P listener.

## Plain-language summary

Today, a phone or home computer can connect out to Handshake but is often
impossible for other users to connect back to because it has no public port.
This proposal lets the device keep an outbound connection to ordinary HNS
peers, advertise a signed temporary route through the HNS network, and receive
incoming encrypted connections through one of those peers.

This can make an Android full node more useful while it is online and can also
support HNS-native sites and applications hosted from devices behind NAT. The
main cost is that volunteer HNS nodes must spend bandwidth, memory, CPU, and
public-IP reputation relaying other users' traffic; if this is not strictly
limited and scheduled below blockchain traffic, it could slow block
propagation and concentrate the network around a small set of powerful relays.

A phone is not required to remain online permanently merely to participate.
When it goes offline, its active reservations stop working and its short-lived
routes expire. A service that requires continuous availability must use
multiple endpoints, replicas, or a future cacheable-content profile.

## Motivation

Handshake's P2P network assumes that some nodes have publicly reachable
listeners. A node behind residential NAT, carrier-grade NAT, a mobile network,
a restrictive firewall, or a frequently changing network can still:

- validate blocks and transactions;
- maintain outbound peers;
- serve data over those already established connections;
- resolve names locally;
- run a wallet or application;
- operate a full node on Android.

It generally cannot advertise a directly reachable address that another peer
can dial. The result is an asymmetric network:

```text
Publicly reachable node
    - initiates outbound connections
    - receives inbound connections

Outbound-only node
    - initiates outbound connections
    - cannot receive new inbound connections
```

An outbound-only full node remains useful to its operator, but it contributes
less connection capacity and is harder for the network to discover. This is
particularly important for Android because opening a listener inside the
application is not the same as making that listener reachable through a router
or cellular carrier.

The same reachability problem applies to application services. A user may wish
to operate:

```text
hnsr://denuoweb/p2p-site/
hnsr://denuoweb/node/
hnsr://denuoweb/messages/
hnsr://denuoweb/preview/
```

from a phone, laptop, Raspberry Pi, or home server without registering a tunnel
account or moving the service to a centralized hosting provider.

A conventional reverse tunnel can solve the transport problem, but it creates
a separately configured external dependency. A generic rendezvous network
would create another bootstrap set, operator set, protocol, and availability
layer. This proposal instead extends the existing Handshake peer network so the
same replaceable peer population used for blocks, headers, transactions,
proofs, and optional DNS relay can also provide bounded discovery and
reachability.

The proposal does not claim that every HNS node will relay traffic or that a
relayed phone becomes an always-on server. Its purpose is to make temporary and
mobile reachability possible without making a single third-party resolver or
tunnel provider mandatory.

## Why intermittent endpoints are still useful

Long-term uptime and inbound reachability are separate properties.

A mobile full node that is online for four hours can, during those four hours:

- accept logical inbound peers;
- serve blocks and proofs;
- provide additional peer diversity;
- answer requests from newly connected nodes;
- contribute relay or rendezvous capacity if configured to do so.

When it disconnects, peers continue using the rest of the network. Handshake
consensus does not require every full node to remain online permanently.

For a website or other singular application service, availability is different:

```text
one phone is the only endpoint
    -> service is unavailable when the phone is offline

phone + home server + VPS endpoint
    -> clients can fail over

signed static content replicated by peers
    -> content can remain available without the publishing phone
```

Version 1 therefore treats route records as short-lived presence information,
not as a promise of long-term uptime.

## Relationship to Handshake P2P DNS Relay

The proposed Handshake P2P DNS Relay carries restricted recursive DNS queries
and raw DNS responses over established Handshake peer connections. It removes
the requirement for a separately configured third-party recursive resolver
when direct authoritative DNS is unavailable.

This proposal operates at a different layer:

```text
P2P DNS Relay
    bounded DNS query-response transport

P2P Rendezvous
    finds a currently online endpoint or relay route

P2P Service Relay
    carries a bounded bidirectional encrypted service stream
```

The proposals intentionally share these properties:

- existing Handshake P2P connections and framing are reused;
- capabilities are negotiated through `version.services`;
- peers remain untrusted and replaceable;
- no new public HTTP, DNS, SOCKS, or tunnel listener is required;
- operator enablement is explicit;
- consensus and block-propagation traffic has higher scheduling priority;
- permanent service-bit and packet-type assignments remain `TBD` while draft.

Rendezvous records MUST NOT be encoded as DNS queries. They are rapidly
changing P2P presence records, not authoritative DNS data. Likewise, the DNS
relay MUST NOT become an unrestricted byte tunnel.

## Relationship to Handshake P2P Oblivious DNS Relay

The separate **Handshake P2P Transport for Oblivious DNS Relay** HIP may use
HNSR as an optional reachability mechanism for an ODoH proxy or target that
cannot accept a directly addressed inbound socket.

The protocol layering is:

```text
Oblivious DNS HIP
    target selection, signed ODoH configuration, HPKE,
    DNS admission, proxy/target privacy split, local validation

HNSR HIP
    current endpoint discovery, relay reservation,
    circuit establishment, flow-controlled opaque transport
```

An ODoH proxy may derive the unnamed HNSR route key from the selected target
peer key, open an `HNS_NODE_V1` circuit, complete inner Brontide and Handshake
`version` / `verack`, and then exchange ordinary `ODNS` packets over the inner
peer session. No new HNSR opcode or mandatory service profile is required.

HNSR route records, relay tickets, and circuits authenticate temporary
reachability only. They do not authenticate ODoH HPKE keys, DNS responses, HNS
state, DNSSEC, or DANE. Those responsibilities remain entirely with the
Oblivious DNS and base DNS Relay HIPs.

Conversely, ODoH does not alter HNSR storage, lookup, tickets, circuits, flow
control, or endpoint authorization. HNSR rendezvous nodes and relays MUST treat
the inner session as opaque and MUST NOT require access to ODoH plaintext.

The integration is optional in both directions:

- a directly reachable ODoH target does not require HNSR;
- an HNSR implementation does not require ODoH;
- HNSR conformance does not imply support for `NODE_DNS_OBLIVIOUS`;
- ODoH conformance may be limited to direct target locators.

HNSR does not strengthen the ODoH anonymity claim. An HNSR relay observes the
ODoH proxy connection, target endpoint, timing, duration, and byte volume.
Rendezvous nodes may observe route-key lookups. Operator collusion and traffic
correlation remain possible.

## Conventions and terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in RFC 2119
and RFC 8174 when, and only when, they appear in all capitals.

- **Endpoint**: a device publishing a service while maintaining one or more
  outbound Handshake connections.
- **Requester**: a client attempting to reach a published service.
- **Relay**: a publicly reachable Handshake peer that accepts endpoint
  reservations and forwards bounded circuits.
- **Rendezvous node**: a Handshake peer that participates in keyed storage and
  lookup of signed route records.
- **Outer connection**: an ordinary Handshake connection between endpoint and
  relay, requester and relay, or two rendezvous peers.
- **Circuit**: one multiplexed bidirectional stream through a relay.
- **Inner session**: the end-to-end authenticated and encrypted session carried
  by a circuit.
- **HNS authority**: a currently authenticated Handshake name resource that
  commits to an HNSR root key.
- **Root key**: the long-lived public key committed by the HNS name resource.
- **Service authorization**: a root-key-signed statement authorizing a service
  name, profile, and service key.
- **Service key**: a key authorized to delegate one named service.
- **Endpoint key**: a short-lived key authorized by the service key and used to
  authenticate the endpoint's inner session.
- **Relay key**: the relay's compressed secp256k1 identity and signing key.
- **Reservation**: relay state binding an endpoint key to a live outer
  connection and resource limits.
- **Relay ticket**: a jointly signed, expiring advertisement for an active
  reservation.
- **Route record**: a signed rendezvous value containing the authorization
  chain and one or more relay tickets.
- **Route key**: the deterministic 32-byte lookup key for a service.
- **Rendezvous node ID**: the deterministic 32-byte identifier used for XOR
  distance in the rendezvous subprotocol.
- **Profile**: a registered definition of what protocol runs inside a circuit.
- **Context ID**: a nonzero 64-bit value identifying one request or circuit on
  an outer connection.

## Goals

1. Make outbound-only Handshake full nodes reachable while online.
2. Allow HNS-authenticated application services to run behind NAT.
3. Use the existing Handshake peer transport and bootstrap population.
4. Avoid a mandatory third-party resolver, rendezvous server, or tunnel
   provider.
5. Authenticate endpoints independently of relays and rendezvous storage nodes.
6. Use short-lived, replay-bounded route records.
7. Support intermittent mobile endpoints without claiming continuous uptime.
8. Preserve core blockchain traffic under relay load.
9. Permit several independent relays and endpoints per service.
10. Prevent the protocol from becoming an arbitrary local-port forwarder.
11. Scale discovery through keyed storage rather than global ticket gossip.
12. Allow gradual deployment without a consensus fork.
13. Permit optional higher-layer protocols such as Oblivious DNS to use an
    authenticated peer circuit without importing their semantics into HNSR.

## Non-goals

This proposal does not define:

- a general-purpose reverse TCP tunnel;
- SOCKS, VPN, SSH, or arbitrary port forwarding;
- wallet RPC, node RPC, shell, database, or LAN exposure;
- anonymity or resistance to all traffic analysis;
- guaranteed service availability;
- a consensus change;
- on-chain rapidly changing endpoint records;
- multi-hop onion routing;
- relay payments or mandatory compensation;
- automatic static-content replication;
- transparent compatibility with unmodified web browsers;
- transparent compatibility with unmodified Handshake dialers;
- a replacement for ordinary direct connections when those work;
- ODoH key discovery, HPKE processing, DNS recursion, DNSSEC, or DANE
  validation.

## Architecture

```text
Authenticated HNS name state
          |
          | authorizes root key
          v
Service authorization
          |
          | delegates endpoint key
          v
Endpoint reservation with relay
          |
          | produces signed relay ticket
          v
Keyed rendezvous storage over HNS P2P
          |
          | requester looks up route
          v
Relay circuit over ordinary HNS connections
          |
          | end-to-end inner security
          v
Handshake node, HNS web site, or another registered profile
```

The blockchain authenticates durable service authority. It does not carry
route updates or service traffic.

The rendezvous layer stores short-lived signed records. A rendezvous node is
not trusted to create or alter those records.

The relay provides connectivity. It is not trusted with inner plaintext or
endpoint identity.

## Roles and trust model

A single implementation may perform several roles, but each role is logically
separate.

### Endpoint

The endpoint is trusted for the service it publishes. It may be intermittent.
It maintains outbound connections, obtains reservations, publishes route
records, accepts or rejects circuits, and terminates the registered profile.

### Relay

The relay is trusted only for availability. It can censor, delay, disconnect,
or observe metadata. It is not trusted for endpoint authentication,
application data, name state, blocks, or proofs.

### Rendezvous node

A rendezvous node stores and returns signed records near a route key. It can
omit, delay, replay within validity, or flood invalid values. Clients validate
all records locally and query several rendezvous paths.

### Requester

The requester authenticates current HNS state when using a named service,
validates the authorization chain and route record, establishes an inner
session, and validates all application data according to the selected profile.

## Protocol assignments

The following permanent assignments are requested:

| Symbol | Registry | Value |
| --- | --- | ---: |
| `NODE_HNSR_RENDEZVOUS` | `version.services` bit | TBD |
| `NODE_HNSR_RELAY` | `version.services` bit | TBD |
| `HNSR` | P2P packet type | TBD |

Only one global packet type is requested. The payload contains a versioned
internal opcode.

Experimental implementations MUST use private assignments on controlled
networks and MUST NOT advertise them as this standard on public mainnet before
maintainer assignment.

The accompanying `hsd` proof of concept uses the following values on regtest
and on explicitly acknowledged, operator-controlled testnet experiments:

| Symbol | Private PoC value |
| --- | ---: |
| HNSR rendezvous service | `0x04000000` |
| HNSR relay service | `0x08000000` |
| HNSR packet type | `0xf3` |

These values are collision-prone implementation values, not requested
assignments. The reference branch rejects mainnet, requires an additional
testnet acknowledgement, and requires publicly routable advertised hosts for
testnet relay and rendezvous roles. A production implementation MUST replace
the private values with assigned values and MUST NOT infer conformance from
these numbers.

## Capability negotiation

`NODE_HNSR_RENDEZVOUS` states that the peer supports version-1 keyed lookup,
storage, and routing messages on the current connection.

`NODE_HNSR_RELAY` states that the peer supports version-1 reservations and
circuits and has a publicly reachable ordinary Handshake listener.

Both bits are directional. A cached address advertisement does not replace the
current `version` exchange.

A rendezvous node MUST NOT advertise its bit unless:

- the feature is explicitly enabled;
- bounded storage and routing tables are initialized;
- signature-verification budgets are enforced;
- it can schedule rendezvous work below core peer work.

A relay MUST NOT advertise its bit unless:

- the feature is explicitly enabled;
- it has a routable Handshake listener;
- it has a valid relay identity key;
- reservation, circuit, queue, and bandwidth limits are enforced;
- relayed data is scheduled below blockchain traffic.

Endpoints and requesters do not require a service bit merely to use the
features.

## HNS authority binding

Named services use the current authenticated HNS name resource as their trust
root.

### Root-key TXT record

A name opts into HNSR named services by publishing exactly one canonical TXT
record:

```text
hnsr1 k=<base32-compressed-secp256k1-public-key>
```

Example:

```text
TXT "hnsr1 k=ak3m..."
```

Requirements:

- `hnsr1` is lowercase ASCII;
- fields are separated by one ASCII space;
- the only version-1 field is `k`;
- the key is a valid compressed 33-byte secp256k1 public key encoded without
  padding;
- a client finding more than one syntactically valid `hnsr1` record MUST treat
  named HNSR service authorization as ambiguous and fail closed;
- unknown fields make the record noncanonical for version 1;
- the TXT record is authenticated by the same HNS state and proof rules the
  client uses for other name resources.

The root key SHOULD be kept offline or in a hardware-backed store. It signs
service authorizations, not frequent route updates.

### Name transfer and root-key update

A requester MUST validate the root key against sufficiently current safe HNS
state. When an `UPDATE` changes or removes the `hnsr1` record, authorizations
under the old key stop being valid once the requester accepts the new safe name
state.

A transfer that leaves the resource unchanged leaves the existing HNSR root key
in effect until the new owner updates it. This matches ordinary DNS-resource
continuity.

## Service naming

A named service is identified by:

```text
HNS root name + service name + profile ID
```

Version-1 service names:

- are 1 through 63 ASCII bytes;
- use lowercase `a-z`, digits, and hyphen;
- MUST NOT begin or end with a hyphen;
- are compared byte-for-byte after lowercase canonicalization;
- do not contain `/`, `.`, `_`, whitespace, or percent escapes.

The canonical URI is:

```text
hnsr://<hns-root>/<service-name>/<optional path>
```

Example:

```text
hnsr://denuoweb/p2p-site/
```

The service name is an application namespace beneath the HNS root; it is not a
second-level DNS registration and is not independently owned on chain.

An HNS-aware browser MAY map a DNS-style presentation such as:

```text
https://p2p-site.denuoweb/
```

to the HNSR service only when explicit browser or resolver policy authorizes
that mapping. Ordinary browsers cannot infer an HNSR route from DNS alone.

## Unnamed peer services

The mandatory full-node profile does not require ownership of an HNS name.
For an unnamed HNS peer service, the endpoint key itself is the service
authority.

Unnamed peer routes are used by upgraded Handshake address managers to sample
outbound-only nodes. They MUST use profile `HNS_NODE_V1` and MUST NOT claim a
human-readable HNS service name.

## Service authorization

A root key authorizes one named service with `ServiceAuthorizationV1`:

```text
ServiceAuthorizationV1 {
    version:                  u8
    network_magic:            u32
    name_hash:                u8[32]
    service_name_length:      u8
    service_name:             u8[service_name_length]
    profile_id:               u16
    service_key:              u8[33]
    flags:                    u16
    serial:                   u64
    valid_from_height:        u32
    valid_until_height:       u32
    max_endpoint_lifetime:    u32
    max_route_lifetime:       u32
    root_signature_length:    u8
    root_signature:           u8[root_signature_length]
}
```

The signature digest is:

```text
BLAKE2b-256(
    "HNSR-SERVICE-AUTH-V1\0"
    || network_magic_u32le
    || all unsigned canonical fields
)
```

Rules:

- `version` MUST be `1`;
- `service_key` MUST be a valid compressed secp256k1 key;
- `flags` MUST contain only profile-defined public-service flags;
- `serial` MUST increase for replacement authorization of the same service;
- `valid_until_height` MAY be zero for no explicit block-height expiry;
- `max_endpoint_lifetime` MUST be 300 through 604,800 seconds;
- `max_route_lifetime` MUST be 60 through 7,200 seconds;
- the root signature MUST verify under the current HNSR root key;
- the profile ID MUST be recognized by the requester;
- an authorization is invalid outside its height interval;
- the complete authorization ID is its BLAKE2b-256 hash.

A service key may be online and rotated more frequently than the root key.

## Endpoint delegation

A service key authorizes an endpoint key with:

```text
EndpointDelegationV1 {
    version:              u8
    authorization_id:     u8[32]
    endpoint_key:         u8[33]
    endpoint_sequence:    u64
    issued_at:            u64
    expires_at:           u64
    max_active_circuits:  u16
    max_bytes_per_circuit:u64
    flags:                u16
    signature_length:     u8
    service_signature:    u8[signature_length]
}
```

The delegation signature digest is:

```text
BLAKE2b-256(
    "HNSR-ENDPOINT-DELEGATION-V1\0"
    || network_magic_u32le
    || all unsigned canonical fields
)
```

It is signed under `service_key`. For an unnamed self-delegation it is signed
under `endpoint_key`.

Rules:

- lifetime MUST NOT exceed the authorization's `max_endpoint_lifetime`;
- `endpoint_sequence` MUST increase for a replacement endpoint delegation;
- the endpoint key authenticates the inner session;
- profile-specific flags MUST be valid;
- an endpoint delegation may authorize several relay tickets;
- a service may publish several valid endpoint delegations for redundancy.

For an unnamed peer route, a compact self-delegation is used and
`authorization_id` is all zeroes. The endpoint key signs its own delegation.

## Route keys

Named route key:

```text
BLAKE2b-256(
    "HNSR-NAMED-ROUTE-V1\0"
    || network_magic_u32le
    || name_hash
    || service_name_length
    || service_name
    || profile_id_u16le
)
```

Unnamed peer route key:

```text
BLAKE2b-256(
    "HNSR-PEER-ROUTE-V1\0"
    || network_magic_u32le
    || endpoint_key
)
```

The route key identifies rendezvous storage. It is not an authentication key.

## Rendezvous subprotocol

The rendezvous subprotocol is a Kademlia-style keyed storage and lookup layer
carried exclusively over ordinary Handshake peer connections. It is not a
separate transport network and has no separate bootstrap list.

### Rendezvous node ID

```text
node_id = BLAKE2b-256(
    "HNSR-RENDEZVOUS-NODE-V1\0"
    || network_magic_u32le
    || peer_identity_key
)
```

Distance is unsigned XOR distance between 32-byte identifiers.

### Routing table

A rendezvous node maintains bounded XOR-distance buckets populated only by
currently or recently connected Handshake peers advertising
`NODE_HNSR_RENDEZVOUS`.

Recommended defaults:

| Parameter | Default |
| --- | ---: |
| Bucket size `k` | 16 |
| Parallelism `alpha` | 3 |
| Record replicas | 8 |
| Maximum lookup steps | 32 |
| Maximum nodes returned | 16 |
| Routing-table entries | 2,048 |

A node MUST NOT create an unbounded number of outbound connections merely to
populate this table. The subprotocol reuses ordinary peer selection and MAY
make a small number of additional rendezvous-purpose connections within local
peer limits.

### Why keyed lookup is required

Global relay-ticket gossip does not scale if many mobile endpoints refresh
routes every few minutes. Keyed storage sends a route record to a deterministic
small set of peers near its route key and lets a requester retrieve it without
every relay learning every endpoint.

## HNSR packet envelope

All messages use the existing Handshake frame and one `HNSR` packet type.

```text
HNSREnvelopeV1 {
    version:    u8
    opcode:     u8
    flags:      u16
    context_id: u64
    body:       u8[remaining]
}
```

Rules:

- `version` MUST be `1`;
- version 1 defines no envelope flags, so `flags` MUST be zero;
- `context_id` MUST be nonzero for requests and circuits;
- unsolicited bounded announcements MAY use context ID zero where specified;
- all integers are unsigned little-endian;
- body boundaries are determined by the outer Handshake payload length;
- a body shorter than the opcode's minimum or with trailing bytes is invalid;
- length limits MUST be checked before allocation;
- an unknown version or opcode fails the exchange but SHOULD NOT alone cause a
  ban because it may represent a future extension.

## Opcodes

| Value | Name | Function |
| ---: | --- | --- |
| 0 | `FINDNODE` | Find rendezvous nodes near a key |
| 1 | `NODES` | Return rendezvous node contacts |
| 2 | `PUTROUTE` | Store a signed route record |
| 3 | `PUTRESULT` | Return route storage result |
| 4 | `GETROUTE` | Retrieve records for a route key |
| 5 | `ROUTES` | Return validated candidate records |
| 6 | `SAMPLEROUTES` | Request random unnamed node routes |
| 7 | `RESERVE` | Request relay capacity |
| 8 | `OFFER` | Relay offers a ticket prefix |
| 9 | `CONFIRM` | Endpoint signs and confirms ticket |
| 10 | `CONFIRMED` | Relay activates reservation |
| 11 | `RENEW` | Renew active reservation |
| 12 | `WITHDRAW` | Withdraw reservation |
| 13 | `OPEN` | Request a circuit |
| 14 | `INCOMING` | Notify endpoint of circuit |
| 15 | `ACCEPT` | Endpoint accepts circuit |
| 16 | `OPENED` | Circuit is ready |
| 17 | `DATA` | Carry opaque inner bytes |
| 18 | `WINDOW` | Add directional flow-control credit |
| 19 | `CLOSE` | Close circuit |
| 20 | `ERROR` | Return request or circuit error |

Values 21 through 255 are reserved.

## Cryptographic primitives

Version 1 uses:

- BLAKE2b-256;
- compressed 33-byte secp256k1 public keys;
- deterministic secp256k1 ECDSA signatures;
- strict DER signature encoding;
- low-S normalization;
- Brontide for mandatory inner sessions in the initial profiles.

A verifier MUST reject invalid public keys, noncanonical DER, high-S
signatures, wrong network magic, unsupported versions, and signatures over any
noncanonical encoding.

Every signature digest includes a unique ASCII domain label with a terminating
zero byte and the active network magic.

## Rendezvous messages

### `FINDNODE`

```text
FindNodeV1 {
    target:      u8[32]
    maximum:     u8
}
```

`maximum` MUST be 1 through 16.

### `NODES`

```text
NodesV1 {
    count:       u8
    contacts:    RendezvousContactV1[count]
}

RendezvousContactV1 {
    node_id:     u8[32]
    host_type:   u8
    host:        u8[16]
    port:        u16
    services:    u64
    peer_key:    u8[33]
    observed_at: u64
}
```

Contacts MUST be publicly routable, canonical, and no older than 24 hours.
A receiver treats them as untrusted address hints.

### `PUTROUTE`

```text
PutRouteV1 {
    route_key:    u8[32]
    record_length:u16
    record:       u8[record_length]
}
```

`record_length` MUST be 1 through 8,192 bytes.

A storage node MUST:

1. verify canonical structure;
2. verify all authorization and route signatures;
3. verify that the record derives the supplied route key;
4. verify expiration and maximum lifetime;
5. apply cheap rate and size checks before signatures;
6. store only within global, per-peer, per-key, and per-prefix limits;
7. return `PUTRESULT`.

It MUST NOT recursively store the record merely because it received one
`PUTROUTE`. The publisher performs iterative lookup and sends directly to the
selected replica set.

### `PUTRESULT`

```text
PutResultV1 {
    status:      u16
    stored_until:u64
}
```

A positive response is not a trust statement and does not guarantee future
availability.

### `GETROUTE`

```text
GetRouteV1 {
    route_key:      u8[32]
    maximum_records:u8
}
```

`maximum_records` MUST be 1 through 16.

### `ROUTES`

```text
RoutesV1 {
    count:   u8
    records: VarBytes16[count]
}
```

The response MUST fit within 65,535 bytes and MUST NOT contain more than 16
records. A requester validates each record independently.

### `SAMPLEROUTES`

This request is limited to unnamed `HNS_NODE_V1` route records so upgraded
nodes can discover random outbound-only full nodes.

```text
SampleRoutesV1 {
    maximum_records:u8
    random_seed:    u8[32]
}
```

A rendezvous node MUST return at most 16 records, sampled from bounded local
storage. It MUST NOT initiate recursive queries to answer. To make a response
reproducible without maintaining per-request state, eligible canonical records
are ordered by the unsigned byte value of:

```text
BLAKE2b-256(
    "HNSR-SAMPLE-ROUTES-V1\0"
    || random_seed
    || canonical_route_record
)
```

The node returns the first `maximum_records` entries after applying expiration
and local admission policy.

## Route record

```text
RouteRecordV1 {
    version:                    u8
    authority_type:             u8
    route_key:                  u8[32]
    profile_id:                 u16
    record_sequence:            u64
    issued_at:                  u64
    expires_at:                 u64
    authorization_length:       u16
    authorization:              u8[authorization_length]
    endpoint_delegation_length: u16
    endpoint_delegation:        u8[endpoint_delegation_length]
    ticket_count:               u8
    tickets:                    RelayTicketV1[ticket_count]
    endpoint_signature_length:  u8
    endpoint_signature:         u8[endpoint_signature_length]
}
```

`authority_type`:

| Value | Meaning |
| ---: | --- |
| 0 | Unnamed endpoint-key authority for `HNS_NODE_V1` |
| 1 | HNS-name authority using `ServiceAuthorizationV1` |

Rules:

- `ticket_count` MUST be 1 through 8;
- all tickets MUST use the delegated endpoint key and profile;
- record lifetime MUST NOT exceed `max_route_lifetime` or 7,200 seconds;
- recommended route lifetime is 900 seconds;
- endpoints SHOULD refresh after one third of the lifetime;
- `record_sequence` MUST increase for replacement records;
- the endpoint signature covers all preceding canonical fields;
- storage nodes MAY retain several records with distinct endpoint keys or relay
  sets for one named service;
- an older sequence for the same endpoint key is ignored;
- route records are deleted no later than expiration.

The endpoint signature digest is:

```text
BLAKE2b-256(
    "HNSR-ROUTE-RECORD-V1\0"
    || all preceding canonical RouteRecordV1 fields
)
```

For unnamed routes, network binding is carried by the signed `route_key`,
which is derived from `network_magic_u32le` and `endpoint_key`.

## Relay reservation

The endpoint establishes an ordinary outbound Handshake connection to a peer
advertising `NODE_HNSR_RELAY`, then requests a reservation.

### `RESERVE`

```text
ReserveV1 {
    endpoint_key:            u8[33]
    profile_id:              u16
    requested_lifetime:      u32
    requested_max_circuits:  u16
    requested_max_bytes:     u64
    endpoint_nonce:          u8[16]
    signature_length:        u8
    endpoint_signature:      u8[signature_length]
}
```

The signature digest is:

```text
BLAKE2b-256(
    "HNSR-RESERVE-V1\0"
    || network_magic_u32le
    || relay_key
    || context_id_u64le
    || all unsigned canonical ReserveV1 fields
)
```

This binds the request to the relay key, context ID, network magic, and all
fields.

Rules:

- lifetime MUST be 300 through 7,200 seconds;
- maximum circuits MUST be 1 through 32;
- maximum bytes MUST be nonzero and bounded by relay policy;
- the relay MAY grant lower limits but not higher limits;
- the relay MUST apply admission limits before allocating provisional state;
- the relay returns `OFFER` or `ERROR`.

## Relay ticket

```text
RelayTicketV1 {
    version:                   u8
    network_magic:             u32
    profile_id:                u16
    relay_transport:           u8
    relay_host_type:           u8
    relay_host:                u8[16]
    relay_port:                u16
    relay_key:                 u8[33]
    endpoint_key:              u8[33]
    reservation_id:            u8[16]
    issued_at:                 u64
    expires_at:                u64
    max_active_circuits:       u16
    max_bytes_per_circuit:     u64
    max_total_bytes:           u64
    flags:                     u16
    relay_signature_length:    u8
    relay_signature:           u8[relay_signature_length]
    endpoint_signature_length: u8
    endpoint_signature:        u8[endpoint_signature_length]
}
```

The relay signs:

```text
BLAKE2b-256(
    "HNSR-RELAY-TICKET-V1\0"
    || all unsigned canonical RelayTicketV1 fields
)
```

The endpoint then signs:

```text
BLAKE2b-256(
    "HNSR-RELAY-CONFIRM-V1\0"
    || all unsigned canonical RelayTicketV1 fields
    || relay_signature
)
```

The ticket ID is BLAKE2b-256 of the complete canonical ticket.

Ticket requirements:

- maximum lifetime is 7,200 seconds;
- recommended lifetime is 1,800 seconds;
- relay host and port MUST be publicly routable;
- profile ID MUST match the endpoint reservation;
- the relay MUST reject the ticket immediately after endpoint disconnect even
  if remote rendezvous caches still hold it;
- endpoint and relay signatures MUST verify;
- limits MUST be finite;
- tickets MUST NOT contain a local target host or port.

### `OFFER`, `CONFIRM`, and `CONFIRMED`

`OFFER` contains a canonical `RelayTicketV1` with the unsigned ticket fields
and relay signature populated, `endpoint_signature_length = 0`, and no endpoint
signature bytes.

`CONFIRM` contains:

```text
ConfirmV1 {
    reservation_id:          u8[16]
    endpoint_signature_length:u8
    endpoint_signature:      u8[endpoint_signature_length]
}
```

`CONFIRMED` contains:

```text
ConfirmedV1 {
    reservation_id:u8[16]
    ticket_id:     u8[32]
    expires_at:    u64
}
```

The endpoint treats the reservation as active only after a matching
`CONFIRMED`.

### Renewal and withdrawal

`RENEW` contains:

```text
RenewV1 {
    previous_reservation_id:u8[16]
    request:                ReserveV1
}
```

The embedded request signature uses this digest instead of the `RESERVE`
digest:

```text
BLAKE2b-256(
    "HNSR-RENEW-V1\0"
    || network_magic_u32le
    || relay_key
    || context_id_u64le
    || previous_reservation_id
    || all unsigned canonical ReserveV1 fields
)
```

The relay MUST require the previous reservation to be active, owned by the
same authenticated outer peer, and bound to the same endpoint key. Renewal
then follows the same `OFFER` / `CONFIRM` / `CONFIRMED` process and issues a
fresh reservation ID and ticket. The previous ticket remains usable until the
new confirmation succeeds. After confirmation, the relay rejects new `OPEN`
requests for the previous ticket; existing circuits MAY finish or be closed by
local policy.

`WITHDRAW` contains:

```text
WithdrawV1 {
    reservation_id:          u8[16]
    ticket_id:               u8[32]
    endpoint_signature_length:u8
    endpoint_signature:      u8[endpoint_signature_length]
}
```

The endpoint signature digest is:

```text
BLAKE2b-256(
    "HNSR-WITHDRAW-V1\0"
    || network_magic_u32le
    || relay_key
    || context_id_u64le
    || reservation_id
    || ticket_id
)
```

`WITHDRAW` is signed by the endpoint and immediately invalidates the local
reservation and its circuits. The relay replies with `ConfirmedV1` containing
the matching reservation and ticket IDs and `expires_at = 0`. Version 1 defines
no global withdrawal broadcast; remote records expire quickly and a requester
receiving a stale ticket gets an `EXPIRED` or `ENDPOINT_GONE` error from the
relay.

## Publishing a route

After confirming one or more relay tickets, an endpoint:

1. builds or loads the service authorization;
2. builds a valid endpoint delegation;
3. constructs a route record containing one through eight tickets;
4. signs the route record with the endpoint key;
5. derives the route key;
6. iteratively finds the closest rendezvous nodes;
7. stores the record at at least eight diverse nodes when possible;
8. confirms at least three successful stores before advertising locally that
   the service is published;
9. refreshes before one third of the route lifetime remains;
10. republishes immediately after a meaningful relay-set change.

A phone that loses connectivity cannot refresh. Its issuing relays invalidate
reservations immediately and rendezvous copies expire naturally.

## Looking up a route

A requester for `hnsr://denuoweb/p2p-site/`:

1. authenticates current HNS state for `denuoweb/`;
2. obtains the canonical HNSR root key from TXT;
3. derives the named route key for `p2p-site` and the desired profile;
4. performs an iterative rendezvous lookup over existing HNS peers;
5. obtains records from several closest nodes;
6. validates root-key service authorization;
7. validates endpoint delegation;
8. validates route-record signature, sequence, and expiry;
9. validates each relay ticket;
10. ranks independent endpoint and relay combinations;
11. connects to one relay as an ordinary HNS peer;
12. sends `OPEN` for the selected ticket;
13. establishes the profile's mandatory inner session;
14. retries independent routes on failure.

A storage node's response is never sufficient by itself. The requester relies
on authenticated HNS state and cryptographic records.

## Circuit establishment

### `OPEN`

```text
OpenV1 {
    ticket_id:       u8[32]
    reservation_id:  u8[16]
    endpoint_key:    u8[33]
    profile_id:      u16
    requester_nonce: u8[16]
    initial_window:  u32
}
```

The relay verifies an active matching reservation and enforces per-requester,
per-endpoint, per-IP, per-netgroup, profile, and global limits. The only target
is the endpoint connection bound to the reservation.

### `INCOMING`

```text
IncomingV1 {
    ticket_id:       u8[32]
    open_request_id: u64
    profile_id:      u16
    requester_nonce: u8[16]
    initial_window:  u32
}
```

The envelope context ID is a relay-generated unpredictable circuit ID.

### `ACCEPT`

```text
AcceptV1 {
    accepted_window:u32
    endpoint_nonce: u8[16]
}
```

The endpoint independently applies local profile and resource policy. It MUST
reply within 10 seconds.

### `OPENED`

```text
OpenedV1 {
    circuit_id:     u64
    accepted_window:u32
    endpoint_nonce: u8[16]
}
```

After `OPENED`, the requester and endpoint begin the mandatory inner session.

## Data plane

### `DATA`

```text
DataV1 {
    bytes:u8[1..16384]
}
```

`DATA` carries opaque inner bytes and does not preserve application message
boundaries.

### `WINDOW`

```text
WindowV1 {
    credit_delta:u32
}
```

Version-1 flow control is directional credit:

- initial credit is 16,384 through 1,048,576 bytes;
- default is 65,536 bytes;
- outstanding credit MUST NOT exceed 1,048,576 bytes;
- the relay MUST NOT buffer more than 65,536 bytes per direction per circuit;
- a sender without credit MUST stop;
- a violation closes the circuit.

### `CLOSE`

```text
CloseV1 {
    reason:       u16
    detail_length:u8
    detail:       u8[detail_length]
}
```

Detail is diagnostic UTF-8 of at most 128 bytes.

## Error reasons

| Value | Name |
| ---: | --- |
| 0 | `NORMAL` |
| 1 | `REFUSED` |
| 2 | `UNSUPPORTED` |
| 3 | `BUSY` |
| 4 | `INVALID` |
| 5 | `NOT_FOUND` |
| 6 | `EXPIRED` |
| 7 | `CAPACITY` |
| 8 | `TIMEOUT` |
| 9 | `PROTOCOL` |
| 10 | `INTERNAL` |
| 11 | `ENDPOINT_GONE` |
| 12 | `AUTH_FAILED` |
| 13 | `FLOW_CONTROL` |
| 14 | `RATE_LIMITED` |
| 15 | `SHUTDOWN` |
| 16 | `PROFILE_DISABLED` |
| 17 | `BYTE_LIMIT` |

Unknown future values are generic failure, never success.

## Service profile registry

| Profile ID | Name | Status |
| ---: | --- | --- |
| 1 | `HNS_NODE_V1` | Mandatory for relay implementations claiming node reachability |
| 2 | `HNS_WEB_V1` | Defined, optional for relays and endpoints |
| 3-65535 | Reserved | Future HIP assignment |

A relay MAY support only `HNS_NODE_V1`. It MUST reject unsupported profiles
before forwarding an incoming circuit.

## Profile 1: `HNS_NODE_V1`

Purpose: carry a complete Handshake peer session to an outbound-only full node.

Requirements:

1. requester is the inner Brontide initiator;
2. endpoint is the responder using `endpoint_key`;
3. Brontide MUST authenticate the endpoint key from the route record;
4. after inner encryption, both sides exchange ordinary Handshake `version` and
   `verack`;
5. ordinary peer messages then follow unchanged;
6. the endpoint exposes an in-process virtual peer, not a local TCP target;
7. node RPC and wallet RPC are never reachable through this profile;
8. inner peer data receives normal consensus validation;
9. relay transport success does not increase trust in blocks or proofs.

The complete inner peer session may carry any packet type negotiated by both
inner peers, including `ODNS` packets from the separate Oblivious DNS HIP. In
that composition, the HNSR `endpoint_key` MUST equal the ODoH target peer key
bound into the signed ODoH target configuration. HNSR authenticates the peer
path but does not validate ODoH configurations or DNS content.

An endpoint MAY publish unnamed peer routes. Upgraded nodes may discover these
through `SAMPLEROUTES` and treat them as relayed address candidates.

A mobile node is useful while online and is not penalized merely for later
disconnecting. Peer managers SHOULD record normal uptime and failure metrics
without treating expected route expiry as protocol misbehavior.

## Profile 2: `HNS_WEB_V1`

Purpose: allow an HNS-aware browser or application to reach a web service on an
outbound-only endpoint.

Canonical URI:

```text
hnsr://<hns-root>/<service-name>/<path>
```

Example:

```text
hnsr://denuoweb/p2p-site/
```

### Inner security

`HNS_WEB_V1` uses an inner Brontide session authenticated by `endpoint_key`.
HTTP is carried only after that session completes.

The endpoint key is not the long-term HNS root key. It is authorized through:

```text
authenticated HNS resource
    -> root key
    -> service authorization
    -> service key
    -> endpoint delegation
    -> endpoint key
```

### HTTP baseline

Version 1 requires HTTP/1.1 semantics over the ordered inner stream.

The requester sends:

```text
Host: <service-name>.<hns-root>
HNSR-Authority: <hns-root>
HNSR-Service: <service-name>
```

The endpoint MUST reject authority or service headers inconsistent with the
route authorization.

The profile supports ordinary request paths, status codes, headers, bodies,
streaming, and connection reuse within configured limits. Upgrade to arbitrary
raw protocols is not permitted in version 1. WebSocket and HTTP upgrade require
a future profile or explicit extension.

### Browser origin

The security origin is based on the authenticated service, not the relay:

```text
(origin scheme, HNS name hash, service name, profile ID)
```

The relay host, IP address, reservation ID, endpoint key, and current ticket do
not change the web origin.

An HNS-aware browser MUST isolate storage, cookies, permissions, and service
workers by this origin tuple. It MUST NOT assign the relay's network origin to
the content.

### Browser compatibility

An ordinary browser cannot use this profile without one of:

- native HNSR support;
- a browser extension with sufficient network capabilities;
- a local HNSR proxy integrated with the browser;
- an operating-system network provider;
- a public gateway.

A public gateway restores conventional browser compatibility but becomes an
optional gateway dependency. It is not part of the trustless native path.

### Availability

A web endpoint is available only while at least one authorized endpoint and
relay route is active. A requester SHOULD try multiple endpoint delegations and
relay tickets.

A publisher seeking continuous availability SHOULD use:

- multiple phones or computers;
- a home server and mobile fallback;
- geographically distinct endpoints;
- a conventional direct origin as another route;
- a future signed-content replication profile for static assets.

## Future service profiles

Separate HIPs may define:

- `HNS_CONTENT`: signed content manifests, chunks, replication, and caching;
- `HNS_MESSAGES`: end-to-end messaging and store-and-forward queues;
- `HNS_API`: constrained request-response APIs;
- `HNS_PREVIEW`: temporary developer preview sites;
- `HNS_REGISTRY`: registry control-plane access;
- private authenticated services;
- direct QUIC or TCP hole-punch upgrades.

New profiles MUST define endpoint behavior, inner authentication, limits,
origin or identity semantics, and whether relays may carry the profile by
default.

## Intermittent endpoint behavior

An endpoint is considered online only while:

- its outer connection to a relay is alive;
- the relay has an active reservation;
- the endpoint can accept the profile;
- at least one unexpired route record is retrievable.

When a phone changes networks or its process stops:

1. outer connections close;
2. relays invalidate reservations immediately;
3. pending and active circuits close;
4. stale rendezvous records may remain until expiration;
5. requesters trying stale tickets receive failure;
6. the phone republishes fresh records after reconnecting.

No blockchain or DNS update is required for ordinary mobility.

Recommended mobile defaults:

| Setting | Default |
| --- | ---: |
| Route lifetime | 15 minutes |
| Relay-ticket lifetime | 30 minutes |
| Refresh interval | 5 minutes |
| Active relays | 3 |
| Maximum node-profile circuits | 8 |
| Maximum web-profile circuits | 4 |
| Wi-Fi route byte budget | implementation policy |
| Cellular route byte budget | disabled or explicitly configured |

The protocol itself does not require continuous uptime. The application may
display current availability accurately rather than presenting an intermittent
endpoint as permanently online.

## Scheduling and protection of blockchain traffic

Application relay traffic shares nodes and outer connections with the
blockchain network. This is the proposal's principal architectural risk.

Every relay MUST maintain separate queues and token buckets.

Recommended priority order:

1. connection liveness and peer control;
2. blocks, headers, compact-block recovery, and chain synchronization;
3. transactions and authenticated proof traffic;
4. bounded P2P DNS relay;
5. `HNS_NODE_V1` relayed peer traffic;
6. `HNS_WEB_V1` and other application-service traffic;
7. rendezvous maintenance and background replication.

A relay MUST:

- reserve bandwidth for its own chain synchronization;
- schedule ordinary direct block traffic before relayed application data;
- prevent one circuit from occupying an outer connection;
- use fair scheduling among circuits;
- bound consecutive relay bytes;
- stop admitting new web circuits before dropping ordinary HNS peers;
- be able to disable application profiles independently of node relay;
- close or throttle relay traffic during sustained overload;
- report `BUSY` or `CAPACITY` instead of accepting unserviceable work.

Relays SHOULD use distinct global budgets for:

```text
core Handshake P2P
HNS node relay
web/application relay
rendezvous storage and lookup
```

Because inner Brontide is end-to-end, an HNSR relay cannot classify individual
inner `ODNS`, block, transaction, or proof packets. It MUST therefore enforce
the `HNS_NODE_V1` circuit budget as a whole. The target endpoint remains
responsible for applying the Oblivious DNS HIP's lower-priority scheduling to
ODoH cryptography and recursion after decryption.

## Resource limits

All implementations MUST use finite bounds.

Recommended relay defaults:

| Resource | Default |
| --- | ---: |
| Reservations per endpoint connection | 4 |
| Reservations per endpoint key | 2 |
| Provisional offers per connection | 2 |
| Active node circuits per reservation | 8 |
| Active web circuits per reservation | 4 |
| Active circuits per requester | 8 |
| Per-direction queue | 65,536 bytes |
| Maximum `DATA` body | 16,384 bytes |
| Initial window | 65,536 bytes |
| Maximum outstanding window | 1,048,576 bytes |
| Pending-open timeout | 10 seconds |
| Inner-handshake timeout | 15 seconds |
| Idle node-circuit timeout | 10 minutes |
| Idle web-circuit timeout | 2 minutes |
| Ticket lifetime | 30 minutes |

Recommended rendezvous defaults:

| Resource | Default |
| --- | ---: |
| Stored route records | 50,000 |
| Records per route key | 16 |
| Maximum record size | 8,192 bytes |
| PUT requests per peer per minute | 32 |
| GET requests per peer per minute | 120 |
| Signature verifications per peer per minute | 128 |
| Maximum returned records | 16 |
| Route lifetime | 15 minutes |

Operators MAY configure lower values.

## Costs and negative effects

This section is normative design input, not merely an implementation note.

### Relay bandwidth and hardware cost

A relay spends bandwidth in both directions and maintains circuit state,
queues, signatures, timers, and public connections. A popular website or large
node synchronization can consume materially more resources than ordinary peer
maintenance.

Open relay participation may therefore be unattractive without payment,
reciprocity, reputation, or operator motivation. The likely result could be
many endpoints depending on a much smaller number of well-provisioned relays.

### Possible harm to block propagation

If relay traffic is not isolated, website downloads, media, or initial block
synchronization through circuits can delay direct blocks and headers. This can
increase stale work, slow synchronization, and weaken the primary blockchain
function of participating nodes.

Conformance therefore requires separate budgets and priority scheduling.
Application relay MUST be optional independently of node relay.

### Relay concentration

Although any public HNS node may opt in, bandwidth cost and abuse exposure may
cause only a few operators to do so. A protocol that is permissionless in
specification can still be operationally centralized.

Clients and endpoints SHOULD measure relay diversity and SHOULD NOT count many
endpoints behind one relay set as many independent ingress paths.

### Denial of service against phones and home devices

The relay intentionally makes previously unreachable devices reachable.
Attackers may open circuits, consume battery and data, hold sessions idle,
request expensive resources, or repeatedly trigger application parsing.

Relays and endpoints MUST independently rate-limit. Android implementations
SHOULD expose battery, metered-network, thermal, and byte-budget controls.

### Abuse complaints and public-IP reputation

Outside observers see relay IP addresses. Relay operators may receive abuse
reports, scanner alerts, denial-of-service traffic, provider complaints, or IP
reputation penalties for encrypted content they cannot inspect.

This burden may discourage participation and concentrate relaying.

### Privacy and metadata

A relay observes endpoint and requester addresses, timing, duration, and byte
counts. Rendezvous nodes observe service route keys, endpoint keys, relay
choices, and approximate online periods.

Binding a human-readable HNS name to a mobile endpoint can create a persistent
association between the name and changing IP addresses, even when content is
encrypted.

### Larger attack surface

The extension adds:

- a DHT-style routing table;
- signed mutable records;
- reservation state;
- multiplexed circuits;
- flow control;
- virtual streams;
- profile dispatch;
- browser-origin integration.

These increase code complexity and create new parsing, memory, state-machine,
and authentication failure modes. Implementations SHOULD isolate HNSR from
consensus code and should fuzz all message decoders.

### Sybil and storage spam

An attacker can create many endpoint keys and signed unnamed node routes.
Signatures prove control, not scarcity. Storage nodes require per-peer,
per-prefix, per-route-key, and global bounds. Named-service lookup is less
susceptible to irrelevant spam because clients derive one exact key from an
authenticated HNS name.

### Stale discovery records

A phone may disconnect immediately after publishing. Remote rendezvous nodes
cannot always receive an authenticated withdrawal before returning the route.
Short lifetimes and relay-side immediate invalidation bound this problem but do
not eliminate failed dials.

### Intermittent website availability

A website hosted only from one phone disappears when the phone is offline,
killed by the operating system, overheated, out of battery, or without network
service. The protocol makes reachability possible; it does not make phones
reliable data centers.

### No built-in economic incentive

Version 1 does not pay relay or rendezvous operators. Heavy users may consume
resources supplied by volunteers. This can become a tragedy-of-the-commons
problem or drive operators toward private allowlists.

### Topology distortion

A topology explorer may see thousands of relayed endpoints but only a few
independent relay ingress points. Monitoring tools SHOULD distinguish:

```text
directly reachable nodes
relayed endpoints
unique relay keys
independent relay netgroups
named application services
```

### Browser compatibility and origin risk

Native HNSR web support requires new browser behavior. Incorrectly assigning a
relay's origin to remote content, or combining storage across HNS services,
would create severe security bugs. `HNS_WEB_V1` origin rules are mandatory.

## Security considerations

### Relay trust

A relay can deny, censor, delay, disconnect, or observe metadata. It cannot
forge the authorization chain or endpoint inner key when signatures and
Brontide are correctly validated.

Endpoints SHOULD maintain at least three relay reservations. Requesters SHOULD
try independent relay and endpoint combinations.

### Rendezvous trust

Storage nodes can omit or replay unexpired records. Requesters query several
paths, select the highest valid endpoint sequence, and validate all signatures.
A storage response is not proof that the endpoint is online.

### End-to-end authentication

Every initial profile requires an inner Brontide session authenticated by the
endpoint key. The relay's outer encryption is insufficient because it
terminates at the relay.

### Replay

A valid route record may be replayed until expiration. The issuing relay still
rejects a locally invalid reservation. Requesters MUST reject expired records,
older endpoint sequences, wrong network magic, and changed HNS root authority.

### Arbitrary forwarding prohibition

No packet contains a requester-selected local address or port. A circuit is
dispatched only by registered profile ID to a configured in-process handler.

A conforming endpoint MUST NOT implement version 1 as:

```text
OPEN -> connect to requester-provided 127.0.0.1:<port>
```

It MAY implement a web handler that internally serves configured content, but
that configuration is local and is never selected by a remote dialer.

### Circuit isolation

Circuit IDs are unpredictable and local to one relay. Each is bound to exactly
one requester outer connection, one endpoint outer connection, one ticket, and
one profile. Data from another connection is invalid.

### Amplification

All messages travel over established bidirectional Handshake connections.
There is no UDP listener. Response sizes, storage, circuits, and byte windows
are bounded.

### Eclipse and route censorship

An attacker controlling the nearest rendezvous nodes can suppress records.
Publishers store at several XOR-near and diversity-selected peers; requesters
query several independent paths. A future version may add signed storage
receipts or alternate replicas.

The extension does not change chain selection. Blocks and proofs from relayed
peers receive ordinary validation.

## Privacy considerations

The protocol is not anonymous.

The endpoint's selected relay knows its network address. Requesters reveal
their address to selected relays. Rendezvous nodes see route keys and records.
End-to-end encryption protects application plaintext but not timing and volume.

When an ODoH proxy uses HNSR to reach an outbound-only target, the HNSR relay
sees the proxy connection and target endpoint but not the original ODoH
requester connection or DNS plaintext. This separation depends on the ODoH
proxy, HNSR relay, and target not combining their observations.

Possible mitigations include:

- several relays;
- endpoint-key rotation;
- short route lifetimes;
- staggered refresh;
- private-service profiles in a future HIP;
- relay-only routes that omit direct addresses;
- multi-hop or blinded lookup in later versions.

## Peer scoring

Misbehavior may include:

- malformed lengths;
- invalid signatures;
- duplicate live context IDs;
- impossible state transitions;
- flow-control violations;
- excessive storage or lookup requests;
- invalid route-key derivation;
- repeated expired-record injection;
- attempts to select arbitrary local targets.

Availability failure alone is not protocol misbehavior. A relay declining work,
a phone going offline, or a stale ticket failing SHOULD affect route quality,
not automatically cause a peer ban.

## Compatibility

This proposal is consensus-compatible and requires no hard fork.

Legacy peers:

- do not advertise HNSR service bits;
- never receive `HNSR` from conforming upgraded peers;
- continue ordinary block and peer behavior;
- cannot discover or dial HNSR-only endpoints;
- cannot browse HNSR web services.

Upgraded implementations may support any combination:

- endpoint;
- requester;
- rendezvous node;
- node-profile relay;
- web-profile relay;
- named-service publisher;
- HNS-aware browser.

Application relay may be disabled while node relay remains enabled.

ODoH carriage requires no HNSR protocol assignment beyond `HNS_NODE_V1`. A
relay or endpoint may support HNSR node circuits without advertising or
implementing the ODoH service bit or packet type.

## Suggested configuration

Illustrative, nonnormative flags:

```text
Rendezvous node:
  --hnsr-rendezvous
  --hnsr-records=50000
  --hnsr-record-bytes=256mb

Node-only relay:
  --hnsr-relay=node
  --hnsr-node-bandwidth=20mbit
  --hnsr-max-reservations=128

Node and web relay:
  --hnsr-relay=node,web
  --hnsr-web-bandwidth=10mbit
  --hnsr-web-max-bytes-per-circuit=64mb

Android endpoint:
  --hnsr-endpoint
  --hnsr-relays=3
  --hnsr-publish-node
  --hnsr-publish-web=denuoweb/p2p-site
  --hnsr-cellular-byte-budget=0
```

No external relay URL, resolver URL, tunnel account, or separate bootstrap list
is configured.

## Implementation architecture

```text
Handshake peer manager
    |
    +-- core block/header/transaction/proof processing
    |
    +-- optional P2P DNS relay
    |
    +-- HNSR controller
          |
          +-- HNS authority validator
          +-- service authorization store
          +-- rendezvous routing table
          +-- route record store
          +-- relay reservation table
          +-- circuit multiplexer
          +-- profile scheduler
          +-- virtual stream adapters
                  |
                  +-- HNS node profile
                  +-- HNS web profile
```

Consensus code SHOULD depend on none of these components.

On Android:

```text
foreground service
    |
embedded hsrd
    |
HNSR endpoint and profile handlers
    |
outbound ordinary HNS peers
    |
relay-capable HNS peers
```

Incoming node circuits become virtual peers. Incoming web circuits enter only
the explicitly configured web handler.

## Reference implementation and regtest evidence

A bounded `hsd` proof of concept is available at:

- draft implementation PR:
  [handshake-org/hsd#960](https://github.com/handshake-org/hsd/pull/960);
- exact implementation commit:
  [`2c26892df9fb437b1c84f39fc4ecfe329b1d4922`](https://github.com/denuoweb/hsd/tree/2c26892df9fb437b1c84f39fc4ecfe329b1d4922);
- exact native Android diagnostic commit:
  [`30137230e67f944861833ee5d6a7ac6783cf46d4`](https://github.com/Denuo-Web/hns-dane-browser/tree/30137230e67f944861833ee5d6a7ac6783cf46d4);
- branch documentation: `docs/experimental-hnsr.md`; and
- reproducible runners: `scripts/run-hnsr-regtest-trial.js` and
  `scripts/run-hnsr-phase2-trial.js`; and
- checked-in evidence: `docs/hnsr-regtest-phase1.json`,
  `docs/hnsr-regtest-phase2.json`, and
  `docs/hnsr-phase2-android.png`.

The reference branches implement the bounded Phase 1A unnamed-node and Phase
1B named-service slices, plus the local engineering portion of Phase 2:

- the version-1 envelope and private regtest or acknowledged-testnet
  capability advertisement;
- iterative `FINDNODE` / `NODES` discovery with XOR ordering, parallelism of
  three, a 32-query ceiling, authenticated contacts, and dialing of discovered
  Handshake peers;
- publisher-driven route replication with an eight-copy default and a
  three-store quorum;
- deterministic bounded `SAMPLEROUTES`;
- `RESERVE`, `OFFER`, `CONFIRM`, `CONFIRMED`, `RENEW`, and signed `WITHDRAW`;
- mutually signed relay tickets using strict-DER, low-S secp256k1 signatures;
- self-authorized unnamed endpoint delegations and route records;
- bounded, expiring, sequence-aware route storage with global, per-key, and
  per-publishing-peer limits;
- `PUTROUTE`, `PUTRESULT`, `GETROUTE`, and `ROUTES`;
- multi-relay records, higher-sequence republishing, and sequential failover;
- `OPEN`, `INCOMING`, `ACCEPT`, and `OPENED`;
- opaque `DATA`, directional `WINDOW`, and `CLOSE`;
- circuit-count, byte, frame, directional-credit, and queue limits;
- yielding relay scheduling, bounded control admission, and observable
  saturation counters;
- immediate local reservation invalidation on endpoint disconnect; and
- ordinary inbound and outbound `hsd` `Peer` objects carrying a complete
  end-to-end inner Brontide and Handshake session;
- current validated, closed HNS name-state lookup and exact canonical
  `hnsr1 k=<base32-key>` TXT parsing with ambiguity rejection;
- root-signed `ServiceAuthorizationV1`, service-signed named endpoint
  delegation, named route-key derivation, and requester-side validation of the
  complete HNS-to-endpoint authority chain;
- replicated named records for two independently keyed endpoints;
- `HNS_WEB_V1` inner endpoint-authenticated Brontide with bounded HTTP/1.1
  request and response framing;
- Host, HNS authority, and service agreement, plus stable origin derivation
  independent of relay, ticket, and endpoint rotation;
- connection reuse up to the configured 16-request circuit limit; and
- sequential named-endpoint failover after primary disconnection;
- persistent 256-bucket XOR routing with `k = 16`, a 2,048-contact cap,
  failure eviction, bounded discovery dials, public-address admission, and
  netgroup diversity;
- optional atomic persistence of contacts, routes, and endpoint sequences,
  with restart recovery and bounded per-prefix state;
- eight-copy route publication, scheduled refresh, immediate network-change
  refresh, and scored relay selection;
- per-prefix control budgets, signature-verification budgets, requester,
  endpoint, IP, netgroup, profile, and global circuit limits;
- separate bounded `HNS_NODE_V1` and `HNS_WEB_V1` relay budgets with 3:1 node
  weighting and operational telemetry; and
- an Android debug-derived build whose Rust/JNI client performs native
  Handshake and HNSR discovery, sampling, exact lookup, and authorization-chain
  verification after Android default-network changes.

The remaining work is separated by milestone rather than treated as one
undifferentiated conformance gap:

- **Phase 2, external testnet operation:** run the checked-in topology across
  several independently administered hosts and networks, exercise real address
  and netgroup diversity, sustain churn and saturation over a soak period, and
  review the resulting operator telemetry and scheduler behavior;
- **Phase 3, clients:** RPC, address-manager, wallet, SPV, production Android
  lifecycle and hardware-backed signing, and browser integration; and
- **public-network readiness:** assigned values, privacy review, production
  abuse controls, reputation or payment policy, deployment gates, and
  sustained public-network measurements.

The Phase 1 topology deliberately remains a four-rendezvous regression. The
Phase 2 topology separately proves persistent XOR buckets, eight initial
stores, durable restart recovery, and five live stores after three rendezvous
failures. Because all twelve keyed processes run on one workstation, that
artifact is not evidence of independent operators or public-network
conformance.

### Reproduction

From the exact `hsd` commit:

```sh
npm ci
NODE_BACKEND=js npm run test-file -- \
  test/hnsr-test.js test/brontide-test.js test/net-test.js
NODE_BACKEND=js node scripts/run-hnsr-regtest-trial.js \
  docs/hnsr-regtest-phase1.json
NODE_BACKEND=js node scripts/run-hnsr-phase2-trial.js \
  docs/hnsr-regtest-phase2.json
```

`NODE_BACKEND=js` selects bcrypto's portable JavaScript backend and is not a
wire-protocol requirement.

The focused suites report 77 passing tests, including 27 HNSR tests. They cover
the envelope, truncation, reserved flags, zero context IDs, unknown versions
and opcodes, reservation and renewal binding, ticket and route round trips,
strict low-S rejection, route-key substitution, sequence replacement,
expiration, canonical and ambiguous HNS root-key TXT records, named authority
round trips and substitution failures, origin stability and separation,
duplicate and oversized HTTP headers, transfer-coding rejection, pipelined
framing, authenticated rendezvous contact encoding and XOR ordering,
deterministic sampling, persistent XOR bucket and netgroup bounds, per-prefix
storage quotas, durable restart recovery, network-change republishing,
profile-weighted relay scheduling, circuit backpressure, role advertisement,
the regtest/testnet gates, Brontide, and the existing network packet suite.

The successful 2026-07-21 trial creates nine actual `hsd` FullNodes with
independent prefixes and identity keys:

```text
Endpoint/fallback web (no listener) ==> Relay A, Relay B, Rendezvous 0
Primary web endpoint (no listener)  ==> Relay A, Relay B, Rendezvous 0
Requester                           ==> Rendezvous 0
Rendezvous 0 ==> Rendezvous 1 ==> Rendezvous 2 ==> Rendezvous 3

Requester == inner HNS peer ==> surviving relay ==> Endpoint
Requester == inner HNS_WEB_V1 ==> relay ==> authenticated named endpoint
```

It verifies:

1. a shared regtest chain auctions `phase1b` and authenticates its canonical
   root key by HNS `UPDATE` at height 47;
2. the root authorizes `p2p-site`, two endpoints publish named records, and
   each publication reaches all four rendezvous nodes;
3. the requester validates the full named authority chain and receives an HTTP
   200 response over inner endpoint-authenticated Brontide;
4. one inner circuit carries 16 ordered request/response exchanges and rejects
   a seventeenth request at the configured limit;
5. a mismatched web authority returns 421, the stable origin excludes relay and
   endpoint identity, and relay-visible DATA excludes response plaintext;
6. disconnecting the primary named endpoint causes the stale candidate to fail
   before the authenticated fallback endpoint returns 200;
7. iterative discovery reaches all four rendezvous nodes from one bootstrap;
8. two relay reservations are stored at all four rendezvous nodes;
9. `SAMPLEROUTES` discovers the unnamed endpoint;
10. both tickets renew, a higher route sequence is replicated, and both old
   reservations withdraw;
11. 72 concurrent lookups produce 64 accepted and 8 rate-limited requests;
12. lookup succeeds from three surviving replicas after one rendezvous stops;
13. opening falls through from stopped Relay A to Relay B;
14. requester and endpoint create ordinary `hsd` `Peer` objects and mutually
   authenticate their inner static keys;
15. 1,000 ordinary Handshake pings saturate the circuit while a control
   reservation completes;
16. a mined block raises only endpoint and requester from height 47 to 48 while
    the relay and rendezvous control chains remain at height 47;
17. the relay queue exceeds one scheduler burst, remains below 65,536 bytes,
    drains over multiple yields, and records zero drops; and
18. endpoint disconnect removes its live reservation while the stale cached
    route remains retrievable and its ticket is rejected.

The checked-in evidence records these results, opcode counts, byte and queue
counters, block convergence, and a hash of all relay-visible opaque `DATA`
frames. Fresh identities, signatures, route values, and block hashes make exact
artifact bytes intentionally unstable between successful runs.

The successful Phase 2 trial creates twelve actual `hsd` FullNodes with
independent identity keys and persistent prefixes: two relays, eight chained
rendezvous nodes, one endpoint, and one requester. It verifies:

1. iterative discovery of all eight rendezvous identities and eight initial
   route stores;
2. recovery of a persisted contact and route after restarting one rendezvous
   process;
3. five live stored copies and successful exact lookup after three rendezvous
   processes stop;
4. immediate higher-sequence republishing after a simulated network change;
5. fallback from a stopped relay to the second authenticated ticket;
6. rejection at the eight-circuit requester bound;
7. mixed acceptance and rejection of 40 validly signed spam records at the
   verification and storage budgets;
8. 64 accepted and 96 rate-limited results during a 160-request lookup burst;
9. a 1 MiB `HNS_WEB_V1` request saturating the web profile while an actual
   block traverses an inner `HNS_NODE_V1` peer on the same relay; and
10. a 49,325-byte peak relay queue, zero drops, and separate relay,
    rendezvous, and endpoint telemetry snapshots.

The companion `hnsrTest` APK was built from the cited Android commit and run on
a physical Pixel 9 through an ADB-reversed connection to the held Phase 2
fixture. Its Rust client completed ordinary Handshake `version` / `verack`,
required the HNSR service bit, performed `FINDNODE`, `SAMPLEROUTES`, and exact
`GETROUTE`, and verified the delegation, both relay tickets, record signature,
route key, and network binding. Disabling and restoring Wi-Fi advanced the
Android default-network generation and produced a third successful fresh
native probe. This is lifecycle and protocol evidence for the dedicated test
client; it is not a claim that browser navigation uses HNSR.

## Deployment

### Phase 1A: unnamed-node regtest

- one endpoint;
- two relays;
- four rendezvous nodes;
- one requester;
- unnamed node route;
- inner Brontide;
- full node peer traffic;
- route renewal and withdrawal;
- rendezvous and relay failure;
- forced endpoint disconnection;
- stale route handling;
- strict load and scheduling tests.

**Reference status (2026-07-21): complete for the bounded unnamed-node
experiment.** The implementation and checked-in evidence cover every item in
this list. Its yielding PoC data scheduler and admission limits are sufficient
for the regtest saturation experiment; the larger profile-aware scheduler and
telemetry exercise is recorded separately by the Phase 2 trial.

### Phase 1B: named-service regtest

- authenticated HNS root-key discovery;
- named service authorization and delegation;
- named web route;
- `HNS_WEB_V1` inner Brontide;
- HTTP request and response;
- authority and browser-origin enforcement;
- named endpoint failover; and
- web-specific limits and adversarial tests.

**Reference status (2026-07-21): complete for the bounded named-service
experiment.** The implementation and checked-in evidence cover every item in
this list at the HSD protocol layer. HTTP messages are deliberately buffered
within the 16 KiB header and 1 MiB body limits; an unbounded streaming API is
not claimed. HSD derives the authenticated origin tuple and enforces its
authority inputs, but native browser isolation of cookies, storage,
permissions, and service workers remains Phase 3 client work.

### Phase 2: testnet

- several independent operators;
- persistent XOR buckets and larger-topology route churn;
- eight-copy replication and diversity under partial failure;
- Android network switching;
- node and web relay limits;
- DDoS simulation;
- production-priority block propagation measurements under relay saturation;
- route spam and Sybil tests.

**Reference status (2026-07-21): complete for the bounded single-workstation
engineering trial; external operator validation remains.** The implementation
and checked-in evidence cover persistent routing, eight-copy churn, Android
network switching, node/web budgets, DDoS admission, valid route spam, and
block propagation under web saturation. Before this milestone can be called a
testnet deployment, the same procedure must run across several independently
administered machines and networks and complete a sustained soak with operator
review.

### Phase 3: opt-in mainnet experiment

- maintainer-approved experimental assignments;
- conservative defaults;
- node profile enabled before web profile;
- aggregate resource telemetry;
- explicit operator opt-in;
- no conformance claim before permanent assignments.

### Phase 4: assigned protocol

- permanent service bits and packet type;
- interoperable implementations;
- published test vectors;
- fuzz corpora;
- Android and desktop clients;
- topology reporting that distinguishes relay dependence.

## Testing requirements

### Encoding and cryptography

- canonical TXT root key parsing;
- service authorization round trip;
- endpoint delegation round trip;
- relay ticket round trip;
- route record round trip;
- minimum, maximum, truncated, and trailing-byte cases;
- strict DER and low-S rejection;
- wrong network magic;
- wrong route key;
- reserved fields and unknown versions.

### Rendezvous

- XOR-distance routing;
- bucket bounds;
- iterative lookup termination;
- eight-replica store;
- partial storage failure;
- record expiration;
- sequence replacement;
- named exact lookup;
- unnamed route sampling;
- Sybil storage pressure;
- per-peer verification budget;
- no global gossip requirement.

### Reservations and circuits

- successful reserve, offer, confirm, and renew;
- endpoint disconnect;
- stale ticket rejection;
- circuit capacity;
- requester limits;
- profile-disabled error;
- flow-control exhaustion;
- queue overflow;
- idle timeout;
- clean shutdown;
- circuit-ID isolation.

### Node profile

- inner Brontide endpoint authentication;
- inner `version` / `verack`;
- blocks, headers, transactions, and proofs;
- invalid block behavior identical to direct peers;
- mobile endpoint disconnect without false ban;
- successful republish after network change;
- optional ODoH target composition over the inner peer session;
- ODoH target peer key equal to the HNSR endpoint key;
- relay and rendezvous nodes never receiving DNS plaintext;
- route and ticket rotation without ODoH HPKE configuration rotation.

### Web profile

- HNS root-key validation;
- service authorization and delegation;
- URI parsing;
- origin isolation;
- HTTP authority checks;
- body and connection limits;
- multiple endpoint failover;
- phone offline response;
- malicious relay byte modification detection.

### Load

- direct block propagation under saturated web relay;
- chain synchronization under rendezvous load;
- fair scheduling across circuits;
- signature-verification exhaustion;
- global byte-budget enforcement;
- Android battery, thermal, and metered-network behavior.

## Rationale

### Why use the existing HNS peer network

The user already needs Handshake peers to validate and use HNS. Reusing those
connections avoids creating a mandatory third-party resolver or tunnel service.
The rendezvous and relay roles remain optional and replaceable.

### Why a keyed rendezvous overlay

Global gossip is simple but scales with every published route. XOR-keyed
storage limits each record to a small deterministic replica set and keeps
lookup traffic proportional to the requested service.

### Why HNS state authorizes named services

A human-readable service such as `denuoweb/p2p-site` needs an authority stronger
than whichever rendezvous node returns a route. The current authenticated HNS
resource commits to the root key, and all short-lived authorization derives
from it.

### Why node reachability does not require an HNS name

A Handshake peer already has a cryptographic transport identity. Requiring a
name for every mobile full node would add cost and friction without improving
ordinary peer validation. Human-readable application services do require a
name binding.

### Why inner Brontide

Outer peer encryption terminates at the relay. A new inner session authenticates
the endpoint and protects application bytes from the relay.

### Why profiles instead of arbitrary streams

A generic stream-to-local-port protocol would become a general reverse tunnel
and expose unrelated services. Profiles define exact handlers, authentication,
origin semantics, and limits that can be reviewed and independently enabled.

### Why web relay is optional

Web traffic can be substantially heavier and more abuse-prone than node peer
traffic. Operators must be able to support node reachability without accepting
arbitrary website bandwidth.

### Why offline phones are acceptable

The protocol publishes presence, not permanence. Temporary nodes still add
capacity while online. Continuous service availability is achieved through
replication and multiple endpoints rather than pretending a phone is always
connected.

## Alternatives considered

### Manual port forwarding

Does not work when the operator lacks gateway control or a public address.

### UPnP, NAT-PMP, and PCP

Useful when available, but not universal and generally ineffective behind
carrier-grade NAT.

### Separate reverse-tunnel provider

Works technically but creates an external account, operator, hostname, and
availability dependency.

### Dedicated libp2p or other rendezvous network

Provides mature primitives but creates another bootstrap and peer population.
Its techniques may inform implementation, but this proposal carries the
subprotocol over HNS peers.

### Dynamic DNS

Can publish an address but cannot make a device behind CGNAT reachable.

### On-chain route records

Too slow, public, and expensive for minute-scale mobile presence changes.

### Authoritative DNS route records

Reintroduce a separately reachable authoritative service and still do not
create an inbound path.

### Global ticket gossip

Does not scale with large numbers of frequently refreshed mobile routes.

### Public port allocation for legacy clients

Requires a separate public port per endpoint and does not scale. Version 1
requires upgraded requesters.

### Multi-hop relay

Increases privacy possibilities but also latency, bandwidth, state, and denial
of service. Deferred.

## Future work

- direct hole-punch upgrade;
- signed storage receipts;
- relay payment and reciprocal accounting;
- static-content replication;
- private and access-controlled services;
- blinded route keys;
- multi-hop privacy routes;
- compact or Schnorr signatures;
- QUIC inner profiles;
- native browser integration;
- service continuity across HNS ownership changes;
- relay reputation and diversity attestations;
- cross-implementation conformance suite;
- a dedicated restricted ODoH target profile only if complete
  `HNS_NODE_V1` sessions prove unnecessarily broad or difficult to budget.

## Reference flows

### Mobile full node

```text
Android node       Relay A       Rendezvous peers       Dialer
     |                |                 |                  |
     |-- outbound --->|                 |                  |
     |-- RESERVE ---->|                 |                  |
     |<- OFFER -------|                 |                  |
     |-- CONFIRM ---->|                 |                  |
     |<- CONFIRMED ---|                 |                  |
     |-- PUTROUTE --------------------->|                  |
     |                |                 |<- GETROUTE ------|
     |                |                 |-- ROUTES -------->|
     |                |<----------------------- OPEN -------|
     |<- INCOMING ----|                 |                  |
     |-- ACCEPT ----->|                 |                  |
     |                |---------------------- OPENED ------>|
     |<========== inner Brontide and HNS peer session =====>|
```

### Outbound-only Oblivious DNS target

```text
ODoH Proxy        HNSR Rendezvous       HNSR Relay       ODoH Target
    |                    |                   |                 |
    |-- GETROUTE ------->|                   |                 |
    |<-- ROUTES ---------|                   |                 |
    |-------------------------------- OPEN ->|                 |
    |                    |                   |-- INCOMING ---->|
    |                    |                   |<-- ACCEPT ------|
    |<------------------------------- OPENED-|                 |
    |<========= inner Brontide + Handshake peer session ======>|
    |-------------------------------- GETCONFIG -------------->|
    |<------------------------------- CONFIG -----------------|
    |-------------------------------- TARGET_QUERY ----------->|
    |<------------------------------- TARGET_RESPONSE --------|
```

The relay ticket and route record are short-lived reachability state. The ODoH
target configuration remains a separate target-signed object and contains no
relay address, reservation ID, or ticket ID.

### Phone-hosted HNS web service

```text
HNS name state
  denuoweb/ -> HNSR root key
          |
          v
root-signed authorization for p2p-site
          |
          v
phone endpoint delegation + relay tickets
          |
          v
route stored near hash(denuoweb, p2p-site, HNS_WEB_V1)
          |
          v
HNS-aware browser opens relay circuit
          |
          v
inner Brontide + HTTP/1.1
          |
          v
phone serves hnsr://denuoweb/p2p-site/
```

## References

1. RFC 2119, *Key words for use in RFCs to Indicate Requirement Levels*.
2. RFC 8174, *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*.
3. Handshake `hsd` peer service-bit negotiation and network addresses.
4. Handshake `hsd` nine-byte P2P framing.
5. Handshake Brontide encrypted transport.
6. Handshake P2P `GETPROOF` / `PROOF` precedent.
7. Draft HIP, *Handshake P2P DNS Relay*.
8. Kademlia, Maymounkov and Mazières, 2002, for XOR-keyed routing concepts.
9. Draft HIP, *Handshake P2P Transport for Oblivious DNS Relay*.
10. `hsd` proof of concept, `handshake-org/hsd#960`, exact commit
    `2c26892df9fb437b1c84f39fc4ecfe329b1d4922`.
11. Native Android HNSR diagnostic, exact commit
    `30137230e67f944861833ee5d6a7ac6783cf46d4`.
