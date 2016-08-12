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

In {{I-D.ietf-tokbind-protocol}}, the signature field of the TokenBinding
struct is defined to be the signature of the EKM value using the corresponding
token binding key. This document changes that signature to be over one of two
possible exporter values.

The first exporter value is the output of the 0-RTT exporter defined above,
which can be used in any TokenBindingMessage. The second is the exporter
defined in section 7.3.3 of {{I-D.ietf-tls-tls13}}, which can only be used
once the handshake is complete. In both cases, the exporter is called with the
following input values:

- Label: The ASCII string “EXPORTER-Token-Binding” with no terminating NUL.
- Context value: NULL (no application context supplied).
- Length: 32 bytes.

These are the same values as defined in section 3 of
{{I-D.ietf-tokbind-protocol}}.

A client chooses which exporter to use for generating a TokenBindingMessage. A
client which is not sending any 0-RTT data on a connection MUST use the
exporter defined in {{I-D.ietf-tls-tls13}} for all TokenBindingMessages on
that connection so that it is compatible with I-D.ietf-tokbind-protocol-08. A
client that sends a TokenBindingMessage in 0-RTT data must use the 0-RTT
exporter defined in this document since the one in {{I-D.ietf-tls-tls13}}
cannot be used at that time. On a connection where 0-RTT data was sent, a
client may continue to use the 0-RTT exporter for TokenBindingMessages sent
after the TLS handshake has completed. This allows a client to re-send 0-RTT
messages without needing to rewrite the TokenBindingMessage therein.

Since it is at the client’s discretion which value gets signed (for any
TokenBindingMessage on a connection that had early data received by the
server), the server must check both values.

Negotiation TLS extension
-------------------------

The Token Binding negotiation TLS extension ({{I-D.ietf-tokbind-negotiation}})
gains new meaning in 0-RTT handshake.  If both the EarlyDataIndication
extension and the token binding negotiation extension are sent in the same
ClientHello, then the Token Binding negotiation extension MUST contain a single
entry in its key_parameters_list. This key parameter must be the same one that
was negotiated in the session that is being resumed.

In the ServerHello, the server may choose to negotiate the same key type that
the client indicated for 0-RTT data, it may negotiate a different key type
(from the list sent in the ClientHello), or it may choose to not negotiate
Token Binding. If the server negotiates Token Binding, it must either support
the key type that the client indicated for 0-RTT data (and send that key type
in its extension), or it must reject the 0-RTT data.

A server that supports Token Binding but not 0-RTT TokenBindingMessages MUST
NOT issue a NewSessionTicket with the allow_early_data flag set. If it does, a
client might try to negotiate both Token Binding and early data, and when the
early data gets rejected, the client might resend the early data with the same
TokenBindingMessage that used the 0-RTT exporter.

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

HTTP2 allows for requests to different domains to share the same TLS connection
if the SAN of the cert covers those domains. If one.example.com supports 0-RTT
and Token Binding, but two.example.com only supports Token Binding as defined
in I-D.ietf-tokbind-protocol-08, those servers cannot share a cert and use
HTTP2.

Alternatives considered
=======================

Require exporter used to match the data it's sent in
----------------------------------------------------

Instead of allowing the client to use whichever exporter is more convenient,
the client could be required to use the 0-RTT exporter only when the
TokenBindingMessage is sent in 0-RTT data, and use the 1-RTT exporter when the
TokenBindingMessage is sent in 1-RTT data. This approach makes implementations
more difficult. Both the client and the server have to know when the switch
from 0-RTT to 1-RTT occurred (which could potentially be in the middle of a
request), and the client has to rewrite requests if initially sent over 0-RTT
and then rejected and re-sent 1-RTT.

Ratcheting exporters
--------------------

To avoid the synchronization issue in the option above, a ratcheting mechanism
could be used by the server for verifying token binding signatures. In this
mode, the server attempts to verify the token binding message with both the
0-RTT and 1-RTT exporter, but once it has seen a token binding message that
verifies with the 1-RTT exporter, it ratchets up to only try verifying with the
1-RTT exporter. At the time the client generates its TokenBindingMessage, it
can use whichever exporter matches the current state of the TLS handshake.

This option ends up moving around the synchronization issue and doesn’t solve
it. If application data is multiplexed on the TLS connection (like in HTTP2),
the ratchet has to be synchronized across the streams. This makes this approach
impractical.

Use only 0-RTT exporter on connections with Token Binding and early data
------------------------------------------------------------------------

Since the client is allowed to use either exporter, the server has to check
both values to see if a TokenBindingMessage is valid. A client could choose to
always use the 0-RTT exporter, so to simplify implementations, the spec could
require using only the 0-RTT exporter if both Token Binding and early data are
used.

Allow new Token Binding negotiation on 0-RTT connection
-------------------------------------------------------

The proposed design has the client indicate a single Token Binding key
parameter in its ClientHello which prevents the key type from changing between
early data and handshake completion. Another option would be to have the client
put the in first entry of their key parameters list the key type being used in
0-RTT, and allow the client and server to potentially negotiate a new type to
use once the handshake is complete. This alternate gains a slight amount of key
type agility in exchange for implementation difficulty.

Token Binding and 0-RTT data are mutually exclusive
---------------------------------------------------

If a TokenBindingMessage is never allowed in 0-RTT data, then no changes are
needed to the exporter or negotiation. A server that wishes to support Token
Binding must not create any NewSessionTicket messages with the allow_early_data
flag set. A client must not send the token binding negotiation extension and
the EarlyDataIndication extension in the same ClientHello.


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

Early data ticket age window
----------------------------

When an attacker with control of the PSK secret replays a TokenBindingMessage,
it has to use the same ClientHello that the client used. The ClientHello
includes an “obfuscated_ticket_age” in its EarlyDataIndication extension, which
the server can use to narrow the window in which that ClientHello will be
accepted. Even if a PSK is valid for a week, the server will only accept that
particular ClientHello for a smaller time window based on the ticket age. A
server should make their acceptance window for this value as small as practical
to limit an attacker’s ability to replay a ClientHello and send new application
data with the stolen TokenBindingMessage.

Acknowledgements
================


--- back
