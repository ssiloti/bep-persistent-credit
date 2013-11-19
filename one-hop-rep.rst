BEP: XXX
Title: Persistent Indirect Reputation
Version: $Revision$
Last-Modified: $Date$
Author:  Steven Siloti <ssiloti@gmail.com>
Status:  Draft
Type:    Standards Track
Requires: 5, 10
Content-Type: text/x-rst
Created: 09-Nov-2013
Post-History:

Abstract
========

This BEP describes a scheme for enabling persistent reciprocation between peers who have exchanged data in one or more torrents with one or more common intermediaries.  It also describes a set of protocol extensions to enable the implementation of this scheme.

This BEP is heavily based on the paper "One hop Reputations for Peer to Peer File Sharing Workloads"[1].


Rationale
=========

The current scheme of reciprocating data transfer within each individual torrent has the shortcoming that it gives peers no incentive to remain active on a torrent after it has finished downloading.  In order to give seed peers an incentive to continue contributing to the swarm a method must be devised to give such peers some token which can be used to gain preferential treatment in future torrents.


Reputation Scheme
=================

Each client generates a 160 bit identifier defined as the SHA1 hash of a cryptographic public key for which the client possesses the corresponding private key. Clients should use ECDSA with curve secp256r1.  The hash must be computed over the public key's representation as the SubjectPublicKeyInfo ASN.1 sequence defined in RFC5280[2] and encoded using DER.  Clients (c) store the following state for each peer identity (p) they discover:


========    ==============================================================================================
Notation    Definition
========    ==============================================================================================
c -> p      bytes of piece data sent directly from the client to the peer
c <- p      bytes of piece data sent directly from the peer to the client
c p> *      bytes of piece data sent to other peers due to the p's recommendation as the intermediary
c <p *      bytes of piece data received by the client from other peers with p active as the intermediary
\* c> p     bytes of piece data sent by any peer to y due to the client's referrals
\* <c p     bytes of piece data sent by y to each of the client's referrals
========    ==============================================================================================

The above state is only maintained for peers who have established a secure channel to the client using the ``starttls`` message.

The first time a peer becomes interested in the client the client sends a ``known_peers`` message.  This message shall contain a list of reputation ids the client has a direct relationship with.  If the client becomes interested in a peer and receives a ``known_peers`` message from it then the client may send one or more ``receipts`` messages were each receipt contains the state of the client at an intermediary present in the ``known_peers`` message.

If the client decides to unchoke a peer on the basis of one or more receipts received from that peer then the client shall, before sending any piece data, send an ``attribution`` message to that peer.  The message shall indicate the weight assigned to each intermediary which was utilized in the decision to unchoke that peer.  The client shall send another ``attribution`` message if it receives additional ``receipts`` messages which change the set of intermediaries used to determine the client's upload rate to that peer.

If the client has determined a target upload rate to a peer based on its reputation it shall send a ``target_rate`` message whenever that rate changes.

While the client is receiving piece data from a peer which has sent an ``attribution`` message it shall periodically send ``receipt`` messages to that peer.  These shall include the client's local state for that peer as well as fractional receipts for all intermediaries listed in the most recently sent ``attribution`` message.

While the client is sending piece data to a peer after having sent an ``attribution`` message it should periodically send ``update_standing`` messages to all intermediaries listed in the ``attribution`` message.  Intermediaries are located by issuing a ``find_node`` DHT query.


Default policy
==============

Clients are not required to implement the default policy.

The client maintains a count of the number of times it observes a peer B, defined as oA(B).  When a session ends where peer B has uploaded piece data to the client, the observation counts of the peers the peer reported in its ``known_peers`` message are incremented by (c <- s) / ((c <24h \*) + r(t)).  Where (c <- s) is the bytes of piece data received during the session, (c <24h \*) is the total bytes of piece data received over the last 24 hours, and r(t) is the number of bytes remaining to be downloaded for the torrent the session took place on.  When a peer fails to respond to an ``update_standing`` message its observation count is reduced by 20% or 2, whichever is larger.

A client A's valuation of an intermediary I is defined as wA(I) = (((A <- I) + (A <I \*))) \* oA(I)) / (((A -> I) + (A I> \*)) \* oA(Max)).  Where oA(Max) is largest observation count of any intermediary the client has local state for.

A client A's valuation of a peer B at intermediary I is defined as vI(B) = ((\* <I B) + (I <- B)) / ((\* <I B) + (I <- B) + (* I> B) + (I -> B)).

A client A's indirect reputation of a peer B is defined as ivalueA(B) equals the sum of wA(I) * vI(B) for each mutual intermediary I, divided by the number of mutual intermediaries being used.

A client A's direct reputation of a peer B is defined as dvalueA(B) = ((A <- B) \* oA(B)) / ((A -> B) \* oA(Max)).

If the client A has an existing direct relationship with a peer B then B's reputation is defined as dvalueA(B), else it is ivalueA(B).  This value is defined as reputationA(B).

When performing an optimistic unchoke the probability of unchoking a peer B is reputationA(B) / reputationA(\*).  Where reputationA(\*) is the sum of reputationA() for all potentially unchoked peers.  Peers without local state at the client are assigned a reputation of 1.

When seeding the client sends piece data to each unchoked and interested peer at rate reputationA(B) / reputationA(U).  Where reputationA(U) is the sum of reputationA(B) for all unchoked and interested peers.  If a peer fails to consume its allocated rate the remainder is distributed to the other peers in proportion to their target rates.  Any upload bandwidth remaining after all peers with an established reputation have been saturated is distributed evenly among all other interested and unchoked peers.  It is expected that clients otherwise retain their existing choking and rate allocation policies.

When incrementing (p <- c) after receipt of piece data, the incremental value is multiplied by ((c -> \*) - (c <- \*)) / r(\*) if it is larger than 1.  Where (c -> \*) is bytes of piece data sent directly by the client to any peer, (c <- \*) is bytes of piece data directly received by the client from any peer, and r(\*) is the total number of bytes remaining in all actively downloading torrents.  This multiplication is not applied when calculating the fractional receipts for intermediaries.

Up to 2000 reputation ids are sent in a ``known_peers`` message.

For each active download session the client sends a ``receipt`` message to the sender every 10 minutes or for every 10MB of data received, whichever happens later.

When the client receives a ``known_peers`` message from a peer which the client is interested in it initially sends a ``receipts`` message with the 10 least observed peers which appear in the ``known_peer`` list received from that peer and which the client has good standing with.  Each minute afterwards the client sends a ``receipts`` message with the next most observed peer until either the client is no longer interested in the peer, the peer unchokes the client and does not indicate a target rate by sending a ``target_rate`` message, or the observed upload rate from the peer drops below 90% of the rate indicated in the most recent ``target_rate`` message.

The client considers at most 10 intermediaries when computing a peer's ivalueA(B).


State representation
====================

When local state is transmitted over the network it is represented as a bencoded dictionary with the following keys:

subject
    The reputation id of the peer whose state this is for.

time
    An integer representing the time this state was generated in POSIX time (Seconds elapsed since 1970-01-01T00:00Z)

ds
    c -> p

dr
    p <- c

is
    c p> *

ir
    c <p *

rs
    \* c> p

rr
    \* <c p

sig
    A cryptographic signature of the dictionary with this key removed.  The signature must be generated using the client's private key.

The client's reputation id is always implied based on context.


Impact on DHT
=============

This BEP assumes that DHT nodes will use their client's reputation identity as their node id.

The following new DHT query is defined:


update_standing
---------------
Used to report a transfer between two peers using the client as an intermediary. The client shall use this information to update its local state for each peer. It has the following parameters:

session
    A randomly generated string of length 4. It is used to uniquely identify a transfer session between two peers.

id
    The reputation id of the peer who sent the piece data.

recipient
    The reputation id of the peer who received the piece data.

intermediary
    The reputation id of the intermediary.

volume
    The total bytes of piece data sent from the sender to the recipient for this session.

sig
    A cryptographic signature of the dictionary with keys "session", "sender", "recipient", "intermediary", and "volume".  The signature must be generated using the private key corresponding to the recipient's reputation id.

The client shall respond with the following keys:

id
    The reputation id of the client.

state
    Local state dictionary for the recipient at the client.


Impact on Bittorrent Protocol
=============================

Per BEP 10, the following extension messages are defined:


starttls
--------
This message has no payload.  The recipient should respond by sending a ``starttls`` message back to the sender. After sending a starttls message no further messages may be sent until the secure channel has been established.  Once the the peer which initiated the connection has both sent and received a starttls message it shall start a TLS handshake by sending a ClientHello message.  Each peer shall use the key pair associated with their reputation id when performing the TLS handshake.  Client authentication is required.  Once a TLS session has been established the stream of bittorrent messages resumes over the secure channel.  Once the secure channel has been established the ``starttls`` message is ignored.  Any previously established stream encryption or obfuscation shall be abandoned after sending the ``starttls`` message.


known_peers
-----------
Indicates the peers with whom the sender has standing and can act as intermediaries.  Its payload is an array of 20-byte reputation ids.  The array should contain the peers which the sender has observed most frequently and be sorted by the sender's wA(I).


receipts
--------
Provides the recipient with proof of the sender's standing with one or more shared intermediaries.  Its payload is a dictionary whose keys are reputation ids and values are the state dictionaries of the sender at the corresponding intermediary.


attribution
-----------
Indicates which intermediaries a the sender considered when unchoking the recipient, and in what proportion each contributed to the decision.  Its payload is a dictionary whose keys are reputation ids and values are integers which must add up to 100.


target_rate
-----------
Tells the recipient what the sender's target upload rate to it is.  Its payload is an integer indicating the target upload rate in bytes/second for the recipient based on the recipient's reputation.  A value of 0 indicates the recipient's reputation does not determine the sender's upload rate.


receipt
-------
During a transfer this message is sent to provide proof of service to the sender.  Its payload is a dictionary with the following keys:

state
    The local state of the sender at the recipient.

receipts
    A list of dictionaries as described in the ``update_standing`` DHT query. One for each of the intermediaries listed in the ``attribution`` message.


Differences from One hop Reputations
====================================

Some key aspects in which this BEP deviates from the paper by Michael Piatek, et. al. are:

- The average rate from y to x is not part of the local state.
- No gossip bit is included in the list of potential intermediaries.
- Receipts are sent by the receiver to the sender at the receiver's leisure rather than requested by the sender.  This is so that receivers can control which intermediaries they wish to utilize based on their bandwidth needs.
- The existing rate based tit-for-tat system is retained while the client is downloading.  Volume based reputation is only used to determine upload rates while seeding and to guide optimistic unchoking.
- vI(B) is modified so that it can never be greater than 1. This so that intermediaries cannot create Sybil identities with arbitrarily large vI(B).
- wA(I) and dvalueA(B) take the observation count of the intermediary/direct peer into account.
- Known peers (top K sets) are sent lazily when the connection enters the appropriate state rather than exchanged at connection time.
- Direct transfer receipts are inflated based on the client's aggregate direct transfer ratio rather than using a fixed multiplier.


Copyright
=========

This document has been placed in the public domain.


.. [1] Michael Piatek, Tomas Isdal, Arvind Krishnamurthy, Thomas Anderson, "One hop Reputations for Peer to Peer File Sharing Workloads",
   NSDI 2008. https://www.usenix.org/legacy/event/nsdi08/tech/full_papers/piatek/piatek_html/

.. [2] Cooper, D., Santesson, S., Farrell, S., Boeyen, S., Housley, R., W. Polk, "Internet X.509 Public Key Infrastructure Certificate and Certificate Revocation List (CRL) Profile",
   RFC 5280, May 2008. http://tools.ietf.org/html/rfc5280


..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:


