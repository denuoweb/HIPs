# HIP-xxxx : Handshake P2P Transport for Oblivious DNS Relay

```text
Number:     HIP-xxxx
Title:      Handshake P2P Transport for Oblivious DNS Relay
Type:       Standards Track
Status:     Draft
Authors:    Jaron Rosenau <@denuoweb>
Created:    2026-07-19
Updated:    2026-07-20
Requires:   Handshake P2P DNS Relay (draft HIP)
Related:    Handshake P2P Rendezvous and Authenticated Service Relay (draft HIP, optional)
```

## Abstract

This document specifies an optional privacy extension to the Handshake P2P DNS
Relay protocol. It carries RFC 9230 Oblivious DNS over HTTPS (ODoH) messages
through ordinary Handshake peer connections instead of through HTTP, separating
the peer that observes the requester's network address from the peer that
decrypts and resolves the DNS question.

A requester encrypts a DNS query to an ODoH target key operated by a
relay-capable Handshake resolver peer, then sends the encrypted message to a
different Handshake peer acting as an oblivious proxy. The proxy forwards the
opaque message to the selected target over the existing Handshake P2P network.
The target sees the DNS question and the proxy connection but not the
requester's direct connection; the proxy sees the requester and selected target
but cannot decrypt the DNS question or response. The requester decrypts the
response and performs the same local HNS-state, DNSSEC, and DANE validation
required by the base Handshake P2P DNS Relay HIP.

This proposal introduces no separately configured DoH resolver, proxy URI,
HTTP service, dedicated rendezvous network, or new public listener. Proxy and
target roles are optional services performed by ordinary Handshake peers. A
target may be reached directly or, when both implementations support the
separate HNSR HIP, through an HNSR `HNS_NODE_V1` circuit. HNSR support is
optional and does not change the ODoH message, validation, or trust model. The
extension requires no consensus change. A future public protocol would need one
Handshake service bit and one versioned P2P packet type, but this draft does
not request them.

## Status and dependency

This HIP is a strict extension of the draft **Handshake P2P DNS Relay** HIP,
referred to throughout this document as the **base HIP**.

The base HIP defines:

- authenticated requester-side HNS state;
- `NODE_DNS_RELAY`;
- the restricted HNS recursive DNS query profile;
- relay admission and bounded recursion;
- raw DNS response handling;
- local DNSSEC and DANE validation;
- resource limits and scheduling;
- direct requester-to-resolver `GETDNSRELAY` / `DNSRELAY` exchanges.

This HIP does not replace or fork those rules. An oblivious target:

1. decrypts an ODoH query;
2. extracts one raw DNS query;
3. processes that query exactly as a base-HIP relay would;
4. places the resulting raw DNS response in an ODoH response;
5. encrypts the response for the requester.

If the base HIP changes its admitted DNS profile, recursion boundary, response
requirements, or local-validation requirements, this HIP inherits those
changes unless an explicit later amendment says otherwise.

The placeholder dependency name MUST be replaced with the assigned HIP number
before this proposal advances from Draft.

The draft **Handshake P2P Rendezvous and Authenticated Service Relay** HIP,
referred to as **HNSR**, is an optional transport dependency only when an
outbound-only target is selected. Direct target operation and conformance to
this HIP do not require HNSR. HNSR supplies temporary peer reachability; this
HIP continues to define target-key authentication, ODoH cryptography, DNS
admission, proxy/target separation, and requester-side validation.

## Motivation

The base HIP removes the need for a separately configured third-party recursive
DoH fallback by allowing a requester to ask an ordinary Handshake peer to
perform bounded HNS recursion.

Its direct privacy relationship is:

```text
Requester  ------------------------>  P2P DNS relay
             plaintext DNS query
```

The relay is not trusted for authenticity because the requester verifies HNS
state, DNSSEC, and DANE locally. However, the relay still learns both:

- the requester's network identity or address; and
- the DNS name and record type being requested.

For a browser, wallet, resolver, or mobile client, repeated direct requests can
reveal:

- browsing destinations;
- services used under an HNS name;
- timing and frequency of access;
- associations among multiple queried names;
- approximate user location from the requester address;
- long-term behavior when the same peer connection is reused.

RFC 9230 addresses this metadata concentration by splitting resolution across a
Client, Proxy, and Target:

```text
Client  ---- encrypted query ---->  Proxy  ---- encrypted query ---->  Target

Proxy learns:
    client address + target identity
    but not plaintext DNS

Target learns:
    proxy address + plaintext DNS
    but not the client's direct connection
```

The standard RFC 9230 deployment uses HTTP and requires discovery of a proxy URI
template and target public keys. Handshake already has:

- encrypted or ordinary persistent peer connections;
- peer discovery and address gossip;
- cryptographic peer identity keys;
- service-bit negotiation;
- the base P2P DNS relay target;
- bounded binary packet framing;
- multiple replaceable peers.

This HIP binds the ODoH message format and HPKE encryption to that existing
network. It does not introduce an HTTP layer merely to transport binary ODoH
messages between peers.

## Privacy objective

The primary objective is that no single non-colluding intermediary learns both:

```text
requester network identity
            +
plaintext DNS query
```

The information split is:

| Participant | Learns requester connection | Learns selected target | Learns plaintext DNS |
| --- | --- | --- | --- |
| Requester | Yes | Yes | Yes |
| Proxy | Yes | Yes | No |
| HNSR rendezvous node, when used | No original requester connection; normally sees the proxy lookup | Route key and candidate records | No |
| HNSR relay, when used | No original requester connection; sees the proxy connection | Endpoint and ticket | No |
| Target | No direct requester connection | Itself | Yes |
| Authoritative DNS server | No direct requester connection | Resolver source | Query needed for recursion |

This property depends on proxy and target non-collusion. It does not provide
anonymity against a global passive observer and does not prevent correlation by
timing, size, rare queries, or other side channels.


When HNSR is used, its relay observes a proxy-to-target circuit and its
rendezvous nodes may observe HNSR route-key lookups. They do not receive the
requester's direct connection or ODoH plaintext under this protocol. Operator
collusion, timing correlation, and a global passive observer remain outside the
privacy guarantee.

## Relationship to RFC 9230

This HIP uses the RFC 9230:

- `ObliviousDoHConfigs` encoding;
- `ObliviousDoHConfig` version `0x0001`;
- `ObliviousDoHMessagePlaintext`;
- query message type `0x01`;
- response message type `0x02`;
- `key_id` derivation;
- HPKE query encryption;
- response-secret derivation;
- encrypted response encoding;
- all-zero plaintext padding requirement;
- mandatory HPKE cipher suite.

This HIP replaces only the HTTP transport and discovery bindings.

The following RFC 9230 HTTP elements do not appear on the Handshake wire:

- HTTP POST;
- `application/oblivious-dns-message`;
- HTTP request or response headers;
- proxy URI templates;
- `targethost`;
- `targetpath`;
- HTTP status codes;
- `Proxy-Status`;
- HTTPS target connections.

Their functions are replaced by:

| RFC 9230 HTTP concept | Handshake P2P binding |
| --- | --- |
| Proxy URI | connected Handshake proxy peer |
| Target host and path | authenticated Handshake target locator |
| HTTP request correlation | hop-local 64-bit request ID |
| HTTP content type | versioned `ODNS` packet opcode |
| HTTP error | bounded `ODNS ERROR` status |
| HTTPS target transport | ordinary Handshake peer connection |
| Target-key discovery | signed ODoH target configuration record |

Although the payload cryptography is ODoH, this transport is not HTTP and does
not claim to implement RFC 8484 HTTP semantics. The precise description is:

> RFC 9230 ODoH messages transported through Handshake P2P as an extension of
> the Handshake P2P DNS Relay.

## Conventions and terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in RFC 2119
and RFC 8174 when, and only when, they appear in all capitals.

Terms:

- **Requester**: the ODoH client and final DNS validator.
- **Proxy**: the first Handshake peer, which observes the requester connection
  and forwards opaque ODoH messages.
- **Target**: a Handshake peer that owns an ODoH private key, decrypts the DNS
  query, and performs base-HIP HNS recursion.
- **Target locator**: a versioned, requester-selected description of how the
  proxy reaches a target peer. Version 1 permits a direct public Handshake
  listener or an optional HNSR unnamed `HNS_NODE_V1` route.
- **Direct target locator**: a public IPv4 or IPv6 Handshake listener and a
  33-byte target peer identity key.
- **HNSR target locator**: a target peer identity key whose current unnamed
  HNSR route is derived and resolved under the HNSR HIP.
- **Target configuration**: one or more RFC 9230 ODoH public-key
  configurations, bound to a target locator and signed by the target peer key.
- **Client request ID**: a request identifier used only on the requester-proxy
  connection.
- **Target request ID**: an independently generated request identifier used
  only on the proxy-target connection.
- **ODoH query**: an RFC 9230 `ObliviousDoHMessage` with message type `0x01`.
- **ODoH response**: an RFC 9230 `ObliviousDoHMessage` with message type
  `0x02`.
- **Base query profile**: the complete DNS query-admission profile specified by
  the base HIP.
- **Authenticated HNS state**: the locally validated state required by the base
  HIP.
- **Peer identity key**: the compressed 33-byte secp256k1 key carried by a
  Handshake network address and used for authenticated peer identity.
- **Netgroup**: a local address-manager grouping used to avoid selecting
  obviously related peers.
- **Live request**: a request accepted but not completed, cancelled, expired, or
  failed.
- **Outer padding**: zero bytes in an `ODNS` packet used to reduce packet-size
  correlation. It is separate from RFC 9230 plaintext padding.
- **Resolver cache**: the target's ordinary recursive DNS cache.
- **Exchange cache**: caching of an encrypted ODoH request-response exchange;
  this is forbidden.

## Goals

1. Extend the base P2P DNS relay without changing its validation or recursion
   model.
2. Prevent a non-colluding proxy from learning plaintext DNS questions.
3. Prevent a non-colluding target from learning the requester's direct
   connection.
4. Use ordinary Handshake peers for proxy and target roles.
5. Avoid a separately configured resolver, HTTP proxy, or proxy URI.
6. Authenticate target ODoH keys using existing Handshake peer identity.
7. Keep request identifiers and connection metadata hop-local.
8. Preserve local requester-side HNS, DNSSEC, and DANE validation.
9. Bound CPU, memory, bandwidth, recursion, and cryptographic work.
10. Prevent use as a general arbitrary TCP or HTTP proxy.
11. Permit incremental deployment and explicit privacy downgrade behavior.
12. Keep block, header, and consensus-related P2P work higher priority than
    DNS privacy services.
13. Optionally reach outbound-only targets through HNSR without changing the
    ODoH wire messages or DNS trust model.

## Non-goals

This HIP does not provide:

- anonymity against a global passive observer;
- protection when proxy and target collude;
- protection when both roles are controlled by one operator despite different
  addresses;
- hiding the target identity from the proxy;
- hiding the DNS query from the target;
- hiding authoritative DNS traffic from authoritative servers;
- a general HTTP proxy;
- the HNSR rendezvous, reservation, ticket, or circuit protocol itself;
- arbitrary target hosts or ports;
- non-HNS open recursion;
- recursive service for arbitrary ICANN questions;
- onion routing or more than one oblivious proxy hop;
- consensus changes;
- new HNS name-resource records;
- on-chain ODoH keys;
- mandatory payment or accounting;
- application user-interface rules;
- silent downgrade to direct DNS relay;
- an exchange cache for encrypted requests or responses.

## Architecture

A normal exchange uses three Handshake peers:

```text
Requester                  Proxy                     Target
    |                        |                          |
    | HNS version/verack     |                          |
    |<---------------------->|                          |
    |                        | HNS version/verack       |
    |                        |<------------------------>|
    |                        |                          |
    |-- encrypted ODoH Q --->|                          |
    |                        |-- encrypted ODoH Q ----->|
    |                        |                          | decrypt
    |                        |                          | base-HIP recursion
    |                        |                          | encrypt
    |                        |<-- encrypted ODoH R -----|
    |<-- encrypted ODoH R ---|                          |
    | decrypt                |                          |
    | local validation       |                          |
```

The requester selects both roles.

The proxy MUST NOT choose a substitute target, because the query is encrypted
to the selected target configuration. A substitute target cannot decrypt it.

The proxy-to-target leg is an ordinary Handshake peer connection. The proxy MAY
reuse an existing connection or establish a new one, subject to address-manager
and resource policy.


The proxy-to-target transport has two permitted forms:

```text
Direct:
Requester -> ODoH Proxy -> direct Handshake target connection

Optional HNSR:
Requester -> ODoH Proxy -> HNSR relay circuit -> outbound-only target
```

In the HNSR form, the proxy is the HNSR circuit requester and the target is an
unnamed `HNS_NODE_V1` endpoint. The inner session is still a complete
authenticated Handshake peer session. `GETCONFIG`, `TARGET_QUERY`,
`TARGET_RESPONSE`, and all target capability checks run unchanged inside that
session. HNSR rendezvous nodes and relays never process ODoH plaintext.

## Required separation

For each oblivious query:

- proxy peer key and target peer key MUST differ;
- the proxy MUST reject a locator whose target peer key equals its own peer key;
- the requester MUST NOT knowingly select the same peer for both roles;
- the requester SHOULD select proxy and target from different netgroups;
- the requester SHOULD select different autonomous systems when that
  information is available;
- the requester SHOULD avoid a target to which it has a direct live peer
  connection;
- the requester SHOULD avoid a target it contacted directly for configuration
  retrieval;
- the requester SHOULD rotate among several proxies and targets over time;
- a node advertising both roles MAY be used as a proxy in one exchange and as a
  target in another, but MUST NOT perform both roles for the same exchange.

Distinct addresses are not proof of distinct operators. Operator independence
cannot be established from protocol data alone.

## Assignments

No permanent assignments are requested by this draft. The identifiers below
document the registry space a future public protocol would need after threat-
model review, multi-operator testing, base-relay validation, and a separate
maintainer decision:

| Symbol | Registry | Value |
| --- | --- | ---: |
| `NODE_DNS_OBLIVIOUS` | `version.services` bit | TBD |
| `ODNS` | Handshake P2P packet type | TBD |

The base target role also requires `NODE_DNS_RELAY` from the base HIP.

Prototype implementations MUST use private experimental assignments on
controlled networks. They MUST NOT advertise those values as this standard on
mainnet. A later standards-track revision may request permanent values only
after the review and deployment gates above are met.

The accompanying regtest-only `hsd` proof of concept uses:

| Symbol | Private PoC value |
| --- | ---: |
| `EXPERIMENTAL_ODOH_SERVICE` | `0x20000000` |
| `EXPERIMENTAL_ODOH` | `0xf2` |

These are collision-prone implementation values, not requested registry
assignments. The PoC refuses to enable proxy or target roles outside regtest.
Future experiments MUST negotiate different private values or establish that
these values remain unused; assigned implementations MUST replace them.

One packet type is used because a versioned internal opcode separates
capability, configuration, query, response, cancellation, and error messages.

## Role negotiation

`NODE_DNS_OBLIVIOUS` states that the peer understands `ODNS` version 1. It does
not by itself state which role is enabled.

After ordinary `version` / `verack`, an interested peer sends `GETCAPS`.

A conforming implementation may enable:

- proxy only;
- target only;
- target-configuration cache only;
- proxy and target;
- all roles.

A target MUST also advertise `NODE_DNS_RELAY`. A peer MUST NOT advertise the
target role merely because it can parse ODoH messages; it requires:

- a ready base-HIP recursive backend;
- sufficiently synchronized HNS state;
- one or more active ODoH private keys;
- a current signed target configuration;
- target-specific limits and replay protection.

A proxy does not require a recursive backend. It requires:

- bounded target connection management;
- request mapping;
- opaque forwarding;
- per-client and per-target limits;
- no arbitrary destination forwarding.

Capability state may change after negotiation. A peer at temporary capacity
returns `BUSY` rather than violating the protocol.

## Existing Handshake framing

`ODNS` uses the ordinary nine-byte Handshake frame:

```text
HandshakeFrame {
    magic:          u32le
    packet_type:    u8
    payload_length: u32le
    payload:        u8[payload_length]
}
```

This HIP does not change the outer framing or general Handshake message-size
limit.

All `ODNS` envelope integers are unsigned little-endian. RFC 9230 structures
retain their specified TLS-style network-byte-order encoding internally.

## `ODNS` envelope

Every `ODNS` payload begins:

```text
ODNSEnvelopeV1 {
    version:    u8
    opcode:     u8
    flags:      u16
    request_id: u64
    body:       u8[remaining]
}
```

Requirements:

- `version` MUST equal `1`;
- version 1 defines no envelope flags, so `flags` MUST be zero;
- `request_id` MUST be nonzero;
- request IDs are scoped to one outer peer connection;
- the payload MUST be at least 12 bytes;
- exact body length is derived from the outer Handshake frame;
- a parser MUST reject truncated, trailing, or structurally inconsistent data;
- limits MUST be checked before allocating declared variable-length fields;
- unknown versions fail the exchange without a ban solely for future-version
  use;
- unknown opcodes fail the exchange;
- repeated malformed messages MAY increase peer misbehavior score.

## Opcodes

| Value | Name | Direction |
| ---: | --- | --- |
| 0 | `GETCAPS` | requester to capable peer |
| 1 | `CAPS` | capable peer to requester |
| 2 | `GETCONFIG` | requester to proxy or proxy to target |
| 3 | `CONFIG` | target or cache to proxy/requester |
| 4 | `CLIENT_QUERY` | requester to proxy |
| 5 | `TARGET_QUERY` | proxy to target |
| 6 | `TARGET_RESPONSE` | target to proxy |
| 7 | `CLIENT_RESPONSE` | proxy to requester |
| 8 | `CANCEL` | requester to proxy or proxy to target |
| 9 | `ERROR` | either direction |

Values 10 through 255 are reserved.

A future opcode requires either compatible version-1 semantics or a new
envelope version.

## Capabilities

### `GETCAPS`

`GETCAPS` has an empty body.

### `CAPS`

```text
CapsV1 {
    roles:                     u8
    max_client_query:          u16
    max_target_response:       u32
    max_live_per_connection:   u16
    max_config_size:           u16
    preferred_query_bucket:    u16
    preferred_response_bucket: u16
}
```

Role bits:

| Bit | Name |
| ---: | --- |
| 0 | `PROXY` |
| 1 | `TARGET` |
| 2 | `CONFIG_CACHE` |
| 3-7 | Reserved |

Requirements:

- at least one defined role bit MUST be set;
- reserved role bits MUST be zero;
- a target role is invalid unless the same peer advertised `NODE_DNS_RELAY`;
- `max_client_query` MUST be between 256 and 8,192;
- `max_target_response` MUST be between 512 and 65,535;
- `max_live_per_connection` MUST be between 1 and 256;
- `max_config_size` MUST be between 128 and 16,384;
- preferred padding buckets MUST be powers of two between 128 and 4,096, or
  zero to indicate no preference;
- capability limits are admission maxima, not guaranteed service.

The requester MUST use the lower of its local limit and the advertised limit.

## Target locator

Version 1 uses a tagged locator so the DNS privacy protocol remains independent
of the mechanism used to reach the selected target:

```text
TargetLocatorV1 {
    locator_type:        u8
    target_peer_key:     u8[33]
    locator_body_length: u16
    locator_body:        u8[locator_body_length]
}
```

Locator types:

| Value | Name | Body |
| ---: | --- | --- |
| 0 | `DIRECT_HANDSHAKE_TCP` | `DirectTargetBodyV1` |
| 1 | `DIRECT_HANDSHAKE_BRONTIDE` | `DirectTargetBodyV1` |
| 2 | `HNSR_NODE_V1` | `HNSRTargetBodyV1` |

Values 3 through 255 are reserved.

The target peer key MUST be a valid compressed secp256k1 key. The body length
MUST exactly match the selected locator type, and no trailing bytes are
permitted.

### Direct target locator

```text
DirectTargetBodyV1 {
    host_type: u8
    host:      u8[16]
    port:      u16
}
```

Host types:

| Value | Meaning |
| ---: | --- |
| 4 | IPv4 encoded as an IPv4-mapped 16-byte address |
| 6 | Native IPv6 |

Direct-locator requirements:

- the host MUST be publicly routable;
- the port MUST be nonzero;
- the address MUST identify an ordinary Handshake P2P listener;
- private, local, multicast, unspecified, documentation-only, or otherwise
  unroutable addresses MUST be rejected on public networks;
- test networks MAY permit explicitly configured private ranges, but MUST NOT
  do so by default;
- the proxy MUST complete the selected direct transport and Handshake
  `version` / `verack` before forwarding.

### HNSR target locator

```text
HNSRTargetBodyV1 {
    profile_id: u16
}
```

Version 1 requires `profile_id` to equal the HNSR profile ID for
`HNS_NODE_V1`. The current unnamed route key is derived without placing
short-lived relay state in the ODoH configuration:

```text
hnsr_route_key =
    BLAKE2b-256(
        "HNSR-PEER-ROUTE-V1\0"
        || network_magic_u32le
        || target_peer_key
    )
```

An HNSR-capable proxy:

1. performs an HNSR lookup for `hnsr_route_key`;
2. validates candidate HNSR route records and relay tickets under the HNSR HIP;
3. selects an active route for the exact `target_peer_key`;
4. opens an `HNS_NODE_V1` circuit;
5. completes inner Brontide with the target as responder;
6. verifies that the authenticated HNSR endpoint key equals
   `target_peer_key`;
7. completes inner Handshake `version` / `verack`;
8. applies the same ODoH target service-bit and `CAPS` checks as a direct
   connection.

The signed target configuration MUST NOT contain an HNSR relay address,
reservation ID, ticket ID, route-record sequence, or other short-lived route
material. Those values are discovered and validated at connection time. An
ODoH target therefore may change networks or relays without rotating its HPKE
configuration solely because its HNSR route changed.

### Common locator requirements

For every locator:

- the current target connection MUST authenticate `target_peer_key`;
- the current target connection MUST advertise both `NODE_DNS_OBLIVIOUS` and
  `NODE_DNS_RELAY`;
- target `CAPS` MUST include `TARGET`;
- the proxy MUST NOT interpret the locator as an HTTP, DNS, SOCKS, or generic
  TCP destination;
- the requester selects the locator and the proxy MUST NOT rewrite the target
  peer key or substitute another locator type;
- for an HNSR locator, the proxy MAY choose among independently valid current
  tickets for the same derived route key;
- failure of one locator does not authorize silent use of a different locator;
  retry through a different locator requires a configuration and query bound
  to that locator.

## Signed target configuration

RFC 9230 leaves target-key discovery outside scope. This HIP defines a signed
Handshake P2P configuration record.

```text
TargetConfigRecordV1 {
    record_version:   u8
    network_magic:    u32
    target_locator:   TargetLocatorV1
    sequence:         u64
    issued_at:        u64
    expires_at:       u64
    configs_length:   u16
    odoh_configs:     u8[configs_length]
    signature_length: u8
    target_signature: u8[signature_length]
}
```

`odoh_configs` is exactly one RFC 9230 `ObliviousDoHConfigs` vector containing
one or more configurations in decreasing preference order.

Requirements:

- `record_version` MUST equal `1`;
- `network_magic` MUST match the active Handshake network;
- target locator rules MUST pass;
- the target MUST issue or return a record only for a locator it controls and
  recognizes;
- a direct locator MUST identify a listener configured for the signing target;
- an HNSR locator requires the signing target to publish or be capable of
  publishing an unnamed `HNS_NODE_V1` route under the same target peer key;
- `sequence` MUST increase for newly issued records under the same target key;
- `issued_at` MUST be no more than 300 seconds in the future;
- `expires_at` MUST be later than `issued_at`;
- record lifetime MUST NOT exceed 172,800 seconds;
- default record lifetime SHOULD be 86,400 seconds;
- `configs_length` MUST be between 1 and 16,384;
- each supported ODoH configuration MUST parse canonically;
- unknown ODoH versions are ignored as required by RFC 9230;
- signature length MUST be between 8 and 72 bytes;
- the signature MUST be strict DER, low-S secp256k1 ECDSA;
- no trailing bytes are permitted.

### Configuration signature

The unsigned record ends immediately after `odoh_configs`.

The signature digest is:

```text
config_digest =
    BLAKE2b-256(
        "HNS-P2P-ODOH-CONFIG-V1\0"
        || canonical_unsigned_record
    )
```

`target_signature` is made by the private key corresponding to the
`target_peer_key` encoded in `target_locator`.

A requester MUST verify the signature before using an ODoH public key.

The target peer identity authenticates the target configuration but is not a
DNS trust anchor. DNS authenticity continues to come from requester-side HNS,
DNSSEC, and DANE validation.

### Configuration identifier

```text
config_record_id =
    BLAKE2b-256(
        complete_canonical_target_config_record
    )
```

The requester places this identifier in `CLIENT_QUERY`. The target uses it to
ensure the query references a record it issued and still recognizes.

## Key rotation

RFC 9230 recommends regular target key rotation. A conforming target:

- SHOULD issue a fresh preferred HPKE key every 24 hours;
- SHOULD retain the immediately previous private key for at least two hours;
- MUST NOT advertise an expired key;
- MUST keep enough overlap to tolerate cached configurations and in-flight
  requests;
- SHOULD publish the new configuration before it becomes preferred;
- MUST increase `sequence` for each newly signed record;
- SHOULD retain at least two configurations during an overlap;
- MUST delete retired private keys according to local secure-key policy.

Very short rotations reduce the client anonymity set. Very long rotations
increase exposure after key compromise. Implementations SHOULD default to one
day unless deployment evidence supports another interval.

## Configuration retrieval

A requester SHOULD retrieve target configurations through the selected proxy,
rather than opening a direct target connection.

### `GETCONFIG`

```text
GetConfigV1 {
    target_locator: TargetLocatorV1
    allow_cached: u8
}
```

`allow_cached` is `0` or `1`.

Requester-to-proxy behavior:

1. The requester selects a target locator.
2. It sends `GETCONFIG` to a proxy.
3. The proxy checks locator and role restrictions.
4. If `allow_cached` is one and the proxy has a current, validated target
   record, it MAY answer from cache.
5. Otherwise, the proxy establishes the target connection selected by the
   locator. For an HNSR locator it performs route lookup, ticket validation,
   circuit opening, inner Brontide, and inner Handshake negotiation first.
6. The proxy sends a hop-local `GETCONFIG` with a fresh target request ID.
7. The target returns `CONFIG` containing the exact requested locator or an
   error; it MUST NOT sign an unrelated locator merely because the request
   arrived over an authenticated connection.
8. The proxy validates basic structure and forwards the complete record
   unchanged.

A proxy MUST NOT rewrite or re-sign a target record.

A configuration cache:

- MAY store only signature-valid, unexpired records;
- MUST evict no later than `expires_at`;
- MUST prefer the highest valid sequence for one target key;
- MUST retain older unexpired records when needed for in-flight requests;
- MUST use bounded storage;
- MUST NOT treat receipt frequency as trust;
- MUST NOT infer DNS authenticity from the record.

### `CONFIG`

```text
ConfigV1 {
    config_record_length: u16
    config_record:        u8[config_record_length]
}
```

The response request ID is hop-local and matches the relevant `GETCONFIG`.

The requester performs complete validation. A proxy MAY perform the same
validation to protect its cache but is not trusted to do so.

## ODoH cryptography

This HIP adopts RFC 9230 and RFC 9180 without defining new cryptography.

A conforming implementation MUST support:

- ODoH configuration version `0x0001`;
- DHKEM(X25519, HKDF-SHA256);
- HKDF-SHA256;
- AES-128-GCM;
- RFC 9230 base-mode HPKE query encryption;
- RFC 9230 response-secret derivation;
- RFC 9230 `key_id` derivation;
- RFC 9230 query type `0x01`;
- RFC 9230 response type `0x02`.

Additional RFC 9230-compatible suites MAY be supported when present in the
signed configuration.

An implementation MUST use established, reviewed HPKE and AEAD libraries. It
MUST NOT implement an ad hoc ECIES construction or reuse the target peer key as
the HPKE key.

The target peer key signs configurations. The independent ODoH key encrypts DNS
queries.

## DNS query construction

Before encryption, the requester constructs one DNS query satisfying every
base-HIP requirement, including:

- one `IN` question;
- admitted opcode and question type;
- EDNS version zero;
- DNSSEC OK;
- no EDNS Client Subnet;
- no TSIG or SIG(0);
- HNS rightmost label;
- request-size bounds;
- base-HIP handling of `CD`;
- base-HIP padding rules.

The requester MUST authenticate current HNS state for the root before accepting
the final answer. It MAY perform that work before or concurrently with the
oblivious exchange.

The DNS query is encoded as an RFC 9230 `ObliviousDoHMessagePlaintext`:

```text
ObliviousDoHMessagePlaintext {
    dns_message<1..2^16-1>
    padding<0..2^16-1>
}
```

The padding bytes MUST all be zero.

The requester encrypts the plaintext under the selected ODoH public key and
constructs an RFC 9230 `ObliviousDoHMessage` query. The resulting message is
opaque to the proxy.

## Query padding

Implementations SHOULD follow RFC 8467:

- queries SHOULD be padded so the plaintext DNS message reaches a multiple of
  128 octets where practical;
- responses SHOULD be padded to a multiple of 468 octets where practical;
- padding MUST be all zero inside the ODoH plaintext;
- the padding option or ODoH padding must be the last padding operation at that
  layer;
- resource-constrained devices MAY use lower padding subject to local privacy
  policy.

Because the Handshake packet frame exposes total size, this HIP also permits
outer padding.

Outer padding is not encrypted end to end and provides only size-class
obfuscation. It MUST contain zeros and is independently regenerated on each
hop.

Recommended outer buckets:

```text
Requests:   512, 1024, 2048, 4096, 8192 bytes
Responses:  1024, 2048, 4096, 8192, 16384, 32768, 65535 bytes
```

A sender selects the smallest bucket that fits. Implementations MAY use the
peer's preferred bucket from `CAPS`.

No padding scheme prevents timing correlation.

## Client query

### `CLIENT_QUERY`

```text
ClientQueryV1 {
    target_locator:       TargetLocatorV1
    config_record_id:     u8[32]
    odoh_message_length:  u16
    odoh_query:           u8[odoh_message_length]
    outer_padding_length: u16
    outer_padding:        u8[outer_padding_length]
}
```

Requirements:

- the envelope request ID is the client request ID;
- the request ID MUST be unique among live requests on the requester-proxy
  connection;
- target locator rules MUST pass;
- `config_record_id` MUST identify the record used for encryption;
- `odoh_query` MUST be structurally parseable as an RFC 9230 message type
  `0x01`, without decrypting it;
- length MUST be within local and advertised limits;
- outer padding MUST contain only zeros;
- no trailing bytes are allowed;
- the packet contains no plaintext DNS name;
- the packet contains no target HTTP path or URI;
- the proxy MUST NOT attempt to decrypt the ODoH message;
- the proxy MUST NOT forward the client request ID to the target.

The proxy MAY reject a query when it lacks a current configuration record for
the supplied ID. It MUST NOT require plaintext inspection.

## Proxy target connection

Before forwarding, the proxy:

1. verifies that the selected target peer key differs from its own;
2. verifies that the target locator is permitted;
3. enforces per-client and global limits;
4. obtains a target connection as follows:
   - for a direct locator, it reuses or establishes the specified ordinary
     Handshake connection;
   - for an HNSR locator, it resolves the derived unnamed route, validates the
     HNSR authorization and tickets, opens an `HNS_NODE_V1` circuit, and
     completes the inner Brontide session;
5. completes or confirms Handshake `version` / `verack` on the target session;
6. authenticates the connected peer as the locator's `target_peer_key`;
7. confirms current `NODE_DNS_OBLIVIOUS` and `NODE_DNS_RELAY`;
8. confirms target `CAPS`;
9. allocates a fresh random target request ID;
10. stores a bounded mapping from target request ID to client request ID and
    requester connection;
11. sends `TARGET_QUERY`.

The target request ID MUST be independent of the client request ID. It MUST NOT
be derived by truncation, increment, encryption, or reversible transformation
of the client request ID.

A proxy SHOULD maintain long-lived multiplexed target connections or HNSR
circuits where policy and route lifetime permit. It SHOULD NOT open and close a
target connection for every DNS query, because per-query connections make
timing correlation easier and consume network resources.

For an HNSR target, loss or expiry of the selected relay ticket closes only the
transport path. It does not invalidate an otherwise current signed ODoH target
configuration. The proxy may resolve a new valid ticket for the same target
peer key and locator type, subject to retry and fresh-query rules.

## Target query

### `TARGET_QUERY`

```text
TargetQueryV1 {
    config_record_id:     u8[32]
    odoh_message_length:  u16
    odoh_query:           u8[odoh_message_length]
    outer_padding_length: u16
    outer_padding:        u8[outer_padding_length]
}
```

Requirements:

- the envelope request ID is the fresh target request ID;
- the request ID is unique among live requests on the proxy-target connection;
- the proxy MUST copy the ODoH query bytes without modification;
- the proxy MAY strip and regenerate outer padding;
- the proxy MUST NOT include the requester IP, peer key, user agent, client
  request ID, or connection nonce;
- the target MUST recognize the configuration record ID;
- the ODoH message MUST use a current or overlap-retained key;
- the target MUST enforce ciphertext and concurrency limits before HPKE work;
- the target MUST not infer requester identity from proxy-supplied metadata,
  because no such metadata is permitted.

## Target processing

On a valid `TARGET_QUERY`, the target:

1. looks up the configuration and private key;
2. performs cheap length and replay checks;
3. decrypts the ODoH query under RFC 9230;
4. verifies all ODoH plaintext padding bytes are zero;
5. parses the embedded DNS message;
6. applies the complete base-HIP DNS query profile;
7. derives the HNS root;
8. applies the base-HIP name-state admission boundary;
9. performs bounded HNS recursion using the base-HIP backend;
10. obtains one raw DNS response or a bounded transport failure;
11. creates an RFC 9230 ODoH response plaintext;
12. applies response padding;
13. derives response secrets under RFC 9230;
14. encrypts the response;
15. sends `TARGET_RESPONSE`.

The target's `AD` bit, validation result, and local chain view are not trust
anchors. The requester performs final validation exactly as in the base HIP.

A target MUST NOT accept a query through this HIP that the base HIP would
reject.

## Target response

### `TARGET_RESPONSE`

```text
TargetResponseV1 {
    odoh_message_length:  u32
    odoh_response:        u8[odoh_message_length]
    outer_padding_length: u16
    outer_padding:        u8[outer_padding_length]
}
```

Requirements:

- the envelope request ID matches the target request ID;
- the message is structurally parseable as RFC 9230 type `0x02`;
- the encrypted response is no larger than 65,535 bytes;
- outer padding contains zeros;
- no trailing bytes are allowed;
- the target returns no plaintext DNS response;
- the target returns no client-specific metadata.

The proxy removes the target mapping and sends `CLIENT_RESPONSE`.

## Client response

### `CLIENT_RESPONSE`

```text
ClientResponseV1 {
    odoh_message_length:  u32
    odoh_response:        u8[odoh_message_length]
    outer_padding_length: u16
    outer_padding:        u8[outer_padding_length]
}
```

Requirements:

- the envelope request ID is the original client request ID;
- the proxy MUST copy the ODoH response bytes unchanged;
- the proxy MAY strip and regenerate outer padding;
- the proxy MUST NOT cache the encrypted exchange;
- the proxy MUST remove request mapping after forwarding;
- duplicate terminal responses are invalid.

The requester:

1. parses the RFC 9230 response structure;
2. derives and opens the response using its saved query context;
3. verifies response padding is all zero;
4. extracts the raw DNS response;
5. validates DNS structure and query correlation;
6. validates the answer against authenticated HNS state;
7. performs DNSSEC validation locally;
8. performs TLSA and DANE validation locally when applicable;
9. ignores the target's `AD` bit as a trust assertion;
10. applies freshness and retry rules from the base HIP.

Only after these steps may the application use the answer.

## Cancellation

### `CANCEL`

```text
CancelV1 {
    reason: u8
}
```

Cancellation reasons:

| Value | Meaning |
| ---: | --- |
| 0 | requester no longer needs the result |
| 1 | local deadline expired |
| 2 | alternate route succeeded |
| 3 | application cancelled |

Requester to proxy:

- the request ID identifies the live client request;
- the proxy deletes or marks the mapping cancelled;
- the proxy MAY send a target-side `CANCEL` using the target request ID.

Proxy to target:

- the request ID identifies the live target request;
- the target SHOULD stop work when safely possible;
- cancellation is best effort;
- cancellation does not permit request ID reuse until state cleanup completes.

A cancelled response received later MUST be discarded.

## Errors

### `ERROR`

```text
ErrorV1 {
    status:      u8
    retry_after: u32
    error_class: u16
}
```

Statuses:

| Value | Name | Origin |
| ---: | --- | --- |
| 0 | `REFUSED` | proxy or target |
| 1 | `UNSUPPORTED` | proxy or target |
| 2 | `BUSY` | proxy or target |
| 3 | `INVALID_OUTER` | receiving hop |
| 4 | `TARGET_UNREACHABLE` | proxy |
| 5 | `TARGET_TIMEOUT` | proxy |
| 6 | `CONFIG_UNKNOWN` | target or proxy |
| 7 | `CONFIG_EXPIRED` | target or proxy |
| 8 | `TARGET_FAILURE` | generic target failure |
| 9 | `RESPONSE_TOO_LARGE` | target or proxy |
| 10 | `RATE_LIMITED` | proxy or target |
| 11 | `CANCELLED` | proxy or target |
| 12 | `INTERNAL_ERROR` | proxy or target |

Values 13 through 255 are reserved.

`retry_after` is seconds and MAY be zero.

`error_class` is diagnostic and MUST NOT contain a DNS RCODE, name, question
type, key-existence oracle, HPKE failure detail, or target stack error.

### Error privacy

The target MAY internally distinguish:

- unknown key;
- invalid HPKE encapsulation;
- AEAD failure;
- invalid ODoH padding;
- malformed decrypted DNS;
- base-HIP query rejection;
- recursion failure.

The proxy-facing result SHOULD expose only the minimum class needed for
resource control.

The requester-facing proxy MUST map decryption, padding, malformed plaintext,
and other target cryptographic failures to generic `TARGET_FAILURE`.

A requester MUST NOT depend on fine-grained cryptographic errors.

Ordinary DNS `NOERROR`, `NXDOMAIN`, `SERVFAIL`, `REFUSED`, truncation, and
other DNS conditions remain inside a successful encrypted ODoH response. They
MUST NOT be exposed as outer transport statuses.

## Request state

### Requester state

```text
IDLE
  |
  | obtain target config
  v
CONFIGURED
  |
  | CLIENT_QUERY
  v
WAITING
  | \
  |  \ CANCEL / timeout / ERROR
  |   v
  |  FAILED
  |
  | CLIENT_RESPONSE
  v
DECRYPT
  |
  | local validation
  v
COMPLETE
```

### Proxy state

```text
CLIENT_ACCEPTED
  |
  | allocate independent target ID
  v
TARGET_PENDING
  |
  | TARGET_RESPONSE
  v
FORWARD_CLIENT
  |
  v
CLOSED
```

Any terminal error, cancellation, timeout, disconnect, or limit failure closes
the mapping.

### Target state

```text
TARGET_QUERY
  |
  | cheap checks
  v
DECRYPT
  |
  | base-HIP admission and recursion
  v
ENCRYPT_RESPONSE
  |
  v
TARGET_RESPONSE
```

Packets inconsistent with state fail the exchange. Repeated impossible state
transitions MAY increase peer misbehavior score.

## Hop-local correlation

The proxy is the only participant that holds the temporary mapping:

```text
(requester connection, client request ID)
                <->
(target connection, target request ID)
```

Requirements:

- mappings MUST be memory-bounded;
- mappings MUST expire no later than the request deadline;
- mappings MUST be deleted after a terminal response;
- mappings MUST NOT be persisted to disk by default;
- target request IDs MUST be random and independent;
- client request IDs MUST never appear on the target leg;
- target request IDs MUST never appear on the client leg;
- user agents, peer nonces, and requester keys MUST not be copied between legs.

## Connection pooling and mixing

A proxy SHOULD maintain multiplexed, long-lived connections to several targets.

For each target connection, the proxy SHOULD:

- carry requests from multiple requester peers;
- permit several requests in flight;
- assign random independent request IDs;
- avoid strict FIFO response assumptions;
- avoid reconnecting per query;
- avoid dedicating a target connection to one requester;
- enforce fair scheduling across requesters.

A proxy MAY introduce small bounded jitter before forwarding, but MUST NOT delay
past the base-HIP recursive deadline.

A proxy MUST NOT claim that connection pooling or jitter defeats a global
observer.

## Target selection

The requester, not the proxy, selects the target.

A requester SHOULD build a target candidate set from peers that:

- advertise `NODE_DNS_OBLIVIOUS`;
- advertise `NODE_DNS_RELAY`;
- return target capability;
- have a valid signed target configuration;
- for an HNSR locator, have at least one currently valid unnamed HNSR route
  record and relay ticket when selection occurs;
- are sufficiently synchronized when that information is available;
- differ from the selected proxy;
- differ from current direct requester peers where possible;
- have acceptable historical availability;
- satisfy netgroup diversity.

For an HNSR locator, the requester SHOULD also prefer route records whose relay
keys and netgroups are distinct from the selected ODoH proxy and from other
selected target operators. Distinct relay and target keys do not prove distinct
operators.

A requester SHOULD randomize within the acceptable set.

It SHOULD NOT always choose the lowest-latency target, because that can create
stable metadata concentration.

## Proxy selection

A requester SHOULD select a proxy from an already established Handshake peer
connection when possible, provided that peer advertises proxy capability.

Using an existing connection:

- avoids a distinctive new connection per DNS query;
- reduces latency;
- increases the mix of DNS and ordinary P2P traffic;
- avoids separately configured proxy infrastructure.

A requester SHOULD rotate proxy choice over time but SHOULD avoid rotating for
every single query when doing so creates distinctive connection patterns.

## Direct requester-target connections

If the requester already maintains a direct blockchain peer connection to a
candidate target, the target knows the requester's network address independently
of ODoH.

The ODoH ciphertext still hides which query came from that requester, but
timing can weaken the intended separation.

Therefore, a requester SHOULD select a target with which it has no direct live
connection. It MAY maintain a local exclusion window after disconnecting from a
target before using it for oblivious queries.

Suggested default exclusion window:

```text
30 minutes
```

This is a privacy recommendation, not a wire rule.

## Replay handling

HPKE confidentiality does not by itself prevent a proxy from replaying an
identical encrypted query.

A target SHOULD maintain a bounded rolling replay filter over:

```text
BLAKE2b-256(
    config_record_id
    || odoh_query
)
```

Suggested replay window:

```text
10 minutes
```

On duplicate detection, the target SHOULD return generic `TARGET_FAILURE` or
silently discard according to local policy.

The replay filter:

- MUST be memory bounded;
- MUST expire entries;
- MUST not become a permanent query log;
- MUST not expose duplicate status to the requester;
- MAY use a probabilistic rolling filter;
- MAY produce false positives, which are availability failures rather than
  authenticity failures.

Each legitimate requester query SHOULD use fresh HPKE encapsulation material.

## Resolver caching

RFC 9230 states that encrypted ODoH request-response messages have no cache
value. Therefore:

- proxy exchange caching is forbidden;
- target encrypted-response caching is forbidden;
- requester exchange reuse is forbidden;
- a proxy MUST NOT replay a cached response for a repeated ciphertext.

The target's ordinary recursive DNS backend MAY use a DNS cache under the same
rules as the base HIP. This cache stores validated or resolver-managed DNS
state, not encrypted ODoH exchanges.

## Base-HIP recursion boundary

After decryption, the target MUST apply all base-HIP constraints, including:

- original question must be an admitted HNS question;
- the rightmost label must be a valid Handshake name;
- local chain state must be sufficiently ready;
- the original root must pass active-name admission;
- arbitrary non-HNS open recursion is forbidden;
- query types and opcodes are restricted;
- EDNS Client Subnet is forbidden;
- authority traffic is bounded;
- alias and out-of-bailiwick behavior follows the base HIP;
- response size and recursive deadline are bounded;
- private-address and test-network rules follow the base HIP;
- DNSSEC material needed by the requester is preserved.

This extension MUST NOT weaken the base HIP's abuse boundary merely because the
target cannot identify the original requester.

## Local requester validation

The oblivious target is an untrusted resolver for authenticity purposes.

The requester MUST:

- validate current HNS state from its full node or verified headers and Urkel
  proof;
- verify the root delegation against that state;
- parse and validate DNSSEC data locally;
- reject bogus DNSSEC;
- process authenticated denial proofs;
- enforce freshness;
- validate TLSA and DANE locally when applicable;
- ignore target `AD` as a trust statement;
- reject a response that does not match the original question;
- apply the same failover and stale-data rules as the base HIP.

ODoH protects metadata separation. It does not replace DNSSEC or HNS-state
authentication.

## Resource limits

Every implementation MUST define finite limits for:

- live client requests per requester connection;
- live target requests per proxy-target connection;
- live requests per proxy IP and netgroup;
- live requests globally;
- config retrievals;
- cached configurations;
- HPKE decryptions per interval;
- target replay-filter size;
- recursive jobs;
- response bytes;
- outer padding;
- target connection attempts;
- mapping memory;
- timeouts;
- error rate.

Recommended defaults:

| Resource | Default |
| --- | ---: |
| Client live requests per connection | 16 |
| Proxy live requests globally | 1,024 |
| Target live requests per proxy connection | 64 |
| Target live recursive jobs globally | 256 |
| Config records cached | 4,096 |
| Configs per target key | 4 |
| Config maximum size | 16,384 bytes |
| ODoH query maximum | 8,192 bytes |
| ODoH response maximum | 65,535 bytes |
| Query deadline | 10 seconds |
| Config retrieval deadline | 5 seconds |
| Proxy-target connect deadline | 5 seconds |
| Replay window | 10 minutes |
| Replay-filter entries | 100,000 |
| Mapping idle timeout | 15 seconds |
| Per-connection target attempts | 8 per minute |

Implementations MAY use lower limits.

## Scheduling

ODoH work is optional application traffic on the blockchain peer network.

A proxy and target MUST schedule:

1. peer liveness and protocol control;
2. blocks, headers, compact-block recovery, and chain synchronization;
3. transaction and proof traffic;
4. direct base-HIP DNS control;
5. oblivious DNS configuration and queries.

Exact scheduling may differ, but ODoH MUST NOT starve consensus-related work.

A target SHOULD return `BUSY` before allowing HPKE or recursion queues to delay
block processing.

A proxy SHOULD preserve a separate token bucket for ODoH bytes and target
connection work.

## DoS resistance

### Proxy exposure

A malicious requester can send:

- large opaque ciphertexts;
- many target locators;
- unreachable targets;
- repeated config requests;
- many concurrent IDs;
- invalid outer padding.

The proxy MUST perform cheap checks before target connection work and MUST use:

- per-peer rates;
- target-attempt limits;
- connection reuse;
- target-locator validation and, when used, HNSR route validation;
- global concurrency limits;
- response-size limits;
- mapping bounds;
- timeouts.

### Target exposure

A malicious proxy or requester can send ciphertexts that consume HPKE
decryption before being rejected.

The target MUST:

- check outer sizes first;
- check configuration ID first;
- check replay filter before decryption;
- apply per-proxy limits;
- limit failed decryptions;
- limit recursive jobs;
- separate HPKE and recursion work pools;
- stop advertising target readiness when saturated;
- avoid detailed decryption oracles.

### No UDP amplification

All messages use established bidirectional Handshake connections. This HIP
defines no UDP listener and no unauthenticated datagram response.

## No arbitrary forwarding

A proxy MUST act only on a valid `TargetLocatorV1`.

For a direct locator, it may connect only to a target that:

- uses the encoded public address and port;
- completes the selected ordinary Handshake transport;
- advertises both required service bits;
- returns target capability;
- authenticates the matching target peer identity key.

For an HNSR locator, it may open only an HNSR `HNS_NODE_V1` circuit that:

- is obtained from the route key derived from the encoded target peer key;
- is backed by a valid, unexpired HNSR route record and relay ticket;
- authenticates the inner endpoint as that exact target peer key;
- completes ordinary inner Handshake negotiation;
- advertises both required ODoH service bits and target capability.

The proxy MUST NOT forward to:

- arbitrary DNS servers;
- HTTP URLs;
- requester-selected paths;
- local services;
- RFC1918 or local-network targets on public networks;
- node RPC;
- wallet RPC;
- SSH;
- SOCKS;
- generic TCP ports.

HNSR does not weaken this rule. The only HNSR destination is the in-process
`HNS_NODE_V1` endpoint bound to a validated route record; no requester-selected
local host or port exists. This prevents the extension from becoming an open
proxy or SSRF primitive.

## Configuration authenticity

A malicious proxy can return a forged or substituted target configuration.

The requester prevents substitution by verifying:

- target locator type and canonical body;
- network magic;
- target peer key;
- target signature;
- timestamps;
- sequence;
- ODoH configuration encoding.

A proxy cannot replace the ODoH public key without the target peer private key.

A compromised target peer key can authorize malicious ODoH configurations for
that target. It does not forge HNS DNSSEC data accepted by a correctly
validating requester.

## Proxy substitution attack

A malicious proxy may forward the ciphertext to a different target.

The substitute cannot decrypt because the query is encrypted to the selected
ODoH key. The proxy can cause failure but cannot recover plaintext through
substitution alone.

The proxy may route to a colluding target that possesses the selected key if
the selected target intentionally shares its private key. Such sharing violates
target operational requirements and defeats non-collusion.

Target private keys SHOULD be unique to one target operator and MUST NOT be
shared with proxy operators.

## Target behavior and logs

A target necessarily sees:

- plaintext DNS query;
- proxy connection;
- timing;
- encrypted response size;
- authority traffic.

A target SHOULD minimize logs and MUST NOT synthesize a client identifier from:

- target request ID;
- ODoH key ID;
- query padding;
- DNS ID;
- resolver cache state.

The target MUST NOT add EDNS Client Subnet.

Shared ODoH configurations MUST be used across many requesters. Per-client
target keys would make queries linkable and are forbidden by this profile.

## Proxy behavior and logs

A proxy sees:

- requester connection;
- target locator;
- for HNSR, route lookup results, relay choice, and ticket metadata;
- ODoH ciphertext size;
- timing;
- response size.

It MUST NOT see plaintext DNS under correct cryptography.

A proxy SHOULD avoid persistent logging of request mappings. If operational
logging is enabled, it SHOULD aggregate counts rather than store per-query
client-target tuples.

The mapping is sensitive metadata even though the DNS name is encrypted.

## Collusion

If proxy and target share their observations, they can associate:

```text
requester connection
    +
plaintext DNS query
```

The protocol cannot cryptographically prevent collusion. When HNSR is used,
collusion among the ODoH proxy, HNSR relay, and target may provide additional
timing and topology evidence. HNSR does not create an additional anonymity
claim.

Mitigations include:

- requester-selected independent peers;
- distinct peer keys;
- netgroup and ASN diversity;
- avoiding dual-role use in one exchange;
- rotating among peers;
- target connection pooling;
- public operator declarations;
- future relay transparency or reputation mechanisms.

No implementation may claim complete query anonymity.

## Timing correlation

A proxy and target—or an observer watching both links—can correlate:

- request times;
- response times;
- packet sizes;
- bursts;
- rare large DNSSEC responses;
- cancellation timing.

Padding, multiplexing, and connection reuse reduce but do not eliminate this
risk.

A proxy SHOULD not forward identifying requester metadata and MAY apply bounded
jitter. A target SHOULD serve many proxies and a proxy SHOULD serve many
requesters to improve the anonymity set.

## Sybil and concentration risks

An attacker can advertise many proxy or target peers. A small number of
well-provisioned operators may also attract most traffic.

Requesters SHOULD use ordinary Handshake anti-eclipse practices:

- address-manager diversity;
- netgroup limits;
- multiple seeds and discovery paths;
- failure history;
- peer rotation;
- avoiding a single permanent proxy or target;
- separate proxy and target candidate sets.

Service bits prove capability claims, not operator independence or honesty.

## Censorship and availability

A proxy can refuse a target. A target can refuse a query. Either can delay or
drop messages.

The requester SHOULD keep multiple proxy and target candidates and MAY retry
with a different pair.

Retries SHOULD avoid sending the exact same HPKE ciphertext. The requester
SHOULD create a fresh ODoH query encryption for each new target attempt.

A retry to the same target through another proxy MAY use a fresh encryption
under the same current configuration.

## Privacy downgrade

A client policy may be:

```text
OBLIVIOUS_REQUIRED
OBLIVIOUS_PREFERRED
DIRECT_ALLOWED
```

Rules:

- `OBLIVIOUS_REQUIRED`: failure of oblivious resolution is a hard failure.
- `OBLIVIOUS_PREFERRED`: the client may ask for explicit application or user
  policy before direct fallback.
- `DIRECT_ALLOWED`: the client may use the base HIP directly.

An implementation MUST NOT silently downgrade when policy requires oblivious
privacy.

A direct base-HIP query reveals requester and DNS question to one relay.

## Security considerations

### Cryptographic scope

HPKE protects the DNS plaintext between requester and target. It does not
authenticate DNS data as true. HNS state, DNSSEC, and DANE provide content
authentication.

### Compromised target ODoH key

A compromised ODoH private key allows decryption of queries encrypted under
that key, subject to the HPKE compromise model. Regular rotation limits
exposure.

Compromise does not let the attacker forge DNSSEC data accepted by the
requester.

### Compromised proxy

A compromised proxy learns requester-target relationships, can censor and
correlate timing, and can replay or drop ciphertext. It cannot decrypt or forge
valid encrypted target responses without target key material.

### Malicious target

A malicious target sees DNS plaintext, can return false data, omit data,
replay still-valid signed DNS data, or censor. Local requester validation
prevents acceptance of unauthenticated DNSSEC or DANE data but cannot force
availability or freshness beyond validation policy.

### Malformed DNS after decryption

The target MUST validate the complete base query profile only after successful
decryption. Failures are generic externally and bounded internally.

### Request-ID attacks

Hop-local random target IDs prevent simple direct correlation. IDs are not
cryptographic secrets. Implementations MUST reject duplicate live IDs and
clean up on disconnect.

### Outer Handshake encryption

Outer Brontide protects each peer link from ordinary passive observers between
those peers, but terminates at the proxy and target. ODoH is still required for
end-to-end query confidentiality across the proxy.

### DNS ID

The requester SHOULD use DNS IDs according to the base HIP. The DNS ID is
inside ODoH encryption and invisible to the proxy.

### Resolver cache inference

A target's response latency can reveal cache state to the proxy. Padding does
not hide timing. A proxy should not be treated as unable to make statistical
inferences about target cache behavior.

### Authority correlation

Authoritative DNS servers see queries originating from the target. A powerful
observer that also sees requester-proxy traffic may attempt correlation.

This HIP does not hide target-to-authority recursion.

## Negative effects and costs

### Additional latency

The exchange adds:

- target-configuration retrieval when not cached;
- requester-to-proxy processing;
- proxy-to-target forwarding;
- HPKE encryption and decryption;
- a second P2P scheduling queue.

Typical cached-config resolution adds one network hop in each direction.

An HNSR target can additionally require route lookup, relay connection, circuit
opening, and inner Brontide negotiation when no reusable circuit exists.

### Two-peer availability

Successful direct resolution requires both a proxy and target. Either can fail.
An HNSR path additionally depends on current route records and an active relay
reservation.

Clients need a larger healthy peer set and retry logic.

### Cryptographic CPU cost

Targets perform HPKE decryption and response encryption in addition to DNS
recursion. Requesters perform HPKE setup and decryption. This may be noticeable
on phones and low-power nodes under high query volume.

### Proxy bandwidth

Proxy nodes carry encrypted requests and responses without receiving DNS-cache
benefit. Operators incur bandwidth and connection-management cost.

### Target concentration

Only nodes with stable recursive backends and adequate CPU may enable the
target role. Traffic may concentrate around a small number of targets, reducing
the practical non-collusion set.

### Increased protocol surface

The extension adds:

- configuration signatures;
- key rotation;
- HPKE;
- proxy mappings;
- target routing;
- cancellation;
- replay filters;
- padding;
- privacy policy.

This creates more parser, state-machine, and interoperability risk than direct
P2P DNS relay.

### Padding overhead

Padding consumes bandwidth, particularly for small mobile queries and large
DNSSEC responses. Lower padding saves resources but weakens size privacy.

### Traffic-analysis limits

Users may misunderstand oblivious resolution as complete anonymity. Timing,
size, collusion, Sybil selection, and global observation remain.

### Blockchain-P2P contention

DNS privacy traffic shares node connections and CPU with blockchain traffic.
Strict lower-priority scheduling is required. Under overload, ODoH must fail
before block propagation or chain synchronization is degraded.

### Config staleness

Rotated or expired keys can cause temporary failures when cached configurations
lag. Overlap and retries reduce but do not eliminate this.

### Direct-peer overlap

A full node requester may already be connected to many candidate targets for
blockchain purposes, weakening network-identity separation and reducing the
available target set.

## Compatibility

This HIP is consensus compatible and requires no hard fork.

Legacy peers:

- do not advertise `NODE_DNS_OBLIVIOUS`;
- never receive `ODNS` from conforming implementations;
- continue ordinary P2P operation.

Base-HIP-only peers:

- may perform direct P2P DNS relay;
- cannot act as ODoH proxy or target;
- remain valid fallback candidates only under client privacy policy.

Upgraded role combinations:

| Roles | Permitted |
| --- | --- |
| Proxy only | Yes |
| Target only | Yes, with base DNS relay |
| Config cache only | Yes |
| Proxy + target | Yes, but not both in one exchange |
| Requester only | Yes |
| All roles | Yes |

A target MUST implement the base HIP's recursive target behavior.

HNSR is optional. A proxy that does not implement HNSR rejects
`HNSR_NODE_V1` locators as `UNSUPPORTED` and remains conformant for direct
locators. A target advertised only through HNSR is reachable only through a
proxy implementing both HIPs. Legacy HNSR-unaware ODoH peers continue to use
direct locators.

## Suggested configuration

Illustrative flags:

```text
Proxy:

  --p2p-odoh-proxy
  --p2p-odoh-proxy-max-live=1024
  --p2p-odoh-proxy-targets=32
  --p2p-odoh-proxy-bandwidth=5mbit
  --p2p-odoh-hnsr-targets

Target:

  --p2p-dns-relay
  --p2p-odoh-target
  --p2p-odoh-key-rotation=24h
  --p2p-odoh-key-overlap=2h
  --p2p-odoh-max-jobs=256
  --hnsr-endpoint
  --hnsr-publish-node

Requester:

  --p2p-odoh
  --p2p-odoh-policy=required
  --p2p-odoh-avoid-direct-targets
```

No proxy URL, target URL, DoH endpoint, or resolver account is configured.

## Implementation architecture

```text
Requester
  |
  +-- authenticated HNS state
  +-- DNS query builder
  +-- ODoH client
  +-- proxy/target selector
  +-- local DNSSEC/DANE validator
  |
  +== Handshake peer connection ==> Proxy
                                      |
                                      +-- target locator validator
                                      +-- request-ID mapper
                                      +-- config cache
                                      +-- target connection pool
                                      +-- optional HNSR locator adapter
                                      |
                                      +== target session (direct or HNSR) ==> Target
                                                                          |
                                                                          +-- signed config manager
                                                                          +-- HPKE key ring
                                                                          +-- replay filter
                                                                          +-- base-HIP admission
                                                                          +-- HNS recursive backend
```

The proxy component MUST not expose generic socket forwarding.

The target SHOULD reuse the base HIP resolver implementation rather than
maintain a second divergent DNS-validation path.

## Complete reference implementation and regtest evidence

The JavaScript `hsd` reference implementation is maintained on
[`denuoweb/hsd:feat/p2p-oblivious-dns-relay`](https://github.com/denuoweb/hsd/tree/feat/p2p-oblivious-dns-relay)
at commit `909311d97c794eb59ed2eb0b095a122607ae078e`. It is based directly
on the prerequisite base relay implementation at commit
`ea31be1554f3235bfa96bdd394e6d33e7dda8080` and is proposed as the stacked
draft [handshake-org/hsd#959](https://github.com/handshake-org/hsd/pull/959).

The independent Rust requester and composed browser acceptance tier are
maintained on
[`Denuo-Web/hns-dane-browser:feat/p2p-oblivious-dns-relay`](https://github.com/Denuo-Web/hns-dane-browser/tree/feat/p2p-oblivious-dns-relay)
at commit `477c4e8092a062a69f56bf348b382b6f4face898`. Together these
implementations complete the mandatory
direct-locator reference profile.

The implementation is kept with this draft at immutable commit links:

- [`lib/net/odoh.js`](https://github.com/denuoweb/hsd/blob/909311d97c794eb59ed2eb0b095a122607ae078e/lib/net/odoh.js), containing strict body codecs,
  signed target configurations, RFC 9230 processing, proxy mappings, target
  replay handling, and requester helpers;
- [`lib/net/packets.js`](https://github.com/denuoweb/hsd/blob/909311d97c794eb59ed2eb0b095a122607ae078e/lib/net/packets.js), containing the version-1
  `ODNS` envelope;
- [`lib/net/pool.js`](https://github.com/denuoweb/hsd/blob/909311d97c794eb59ed2eb0b095a122607ae078e/lib/net/pool.js) and
  [`lib/node/fullnode.js`](https://github.com/denuoweb/hsd/blob/909311d97c794eb59ed2eb0b095a122607ae078e/lib/node/fullnode.js), containing dynamic
  service negotiation, packet dispatch, role configuration, and lifecycle
  integration;
- [`test/odoh-test.js`](https://github.com/denuoweb/hsd/blob/909311d97c794eb59ed2eb0b095a122607ae078e/test/odoh-test.js), containing codec,
  cryptographic, signature, rotation/overlap, privacy-split, mapping, replay,
  and deterministic-vector tests;
- [`test/data/odoh-v1-vectors.json`](https://github.com/denuoweb/hsd/blob/909311d97c794eb59ed2eb0b095a122607ae078e/test/data/odoh-v1-vectors.json),
  containing shared locator, RFC 9230 encoding/KDF, and signed-target-record
  vectors verified by JavaScript and Rust;
- [`scripts/run-odoh-regtest-trial.js`](https://github.com/denuoweb/hsd/blob/909311d97c794eb59ed2eb0b095a122607ae078e/scripts/run-odoh-regtest-trial.js),
  containing the reproducible three-FullNode trial;
- [`docs/experimental-odoh.md`](https://github.com/denuoweb/hsd/blob/909311d97c794eb59ed2eb0b095a122607ae078e/docs/experimental-odoh.md), containing
  the implementation runbook and explicit claim boundary.

The browser-side reference is in
[`rust/crates/hns-p2p/src/odoh.rs`](https://github.com/Denuo-Web/hns-dane-browser/blob/477c4e8092a062a69f56bf348b382b6f4face898/rust/crates/hns-p2p/src/odoh.rs),
with resolver integration in
[`rust/crates/hns-browser-runtime`](https://github.com/Denuo-Web/hns-dane-browser/tree/477c4e8092a062a69f56bf348b382b6f4face898/rust/crates/hns-browser-runtime),
platform security path mappings for Android and iOS, and the reproducible full
topology in
[`scripts/test-experimental-p2p-dns-relay-full.sh`](https://github.com/Denuo-Web/hns-dane-browser/blob/477c4e8092a062a69f56bf348b382b6f4face898/scripts/test-experimental-p2p-dns-relay-full.sh).

The cryptographic implementation uses the maintained `@hpke/core` and
`@hpke/dhkem-x25519` packages for RFC 9180 HPKE. The P2P-specific code performs
RFC 9230 message encoding, key-ID derivation, response-secret derivation, and
target-record signing around that library. It does not implement an ad hoc
public-key encryption construction.

### Implemented direct-locator profile

The current branch implements:

- all version-1 opcodes and strict envelope/body length checks;
- proxy and target capability negotiation;
- requester-selected direct Brontide target locators;
- low-S strict-DER secp256k1-signed target records;
- the mandatory RFC 9230 cipher suite;
- independent cryptographically random hop-local request IDs;
- bounded per-connection and global requester/proxy state;
- target deadlines, cancellation, disconnect cleanup, and replay detection;
- automatic 22-hour target HPKE key rotation, a two-hour old-key/record
  overlap, monotonically increasing sequences, and retired-key wiping;
- generic target failure mapping without an HPKE or DNS parsing oracle;
- target handoff to the existing base-HIP `DNSRelayService` after decryption;
- requester-side decrypted-response structure, recursion flag, DNS ID, and
  question correlation checks;
- an explicit regtest-only private/loopback target exception;
- selection only of an already-connected outbound Brontide peer whose numeric
  address, port, authenticated peer key, service bits, and `TARGET` capability
  match the locator;
- an independent Rust requester using X25519/HKDF-SHA256/AES-128-GCM and the
  same strict target-record and DNS-correlation checks; and
- browser-runtime provenance and security paths that remain beneath local
  Urkel, DNSSEC, HTTPS/SVCB, TLSA, and DANE validation.

The reference proxy deliberately forwards only to a preconnected,
authenticated target that exactly matches the requester-selected locator. It
never becomes a generic socket forwarder. HNSR locators, on-demand target
connection establishment, config caching, persistent target keys, multi-target
scheduling, outer bucket padding, and production telemetry are optional
extensions outside the mandatory direct-locator profile; their absence does
not reduce direct-locator conformance.

### Trial command and topology

From the `hsd` feature branch:

```sh
npm ci
node scripts/run-odoh-regtest-trial.js ../artifacts/regtest-trial.json
```

The successful 2026-07-20 trial created three actual `hsd` FullNodes with
independent persistent test identities and disposable prefixes:

```text
Requester == authenticated Brontide ==> Proxy
                                      == authenticated Brontide ==> Target
```

The requester observed the proxy role, retrieved the target's signed
configuration through the proxy, verified its signature, network, lifetime,
record ID, and exact locator, encrypted a `www.relaytest. A` query, and sent a
`CLIENT_QUERY`. The proxy replaced the client request ID with an independent
target request ID and sent a `TARGET_QUERY`. The target decrypted the query,
passed it through the prerequisite base relay's complete wire-query parser,
active-name admission hook, resource scheduler, and backend interface,
encrypted the returned DNS response, and returned it through the proxy.

The trial asserted and recorded:

| Observation | Result |
| --- | --- |
| Target advertised both private ODoH and base-relay service bits | Pass |
| Target configuration signature and locator verified | Pass |
| Target sequence increased across forced rotation | `1` to `2` |
| Current configuration advertised old/new overlap | `2` configurations |
| Previous signed record accepted during overlap | Pass |
| Raw query bytes absent from proxy-observed ODoH ciphertext | Pass |
| Proxy plaintext byte counter | `0` |
| Target received the exact admitted raw DNS query | Pass |
| Client and target request IDs were different | Pass |
| Base relay accepted/succeeded exactly once | Pass |
| Requester decrypted the expected correlated response | Pass |
| Proxy mappings after completion | `0` client, `0` target |

The runner writes machine-readable evidence to `artifacts/regtest-trial.json`
when invoked from the shared workspace. Target identity, HPKE key, request IDs,
configuration ID, response nonce, and ciphertext are fresh per run. Stable
interoperability inputs are instead published in the linked
`test/data/odoh-v1-vectors.json` and verified by both reference
implementations.

### Composed browser trial

The focused three-node run isolates framing, cryptography, privacy views, and
rotation. The companion browser repository supplies the composed acceptance
tier and runs an independent native Rust requester against four actual patched
`hsd` FullNodes with independent prefixes and identities.

The successful 2026-07-20 run:

1. mined the complete `relaytest` name lifecycle to regtest height 91;
2. required all four nodes to converge on the same chain and Urkel tree root;
3. verified a current `TYPE_EXISTS` proof and the registered NS/GLUE4/DS
   resource;
4. used `hsd-relay-bad` as the requester-facing ODoH proxy and the distinct
   Brontide listener on `hsd-owner-good` as the authenticated target;
5. prevented the browser namespace from reaching authoritative or external
   UDP/TCP port 53;
6. recursively resolved the live signed child zone through ODoH;
7. validated DS/DNSKEY/RRSIG, TLSA, and DANE locally in the browser runtime;
8. fetched `https://www.relaytest:18443/` with status 200; and
9. observed zero contacts to the configured legacy DoH sentinel.

The runner writes its machine-readable result to
`artifacts/browser-odoh-full-tier/full-tier-result.json` in the shared
workspace. It records `urkelProof: verified`, `resolutionSource: p2p_odoh`,
`odoh.verified: true`, `dnssec: secure`, `dane: verified`, status 200, distinct
proxy/target sockets, and `legacyDohContact: false`. The adjacent proof,
network-isolation, authority, origin, and sentinel artifacts are checked by the
runner before it reports success.

The complete reference claim is the combination of these two trials. It covers
the mandatory direct-locator profile and inherited base-HIP validation path.
Optional HNSR reachability, public canaries, anonymity-set measurement, and
Android/iOS application-binary packaging are separate deployment exercises,
not missing protocol behavior.

### Verification record

The reference implementations pass:

```sh
npx --yes eslint@9 \
  lib/net/odoh.js lib/net/common.js lib/net/packets.js lib/net/parser.js \
  lib/net/pool.js lib/node/fullnode.js lib/net/index.js \
  test/odoh-test.js scripts/run-odoh-regtest-trial.js

npm run test-file -- \
  test/odoh-test.js test/dns-relay-test.js test/net-test.js
```

The focused suite covers canonical envelope and opcode bodies, malformed
envelopes, target-locator restrictions, all-zero plaintext padding, signed
record validation, wrong HPKE keys, encrypted query/response round trips,
decrypted DNS response correlation, three-role forwarding, independent IDs,
exact ciphertext replay rejection, and the prerequisite base-relay and network
packet regression suites. The focused hsd command reports 102 passing tests.
The Rust peer, browser, Android-bridge, and iOS-ABI suites report 206 passing
tests; Android reports 192 passing Kotlin unit tests; strict Rust Clippy is
clean; and the composed four-node runner reports success.

## Deployment plan

### Phase 1: cryptographic unit implementation

- RFC 9230 ODoH encoding;
- RFC 9180 HPKE suite;
- target config signing;
- key rotation;
- test vectors;
- malformed input handling.

**Reference status (2026-07-20): complete.** The mandatory suite, target
signatures, canonical encodings, wrong-key behavior, zero padding, malformed
outer inputs, automatic rotation/overlap, retired-key wiping, and published
JavaScript/Rust vectors are implemented and tested.

### Phase 2: isolated regtest

At least:

- requester node;
- proxy node;
- target node;
- HNS authority;
- test HTTPS/DANE origin where applicable.

Verify:

- query plaintext absent from proxy;
- requester address absent from target application state;
- complete base-HIP recursion;
- local DNSSEC and DANE validation;
- no legacy resolver contact.

**Reference status (2026-07-20): complete.** The focused three-FullNode run
verifies the encrypted two-hop transport, privacy-view split, target
authentication, rotation, and inherited base-relay admission/scheduling. The
composed four-FullNode browser run verifies the registered name, live
recursion, current Urkel proof, local DNSSEC and DANE, HTTPS, network isolation,
and zero legacy-resolver contact.

### Phase 3: optional multi-proxy and HNSR exercises

- several proxies;
- several targets;
- independent target request IDs;
- connection pooling;
- retries;
- pair rotation;
- key overlap;
- replay tests;
- target failure;
- proxy failure;
- outbound-only target through HNSR `HNS_NODE_V1`;
- HNSR route expiry and relay failover without HPKE configuration rotation.

These exercises measure optional deployment policies and the optional HNSR
locator. They are not prerequisites for direct-locator implementation or
conformance.

### Phase 4: bounded testnet canary

- private experimental service and packet values;
- conservative rate limits;
- aggregate performance metrics;
- block propagation measurements;
- privacy correlation experiments;
- no public-network conformance claim.

This phase is an operator deployment gate, not unfinished reference code.

### Phase 5: post-review mainnet deployment

- permanent service bit and packet type, requested only after prior gates pass;
- published interoperability vectors;
- at least two independent implementations;
- audited HPKE library integration;
- explicit operator configuration.

This phase begins only after the prior gates pass and the Handshake community
accepts a later revision and assigns permanent values. Draft governance status
is intentionally not represented as an implementation gap.

## Testing requirements

### Encoding

- every opcode round trip;
- minimum and maximum lengths;
- truncated field rejection;
- trailing-byte rejection;
- nonzero request ID;
- reserved flags;
- invalid target locator;
- unknown locator type;
- malformed locator body length;
- invalid host type;
- HNSR profile other than `HNS_NODE_V1`;
- invalid key;
- invalid DER signature;
- high-S signature;
- wrong network magic;
- expired config;
- future config;
- sequence behavior.

### RFC 9230

- config vector parsing;
- key ID derivation;
- query encryption;
- target decryption;
- response derivation;
- response decryption;
- mandatory HPKE suite;
- invalid ODoH padding;
- wrong message type;
- wrong key ID;
- invalid encapsulated key;
- AEAD failure.

### Role negotiation

- proxy only;
- target only;
- dual role;
- target missing `NODE_DNS_RELAY`;
- role disappearing after connection;
- stale capability observation.

### Configuration retrieval

- proxy cache hit;
- proxy cache miss;
- target retrieval;
- forged config;
- substituted target key;
- expired config;
- key overlap;
- rotation during live request;
- target unavailable;
- HNSR route lookup failure;
- expired HNSR ticket;
- HNSR endpoint-key mismatch;
- successful HNSR re-resolution for the same target peer key.

### Query privacy

Instrumentation MUST demonstrate:

Proxy-visible data:
- requester connection;
- target locator;
- HNSR route and relay metadata when that locator is used;
- ciphertext;
- size and timing.

Proxy-hidden data:
- QNAME;
- QTYPE;
- plaintext response;
- DNSSEC records.

Target-visible data:
- proxy connection;
- QNAME;
- QTYPE;
- resolver result.

Target-hidden protocol data:
- requester connection;
- client request ID;
- requester peer key;
- requester user agent.

### Base-HIP inheritance

- every admitted base-HIP query type;
- every rejected base-HIP query type;
- HNS root admission;
- DNSSEC success;
- DNSSEC bogus;
- denial proof;
- TLSA and DANE;
- CNAME and DNAME behavior;
- out-of-bailiwick nameserver behavior;
- recursion deadline;
- response size.

### Request mapping

- random independent IDs;
- simultaneous requests;
- out-of-order responses;
- duplicate client ID;
- duplicate target ID;
- requester disconnect;
- target disconnect;
- proxy restart;
- cancellation;
- late response;
- mapping cleanup.

### Replay

- exact ciphertext replay;
- replay-filter expiry;
- bounded filter;
- false-positive behavior;
- fresh encryption retry.

### Resource tests

- ciphertext flood;
- config-request flood;
- unreachable-target flood;
- HPKE-failure flood;
- recursive-job saturation;
- response-size limit;
- padding limit;
- mapping exhaustion;
- target connection exhaustion.

### Scheduling

Under ODoH saturation, measure:

- block propagation;
- header synchronization;
- compact-block recovery;
- peer ping latency;
- proof serving;
- direct base-HIP DNS latency.

Core blockchain traffic MUST retain its configured service guarantees.

### Collusion and correlation experiments

A reference implementation SHOULD measure:

- size correlation with and without padding;
- timing correlation with and without pooling;
- effect of bounded jitter;
- effect of target connection reuse;
- anonymity-set size by proxy and target;
- dual-AS versus same-AS pairs.

These experiments do not create normative guarantees but inform safer defaults.

## Reference flow

```text
Requester                       Proxy                         Target
    |                             |                              |
    |-- version / verack -------->|                              |
    |-- GETCAPS ----------------->|                              |
    |<-- CAPS(PROXY) -------------|                              |
    |                             |                              |
    |-- GETCONFIG(target) ------->|                              |
    |                             |-- version / verack --------->|
    |                             |-- GETCAPS ------------------>|
    |                             |<-- CAPS(TARGET) -------------|
    |                             |-- GETCONFIG ---------------->|
    |                             |<-- CONFIG(signed) ------------|
    |<-- CONFIG(signed) ----------|                              |
    |                             |                              |
    | verify target signature     |                              |
    | construct base-HIP DNS Q    |                              |
    | ODoH encrypt to target key  |                              |
    |                             |                              |
    |-- CLIENT_QUERY [CID=41] --->|                              |
    |                             | allocate TID=0x9af3...        |
    |                             |-- TARGET_QUERY [TID] -------->|
    |                             |                              | decrypt
    |                             |                              | validate
    |                             |                              | recurse
    |                             |                              | encrypt
    |                             |<-- TARGET_RESPONSE [TID] -----|
    |<-- CLIENT_RESPONSE [CID=41]-|                              |
    |                             |                              |
    | decrypt response            |                              |
    | validate HNS/DNSSEC/DANE    |                              |
```


### Optional outbound-only HNSR target

```text
Requester             ODoH Proxy          HNSR peers/relay          Target
    |                      |                       |                    |
    |-- GETCONFIG -------->|                       |                    |
    |                      |-- GETROUTE ---------->|                    |
    |                      |<-- ROUTES ------------|                    |
    |                      |-- OPEN -------------->|                    |
    |                      |<======== inner Brontide + HNS peer =======>|
    |                      |-- GETCONFIG ------------------------------>|
    |                      |<-- CONFIG --------------------------------|
    |<-- CONFIG -----------|                       |                    |
    |-- CLIENT_QUERY ----->|                       |                    |
    |                      |-- TARGET_QUERY --------------------------->|
    |                      |<-- TARGET_RESPONSE ------------------------|
    |<-- CLIENT_RESPONSE --|                       |                    |
```

The HNSR route and ticket may change while the signed ODoH configuration
remains valid because the configuration binds the stable target peer key and
locator type, not short-lived relay material.

## Rationale

### Why extend the base HIP

The base HIP already defines the difficult DNS-specific boundary:

- what may be queried;
- how the HNS root is established;
- how recursion is bounded;
- what DNSSEC data is returned;
- what the requester validates.

Duplicating those rules would create two diverging recursive protocols.

### Why use RFC 9230

RFC 9230 provides a reviewed message format and HPKE construction designed for
exactly the proxy/target privacy split required here. Inventing a new encrypted
DNS envelope would add cryptographic and interoperability risk.

### Why no HTTP

Handshake peers already have:

- persistent connections;
- framing;
- service negotiation;
- target identity;
- request multiplexing;
- error handling.

Adding HTTP between Handshake peers would increase dependencies without adding
the web-origin semantics that DoH normally uses.

### Why HNSR is an optional locator

ODoH target identity and HPKE keys should remain stable when a phone changes
networks or relay reservations. HNSR already defines short-lived authenticated
reachability for an outbound-only Handshake peer. Binding only the HNSR locator
type and target peer key into the ODoH target configuration preserves that
separation: ODoH authenticates the target and DNS privacy state, while HNSR
finds a current path to the same authenticated peer.

Making HNSR mandatory would unnecessarily couple direct targets to a larger
rendezvous and circuit protocol. Therefore, direct and HNSR locators coexist,
and implementations may support only direct locators.

### Why a signed target config

The requester needs an authentic target HPKE public key. The target's existing
Handshake peer identity can sign short-lived ODoH configurations without an
on-chain update.

### Why the requester chooses the target

If the proxy selected the target, it could always select a colluding resolver.
Requester selection does not eliminate collusion, but it permits diversity and
prevents undetectable target substitution because encryption binds the query to
the selected key.

### Why one proxy hop

One non-colluding proxy separates requester identity and query plaintext.
Additional hops add latency, resource use, and route complexity. A future HIP
may define multi-hop forwarding.

### Why local validation remains mandatory

ODoH protects who queried what. It does not prove that the returned DNS data is
correct. Handshake state, DNSSEC, and DANE remain the authenticity system.

## Alternatives considered

### Direct P2P DNS relay

Simpler and faster, but one peer sees both requester and query.

### Public ODoH over HTTPS

Provides the privacy split but requires separately configured HTTP proxy and
target infrastructure outside the Handshake peer network.

### Tor

Can conceal requester address from the direct DNS relay, but introduces a
separate overlay and does not define the base-HIP query and validation
relationship.

### VPN

Moves trust to the VPN and does not prevent the resolver from seeing DNS
plaintext.

### Proxy-selected target

Simpler proxy API but permits the proxy to direct every query to a colluding
target. Rejected.

### Target key in HNS name resource

Provides on-chain binding but is poorly suited to daily key rotation, consumes
name-resource space, and creates update latency. Signed peer configurations are
short-lived and replaceable.

### Reuse target peer key as HPKE key

Rejected. Peer identity and HPKE encryption require separate key lifecycles and
algorithms.

### Encrypt the existing `GETDNSRELAY` body directly

Insufficient. It lacks a two-hop proxy/target routing and response-key
construction, and would invite an ad hoc cryptographic design.

### Global passive anonymity claim

Rejected. The protocol does not defeat global timing and size observation.

## Future extensions

Possible later HIPs:

- two-proxy or multi-hop oblivious forwarding;
- blinded target discovery;
- privacy-preserving target reputation;
- verifiable operator-separation attestations;
- batch and mix-delay profiles;
- cover traffic;
- post-quantum HPKE configurations;
- proxy payment or reciprocal service accounting;
- compact target-locator references;
- a dedicated restricted HNSR ODoH target profile if complete `HNS_NODE_V1`
  sessions prove unnecessarily broad or difficult to budget;
- encrypted client-target error details;
- oblivious proof retrieval in addition to DNS;
- private information retrieval for target configurations.

Future extensions MUST preserve requester-side validation and MUST not silently
weaken the proxy/target separation.

## References

1. RFC 2119, *Key words for use in RFCs to Indicate Requirement Levels*.
2. RFC 8174, *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words*.
3. RFC 8484, *DNS Queries over HTTPS (DoH)*.
4. RFC 8467, *Padding Policies for Extension Mechanisms for DNS (EDNS(0))*.
5. RFC 9180, *Hybrid Public Key Encryption*.
6. RFC 9230, *Oblivious DNS over HTTPS*.
7. Draft HIP, *Handshake P2P DNS Relay*.
8. Handshake `hsd`, peer framing, service negotiation, and network addresses.
9. Handshake `hnsd`, SPV name-proof retrieval.
10. Draft HIP, *Handshake P2P Rendezvous and Authenticated Service Relay*.
11. `@hpke/core` and `@hpke/dhkem-x25519`, RFC 9180 HPKE implementation used
    by the regtest proof of concept.
12. Denuo Web `hsd`, `feat/p2p-dns-relay` at
    `ea31be1554f3235bfa96bdd394e6d33e7dda8080`, prerequisite responder
    implementation.
