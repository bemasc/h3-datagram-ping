---
title: "HTTP Datagram PING"
abbrev: "HTTP Datagram PING"
docname: draft-schwartz-masque-h3-datagram-ping-latest
category: std

ipr: trust200902
area: General
workgroup: masque
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: B. Schwartz
    name: Benjamin Schwartz
    organization: Google LLC
    email: bemasc@google.com


--- abstract

This draft defines an HTTP Datagram Format Type for measuring the functionality of a Datagram path.

--- middle

# Conventions and Definitions

{::boilerplate bcp14}

# PING Datagram Format Type

PING is an HTTP Datagram Format Type {{!I-D.draft-ietf-masque-h3-datagram}}.  It has no Additional Data.

## Format

PING Datagrams have the following format:

~~~
PING {
  Sequence Number (i),
  Opaque Data (..),
}
~~~
{: #ping-datagram-format title="PING Datagram Format"}

All `Sequence Number` and `Opaque Data` values are potentially valid.

## Use

The sender emits a PING Datagram with any even `Sequence Number` and any `Opaque Data`.  Upon receiving a PING Datagram with an even `Sequence Number`, the recipient MUST reply with a PING Datagram whose `Sequence Number` is one larger, with empty `Opaque Data`.

Intermediaries MUST forward PING Datagrams without modification, just like any other HTTP Datagram.

# Use cases

PING Datagrams can be used to characterize the end-to-end HTTP Datagram path associated with an HTTP request.  For example, HTTP endpoints can easily use PING Datagrams to estimate the round-trip time and loss rate of the HTTP Datagram path.

PING Datagrams are also suitable for use as DPLPMTUD Probe Packets {{?RFC8899}}.  This enables endpoints to estimate the HTTP Datagram MTU of each Datagram path, in order to avoid sending HTTP Datagrams that will be dropped.

Note that these path characteristics can differ from those inferred from the underlying transport (e.g. QUIC), if the HTTP request traverses one or more HTTP intermediaries (see {{Section 3.7 of ?I-D.draft-ietf-httpbis-semantics}}).

# IANA considerations

IANA is directed to add the following entry to the "HTTP Datagram Format Types" registry:

* Type: PING
* Value: TBD
* Reference: (This document)

--- back

# Acknowledgments
{:numbered="false"}

TODO
