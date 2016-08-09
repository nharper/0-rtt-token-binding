---
title: Token Binding for 0-RTT TLS 1.3 Connections
abbrev: 0-RTT Token Binding
docname: draft-nharper-0-rtt-token-binding
date: 2016-08-02
category: std

ipr: trust200902
area: General
workgroup: Token Binding Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs, compact]

author:
 -
    ins: N. Harper
    name: Nick Harper
    organization: Google Inc.
    country: USA
    email: nharper@google.com

normative:
  RFC2119:
  RFC5705:
  I-D.ietf-tokbind-protocol:
  I-D.ietf-tokbind-negotiation:
  I-D.ietf-tls-tls13:

informative:


--- abstract

This document describes how Token Binding can be used in the 0-RTT data of a
TLS 1.3  connection. This involves defining a 0-RTT exporter for TLS 1.3 and
updating how Token Binding negotiation works. A TokenBindingMessage sent in
0-RTT data has different security properties than one sent after the TLS
handshake has finished, which this document also describes.

--- middle

Introduction        {#intro}
============

Token Binding ({{I-D.ietf-tokbind-protocol}}) cryptographically binds security
tokens (e.g. HTTP cookies, OAuth tokens) to the TLS layer on which they are
presented. It does so by signing an {{RFC5705}} exporter value from the TLS
connection. TLS 1.3 introduces a new mode that allows a client to send
application data on its first flight. If this 0-RTT data contains a security
token, the client would want to prove possession of its private key. However,
the TLS exporter cannot be run until the handshake has finished. This document
describes changes to Token Binding to allow for a client to send a proof of
possession in its 0-RTT application data, albeit with weaker security
properties.

Requirements language
---------------------

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
{{RFC2119}}.


Proposed design
===============

A TokenBinding struct as defined in {{I-D.ietf-tokbind-protocol}} contains a
signature of the EKM value from the TLS layer. When a client is building 0-RTT
data to send on a TLS 1.3 connection, there is no EKM value available. This
design changes the definition of the TokenBinding.signature field to use a
different exporter for 0-RTT data, as well as defines that exporter. Since no
negotiation for the connection can happen before the client sends this
TokenBindingMessage in 0-RTT data, this document also describes how a client
decides what TokenBindingMessage to send in 0-RTT data and how a server should
interpret that message.

0-RTT Exporter
--------------

In the key schedule for TLS 1.3, step is added between Early Secret and
HKDF-Extract(ECDHE, Early Secret) to derive a value early_exporter_secret. With
this modification, the key schedule (from {{I-D.ietf-tls-tls13}} section 7.1)
looks like the following:

~~~


                     0
                     |
                     v
       PSK ->  HKDF-Extract
                     |
                     v
               Early Secret
                     |
                     +---------> Derive-Secret(., "early traffic secret",
                     |                         ClientHello)
                     |                         = early_traffic_secret
                     |
                     +---------> Derive-Secret(., “early exporter secret”,
                     |                         ClientHello)
                     |                         = early_exporter_secret
                     v
    (EC)DHE -> HKDF-Extract
                     |
                     ...
~~~


This definition does not affect the value of anything else derived in this key
schedule.

The 0-RTT exporter is defined similarly to exporter in section 7.3.3, and has
the same interface as the RFC 5705 exporter. It is defined as:

~~~


    HKDF-Expand-Label(early_exporter_secret,
                      label, context_value, key_length)
~~~

Where HKDF-Expand-Label is the same function defined in {{I-D.ietf-tls-tls13}}.

TokenBinding signature definition
---------------------------------

In {{I-D.ietf-tokbind-protocol}}, the signature field of the TokenBinding struct
is defined to be the signature of the EKM value using the corresponding token
binding key. This document changes that signature to be over a value
exporter_value dependent on the state of the TLS handshake.

If the handshake has not completed, then the exporter_value is the output of
the 0-RTT exporter defined above, and once the handshake has completed, it is
the output of the exporter defined in section 7.3.3 of {{I-D.ietf-tls-tls13}}.
In both cases, the exporter is called with the following input values:

- Label: The ASCII string “EXPORTER-Token-Binding” with no terminating NUL.
- Context value: NULL (no application context supplied).
- Length: 32 bytes.

These are the same values as defined in section 3 of {{I-D.ietf-tokbind-protocol}}.

Negotiation TLS extension
-------------------------

The Token Binding negotiation TLS extension ({{I-D.ietf-tokbind-negotiation}})
gains new meaning in 0-RTT handshake. If Token Binding is negotiated, the first
TokenBindingKeyParameters in the TokenBindingParameters struct MUST be the same
parameters that were negotiated in the session that is being resumed. Any
TokenBindingMessage that is sent in 0-RTT data MUST use a TokenBindingID with
the same TokenBindingKeyParameters that is in the first position of the
key_parameters_list in TokenBindingParameters.

In the ServerHello, the server may choose to negotiate the same key type that
the client indicated for 0-RTT data, it may negotiate a different key type
(from the list sent in the ClientHello), or it may choose to not negotiate
Token Binding. If the server negotiates Token Binding, it must either support
the key type that the client indicated for 0-RTT data (and send that key type
in its extension), or it must reject the 0-RTT data.

A client may send the Token Binding negotiation TLS extension in a ClientHello
without including a TokenBindingMessage in the application layer data if the
client does not have a key to use for that connection. This is to allow a
client to attempt to negotiate Token Binding on resumption with a server that
previously didn’t support Token Binding without the client having to
speculatively create a keypair and send a TokenBindingMessage.

Implementation challenges
=========================

The server needs to know whether the application data it received was 0-RTT or
1-RTT data to know which exporter to use to verify the signature.

The client has to be able to modify the message it sends in 0-RTT data if the
0-RTT data gets rejected and needs to be retransmitted in 1-RTT data. Even if
the Token Binding integration with 0-RTT were modified so that Token Binding
never caused a 0-RTT reject that required rewriting a request, the client still
has to handle the server rejecting the 0-RTT data for other reasons.

The client has to know whether the TokenBindingMessage it is sending in
application data is being sent as part of 0-RTT data or 1-RTT data to know
which exporter to use. More broadly, it has to be aware that the state of the
token binding negotiation could change between sending 0-RTT data and finishing
the handshake. The handshake could negotiate a different key parameter than
used in 0-RTT data, or it could result in token binding not being used at all.
The client could also send 0-RTT data without a TokenBindingMessage, but
negotiate token binding in the handshake and send a TokenBindingMessage in
subsequent application data.


Alternatives considered
=======================

Ratcheting exporters
--------------------

To avoid the synchronization issue between application data and knowledge of
whether the data is sent 0-RTT or 1-RTT, a ratcheting mechanism could be used
by the server for verifying token binding signatures. In this mode, the server
attempts to verify the token binding message with both the 0-RTT and 1-RTT
exporter, but once it has seen a token binding message that verifies with the
1-RTT exporter, it ratchets up to only try verifying with the 1-RTT exporter.
At the time the client generates its TokenBindingMessage, it can use whichever
exporter matches the current state of the TLS handshake.

This ratcheting mechanism needs to be done per application layer stream, and
not per TLS connection. Consider for example multiple http2 streams on a single
TLS connection. If there are multiple requests that were started in 0-RTT data
(but have the TokenBindingMessage sent after the handshake) and a single
request multiplexed with them that was started after the handshake was finished
(so it has a TokenBindingMessage with a signature over the standard exporter),
this single request might get sent and processed before one of the
TokenBindingMessages from a request that started in 0-RTT data. A
per-connection ratchet would reject those TokenBindingMessages, while a
per-stream ratchet would allow them.

Remember negotiation from previous connection
---------------------------------------------

If the client and server remember what token binding key parameter was
negotiated on the previous connection, the client could assume to use the same
key type for token binding on the new (resumed) connection, and the negotiation
extension in the handshake for the connection would only apply to 1-RTT data.
This requires keeping more state on the server and client, but only slightly
simplifies negotiation (effectively, the 0-RTT Token Binding key parameters are
identical to the previous connection, and the negotiation determines the
parameters going forward after the handshake).


Security Considerations
=======================

Token Binding messages that use the 0-RTT exporter have weaker security
properties than with the RFC 5705 exporter. If either party of a connection
using Token Binding does not wish to use 0-RTT token bindings, they can do so:
a client can choose to never send 0-RTT data on a connection where it uses
token binding, and a server can choose to reject any 0-RTT data sent on a
connection that negotiated token binding.

0-RTT data in TLS 1.3 is not protected against replays. Using token binding in
that 0-RTT data does not provide any additional protection against replays.

Exporter weaknesses
-------------------

The exporter specified in {{I-D.ietf-tokbind-protocol}} is chosen so that a client and server
have the same exporter value only if they are on the same TLS connection. This
prevents an attacker who can read the plaintext of a TokenBindingMessage sent
on that connection from replaying that message on another connection (without
also having the token binding private key). The 0-RTT exporter only covers the
ClientHello and the PSK of the connection, so it does not provide this
guarantee.

An attacker with possession of the PSK secret and a transcript of the
ClientHello and early data sent by a client under that PSK can extract the
TokenBindingMessage, create a new connection to the server (using the same
ClientHello and PSK), and send different application data with the same
TokenBindingMessage. Note that the ClientHello contains public values for the
(EC)DHE key agreement that is used as part of deriving the traffic keys for the
TLS connection, so if the attacker does not also have the corresponding private
values, they will not be able to read the server’s response or send a valid
Finished message in the handshake for this TLS connection. Nevertheless, by
that point the server has already processed the attacker’s message with the
replayed TokenBindingMessage.

This sort of replayability of a TokenBindingMessage is different than the
replayability caveat of 0-RTT application data in TLS 1.3. A network observer
can replay 0-RTT data from TLS 1.3 without knowing any secrets of the client or
server, but the application data that is replayed is untouched. This replay is
done by a more powerful attacker who is able to view the plaintext and then
spoof a connection with the same parameters so that the replayed
TokenBindingMessage still validates when sent with different application data.

Acknowledgements
================


--- back
