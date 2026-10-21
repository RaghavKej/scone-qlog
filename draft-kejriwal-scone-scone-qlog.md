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
    email: "raghavkejriwal@gmail.com"

normative:
  SCONE: I-D.ietf-scone-protocol
  INVARIANTS: RFC8999
  QLOG-SCHEMA: I-D.ietf-quic-qlog-main-schema
  CDDL: RFC8610

informative:

...

--- abstract

This document specifies a qlog logging format for sending and receiving SCONE packets.


--- middle

# Introduction {#intro}

This document defines a way to provide logging for the SCONE protocol ({{SCONE}}) using the qlog format ({{QLOG-SCHEMA}}).

({{SCONE}}) describes a protocol where on-path network elements can communicate their perspective on the maximum sustainable thorughput for QUIC flows to endpoints. This thorughput advice suggests an upper bound on long-term average throughput, independent of and complementary to real-time congestion control signals.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
