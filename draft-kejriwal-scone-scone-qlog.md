---
title: "QUIC Logging for the SCONE Protocol"
abbrev: "scone-qlog"
category: std

docname: draft-kejriwal-scone-scone-qlog-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Standard Communication with Network Elements"
keyword:
venue:
  group: "Standard Communication with Network Elements"
  type: "Working Group"
  mail: "scone@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/scone"
  github: "RaghavKej/scone-qlog"
  latest: "https://RaghavKej.github.io/scone-qlog/draft-kejriwal-scone-scone-qlog.html"

author:
 -
    fullname: "Raghav Kejriwal"
    organization: University of Aberdeen
    extaddr: School of Engineering
    street: Fraser Noble Building
    city: Aberdeen
    code: AB24 3UE
    country: UK
    email: "ietf@raghavkejriwal.com"
 -
    fullname: "Godred Fairhurst"
    organization: University of Aberdeen
    extaddr: School of Engineering
    street: Fraser Noble Building
    city: Aberdeen
    code: AB24 3UE
    country: UK
    email: "gorry@erg.abdn.ac.uk"
    uri: https://www.erg.abdn.ac.uk
 -
    fullname: "Matthias Hofstätter"
    ascii: "Matthias Hofstaetter"
    organization: FAU Erlangen-Nürnberg
    extaddr: Computer Science 7
    street: Martensstraße 3
    city: Erlangen
    code: 91058
    country: Germany
    email: "matthias.hofstaetter@fau.de"
    uri: https://www.cs7.tf.fau.de

normative:
  SCONE: I-D.ietf-scone-protocol
  QLOG-SCHEMA: I-D.ietf-quic-qlog-main-schema
  QLOG-QUIC: I-D.ietf-quic-qlog-quic-events
  CDDL: RFC8610

informative:

...

--- abstract

This document defines qlog events for the Standard Communication with Network Elements (SCONE) protocol. The events record the sending and reception of SCONE packets and changes to throughput advice.

--- middle

# Introduction {#intro}

{{SCONE}} describes a protocol where on-path network elements can communicate their perspective on the maximum sustainable throughput for QUIC flows to endpoints. This throughput advice suggests an upper bound on long-term average throughput, independent of and complementary to real-time congestion control signals.

This document defines the qlog namespace and events that record the sending and reception of SCONE packets and changes to throughput advice, allowing for debugging of implementations and reconstruction of the packets received.

The event namespace with the identifier scone is defined in {{namespace}}.

{{tab-scone-event-overview}} summarises the qlog events described in this document.


| Event Name | Importance | Definition |
|:-----------|:-----------|:-----------|
| scone:packet_sent | Core | {{sent}} |
| scone:packet_received | Core | {{received}} |
| scone:advice_updated | Base | {{updated}} |
| scone:advice_expired | Base | {{expired}} |
{: #tab-scone-event-overview title="Overview of SCONE qlog events."}

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The event and data structure definitions in this document are expressed in the Concise Data Definition Language {{CDDL}} and its extensions described in {{Section 1.2 of QLOG-SCHEMA}}.

This document follows the language used in {{SCONE}}:

- "Throughput advice" refers to the information communicated to endpoints by network elements in the form of a 7-bit rate signal
- A "monitoring period" is the time over which rate signals apply (67 seconds {{SCONE}}).

# SCONE Rate Signal Data Type

A Rate Signal n in the range 0 to 126 maps to a ceiling bitrate of 100,000 * 10^(n/20) bits per second {{SCONE}}. The value 127 carries no throughput advice.

~~~cddl
SCONERateSignal = 0..127
SCONEKnownRateSignal = 0..126
~~~
{: #fig-rate-signal-type title="CDDL Description of SCONE Rate Signal types."}

# SCONE Event Schema Definition {#namespace}

This document describes how the SCONE protocol can be expressed in qlog using the event schema. Per the requirements in {{Section 8 of QLOG-SCHEMA}}, this document registers the `scone` namespace. The event schema URI is `urn:ietf:params:qlog:events:scone`.

## Draft Event Schema Identification
RFC-Editor NOTE:: Please remove this section when this is published as an RFC.

Only implementations of the final, published RFC can use the events belonging to the event schema with the URI `urn:ietf:params:qlog:events:scone`. Until such an RFC exists, implementations MUST NOT identify themselves using this URI.

Implementations of draft versions of the event schema MUST append the string "-" and the corresponding draft number to the URI. For example, draft 00 of this document is identified using the URI `urn:ietf:params:qlog:events:scone-00`.

The namespace identifier itself is not affected by this requirement.

# SCONE Events {#scone-events}

SCONE events extend the `$ProtocolEventData` extension point defined in {{Section 8.2 of QLOG-SCHEMA}}. Additionally, they allow for direct extensibility by their use of per-event extension points via the `$$` CDDL "group socket" syntax, as also described in {{Section 8.2 of QLOG-SCHEMA}}.

~~~cddl
SCONEEventData = SCONEPacketSent /
                 SCONEPacketReceived /
                 SCONEAdviceUpdated /
                 SCONEAdviceExpired
$ProtocolEventData /= SCONEEventData
~~~
{: #scone-events-def title="SCONEEventData definition and ProtocolEventData extension"}

## scone:packet_sent {#sent}

Importance: Core

The packet_sent event is emitted when an endpoint includes a SCONE packet in a datagram it sends. The event records the rate signal in the sent SCONE packet.

~~~cddl
SCONEPacketSent = {
  ? datagram_id: uint32
  ; derivable from rate_signal: 0x6f7dc0fd when even, 0xef7dc0fd when odd
  ? version: QuicVersion
  rate_signal: SCONERateSignal
  ; expected to be 127 when a packet is sent by an endpoint
  ? trigger: "loss_detected"
  ; indicates that the SCONE packet is being sent again
  ; because a coalesced QUIC packet was detected to be lost

  * $$scone-packetsent-extension
}
~~~
{: #fig-sent title="CDDL Description of the SCONEPacketSent Data."}

Note: The `version` field is optional, because it is fully determined by `rate_signal`: the two SCONE versions differ only in their most significant bit, which is the least significant bit of the Rate Signal ({{Section 5.1 of SCONE}}). It uses the `QuicVersion` type defined in {{Section 8.1 of QLOG-QUIC}}.

## scone:packet_received {#received}

Importance: Core

The packet_received event is emitted when an endpoint receives a datagram that begins with a SCONE packet, once the endpoint has processed the coalesced QUIC packets in the datagram and therefore knows whether the SCONE packet is accepted or discarded ({{Section 5.3 of SCONE}}). The event records the rate signal carried in the received SCONE packet and the outcome of processing it, regardless of whether the packet is accepted or discarded.

~~~cddl
SCONEPacketReceived = {
  ? datagram_id: uint32
  ; derivable from rate_signal: 0x6f7dc0fd when even, 0xef7dc0fd when odd
  ? version: QuicVersion
  rate_signal: SCONERateSignal
  ; bits per second encoded by the rate_signal
  ; absent when rate_signal is 127
  ? throughput_advice: uint64
  ; accepted implies the packet was valid and contained a
  ; SCONEKnownRateSignal regardless of whether the advice was applied
  outcome: "accepted" / "discarded"

  ? reason: "destination_connection_id_mismatch" /
            "source_connection_id_mismatch" /
            ; no coalesced QUIC packet in the
            ; datagram was successfully processed
            "coalesced_packet_not_processed" /
            ; multiple scone packets were received in one datagram
            "multiple_scone_packets_received" /
            ; rate signal was 127
            "unknown_rate_signal" /
            ; this endpoint did not send the
            ; scone_supported transport parameter
            "not_negotiated" /
            ; SCONE packet was not the first packet in the datagram
            "not_first_packet" /
            ; discarded for a reason not listed here
            "unspecified"

  * $$scone-packetreceived-extension
}
~~~
{: #fig-received title="CDDL Description of the SCONEPacketReceived Data."}

The `reason` field MUST be present when `outcome` is "discarded" and MUST NOT be present when `outcome` is "accepted".

## scone:advice_updated {#updated}

Importance: Base

The advice_updated event is emitted by a SCONE-receiving endpoint when the effective throughput advice changes as a result of processing a received SCONE packet or applying the monitoring-period rules defined in {{SCONE}}. Following the convention of other qlog `*_updated` events, the event records both the previous and the new effective advice, and an optional `trigger` field indicates the cause of the change ({{Section 8.2.1 of QLOG-SCHEMA}}).

~~~cddl
SCONEAdviceUpdated = {
  ; previous effective throughput advice in bits per second;
  ; absent when no advice was in effect
  ? old: uint64
  ; new effective throughput advice in bits per second
  new: uint64
  ; rate signal corresponding to the new effective advice
  rate_signal: SCONEKnownRateSignal
  ? trigger:
    ; a received SCONE packet changed the effective advice
    "packet_received" /
    ; advice from the previous monitoring period expired and
    ; advice from the current period became effective
    "period_expired"

  * $$scone-adviceupdated-extension
}
~~~
{: #fig-updated title="CDDL Description of the SCONEAdviceUpdated Data."}

This event could be emitted in the following common cases (non-exhaustive):

- When throughput advice first becomes effective, for example, after processing a SCONE packet carrying a known rate signal. In this case `old` is absent and `trigger` is "packet_received".
- Processing a SCONE packet that lowers the effective throughput advice. In this case `trigger` is "packet_received".
- Advice from a previous monitoring period expires and advice from the current monitoring period becomes effective. In this case `trigger` is "period_expired".

This event SHOULD NOT be emitted when advice from a previous monitoring period expires, but the advice in the current monitoring period does not change the effective advice; that is, `old` and `new` SHOULD differ.

Note: This event aligns with an API transition. It might or might not align with the boundary of the 67 second monitoring period. For example, implementations that do not use timers for SCONE monitoring periods could choose to emit this event when the application is next informed of the SCONE advice.

## scone:advice_expired {#expired}

Importance: Base

The advice_expired event is emitted by a SCONE-receiving endpoint when previously effective SCONE advice expires because no applicable throughput advice was received during the monitoring period, as defined in {{SCONE}}.

This event is emitted only when the end of a monitoring period leaves no effective throughput advice ({{Section 5.4 of SCONE}}). If other advice becomes effective at that point, for example advice received during the monitoring period that just ended, the advice_updated event ({{updated}}) is emitted instead; the two events are never both emitted for the same transition.

~~~cddl
SCONEAdviceExpired = {
  previous_rate_signal: SCONEKnownRateSignal
  ; rate signal corresponding to the expired advice
  previous_throughput_advice: uint64
  ; The effective previous advice in bits per second

  * $$scone-adviceexpired-extension
}
~~~
{: #fig-expired title="CDDL Description of the SCONEAdviceExpired Data."}

Note: For the same reasons as in {{updated}}, this event might or might not align with the boundary of the 67 second monitoring period.

# Security Considerations {#security}

The security and privacy considerations discussed in {{Section 14 of QLOG-SCHEMA}} apply to this document.

The events defined in this document record the rate signals and throughput advice that a network applies to the traffic of the logging endpoint. A qlog file containing these events therefore reveals the policy that an access network applied to that endpoint, and can help identify the network or infer the endpoint's subscription, as discussed in {{Section 10 of SCONE}}. When qlog files are shared beyond the logging endpoint, the data minimization and anonymization guidance in {{Section 14.3 of QLOG-SCHEMA}} is RECOMMENDED for these fields.

# IANA Considerations {#iana}

This document registers a new entry in the "qlog event schema URIs" registry (created in {{Section 15 of QLOG-SCHEMA}}).

Event schema URI:
: urn:ietf:params:qlog:events:scone

Namespace
: scone

Event Types
: packet_sent, packet_received, advice_updated, advice_expired

Description:
: Event definitions related to the SCONE protocol.

Reference:
: This Document

--- back

# Acknowledgments
{:numbered="false"}

The authors thank the European Space Agency (ESA) for supporting the QUICOPTSAT project, under which an evaluation of SCONE over satellite networks was undertaken.
