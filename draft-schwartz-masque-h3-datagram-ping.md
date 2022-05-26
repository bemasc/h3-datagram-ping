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

This draft defines new mechanisms for measuring the functionality and performance of an HTTP Datagram path.  These mechanisms can be used with CONNECT-UDP, CONNECT-IP, or any other instantiation of the Capsule Protocol.

--- middle

# Conventions and Definitions

{::boilerplate bcp14}

# PING

PING Datagrams can be used to characterize and monitor the end-to-end HTTP Datagram path associated with an HTTP request.  For example, HTTP endpoints can easily use PING Datagrams to estimate the round-trip time and loss rate of the HTTP Datagram path.

PING Datagrams are also suitable for use as DPLPMTUD Probe Packets {{?RFC8899}}.  This enables endpoints to estimate the HTTP Datagram MTU of each request-response pair, in order to avoid sending HTTP Datagrams that will be dropped.

Note that these path characteristics can differ from those inferred from the underlying transport (e.g. QUIC), if the HTTP request traverses one or more HTTP intermediaries (see {{Section 3.7 of ?I-D.draft-ietf-httpbis-semantics}}).

## Registration

Endpoints indicate support for the PING Datagram type using the Item Structured Field "DG-Ping" in the HTTP Request and Response headers.  Its value MUST be an integer indicating the Context ID allocated for PING datagrams.  (See {{Section 3.3.1 of !RFC8941}} for information about the integer format.)

Endpoints MUST NOT allocate more than one Context ID for PING Datagrams.  As a side effect, this means that only the HTTP client can choose the Context ID used for PING Datagrams.

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

Endpoints indicate support for TIMESTAMP Datagram type by including the boolean-valued Item Structured Field "DG-Timestamp: ?1" in the HTTP Request and Response headers.  (See {{Section 3.3.6 of !RFC8941}} for information about the boolean format.)

A TIMESTAMP Datagram context is opened by a REGISTER_TIMESTAMP_CONTEXT Capsule with the following structure:

~~~
REGISTER_TIMESTAMP_CONTEXT Capsule {
  Context ID (i)
  Inner Context ID (i)
  Short Format (1)
}
~~~

"Inner Context ID" specifies how to interpret the payload after the timestamp.  It MUST be smaller than "Context ID", and MUST already be registered (although that registration does not need to have been confirmed yet).

If "Short Format" is 1 (i.e. true), timestamps MUST use the NTP Short Format ({{Section 6 of !RFC5905}}).  Otherwise, the full NTP Timestamp Format MUST be used.

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

Endpoints SHOULD close any TIMESTAMP context before closing its Inner Context.  If the Inner Context is closed first, datagrams subsequently received on the TIMESTAMP context MUST be dropped.

## Format

TIMESTAMP Datagrams have the following format:

~~~
TIMESTAMP Datagram {
  Context ID (i),
  Timestamp (32..64),
  Inner Data (..),
}
~~~

"Timestamp" is an NTP timestamp in the short or full format, as specified at registration.  The NTP Short Format occupies 4 bytes and provides a resolution of 15 microseconds; the full NTP Timestamp Format occupies 8 bytes and provides a resolution of 232 picoseconds.

"Inner Data" is a payload to be interpreted in accordance with this context's "Inner Context ID".

# Examples

This example shows the PING and TIMESTAMP types used in combination.  Note that the client is using a "false start" pattern, creating and using two registrations before either is confirmed.

~~~
Client                                                          Origin

# Headers
Capsule-Protocol: ?1
DG-Timestamp: ?1
DG-Ping: 42

# Capsules
REGISTER_TIMESTAMP_CONTEXT(Context ID = 6, Inner ID = 42, Short = 1) ==>

# Datagrams
[Context ID(6) + Timestamp(X) + Sequence Number(0) + Opaque Data] --->

# Headers
                                                   Capsule-Protocol: ?1
                                                   DG-Timestamp: ?1
                                                   DG-Ping: 42

# Capsules
              <== ACK_TIMESTAMP_CONTEXT(Context ID = 6, Error Code = 0)

# Datagrams
               <--- [Context ID(6) + Timestamp(Y) + Sequence Number(1)]
~~~
{: #timestamp-ping example title="TIMESTAMP and PING example"}

TIMESTAMP can also be applied to other payload types, such as UDP packets.  In CONNECT-UDP, these are pre-allocated with Context ID 0.  This example similarly shows a "false start" pattern, sending a datagram before its context registration, or support for this format, is confirmed.

~~~
Client                                               CONNECT-UDP Server

# Headers
:method = CONNECT
:protocol = connect-udp
Capsule-Protocol: ?1
DG-Timestamp: ?1

# Capsules
REGISTER_TIMESTAMP_CONTEXT(Context ID = 2, Inner ID = 0, Short = 1) ==>

# Datagrams
[Context ID(2) + Timestamp(X) + UDP Payload] --->

# Headers
                                                   Capsule-Protocol: ?1
                                                   DG-Timestamp: ?1

# Capsules
             <=== ACK_TIMESTAMP_CONTEXT(Context ID = 2, Error Code = 0)

# Datagrams
# ... server waits for a UDP response packet.
                      <--- [Context ID(2) + Timestamp(Y) + UDP Payload]
~~~
{: #timestamp-UDP example title="TIMESTAMP and UDP example"}

# IANA considerations

## Capsule types

IANA is directed to add the following entries to the "HTTP Capsule Types" registry:

| Capsule Type               | Value | Specification   |
| -------------------------- | ----- | --------------- |
| REGISTER_TIMESTAMP_CONTEXT | TBD   | (This document) |
| ACK_TIMESTAMP_CONTEXT      | TBD   | (This document) |
| CLOSE_TIMESTAMP_CONTEXT    | TBD   | (This document) |

## HTTP headers

IANA is directed to add the following entries to the "Hypertext Transfer Protocol (HTTP) Field Name Registry":

| Field Name   | Template | Status    | Reference       | Comments |
| ------------ | -------- | --------- | --------------- | -------- |
| DG-Ping      |          | permanent | (This document) |          |
| DG-Timestamp |          | permanent | (This document) |          |

--- back

# Acknowledgments
{:numbered="false"}

Thanks to Alex Chernyakhovsky for constructive input.
