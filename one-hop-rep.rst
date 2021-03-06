BEP: XXX
Title: Persistent Indirect Reputation
Version: $Revision$
Last-Modified: $Date$
Author:  Steven Siloti <ssiloti@gmail.com>
Status:  Draft
Type:    Standards Track
Requires: 5, 10, 44
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

It is also hoped that decoupling the incentive scheme from individual swarms will enable bittorrent to be effectively utilized for applications beyond bulk file transfer such as live streaming and decentralized databases.

Reputation Scheme
=================

Each client generates a 160 bit identifier defined as the SHA1 hash of an Ed25519 public key for which the client possesses the corresponding private key.  The hash MUST be computed over the public key's representation as generated by this `ed25519 library`_.  Clients (c) store the following state for each peer identity (p) they discover:


========    ==============================================================================================
Notation    Definition
========    ==============================================================================================
c -> p      bytes of piece data sent directly from the client to the peer
c <- p      bytes of piece data sent directly from the peer to the client
c p> *      bytes of piece data sent to other peers due to p's recommendation as the intermediary
c <p *      bytes of piece data received by the client from other peers with p active as the intermediary
\* c> p     bytes of piece data sent by any peer to p due to the client's referrals
\* <c p     bytes of piece data sent by p to each of the client's referrals
========    ==============================================================================================

The above state is only maintained for peers who have sent an ``identify`` message.

The first time a peer becomes interested in the client the client sends a ``known_peers`` message.  This message SHALL contain a list of reputation ids the client has a direct relationship with.  If the client becomes interested in a peer and receives a ``known_peers`` message from it then the client MAY send one or more ``my_standing`` messages with the state of the client at intermediaries present in the ``known_peers`` message.

If the client decides to unchoke a peer on the basis of its standing with one or more intermediary received from that peer then the client SHALL, before sending any piece data, send an ``attribution`` message to that peer.  The message SHALL indicate the weight assigned to each intermediary which was utilized in the decision to unchoke that peer.  The client SHALL send another ``attribution`` message if it receives additional ``my_standing`` messages which change the set of intermediaries used to determine the client's upload rate to that peer.

While the client is receiving piece data from a peer which has sent an ``attribution`` message it SHALL periodically send ``receipt`` messages to that peer.  These SHALL include the client's local state for that peer as well as fractional receipts for all intermediaries listed in the most recently sent ``attribution`` message.

When the client receives a ``receipt`` message it SHOULD forward all of the receipts to their corresponding intermediaries using ``update_standing`` messages.  Intermediaries are located by issuing a ``get`` DHT query with their reputation id as the target.  The response to this query MUST be a mutable item signed by the key used to generate the intermediary's reputation id.

The value (c <- p) + (\* <c p) - (c -> p) - (\* c> p) is referred to as a peer's balance at the client.  A standing update cannot cause a peer's balance to become negative.  Any standing update which would violate this constraint is rejected either in part or in full.

A session refers to a continuous period of time over which two peers have at least one connection open to each other.  Multiple concurrent connections between the same peers on different torrents are considered to be part of a single, shared session.


Default policy
==============

Clients are not required to implement the default policy.

The client maintains a count of the number of times it observes a peer B, defined as oA(B).  When a session ends where peer B has uploaded piece data to the client, the observation counts of the peers the peer reported in its ``known_peers`` message are incremented by (c <- s) / ((c <24h \*) + r(t)).  Where (c <- s) is the bytes of piece data received during the session, (c <24h \*) is the total bytes of piece data received over the last 24 hours, and r(t) is the number of bytes remaining to be downloaded for the torrent the session took place on.  When a peer fails to respond to an ``update_standing`` message its observation count is reduced by 20% or 2, whichever is larger.

A client A's direct reputation of a peer B is defined as dvalueA(B) = ((A <- B) - (A -> B) + (\* <A B) - (\* A> B)) \* (oA(B) / oA(Max)).  Where oA(Max) is largest observation count of any intermediary the client has local state for.

A client A's valuation of an intermediary I is defined as wA(I) = ((A <- I) - (A -> I) + (A <I \*) - (A I> \*)) \* (oA(I) / oA(Max)).

A client A's valuation of a peer B at intermediary I is defined as vI(B) = (I <- B) - (I -> B) + (\* <I B) - (\* I> B).

A client A's indirect reputation of a peer B is defined as ivalueA(B) equals the sum of wA(I) * (vI(B) / sum(vI(*)) for each mutual intermediary I, divided by the number of mutual intermediaries being used.

If the client A has an existing direct relationship with a peer B then B's reputation is defined as dvalueA(B), else it is ivalueA(B).  This value is defined as reputationA(B).  If a peer has not sent an ``identify`` message, does not support the ``attribution`` and ``receipt`` messages, or does not have any local state at the client it is assigned a reputationA(B) of 1.

When performing an optimistic unchoke the probability of unchoking a peer B is reputationA(B) / reputationA(\*).  Where reputationA(\*) is the sum of reputationA() for all potentially unchoked peers.

When seeding the client sends piece data to each unchoked and interested peer at rate reputationA(B) / reputationA(U) * U(*).  Where reputationA(U) is the sum of reputationA(B) for all unchoked and interested peers and U(*) is the client's total upload rate or the user configured upload rate limit.  If the user has configured an upload rate limit the client sends ``target_rate`` messages to each eligible peer whenever the peer's target rate changes.  If a peer fails to consume its allocated rate the remainder is distributed to the other peers in proportion to their target rates.  It is expected that clients otherwise retain their existing choking and rate allocation policies.

When incrementing (p <- c) after receipt of piece data, the incremental value is multiplied by ((c -> \*) - (c <- \*)) / r(\*) if it is larger than 1.  Where (c -> \*) is bytes of piece data sent directly by the client to any peer, (c <- \*) is bytes of piece data directly received by the client from any peer, and r(\*) is the total number of bytes remaining in all actively downloading torrents.  This multiplication is not applied when calculating the fractional receipts for intermediaries.

Up to 2000 reputation ids are sent in a ``known_peers`` message.

For each active download session the client sends a ``receipt`` message to the sender every 10 minutes or for every 10MB of data received, whichever happens later.

When the client receives a ``known_peers`` message from a peer which the client is interested in it initially sends a ``my_standing`` message with the 10 least observed peers which appear in the ``known_peer`` list received from that peer and which the client has good standing with.  Each minute afterwards the client sends a ``my_standing`` message with the next most observed peer until either the client is no longer interested in the peer, the peer unchokes the client and does not indicate a target rate by sending a ``target_rate`` message, or the observed upload rate from the peer drops below 90% of the rate indicated in the most recent ``target_rate`` message.

The client considers at most 10 intermediaries when computing a peer's ivalueA(B).

After unchoking a peer and sending an ``attribution`` message, the client sends a ``get_standing`` query to each intermediary listed in the ``attribution`` message for the peer's state.  Upon receiving the state the client forwards it to the peer in a ``your_standing`` message.  If the ``get_standing`` query fails, or the standing received from the intermediary indicates that the peer has a non-positive balance, the peer is choked.

State representation
====================

Local state is represented as a bencoded dictionary with the following keys:

subject
    The reputation id of the peer whose state this is for.  This key SHOULD be omitted when sending state over the network and implied based on context.  Clients MUST validate this key if it is present.

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
    A cryptographic signature of the dictionary with this key removed.  The signature format is as produced by the `ed25519 library`_.

The signer's reputation id is always implied based on context.  When the client receives a state dictionary for a peer at an intermediary for which the client already has a state stored locally the new state supersedes the old state only if all state values are greater-than-or-equal-to those in the stored state.


Receipt representation
======================

When piece data is transfered based on a peer's standing with an intermediary the recipient generates one or more receipts attesting to the transfer having taken place.  It is represented as a bencoded dictionary with the following keys:

seq
    A monotonically increasing integer which uniquely identifies the receipt.  The client SHOULD generate this value using a global counter which is incremented each time a new receipt is generated.

sender
    The reputation id of the peer who sent the piece data.

recipient
    The reputation id of the peer who received the piece data.

intermediary
    The reputation id of the intermediary.

volume
    Bytes of piece data sent from the sender to the recipient since the last receipt was generated.

sig
    A cryptographic signature of the dictionary with this key removed.  The signature format is as produced by the `ed25519 library`_.  The signature MUST be generated using the private key corresponding to the recipient's reputation id.


Contact Information
===================

The client MUST store its contact info as a mutable item as defined by BEP 44.  The item MUST be signed using the same key as used to generate the client's reputation id.  The item's value is the IP and port which the client is listening for DHT messages on.  It may have one of three formats depending on which IP versions the peer is listening on.  All values are stored in "compact" format.

IPv4 only
    IPv4 address followed by port for a total of 6 bytes.

IPv6 only
    IPv6 address followed by port for a total of 18 bytes.

IPv4 and IPv6
    IPv4 address followed by IPv6 address followed by port for a total of 22 bytes.  If an item's value is larger than 22 bytes, the first 22 bytes are assumed to follow this format.


Impact on DHT
=============

The following new DHT queries are defined.  These message MUST be sent directly to the intended recipient, they are never sent as part of a DHT traversal.  The DHT message format is used here only for convenience.


get_standing
------------
Used to retrieve a peer's current state at another peer.  The message's payload is a dictionary with the following keys:

for
    The reputation id of the peer whose state is being requested.

id
    The sender's DHT node id.

sender
    The sender's reputation id.

state
    The local state representation of the recipient at the sender.

The client SHALL respond with the following keys:

id
    The client's DHT node id.

state
    Local state representation for the requested peer at the client.


update_standing
---------------
Used to report a transfer between two peers using the client as an intermediary. The client SHALL use this information to update its local state for each peer.  The message's payload is a dictionary with the following keys:

id
    The sender's DHT node id.

state
    The local state representation of the intermediary at the sender.

receipt
    Receipt representation.  Clients SHOULD omit the intermediary key.  Clients MUST validate the intermediary key if it is present.  The receipt SHOULD be rejected by the intermediary if the sequence number is less-than-or-equal-to the largest value previously received for this pairing of sender and recipient.

The client SHALL respond with the following keys:

id
    The client's DHT node id.

state
    Local state representation for the recipient at the client.


Impact on Bittorrent Protocol
=============================

Per BEP 10, the following extension messages are defined.  All messages except ``identify`` MUST only be sent after an ``identify`` message has been sent.  All messages except ``identify`` MUST be ignored if received on a connection on which an ``identify`` has not been received.  All messages except ``target_rate`` are required.  A peer which does not support all required messages SHOULD be treated as if it does not support any of them.


identify
--------
Provides the identity of the sender and requests the identity of the recipient.  The recipient MUST respond by sending an ``identify`` message back to the sender if it has not already done so.  Its payload is a dictionary with the following keys:

pk
    The sender's public key in the format generated by the `ed25519 library`_.

nonce
    A randomly generated 24 byte string.

After the first ``identify`` message is received on a connection any subsequent ``identify`` messages are ignored.

Any MSE/PE obfuscation is abandoned after sending an identify message.  After an identify message is sent the peer protocol becomes a series of encrypted and authenticated packets.  The first 4 bytes are the length of the packet including the tag.  The next 16 bytes are a Poly1305 tag computed over the remaining, encrypted, payload.  The payload is encrypted using ChaCha20.  Each packet contains one-or-more length prefixed Bittorrent messages.  Bittorrent messages MAY span multiple packets.

The ChaCha20 secret key is the SHA256 hash of an 80 byte string where the first 32 bytes are the output of the function ``ed25519_key_exchange`` provided by the `ed25519 library`_ using the sender's private key and the public key received in the ``identify`` message, the next 24 bytes are the nonce sent by the peer which initiated the connection, and the last 24 bytes are the nonce of the peer which accepted the connection.

Each packet uses a unique nonce for ChaCha20.  The nonce is a 64-bit, unsigned, little endian integer.  Its initial value is 1 for the peer which initiated the connection and 2 for the peer which accepted it.  After each packet the nonce is incremented by 2.

The Poly1305 key used for each packet is generated by taking the first 32 bytes of the output from ChaCha20 with the block counter set to zero.  The remaining 32 bytes of output from the first block are discarded.

The packet body is encrypted by XORing the plaintext with the output of ChaCha20 with the initial block counter set to one.


known_peers
-----------
Indicates the peers with whom the sender has standing and can act as intermediaries.  Its payload is an array of 20-byte reputation ids.  The array SHOULD contain the peers which the sender has observed most frequently and be sorted by the sender's wA(I).


my_standing
-----------
Provides the recipient with proof of the sender's standing with one or more shared intermediaries.  Its payload is a dictionary whose keys are reputation ids and values are the state representations of the sender at the corresponding intermediary.  Clients MUST ignore any items in the state representations which it does not understand, except to include them when verifying the signature.  This message SHOULD only be sent on a connection which the client has received a ``known_peers`` message.

your_standing
-------------
Same as my_standing except the state representations are for the recipient instead of the sender.

attribution
-----------
Indicates which intermediaries a the sender considered when unchoking the recipient, and in what proportion each contributed to the decision.  Its payload is a dictionary whose keys are reputation ids and values are integers which MUST add up to 100.  Clients which implement this message MUST implement the ``update_standing`` DHT query.


target_rate
-----------
Tells the recipient what the sender's target upload rate to it is.  Its payload is an integer indicating the target upload rate in bytes/second for the recipient based on the recipient's reputation.  A value of 0 indicates the recipient's reputation does not determine the sender's upload rate.  This message is optional.  This message MUST only be sent on a connection which the sender has sent an ``attribution`` message.


receipt
-------
During a transfer this message is sent to provide proof of service to the sender.  Its payload is a dictionary with the following keys:

state
    The local state of the sender at the recipient.

receipts
    A list of receipt representations, one for each of the intermediaries listed in the ``attribution`` message.  Clients SHOULD omit the id and recipient keys.  Clients MUST validate the id and recipient keys if they are present.

This message MUST only be sent on a connection which the client has received an ``attribution`` message on.  This message MUST be ignored if received on a connection which the client has not sent an ``attribution`` message on.


Differences from One hop Reputations
====================================

Some key aspects in which this BEP deviates from the paper by Michael Piatek, et. al. are:

- The average rate from y to x is not part of the local state.
- No gossip bit is included in the list of potential intermediaries.
- Proof of standing is sent by the receiver to the sender at the receiver's leisure rather than requested by the sender.  This is so that receivers can control which intermediaries they wish to utilize based on their bandwidth needs.
- The existing rate based tit-for-tat system is retained while the client is downloading.  Volume based reputation is only used to determine upload rates while seeding and to guide optimistic unchoking.
- The default policy uses summation rather than division to compute reputation values.  This is more resistant to whitewashing attacks.
- vI(B) is modified so that it can never be greater than 1. This so that intermediaries cannot create Sybil identities with arbitrarily large vI(B).
- wA(I) and dvalueA(B) take the observation count of the intermediary/direct peer into account.
- Known peers (top K sets) are sent lazily when the connection enters the appropriate state rather than exchanged at connection time.
- Direct transfer receipts are inflated based on the client's aggregate direct transfer ratio rather than using a fixed multiplier.


Copyright
=========

This document has been placed in the public domain.


.. [1] Michael Piatek, Tomas Isdal, Arvind Krishnamurthy, Thomas Anderson, "One hop Reputations for Peer to Peer File Sharing Workloads",
   NSDI 2008. https://www.usenix.org/legacy/event/nsdi08/tech/full_papers/piatek/piatek_html/

.. _ed25519 library: https://github.com/nightcracker/ed25519


..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:


