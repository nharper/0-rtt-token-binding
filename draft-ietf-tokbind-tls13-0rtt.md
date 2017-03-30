---
title: Token Binding for 0-RTT TLS 1.3 Connections
abbrev: 0-RTT Token Binding
docname: draft-ietf-tokbind-tls13-0rtt-latest
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

This document describes how Token Binding can be used in the 0-RTT data of a TLS
1.3 connection. This involves updating how Token Binding negotiation works and
adding a mechanism for indicating whether a server prevents replay. A
TokenBindingMessage sent in 0-RTT data has different security properties than
one sent after the TLS handshake has finished, which this document also
describes.

--- middle

Introduction
============

Token Binding ({{I-D.ietf-tokbind-protocol}}) cryptographically binds security
tokens (e.g. HTTP cookies, OAuth tokens) to the TLS layer on which they are
presented. It does so by signing an {{RFC5705}} exporter value from the TLS
connection. TLS 1.3 introduces a new mode that allows a client to send
application data on its first flight. If this 0-RTT data contains a security
token, then a client using Token Binding would want to prove possession of its
Token Binding private key so that the server can verify the binding. The
{{RFC5705}}-style exporter provided by TLS 1.3 cannot be run until the
handshake has finished. TLS 1.3 also provides an exporter that can be used with
0-RTT data, but it requires that the application explicitly specify that use.
This document specifies how to use the early_exporter_secret with Token Binding
in TLS 1.3 0-RTT data.

Requirements Language
---------------------

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
{{RFC2119}}.


Proposed Design
===============

A TokenBinding struct as defined in {{I-D.ietf-tokbind-protocol}} contains a
signature of the EKM value from the TLS layer. Under normal circumstances, a
TokenBinding on a TLS 1.3 connection would use the exporter_secret to derive the
EKM value. When 0-RTT data is assembled to be sent, the exporter_secret is not
yet available.  This design changes the definition of the TokenBinding.signature
field to use the exporter with either early_exporter_secret or exporter_secret.
Since no negotiation for the connection can happen before the client sends this
TokenBindingMessage in 0-RTT data, this document also describes how a client
decides what TokenBindingMessage to send in 0-RTT data and how a server should
interpret that message.

If a client does not send any 0-RTT data, or if the server rejects the client’s
0-RTT data, then the client MUST use the 1-RTT exporter, as defined in
{{I-D.ietf-tokbind-protocol}}.

TokenBinding Signature Definition
---------------------------------

In {{I-D.ietf-tokbind-protocol}}, the signature field of the TokenBinding
struct is defined to be the signature of a concatentation that includes the EKM
value. Depending on the circumstances, the exporter value in section 7.3.3 of
{{I-D.ietf-tls-tls13}} is computed using either exporter_secret or
early_exporter_secret as the Secret.

When early_exporter_secret is used as the Secret, the client MUST indicate this
use so the server knows which secret to use in signature verification. This
indication is done through a new Token Binding extension, "early_exporter" (with
extension type TBD). This extension always has 0-length data, so the full
Extension struct is the bytes {0xTBD, 0x00, 0x00}. The early_exporter extension
MUST be present in every TokenBinding struct where the exporter that is signed
uses the early_exporter_secret, and it MUST NOT be present in any other
TokenBinding structs.

### Selecting Which Exporter Secret to Use

A client which is not sending any 0-RTT data on a connection MUST use the
exporter defined in {{I-D.ietf-tls-tls13}} (using exporter_secret as the Secret)
for all TokenBindingMessages on that connection so that it is compatible with
{{I-D.ietf-tokbind-protocol}}.

When a client sends a TokenBindingMessage in 0-RTT data, it must use the
early_exporter_secret. After the client receives an application-layer response
from the server, it must use the exporter_secret for all future token bindings
on that connection. Requests sent after the client's TLS Finished message, but
before the client processes any application-layer response from the server, may
use either exporter secret in their token bindings.

A server may choose to reject an application message containing a Token Binding
that uses the early_exporter_secret. If it chooses to do so, it may send an
application message indicating that the client should re-send the request (with
a new Token Binding). In HTTP, this could be done with a 307 status code.


Negotiating Token Binding
-------------------------

### Negotiation TLS Extension

The behavior of the Token Binding negotiation TLS extension does not change for
a 0-RTT connection: the client and server should process this extension the
same way regardless of whether the client also sent the EarlyDataIndication
extension.

For the sake of choosing a key parameter to use in 0-RTT data, the client MUST
use the same key parameter that was used on the connection during which the
ticket (now being used for resumption) was established. The server MUST NOT
accept early data if the negotiated Token Binding key parameter does not match
the parameter from the initial connection. This is the same behavior as ALPN
and SNI extensions.

If 0-RTT data is being sent with Token Binding using a PSK obtained out-of-band,
then the Token Binding key parameter to use with that PSK must also be
provisioned to both parties, and only that key parameter must be used with that
PSK.

Implementation Challenges
=========================

The client has to be able to modify the message it sends in 0-RTT data if the
0-RTT data gets rejected and needs to be retransmitted in 1-RTT data. Even if
the Token Binding integration with 0-RTT were modified so that Token Binding
never caused a 0-RTT reject that required rewriting a request, the client still
has to handle the server rejecting the 0-RTT data for other reasons.

HTTP2 allows for requests to different domains to share the same TLS connection
if the SAN of the cert covers those domains. If one.example.com supports 0-RTT
and Token Binding, but two.example.com only supports Token Binding as defined
in {{I-D.ietf-tokbind-protocol}}, those servers cannot share a cert and use
HTTP2.


IANA Considerations
===================

This document defines a new Token Binding extension "early_exporter", which
needs to be added to the IANA "Token Binding Extensions" registry.

Security Considerations
=======================

Token Binding messages that use the 0-RTT exporter have weaker security
properties than with the {{RFC5705}} exporter. If either party of a connection
using Token Binding does not wish to use 0-RTT token bindings, they can do so:
a client can choose to never send 0-RTT data on a connection where it uses
token binding, and a server can choose to reject any 0-RTT data sent on a
connection that negotiated token binding.

0-RTT data in TLS 1.3 has weaker security properties than other kinds of
TLS data. Specifically, TLS 1.3 does not guarantee non-replayability of
data between connections. Token Binding has similar replayability issues
when in 0-RTT data, but preventing replay of Token Binding and preventing
replay of 0-RTT data are two separate problems. Token Binding is not
designed to prevent replay of 0-RTT data, although solutions for
preventing the replay of Token Binding might also be applicable to 0-RTT
data.

Proof of Possession of Token Binding Key
----------------------------------------

When a Token Binding signature is generated using the exporter with
early_exporter_secret, the value being signed is under the client's control. An
attacker with temporary access to the Token Binding private key can generate
Token Binding signatures for as many future connections as it has
NewSessionTickets for. An attacker can construct these to be usable at any time
in the future up until the NewSessionTicket's expiration. Section 4.6.1 of
{{I-D.ietf-tls-tls13}} requires that a NewSessionTicket be valid for a maximum
of 7 days.

Unlike in {{I-D.ietf-tokbind-protocol}}, where the proof of possession of the
Token Binding key proves that the client had possession at the time the TLS
handshake finished, 0-RTT Token Binding only proves that the client had
possession of the Token Binding key at some point after receiving the
NewSessionTicket used for that connection.

Attacks on PSK-only Key Exchange and Token Binding
--------------------------------------------------

An attacker who possesses the PSK can eavesdrop on an existing connection that
uses that PSK to obtain a TokenBindingMessage that is valid on the connection
and then hijack the connection to send whatever attacker-controlled data it
wishes. Because the regular exporter closes over the server random, this
TokenBindingMessage is valid only for that connection.

If the attacker does the same thing with a pure-PSK connection and 0-RTT Token
Binding, the attacker can replay the original ClientHello and the exporter will
stay the same, allowing the attacker to obtain a TokenBindingMessage from one
connection and replay it on future connections. The only way for a server to
prevent this replay is to prevent the client from ever repeating a client random
in the handshake.

If a server accepting connections with PSK-only key establishment is concerned
about the threat of PSK theft and also implements Token Binding, then that
server must either reject all 0-RTT token bindings, or implement some form of
preventing reuse of a client random.

Exporter Replayability
----------------------

The exporter specified in {{I-D.ietf-tokbind-protocol}} is chosen so that a
client and server have the same exporter value only if they are on the same TLS
connection. This prevents an attacker who can read the plaintext of a
TokenBindingMessage sent on that connection from replaying that message on
another connection (without also having the token binding private key). The
0-RTT exporter only covers the ClientHello and the PSK of the connection, so it
does not provide this guarantee.

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

Replay Mitigations
------------------

This section presents multiple ways that a client or server can mitigate
the replay of a TokenBinding while still using Token Binding with 0-RTT
data. Note that even with replay mitigations, 0-RTT Token Binding is
vulnerable to other attacks.

### Server Mitigations

If a server uses a session cache instead of stateless tickets, it can
enforce that a PSK generated for resumption can only be used once. If an
attacker tries to replay 0-RTT data (with a TokenBindingMessage), the
server will reject it because the PSK was already used.

Preventing all replay of 0-RTT data is not necessary to prevent replay of
a TokenBinding. A server could implement a mechanism to prevent a
particular TokenBinding from being presented on more than one connection.
In cases where a server's TLS termination and application layer processing
happen in different locations, this option might be easier to implement,
especially when not all requests have bound tokens. This processing can
also take advantage of the structure of the bound token, e.g. a token that
identifies which user is making a request could shard its store of which
TokenBindings have been seen based on the user ID.

A server can prevent some, but not all, 0-RTT data replay with a tight
time window for the ticket age that it will accept. See {{ticket-age}} for
more details.

### Client Mitigations

A client cannot prevent a sufficiently motivated attacker from replaying a
TokenBinding, but it can make it so difficult to replay the TokenBinding
that it is easier for the attacker to steal the Token Binding key
directly. If the client secures the resumption secret with the same level
of protection as the Token Binding key, then the client has made it not
worth the effort of the attacker to attempt to replay a TokenBinding.
Ideally the resumption secret (and Token Binding key) are protected
strongly and virtually non-exportable.

Early Data Ticket Age Window {#ticket-age}
----------------------------

When an attacker with control of the PSK secret replays a TokenBindingMessage,
it has to use the same ClientHello that the client used. The ClientHello
includes an "obfuscated_ticket_age" in its EarlyDataIndication extension, which
the server can use to narrow the window in which that ClientHello will be
accepted. Even if a PSK is valid for a week, the server will only accept that
particular ClientHello for a smaller time window based on the ticket age. A
server should make their acceptance window for this value as small as practical
to limit an attacker’s ability to replay a ClientHello and send new application
data with the stolen TokenBindingMessage.

Acknowledgements
================

The author would like to thank David Benjamin, Steven Valdez, Bill Cox,
and Andrei Popov for their feedback and suggestions.

--- back
