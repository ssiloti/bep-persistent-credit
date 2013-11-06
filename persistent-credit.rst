BEP: XXX
Title: Persistent peer crediting
Version: $Revision$
Last-Modified: $Date$
Author:  Steven Siloti <ssiloti@gmail.com>
Status:  Draft
Type:    Standards Track
Requires: 5, 10
Content-Type: text/x-rst
Created: 07-Jul-2011
Post-History:

Abstract
========

This BEP describes a scheme for establishing persistent, indirectly applicable credit between peers which have exchanged data in one or more torrents. This information is intended to enhance clients' choking algorithms so that they take into account past contributions made by interested peers. It also describes a set of protocol extensions to enable the implementation of this scheme.

Rationale
=========

The current scheme of reciprocating data transfer within each individual torrent has the shortcoming that it gives peers no incentive to remain active on a torrent after it has completed downloading. In order to give seed peers an incentive to continue contributing to the swarm a method must be devised to give such peers some token which can be used to gain preferential treatment in future torrents.

Credit Scheme
=============

Credits do not resemble a real-world currency. The client does not forfeit credits in order to obtain preferential treatment from a peer. Instead credits decay with time. A peer's credit id is 160 bits long and is defined as the SHA1 hash of a cryptographic public key for which the peer possesses the corresponding private key. An individual credit consists of a record containing the public key of the receiving peer and a timestamp. This record is signed using the private key of the issuing peer. Distance between credit ids is determined using the same XOR metric used by Kademila.

The timestamp defines the value of the credit. A higher timestamp indicates higher value. Credits with timestamps in the past are not necessarily worthless.

Timestamps should only be compared with those issued by the same credit id. Comparisons between different issuing credit ids are not reliable since they may be using different algorithms to generate timestamps.

When the client downloads data it rewards the uploader by issuing it a credit with a timestamp some distance into the future. The distance is determined by an implementation defined means based on the amount of data transfered and the distance between the client's and the uploader's credit ids. Larger credit id distance should generate larger timestamps. This is to counteract the incentive to game the system by generating a credit id very close to a highly active node.

Indirect Credit
===============

The probability of two peers connecting on unrelated torrents is negligible. There must be some means of applying credit between peers who have not previously connected directly. To do this the client first sends a request to a peer it is both interested in and choked to obtain a list of the best credits which the peer has issued. The client then performs a recursive bidirectional search by using the DHT to request credits along the graph of all credits issued by the interesting peer and to the client. Once the client finds a list of credits which links itself to the interesting peer it sends this list to the peer. The peer uses these lists to aid in determining which peer to unchoke next.

Credits should only be considered valid if the peer connection they are received on has been secured using the ``starttls`` message defined below.

Credits not issued by the client should be ranked by comparing the timestamp to all other timestamps known to have been issued by the same credit id. The largest known timestamp is assigned the highest possible rank and the least known timestamp the lowest.

The best credits for a requesting peer should be determined by taking the rank of a credit and factoring in the number of eligible credits with credit ids closer to the target credit id. The target credit id is either the interesting peer or the client depending on the direction of the request.

Impact on Choking
=================

When unchoking peers, two new factors should be taken into consideration:

- Peers which have submitted higher ranked credit lists should be unchoked before peers with lesser ranked ones.
- Peers with closer credit ids to the client should be unchoked before those with farther ones.

The precise manner in which these factors interact with existing choking logic is implementation defined. In general they should only be considered when selecting a peer to optimistically unchoke. Greater influence may be afforded while seeding. Credit id distance should be considered last, it is included to increase the likely hood of establishing useful credits for future indirect application.

Impact on Trackers
==================

Trackers can optionally be modified to record the credit id of the peers it tracks and to automatically return peers which have credit ids close to that of the requesting peer.

The following new key is defined for tracker GET requests and the peer dictionaries in tracker responses:

credit_id
	A string of length 20 which contains the credit id of the peer.

Impact on DHT
=============

This BEP assumes that DHT nodes will use their client's credit id as their node id.

The following new DHT query is defined:

get_credits
-----------
Get up to 6 credits issued by/to a given credit id to/by credit ids close to another id. Credits with timestamps in the past should not be returned. ``get_credits`` has three arguments:

d
	The direction of the search. If 'b' credits issued by the target id are requested. If 't' credits issued to target id are requested.

t
	The "target" id. The credit/node id which credits issued by/to are being requested.

g
	The "goal" id. The credit/node id which is the goal of the search, the best credits should be issued by/to ids close to this id.

If the queried node has at least one credit issued by/to the target credit id it shall return a key "credits" containing a list of up to 6 credits in a compact format as follows:

+-----------+------------------------------------------------------------------------------+
| Byte      | Value                                                                        |
+===========+==============================================================================+
| 0         | The number of unique peer public keys to follow (P)                          |
+-----------+------------------------------------------------------------------------------+
| 1         | The number of bytes in the first peer key                                    |
+-----------+------------------------------------------------------------------------------+
| 2..A      | The first peer key in DER format (Index 0)                                   |
+-----------+------------------------------------------------------------------------------+
| ...       + ...                                                                          |
+-----------+------------------------------------------------------------------------------+
| B         | The number of bytes in the last peer key                                     |
+-----------+------------------------------------------------------------------------------+
| B+1..C    | The second peer key in DER format (Index P-1)                                |
+-----------+------------------------------------------------------------------------------+
| C+1       | The number of credits to follow (R)                                          |
+-----------+------------------------------------------------------------------------------+
| C+2       | The peer key index which issued the credit                                   |
+-----------+------------------------------------------------------------------------------+
| C+3..C+10 | The timestamp of the credit as an unsigned 64-bit big endian integer         |
+-----------+------------------------------------------------------------------------------+
|           | representing the number of seconds elapsed since midnight UTC 1 January 1970 |
+-----------+------------------------------------------------------------------------------+
|           | I.e. POSIX time                                                              |
+-----------+------------------------------------------------------------------------------+
| C+11      | The peer key index to which the credit was issued                            |
+-----------+------------------------------------------------------------------------------+
| C+12      | The number of bytes in the credit signature                                  |
+-----------+------------------------------------------------------------------------------+
| C+13..D   | A signature taken over the big endian representation of the                  |
+-----------+------------------------------------------------------------------------------+
|           | timestamp followed by one byte indicating the length of the                  |
+-----------+------------------------------------------------------------------------------+
|           | recipient's public key followed by the public key in DER format              |
+-----------+------------------------------------------------------------------------------+
|           | I.e. replace byte C+11 with the corresponding public key then take the       |
+-----------+------------------------------------------------------------------------------+
|           | signature over the range [C+3,C+12)                                          |
+-----------+------------------------------------------------------------------------------+
| ...       + ...                                                                          |
+-----------+------------------------------------------------------------------------------+

The following peer key indexes have special meaning:

254. Refers to the public key of the client. I.e. the originator of the request.
255. Refers to the public key corresponding to the "target" credit id.

If the queried node has no credits from the given issuer_id a key "nodes" is returned containing the K nodes in the queried nodes routing table closest to the target id supplied in the query. See BEP 5 for the format of the "nodes" key.

Impact on Bittorrent Protocol
=============================

Per BEP 10, the following extension messages are defined:

starttls
	This message has no arguments. The receiver should respond by sending a ``starttls`` message back to the originating peer. After sending a starttls message no further messages may be sent until the secure channel has been established. Once the the peer which initiated the connection has both sent and received a starttls message it shall start a TLS handshake by sending a ClientHello message. Each peer shall use the key pair which determines their credit id when performing the TLS handshake. Client authentication is required. Once a TLS session has been established the stream of bittorrent messages resumes over the secure channel. Once a secure channel has been established the ``starttls`` message is ignored. Any previously established stream encryption or obfuscation shall be abandoned once the secure channel is established.

pc_credit
	This message is sent to peers the client is interested in but choked in order to provide an indirect credit list. It has a single argument "credits" which contains a compact list of credits in the same format as described above for the ``get_credits`` response. The list shall be ordered by issuer with the first credit being issued by the receiving peer and the last being issued to the client. The "target" public key index refers to the receiving peer.

Copyright
=========

This document has been placed in the public domain.



..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:


