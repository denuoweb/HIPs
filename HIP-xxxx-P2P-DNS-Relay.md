# HIP-xxxx : Handshake P2P DNS Relay

```
Number:  HIP-xxxx
Title:   Handshake P2P DNS Relay
Type:    Standards Track
Status:  Draft
Authors: Jaron Rosenau <@denuoweb>
Created: 2026-07-16
```

## Abstract

This document specifies an optional Handshake peer-to-peer service for carrying
restricted recursive DNS queries and raw DNS responses over an established
Handshake peer connection. It is intended for state-authenticating clients that
can reach Handshake peers but cannot reliably reach authoritative DNS servers
on UDP or TCP port 53.

The relay is an untrusted transport. Before using the service, a requester
authenticates current Handshake name state, either from its own validated full
node state or from a verified header chain and Urkel name proof. It validates
returned DNSSEC data locally. Applications using TLSA and DANE also perform
those checks locally. A relay's service advertisement, recursive validation
result, and DNS Authenticated Data (`AD`) bit are never trust anchors.

This extension is opt-in, introduces no consensus changes, and creates no new
public listener. It reuses Handshake's existing peer framing and version
service negotiation with one service-bit assignment and two packet-type
assignments.

## Motivation

Handshake names commonly delegate resolution from proof-backed root data to
authoritative DNS servers. Mobile networks, VPNs, captive networks, and
middleboxes may block, redirect, or interfere with direct DNS on port 53 even
when ordinary TCP connections to Handshake peers and HTTPS origins work.

A client can use a third-party recursive DNS-over-HTTPS service as a fallback,
but that concentrates availability and query metadata in a separately operated
service. A capable Handshake full node already has the chain state and resolver
machinery needed to perform bounded HNS recursion. Reusing the existing peer
connection makes multiple independently operated relay choices possible without
turning any relay into a trusted resolver.

This proposal separates availability from authenticity:

- the relay provides recursive DNS reachability;
- locally authenticated Handshake name state establishes the root delegation;
- local DNSSEC validation authenticates delegated DNS data and denial proofs;
- local TLSA and DANE validation authenticates TLS when an application uses
  DANE.

A lying relay can omit, delay, censor, replay, or corrupt data. Local validation
prevents unsigned or incorrectly signed data from being accepted as secure,
subject to DNSSEC signature-validity and freshness limits. It does not prevent
replay of an older, still-valid signed RRset. Requesters can retry a different
relay when transport or availability fails.

## Conventions and terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in RFC 2119
and RFC 8174 when, and only when, they appear in all capitals.

The following terms are used:

- **Requester**: a Handshake peer that sends a relayed DNS query.
- **Relay**: a Handshake peer that advertises and serves this extension.
- **HNS root**: the rightmost DNS label of the requested name, interpreted as a
  Handshake name.
- **Authenticated HNS state**: a sufficiently current name resource obtained
  either from locally validated full-node state or from a verified Urkel proof
  anchored in a locally validated Handshake header chain.
- **Peer**, in resource-limit requirements: one established Handshake peer
  connection, unless an implementation can apply a stronger stable identity
  safely.
- **Transport status**: the one-byte relay result outside the embedded DNS
  message. It is distinct from a DNS RCODE.
- **Embedded DNS message**: a complete DNS query or response in DNS network
  byte order carried inside a relay packet.

## Scope

This proposal defines:

- relay capability negotiation;
- request and response packet payloads;
- a restricted HNS recursive-query profile;
- transport statuses and correlation rules;
- minimum requester-side validation requirements;
- resource, privacy, and compatibility requirements.

This proposal does not define:

- a new Handshake consensus rule or name-resource encoding;
- how clients obtain or synchronize authenticated HNS state;
- a general-purpose recursive resolver service;
- a new DNS, HTTP, or management listener;
- an oblivious or two-hop relay protocol;
- a new encrypted peer transport;
- application UI, exact peer scores, or a fixed resolution fallback order;
- test-network exceptions for private authoritative addresses.

## Protocol overview

A successful exchange proceeds as follows:

1. The requester authenticates current HNS state for the requested root, then
   derives an acceptable delegation from that state.
2. The requester completes the ordinary Handshake `version` / `verack`
   exchange with a peer that currently advertises `NODE_DNS_RELAY`.
3. The requester sends `GETDNSRELAY` with a nonzero request identifier and one
   complete DNS query.
4. The relay applies cheap syntax and rate checks, confirms that the rightmost
   label is an active HNS name in its current chain state, and performs bounded
   HNS recursion.
5. The relay sends `DNSRELAY` on the same connection with either a raw DNS
   response or an empty transport error.
6. The requester checks transport correlation and DNS structure, then passes
   the raw response through its authenticated HNS state, DNSSEC, and
   application-security validation.

Multiple requests MAY be outstanding on one connection. Request identifiers
correlate responses, and responses MAY arrive in a different order from
requests. Relay processing MUST NOT block unrelated Handshake peer traffic on
the connection.

## Assignments

The following permanent assignments are requested from Handshake protocol
maintainers:

| Symbol | Registry | Value |
| --- | --- | ---: |
| `NODE_DNS_RELAY` | `version.services` bit | TBD |
| `GETDNSRELAY` | P2P packet type | TBD |
| `DNSRELAY` | P2P packet type | TBD |

Values are deliberately unassigned while this HIP is a draft. Prototype
implementations use private experimental values on controlled networks. Those
values are not protocol assignments and MUST NOT be advertised as this
standard on a public network unless maintainers explicitly assign them.
The placeholders are appropriate for initial HIP submission, but no
implementation can claim public-network conformance until all three permanent
assignments have been made and implemented.

## Capability negotiation

`NODE_DNS_RELAY` is directional: a peer setting the bit states that it can
receive `GETDNSRELAY` requests on that connection. It says nothing about the
other peer's capabilities.

A relay:

- MUST require explicit operator enablement;
- MUST complete ordinary Handshake negotiation before accepting a request;
- MUST have a ready HNS recursive backend and sufficiently synchronized local
  chain state when advertising the service;
- MUST NOT advertise merely because it can decode the packet;
- SHOULD stop advertising on new connections while its backend is unavailable
  or its global work bound is exhausted.

Readiness can change after the version exchange. An already connected relay
that becomes unavailable returns `RESOLVER_UNAVAILABLE` or `BUSY` as
appropriate.

A requester MUST send `GETDNSRELAY` only after a completed `version` / `verack`
exchange on a live connection whose received `version` message contains
`NODE_DNS_RELAY`. A cached observation from an older connection MUST NOT
override the current handshake.

The capability is not an authentication or quality signal. Requesters MUST
treat all capable peers as untrusted until each response passes local
validation.

## Message framing

`GETDNSRELAY` and `DNSRELAY` use the existing nine-byte Handshake peer frame:
a four-byte little-endian network magic value, a one-byte packet type, and a
four-byte little-endian unsigned payload length. The outer payload length MUST
equal the number of relay-envelope bytes that follow. This HIP changes neither
the peer header nor the maximum general Handshake message size.

All relay-envelope integers are unsigned and little-endian. Embedded DNS
messages retain the network byte order defined by DNS. A request identifier is
an opaque eight-byte value; numeric notation in this document is only a
convenience.

### `GETDNSRELAY`

```text
GetDnsRelay {
    request_id:  u64
    query_length: u16
    query:        u8[query_length]
}
```

| Offset | Size | Field | Requirement |
| ---: | ---: | --- | --- |
| 0 | 8 | `request_id` | Nonzero and unique among live requests on this connection |
| 8 | 2 | `query_length` | 1 through 4,096 |
| 10 | variable | `query` | Complete DNS query of exactly `query_length` bytes |

The maximum `GETDNSRELAY` payload is 4,106 bytes.

### `DNSRELAY`

```text
DnsRelay {
    request_id:     u64
    status:         u8
    response_length: u16
    response:       u8[response_length]
}
```

| Offset | Size | Field | Requirement |
| ---: | ---: | --- | --- |
| 0 | 8 | `request_id` | Exact request identifier being answered |
| 8 | 1 | `status` | A defined transport status |
| 9 | 2 | `response_length` | 1 through 65,535 for `OK`; zero otherwise |
| 11 | variable | `response` | Complete DNS response for `OK`; absent otherwise |

The maximum `DNSRELAY` payload is 65,546 bytes.

Both packet types use exact-length encoding. A parser MUST reject:

- a zero request identifier;
- a declared length beyond the applicable limit;
- a truncated body;
- any byte after the declared body;
- `OK` with an empty body;
- an error status with a nonempty body;
- an unknown status.

Length and structural limits MUST be checked before allocating or copying the
declared body. A receiver MAY close the connection for a non-canonical packet,
except that an unknown status alone SHOULD fail only the exchange as described
below.

A requester MUST NOT reuse an identifier while the earlier request remains
live on the same connection. Identifiers SHOULD be generated from a source that
makes accidental reuse unlikely. A relay that receives a duplicate live
identifier returns `INVALID_QUERY` and MUST NOT start duplicate recursive work.

## Transport statuses

Transport statuses describe the relay exchange, not the DNS result:

| Value | Name | Meaning |
| ---: | --- | --- |
| 0 | `OK` | A nonempty raw DNS response is present. |
| 1 | `REFUSED` | Relay admission policy refused the HNS question. |
| 2 | `UNSUPPORTED` | The operation or recursive backend is not supported. |
| 3 | `BUSY` | A rate or concurrency bound was reached. |
| 4 | `INVALID_QUERY` | The query or live request identifier is invalid. |
| 5 | `RESOLVER_UNAVAILABLE` | The recursive backend or required chain state is unavailable. |
| 6 | `TIMEOUT` | The relay's bounded recursive deadline expired. |
| 7 | `INTERNAL_ERROR` | A bounded internal failure prevented a DNS response. |

Values 8 through 255 are reserved for future standards. An unrecognized value
makes the exchange unusable: a requester MUST fail that exchange and MUST NOT
pass its body to DNS processing. It SHOULD NOT penalize a peer solely for an
unknown status value, because the value may have been assigned by a later
standard. A future standard assigning a status in this range MUST also define
capability or version negotiation that prevents it from being sent to an older
requester.

DNS `NOERROR`, `NXDOMAIN`, `SERVFAIL`, `REFUSED`, truncation, and other RCODEs
or DNS conditions remain inside a syntactically valid `OK` response. A
requester MUST NOT translate a DNS RCODE into a transport status or vice versa.

## DNS query profile

An admitted embedded query MUST satisfy all of the following:

- it is a complete, syntactically valid DNS message with no trailing bytes;
- `QR` is clear, the opcode is standard `QUERY`, and the header RCODE is zero;
- `AA`, `TC`, `RA`, and the reserved header `Z` bit are clear;
- `RD` is set;
- there is exactly one `IN`-class question;
- the answer and authority sections are empty;
- the additional section contains exactly one root-owner EDNS OPT record;
- EDNS version is zero, the extended RCODE is zero, and `DO` is set;
- the EDNS advertised payload size is between 512 and 4,096 bytes inclusive;
- EDNS reserved flags are clear and EDNS Client Subnet is absent;
- TSIG and SIG(0) are absent;
- the rightmost label is a syntactically valid Handshake name.

The requester SHOULD set Checking Disabled (`CD`) so that a validating relay
returns material for local validation even when it considers the answer bogus.
A relay MUST accept either `CD` state. When `CD` is set, a relay MUST NOT
suppress otherwise available DNSSEC records or synthesize a failure solely
because its own DNSSEC validation failed. Resolution or transport failures may
still produce a DNS or transport failure. A response is not required to echo
`CD`.

The following browser- and DNSSEC-relevant question types are the mandatory
baseline for this service. A relay MUST admit each type when the rest of the
query is valid. A relay whose backend persistently cannot execute one of these
types MUST NOT advertise `NODE_DNS_RELAY`; a transient inability after
advertisement may return `UNSUPPORTED`.

```text
A AAAA CNAME DNAME NS SOA DS DNSKEY RRSIG NSEC NSEC3 NSEC3PARAM
TLSA SVCB HTTPS TXT MX SRV CAA
```

Relays MUST reject all other types with `INVALID_QUERY` or `REFUSED`. In
particular, they MUST reject `ANY`, `AXFR`, `IXFR`, `TKEY`, and `TSIG`, and MUST
reject non-query opcodes such as `UPDATE` and `NOTIFY`.

An empty EDNS option list and a well-formed EDNS Padding option MUST be
accepted. A relay MAY strip Padding before authority traffic. EDNS Client
Subnet MUST NOT be added by a requester or relay. A relay MAY reject other EDNS
options with `INVALID_QUERY`, and it MUST NOT forward a requester-supplied EDNS
option to an authority unless that option is explicitly safe and required for
the recursive operation.

The request carries no destination IP address, transport, or port. The relay,
not the requester, selects authoritative endpoints during HNS recursion.
The requester's EDNS UDP payload size is part of the admitted query profile; it
is not an instruction to copy that OPT record or size into authority-facing
queries.

## Relay admission and recursion

Before starting recursive authority traffic, a relay MUST derive the HNS root
from the single question and consult its own current, validated Handshake name
state. It MUST refuse a root that is invalid, reserved for the ICANN root,
expired, unregistered, blacklisted by Handshake root policy, or lacks a
non-empty name resource. This lookup is an abuse-control boundary; it does not
prove that the resource contains a usable delegation and is not evidence the
requester can trust. A resource that cannot produce a usable delegation may
subsequently result in `REFUSED` or an ordinary DNS failure response.

The recursive backend MUST be rooted in Handshake state and restricted to the
admitted original question. It MUST NOT expose a direct interface for arbitrary
non-HNS recursive questions.

The HNS-root restriction applies to the original question. Ordinary recursive
processing of that question MAY follow CNAME or DNAME aliases and MAY resolve
out-of-bailiwick nameserver addresses across another DNS namespace, including
the ICANN namespace. This indirect traversal is necessary for some valid HNS
delegations, but a cooperating HNS zone can use it to induce lookups of other
names. Relays MUST apply the egress, depth, rate, and concurrency bounds in this
document to every such lookup and MUST NOT accept a requester-specified
secondary destination outside the DNS message.

Except for a fixed operator-configured local HNS root interface, external
authority egress MUST be limited to UDP and TCP port 53 at globally routable
addresses derived through the admitted recursion. A production relay MUST
reject loopback, link-local, private-use, shared-address, documentation,
benchmarking, multicast, unspecified, metadata-service, and otherwise
non-public endpoints immediately before every referral or nameserver-address
network operation. IPv4 and IPv6 support are implementation choices, but the
same public-address rule applies to each supported family. The local-root
exception MUST NOT be selected or influenced by the requester.

This egress rule governs DNS network operations performed by the relay. It does
not authorize a requester or application to connect to an address found in a
returned A or AAAA RRset; applications need their own destination policy.

Private-address exceptions MAY exist on isolated test networks, but they are
not part of this protocol and MUST NOT be enabled as a public relay mode.

A relay SHOULD retry a truncated UDP response over TCP. If it cannot complete
the retry, it MAY return the original truncated DNS response as `OK`; DNS
truncation remains a DNS-layer condition.

The backend MAY validate DNSSEC and MAY cache DNS data according to DNS TTLs,
but it MUST return the requested DNSSEC material without treating its own
validation result as authoritative for the requester.

The relay MUST select an independently bounded authority-facing EDNS UDP
payload size appropriate to its own network policy. That size MUST be between
512 and 4,096 bytes inclusive. The relay MUST NOT blindly forward the
requester's OPT record or options, and the authority-facing size need not equal
the size in the embedded request.

## DNS response requirements

For an `OK` result, the relay MUST return a complete, syntactically valid DNS
response that:

- uses the embedded query's DNS transaction identifier;
- sets `QR` and uses the standard `QUERY` opcode;
- echoes `RD` and sets `RA`;
- contains exactly one question matching the query name, type, and class;
- is no larger than 65,535 bytes.

Question-name comparison MUST be case-insensitive after decompression. The type
and class values MUST match exactly.

If its backend changes the DNS transaction identifier internally, the relay
MUST restore the requester's identifier before sending the response. It MUST
return an error status rather than an empty, malformed, or oversized DNS body.

The relay MAY preserve or set `AD`, but `AD` has no trust meaning in this
protocol.

Neither the embedded query's EDNS UDP payload size nor the relay's independently
selected authority-facing size caps the P2P-carried response. Subject to the
65,535-byte relay limit, a relay MAY return the complete response it obtained
over TCP.

## Requester validation

### Transport and DNS structure

A requester MUST accept `DNSRELAY` only when all of the following hold:

- it arrived on the same live connection as the request;
- its request identifier matches one currently outstanding request;
- its envelope status and body-length combination is canonical;
- an `OK` body parses as one complete DNS message;
- the DNS transaction identifier matches;
- `QR`, opcode, echoed `RD`, and `RA` have the required values;
- the single question matches the original name, type, and class.

Unknown-status, unsolicited, duplicate, late, mismatched, and malformed
responses MUST NOT be passed to the DNS validator. The requester SHOULD discard
or penalize a peer for unsolicited, duplicate, late, mismatched, or malformed
responses and SHOULD close the affected relay connection when safe correlation
has been lost. An unknown status alone is handled as specified in **Transport
statuses** and is not sufficient reason to penalize the peer.

Other ordinary Handshake packets may arrive while a request is live. A
requester MUST NOT assume that the next frame is its DNS response. At minimum,
it MUST answer peer liveness packets required by the base protocol and tolerate
a bounded number of advisory, address, acknowledgement, and unknown packets
without extending the relay deadline. It MAY fail the relay exchange on a
packet that requires a separate synchronization or proof state machine, but it
MUST NOT interpret that packet as a relay response. This extension does not
waive obligations created by any other service bit the requester advertises.

### Local authentication

Before sending a relay request, a requester MUST have authenticated a
sufficiently current HNS name resource for the root. It may do so from its own
validated full-node state or from a verified Urkel proof anchored in a locally
validated Handshake header chain. "Sufficiently current" is determined by the
requester's existing chain-staleness policy; the relay does not weaken that
policy.

Every relayed DNS response is untrusted input. A requester MUST NOT mark an
answer secure because of:

- the relay service bit;
- the peer's identity or score;
- an encrypted peer connection;
- a successful transport status;
- the DNS `AD` bit;
- a relay's claimed validation result.

The authenticated HNS resource authenticates the NS, DS, and glue data that it
contains. To mark RRsets below a secure delegation as secure, the requester
MUST build and validate the DNSSEC authentication chain from the authenticated
DS. Such
positive RRsets require their applicable DNSKEY and RRSIG validation; this does
not imply that delegation NS or glue RRsets are themselves signed. Negative
answers require locally validated NSEC or NSEC3 denial. When HTTPS/SVCB or TLSA
records affect connection policy, those RRsets and their aliases MUST be
covered by the same local validation chain. DANE applications MUST validate
the TLSA result against the presented certificate under the applicable DANE
RFCs. Data below a delegation without a validated DS MUST NOT be labeled secure,
though an application MAY use it under a separately defined insecure-data
policy.

The relay cannot compensate for stale authenticated HNS state, a proof/root
mismatch when a proof is used, an invalid delegation, unverified nameserver
addresses, invalid DNSSEC, an invalid denial proof, invalid TLSA data, or a
failed DANE match.

## Failure, retry, and peer selection

Requesters MUST use bounded timeouts. Relays MUST apply one absolute bound to
admission plus recursive work. Three seconds per relay exchange is the
RECOMMENDED default, but implementations MAY tune this for their network and
resource constraints.

`INVALID_QUERY` is not retryable without changing the query. `REFUSED`,
`UNSUPPORTED`, `BUSY`, `RESOLVER_UNAVAILABLE`, `TIMEOUT`, `INTERNAL_ERROR`, a
transport failure, or a malformed response MAY be retried through another live
capable peer. Requesters SHOULD keep the number of attempts small and MUST NOT
retry indefinitely.

Peer selection SHOULD consider existing peer quality and address-group
diversity. When an Urkel proof was used, it MAY prefer a relay other than the
peer that supplied that proof. One capable peer is sufficient for
interoperability; implementations MUST NOT require a centralized discovery
service.

Successful exchanges MAY improve a peer's local score. Timeouts and transport
failures SHOULD cause temporary backoff. Malformed or mismatched responses
SHOULD receive a stronger penalty. Exact scores and cooldown intervals are
local policy, not wire protocol.

## Resource limits and backpressure

A relay MUST bound, at minimum:

- request rate per peer;
- aggregate admitted-request rate across peers;
- concurrent physical work per peer;
- concurrent physical work globally;
- outbound request rate per authoritative address or an equivalent
  destination-abuse budget;
- query and response bytes;
- recursive duration;
- referral and alias-chase depth;
- cache memory and diagnostic state.

The following defaults have been validated by the prototype implementation and
are RECOMMENDED as a conservative starting profile:

| Limit | Recommended default |
| --- | ---: |
| Per-peer request rate | 20 attempts/second |
| Per-peer burst | 40 attempts |
| Per-peer physical in-flight work | 16 |
| Global physical in-flight work | 64 |
| Admission plus recursive deadline | 3 seconds |

Operators MAY lower these values and MAY raise them after measuring capacity,
but all bounds MUST remain finite. Aggregate-rate, destination-abuse, and
recursion-depth limits are implementation and deployment policy; an
implementation MUST provide finite defaults even though this HIP does not set
one universal numeric profile for them.

Concurrency backpressure SHOULD return `BUSY`. A rate-limited relay MAY drop a
request without a response. To avoid turning rate notices into excess traffic,
a relay SHOULD send at most one rate-limit `BUSY` response per peer per second
and silently drop further over-limit attempts in that interval.

Disconnecting a requester or expiring its logical deadline MUST suppress a
late response. If underlying admission or resolver work cannot be physically
cancelled, the relay MUST continue charging it to the originating peer and
global work bounds until it actually settles. This prevents disconnects from
bypassing concurrency limits.

Because requests and responses remain on an established TCP peer connection,
the protocol does not introduce a UDP source-spoofing reflection surface.

## Connection reuse, coalescing, and caching

Requesters SHOULD reuse a bounded number of capable peer connections. They MAY
coalesce simultaneous identical DNS wire questions after normalizing the DNS
transaction identifier, provided each caller receives its own identifier and
normal validation is applied.

No requester-side cross-request DNS cache is required by this HIP. A requester
that caches relayed data:

- MUST admit only data that passed local validation required by its policy;
- MUST honor DNS TTLs;
- MUST expire secure data no later than the earliest applicable DNS TTL, RRSIG
  expiration, or authenticated-HNS-state freshness limit;
- MUST bind the entry to the authenticated HNS name-tree state and exact
  delegation resource;
- MUST invalidate or revalidate it when that authenticated state or delegation
  resource changes;
- MUST cache negative results only after local NSEC or NSEC3 validation and in
  accordance with the negative-caching limits in RFC 2308;
- MUST NOT cache a transport status as DNS data.

## Privacy considerations

This is a one-hop relay. The relay necessarily learns the queried name, type,
timing, and response size. It is not Oblivious DoH and MUST NOT be described as
such.

Handshake's ordinary TCP peer listener is plaintext. When an implementation
uses an existing encrypted peer transport such as Brontide, an on-path observer
gets less query visibility, but the relay still sees the query. This HIP does
not define or require a new encrypted transport.

Implementations:

- MUST NOT add EDNS Client Subnet;
- SHOULD omit full qnames and raw DNS messages from normal logs and persisted
  diagnostics;
- SHOULD avoid stable requester identifiers in relay diagnostics;
- SHOULD restrict normal metrics to aggregate statuses, latency and size
  buckets, retry counts, and validation stages;
- SHOULD NOT add speculative prefetch, telemetry, or duplicate measurement
  queries solely for this feature;
- SHOULD make any query-bearing diagnostic trace bounded, ephemeral, and
  explicitly user- or operator-enabled and access-controlled.

Verbose DNS debugging can reveal names and responses. Operators enabling such
logging are responsible for the resulting privacy exposure.

## Security considerations

### Malicious relays

A relay can censor, delay, truncate, replay, reorder, or fabricate DNS bytes.
Strict request correlation and local cryptographic validation limit these
actions to availability, metadata, and the bounded replay/freshness effects
described below. Requesters MUST fail closed when validation does not complete.

### Stale or inconsistent chain state

A relay and requester may have different chain tips. Relay-side HNS admission
reduces abuse but does not establish correctness. Only the requester's locally
authenticated HNS state controls whether data is accepted or cached.

### Replay and freshness

DNSSEC authenticates data and signature validity; it does not guarantee that a
relay returned the newest RRset while an older signature remains valid. A
requester MUST enforce RRSIG inception and expiration, DNS TTLs, and its HNS
state-freshness policy. Security-sensitive applications SHOULD account for the
resulting replay window when rotating DNSKEY, HTTPS/SVCB, TLSA, or address data.

### DNS rebinding and server-side request forgery

The request contains no destination endpoint. Public-address and port-53
filtering MUST be repeated after every referral and nameserver-address result,
not only on the first delegation. A test-only private-network exception MUST
never become a production default.

### Resource exhaustion and Sybil peers

Per-peer rate limits alone can be bypassed with many connections or identities.
Relays need global request-rate, physical-work, memory, recursion-depth, and
per-destination abuse bounds. This is especially important because an HNS zone
can induce cross-namespace alias or nameserver-address lookups. Requesters also
need bounded connection pools, deadlines, response sizes, and retries.

### Traffic correlation

When an Urkel proof is used, using the same peer for that proof and DNS relay
lets the peer correlate the root proof with the full DNS question. Selecting
diverse peers can reduce that single-peer correlation, though it may reveal
portions of the operation to two peers. This is a requester policy tradeoff.

### Peer encryption

Peer encryption protects a transport hop; it does not authenticate DNS data or
make the relay trustworthy. Plain peer transport exposes queries to on-path
observers and active modification, which local validation MUST reject.

## Backward compatibility and deployment

This extension changes neither blockchain consensus nor existing name
resources. Legacy peers do not advertise `NODE_DNS_RELAY`, and conforming
requesters never send the new packet to them. Implementations that preserve
unknown Handshake packet types can continue the connection if a packet is
received unexpectedly; disconnecting an unexpected packet is also permissible
for a peer that never advertised the service.

The service SHOULD ship disabled on relay nodes until an operator opts in and
allocates resolver capacity. Requester applications MAY enable relay fallback
by default because capability negotiation prevents requests to legacy peers,
provided they retain a safe behavior when no capable peer is available.

During rollout, requesters SHOULD retain at least one independently controlled
resolution fallback appropriate to their security policy. Removing a legacy
fallback requires measured relay diversity, availability, latency, and
interoperability; this HIP does not mandate that product decision.

Prototype implementations using private service or packet values MUST migrate
to the permanent assignments before claiming conformance. A private prototype
and the final protocol MUST NOT share a service advertisement unless their wire
formats are identical and the permanent assignments have been made.

## Prototype implementations and interoperability evidence

An experimental requester, responder, deterministic fixture set, and
verification harness accompany this proposal:

- [Denuo Web HNS DANE Browser requester](https://github.com/Denuo-Web/hns-dane-browser/tree/021bc5edfea6d250aaa3fd1796b0a1875b3f3b7c)
- [Denuo Web `hsd` responder](https://github.com/denuoweb/hsd/tree/ea31be1554f3235bfa96bdd394e6d33e7dda8080)
- [Protocol and security design](https://github.com/Denuo-Web/hns-dane-browser/blob/021bc5edfea6d250aaa3fd1796b0a1875b3f3b7c/docs/experimental-hns-p2p-dns-relay.md)
- [Verification and canary instructions](https://github.com/Denuo-Web/hns-dane-browser/blob/021bc5edfea6d250aaa3fd1796b0a1875b3f3b7c/docs/experimental-p2p-dns-relay-runbook.md)
- [Full four-node acceptance runner](https://github.com/Denuo-Web/hns-dane-browser/blob/021bc5edfea6d250aaa3fd1796b0a1875b3f3b7c/scripts/test-experimental-p2p-dns-relay-full.sh)

The prototype uses a Rust requester and JavaScript `hsd` responder that consume
the same relay-envelope codec fixtures. Its full positive-path acceptance tier
creates four independent regtest `hsd` full nodes, completes an auction and
registration,
commits NS, glue, and DS data through a safe tree-root interval, verifies a
current existence proof from every node, and resolves a DNSSEC-signed child
zone through bad-to-good relay failover. The requester then validates A,
HTTPS, TLSA, and DANE locally and completes HTTPS without contacting the legacy
DoH sentinel.

A separate physical Android-device run exercised the packaged application.
These runs demonstrate implementation feasibility; they are not consensus
tests and do not make the Docker topology, fixture name, addresses, keys,
certificate, UI, or test-only private-authority option normative. The current
full tier covers a positive DNSSEC chain. It does not claim full-tier NSEC or
NSEC3 denial coverage, an `hnsd` implementation, a public canary, or encrypted
peer-path acceptance.

## Test vectors

The vectors in this section are relay packet payloads. They exclude the
Handshake network magic, packet type, and four-byte peer-frame payload length.
SHA-256 digests cover decoded binary bytes, not ASCII hex text.

These vectors first test the relay-envelope codec. The basic request and
response also satisfy the DNS profile in this HIP. The large boundary vectors
exercise length handling; unless explicitly stated, their inclusion does not
assert that every embedded DNS message is suitable for every application
policy. A conforming DNS parser still needs to support the DNS wire-name limits
defined by RFC 1035.

### Basic request

```text
08070605040302012a00123401100001000000000001037777770972656c617974657374000001000100002904d0000080000000
```

SHA-256:
`d38597681c22abdb6c7ab72454658f5c0a07ff74fb53d5ce72e81b922f23e520`

This 52-byte payload contains:

- request identifier `0x0102030405060708`, encoded little-endian;
- query length 42 (`2a00`);
- DNS identifier `0x1234` in network byte order;
- DNS flags `RD | CD`;
- one `www.relaytest.` `A IN` question;
- one root-owner OPT with UDP size 1232, EDNS version zero, `DO`, and no
  options.

### Successful response with untrusted `AD`

```text
0807060504030201003a00123481b00001000100000001037777770972656c6179746573740000010001c00c000100010000003c0004c000020100002904d0000080000000
```

SHA-256:
`63d59820b3a078f214ff6870d9322443c6131a84be4b53ebc1b35b7f36746048`

This 69-byte payload echoes the request identifier, has status `OK`, and
declares a 58-byte DNS response. The DNS response sets `QR | RD | RA | AD |
CD` and returns `www.relaytest. A 192.0.2.1` with TTL 60. The deliberately set
`AD` bit MUST NOT be trusted.

### `BUSY` response

```text
0807060504030201030000
```

SHA-256:
`37b13837928fca8d3a1427f7c2ef6bf7917fad052a88515a4f25c7e25ff2e27c`

This 11-byte payload echoes the request identifier, has status `BUSY` (3), and
has an empty response body.

### Invalid payloads

| Case | Bytes | Hex | SHA-256 |
| --- | ---: | --- | --- |
| Declared request length 43 with only 42 bytes | 52 | `08070605040302012b00123401100001000000000001037777770972656c617974657374000001000100002904d0000080000000` | `203d58767d89c396b85ef71398c186043006dcd0d42e6f381638eee30e69cbac` |
| One trailing request byte | 53 | `08070605040302012a00123401100001000000000001037777770972656c617974657374000001000100002904d0000080000000ff` | `cebc2f53063ff1e8ec35d3529fa038690793735086833ac895fa314e1baebec9` |
| Unknown response status 255 | 11 | `0807060504030201ff0000` | `6f763985eab74de92fe521f52ce882451a366d759b1e10f9a6e381d2c2d117fa` |
| Declared request length 4,097 | 10 | `08070605040302010110` | `0d8b7866de6682bcfa9e9ad6c215225965d959b241a20ab9b48e26808b61bcdc` |
| Zero response request identifier | 11 | `0000000000000000030000` | `5c66e18d8148d655710b2cee7313aa7110611b3ac663abb3771abd1baa9394f3` |

### Boundary-vector digests

Large deterministic vectors are not reproduced inline:

| Case | Bytes | SHA-256 |
| --- | ---: | --- |
| Maximum encoded request envelope | 4,106 | `2dc036004171fc77651a1732363e8260bba636d98d5a2678b744d4e579fa5bef` |
| Maximum encoded response envelope | 65,546 | `0a6bbf5e85b49a03f3914d12a4eb54e2c51bf18e6e9c757334025b4090f30d12` |
| 255-octet DNS wire-name request envelope | 292 | `b971391700b4b12aaf9521bccf78c0ca2738be81d8a97b03eedbd8ce1d0530b2` |
| Response with one byte after a maximum body | 65,547 | `deb7812c42fa0d8617e19d7a7d6d36524bc9c2c67da7f9d4bf61b7b6482cc3b5` |

The last vector retains a legal `ffff` response-length field and physically
contains one extra byte. It is rejected for trailing data; it does not encode
65,536 in a `u16`.

## References

- [HIP repository and submission process](https://github.com/handshake-org/HIPs)
- [HIP-0017: Stateless DANE clients](https://github.com/handshake-org/HIPs/blob/master/HIP-0017.md)
- [Handshake `hsd`](https://github.com/handshake-org/hsd)
- [RFC 1035: Domain Names - Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)
- [RFC 2119: Key words for use in RFCs](https://www.rfc-editor.org/rfc/rfc2119)
- [RFC 2308: Negative Caching of DNS Queries](https://www.rfc-editor.org/rfc/rfc2308)
- [RFC 4033: DNS Security Introduction and Requirements](https://www.rfc-editor.org/rfc/rfc4033)
- [RFC 4034: Resource Records for DNS Security Extensions](https://www.rfc-editor.org/rfc/rfc4034)
- [RFC 4035: Protocol Modifications for DNS Security Extensions](https://www.rfc-editor.org/rfc/rfc4035)
- [RFC 5155: DNS Security (DNSSEC) Hashed Authenticated Denial of Existence](https://www.rfc-editor.org/rfc/rfc5155)
- [RFC 6698: The DNS-Based Authentication of Named Entities (DANE) TLSA Protocol](https://www.rfc-editor.org/rfc/rfc6698)
- [RFC 6890: Special-Purpose IP Address Registries](https://www.rfc-editor.org/rfc/rfc6890)
- [RFC 6891: Extension Mechanisms for DNS (EDNS(0))](https://www.rfc-editor.org/rfc/rfc6891)
- [RFC 7671: The DNS-Based Authentication of Named Entities (DANE) Protocol: Updates and Operational Guidance](https://www.rfc-editor.org/rfc/rfc7671)
- [RFC 7830: The EDNS(0) Padding Option](https://www.rfc-editor.org/rfc/rfc7830)
- [RFC 7871: Client Subnet in DNS Queries](https://www.rfc-editor.org/rfc/rfc7871)
- [RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words](https://www.rfc-editor.org/rfc/rfc8174)
- [RFC 9230: Oblivious DNS over HTTPS](https://www.rfc-editor.org/rfc/rfc9230)
- [RFC 9460: Service Binding and Parameter Specification via the DNS](https://www.rfc-editor.org/rfc/rfc9460)
