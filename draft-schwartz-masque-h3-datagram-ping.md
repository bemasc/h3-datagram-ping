---
title: "HTTP Datagram PING and TIMESTAMP"
abbrev: "HTTP Datagram PING and TIMESTAMP"
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

This draft defines new mechanisms for measuring the functionality and performance of an HTTP Datagram path.  These mechanisms can be used with CONNECT-UDP, CONNECT-IP, or any other CONNECT protocol that employs Context IDs and Capsules.

--- middle

# Conventions and Definitions

{::boilerplate bcp14}

# PING

PING Datagrams can be used to characterize the end-to-end HTTP Datagram path associated with an HTTP request.  For example, HTTP endpoints can easily use PING Datagrams to estimate the round-trip time and loss rate of the HTTP Datagram path.

PING Datagrams are also suitable for use as DPLPMTUD Probe Packets {{?RFC8899}}.  This enables endpoints to estimate the HTTP Datagram MTU of each Datagram path, in order to avoid sending HTTP Datagrams that will be dropped.

Note that these path characteristics can differ from those inferred from the underlying transport (e.g. QUIC), if the HTTP request traverses one or more HTTP intermediaries (see {{Section 3.7 of ?I-D.draft-ietf-httpbis-semantics}}).

## Registration

Endpoints indicate support for PING Datagram type by including the header `"Datagram-Extension-Ping: ?1"` in the HTTP Request or Response.

A PING Datagram context is registered by a REGISTER_PING_CONTEXT Capsule with the following structure:

~~~
REGISTER_PING_CONTEXT Capsule {
  Context ID (i)
}
~~~

Registration is confirmed by an ACK_PING_CONTEXT Capsule:

~~~
ACK_PING_CONTEXT Capsule {
  Context ID (i)
  Error Code (i)
}
~~~

Error Code 0 means registration succeeded.  Error Code 1 means registration failed.  All other error code values also mean failure, but they are reserved for future use.

## Format

PING Datagrams have the following format:

~~~
PING Datagram {
  Context ID (i),
  Sequence Number (i),
  Opaque Data (..),
}
~~~

All `Sequence Number` and `Opaque Data` values are potentially valid.

## Use

The sender emits a PING Datagram with any even `Sequence Number` and any `Opaque Data`.  Upon receiving a PING Datagram with an even `Sequence Number`, the recipient MUST reply with a PING Datagram whose `Sequence Number` is one larger, with empty `Opaque Data`.

Intermediaries MUST forward PING Datagrams without modification, just like any other HTTP Datagram.

# TIMESTAMP

The TIMESTAMP Datagram extension allows marking any datagram with a timestamp indicating the time that it was sent.  Where PING allows measurement of the round-trip time between peers, TIMESTAMP allows peers to observe changes in the one-way latency.  Increasing one-way latency can indicate congestion on that path, informing peers' congestion control decisions.

## Registration

Endpoints indicate support for TIMESTAMP Datagram type by including the header `"Datagram-Extension-Timestamp: ?1"` in the HTTP Request or Response.

A TIMESTAMP Datagram context is registered by a REGISTER_TIMESTAMP_CONTEXT Capsule with the following structure:

~~~
REGISTER_TIMESTAMP_CONTEXT Capsule {
  Context ID (i)
  Inner Context ID (i)
}
~~~

`Inner Context ID` specifies how to interpret the payload after the timestamp.  It MUST be smaller than `Context ID`, and MUST already be registered (although that registration does not need to have been confirmed yet).

Registration is confirmed by an ACK_TIMESTAMP_CONTEXT Capsule:

~~~
ACK_TIMESTAMP_CONTEXT Capsule {
  Context ID (i)
  Error Code (i)
}
~~~

Error Code 0 means registration succeeded.  Error Code 1 means registration failed.  All other error code values also mean failure, but they are reserved for future use.

Registrations can be closed by a CLOSE_TIMESTAMP_CONTEXT Capsule:

~~~
CLOSE_TIMESTAMP_CONTEXT Capsule {
  Context ID (i)
}
~~~

Endpoints SHOULD close any TIMESTAMP context before closing its inner context.

## Format

TIMESTAMP Datagrams have the following format:

~~~
TIMESTAMP Datagram {
  Context ID (i),
  Timestamp (32),
  Inner Data (..),
}
~~~

`Timestamp` is an NTP short-format timestamp ({{Section 6 of !RFC5905}}), which provides a resolution of ~15 microseconds.  `Inner Data` is a payload to be interpreted in accordance with this context's `Inner Context ID`.

# Examples

PING and TIMESTAMP can be combined as follows.  Note that the client is using a "false start" pattern, sending a dependent registration before the first is confirmed.

~~~
Client                                                    MASQUE Server

# Capsules
REGISTER_PING_CONTEXT(Context ID = 4) ===>
REGISTER_TIMESTAMP_CONTEXT(Context ID = 6, Inner Context ID = 4) ===>
                  <=== ACK_PING_CONTEXT(Context ID = 4, Error Code = 0)
             <=== ACK_TIMESTAMP_CONTEXT(Context ID = 6, Error Code = 0)

# Datagrams
[Context ID(6) + Timestamp(X) + Sequence Number(0) + Opaque Data] --->
               <--- [Context ID(6) + Timestamp(Y) + Sequence Number(1)]
~~~
{: #timestamp-ping example title="TIMESTAMP and PING example"}

TIMESTAMP can also be applied to other payload types, such as UDP packets.  In CONNECT-UDP, these are pre-allocated with Context ID 0.  This example shows a different "false start" pattern, sending a datagram before its context registration is confirmed.

~~~
Client                                                    MASQUE Server

REGISTER_TIMESTAMP_CONTEXT(Context ID = 2, Inner Context ID = 0) ===>
[Context ID(2) + Timestamp(X) + UDP Payload] --->
             <=== ACK_TIMESTAMP_CONTEXT(Context ID = 2, Error Code = 0)

                              # Server waits for a UDP response packet.
                      <--- [Context ID(2) + Timestamp(Y) + UDP Payload]
~~~
{: #timestamp-UDP example title="TIMESTAMP and UDP example"}

# IANA considerations

IANA is directed to add the following entriess to the "HTTP Capsule Types" registry:

| Capsule Type               | Value | Specification   |
| -------------------------- | ----- | --------------- |
| REGISTER_PING_CONTEXT      | TBD   | (This document) |
| ACK_PING_CONTEXT           | TBD   | (This document) |
| REGISTER_TIMESTAMP_CONTEXT | TBD   | (This document) |
| ACK_TIMESTAMP_CONTEXT      | TBD   | (This document) |
| CLOSE_TIMESTAMP_CONTEXT    | TBD   | (This document) |

--- back

# Acknowledgments
{:numbered="false"}

Thanks to Alex Chernyakhovsky for constructive input.
