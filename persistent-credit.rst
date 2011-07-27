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

Credits do not resemble a real-world currency. The client does not forfeit credits in order to obtain preferential treatment from a peer. Instead credits decay with time. A peer's credit id is 160 bits long and is defined as the SHA1 hash of a cryptographic public key for which the peer possesses the corresponding private key. An individual credit consists of a record containing the public key of the receiving peer and a timestamp. This record is signed using the private key of the issuing peer. Distance between credit ids is determined using the same XOR metric as used by Kademila.

The timestamp defines the value of the credit. A higher timestamp indicates higher value. Credits with timestamps in the past are not necessarily worthless.

When the client downloads data it rewards the uploader by issuing it a credit with a timestamp some distance into the future. The distance is determined by an implementation defined means based on the amount of data transfered and the distance between the client's and the uploader's credit ids. Larger credit id distance should generate larger timestamps. This is to counteract the incentive to game the system by generating a credit id very close to a highly active node.

Indirect Credit
===============

The probability of two peers connecting on unrelated torrents is negligible. This clearly requires that there be some means of applying credit between peers who have not previously connected directly. To do this the client first sends a request to a peer it is both interested in and choked to obtain a list of the best credits which the peer has issued. The client then performs a recursive depth first search by using the DHT to request the best credits issued by the peers returned in the initial response. Once the client finds a list of credits which links itself to the interesting peer it sends this list to the peer. The peer uses these lists to aid in determining which peer to unchoke next.

Credits should only be considered valid if the peer connection they are received on has been secured using the ``starttls`` message defined below.

Credit lists are ranked in descending order according to the minimum timestamp in each list. Multiple credit lists with the same minimum timestamp are ranked in ascending order according to the number of credits in the list.

Impact on Choking
=================

When unchoking peers, two new factors should be taken into consideration:

- Peers which have submitted higher ranked credit lists should be unchoked before peers with lesser ranked ones.
- Peers with closer credit ids to the client should be unchoked before those with farther ones.

The precise manner in which these factors interact with existing choking logic is implementation defined. In general they should only be considered when selecting a peer to optimistically unchoke. Greater influence may be afforded while seeding. Credit id distance should be considered last, it is included to increase the likely hood of establishing useful credits for future indirect application.

Impact on Trackers
==================

Ideally trackers will be modified to record the credit id of the peers it tracks and to automatically return peers which have credit ids close to that of the requesting peer.

The following new key is defined for tracker GET requests and the peer dictionaries in tracker responses:

credit_id
	A string of length 20 which contains the credit id of the peer.

Impact on DHT
=============

This BEP assumes that DHT nodes will use their client's credit id as their node id.

The following new DHT query is defined:

get_credits
-----------
Get the K credits issued by a given credit id to credit ids closest to the querying node's. Credits with timestamps in the past should not be returned. ``get_credits`` has two arguments:

id
	The credit/node id of the querying node.
	
issuer_id
	The credit/node id who's issued credits are being requested.
	
If the queried node has at least one credit issued by the requested credit id it shall return a key "credits" containing a list of up to K credits as dictionaries with the following keys:

recipient_key
	A string containing the public key of the peer which received the credit.
	
sig
	A string containing the signature of the bencoded dictionary with this key removed. It must be generated by the private key possessed by the issuing peer.
	
time
	An integer representing the time value of the credit in POSIX time (Seconds elapsed since midnight UTC 1 January 1970).
	
Credits with timestamps in the past should not be returned since their value will deteriorate rapidly. It is assumed that the querying node will already poses the issuer's public key from a previous query.
	
If the queried node has no credits from the given issuer_id a key "nodes" is returned containing the K nodes in the queried nodes routing table closest to the ``issuer_id`` supplied in the query. See BEP 5 for the format of the "nodes" key.

Impact on Bittorrent Protocol
=============================

Per BEP 10, the following extension messages are defined:

starttls
	This message has no arguments. The receiver should respond by sending a ``starttls`` message back to the originating peer. Once a peer has both sent and received a starttls message it will perform a lexicographical comparison between their peer id and that of the remote peer. The peer whose id comes first shall start a TLS handshake by sending a ClientHello message. Each peer shall use the key pair which determines their credit id when performing the TLS handshake. Client authentication is required. Once a TLS session has been established the stream of bittorrent messages resumes over the secure channel. Once a secure channel has been established the ``starttls`` message is ignored. Any previously established stream encryption or obfuscation shall be abandoned once the secure channel is established.
	
pc_credit
	This message is sent to peers the client is interested in but choked in order to provide an indirect credit list. It has a single argument "credits" which contains a list of dictionaries in the same format as described above for the ``get_credits`` response. The list shall be ordered by issuer with the first credit being issued by the destination peer and the last being issued to the sending peer.

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


