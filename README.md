# p2p




Internet Draft                                                   B. Ford
Document: draft-ford-midcom-p2p-01.txt                            M.I.T.
Expires: April 27, 2004                                     P. Srisuresh
                                                          Caymas Systems
                                                                D. Kegel
                                                               kegel.com
                                                            October 2003


              Peer-to-Peer (P2P) communication across middleboxes


Status of this Memo

   This document is an Internet-Draft and is subject to all provisions
   of Section 10 of RFC2026.  Internet-Drafts are working documents of
   the Internet Engineering Task Force (IETF), its areas, and its
   working groups.  Note that other groups may also distribute working
   documents as Internet-Drafts.

   Internet-Drafts are draft documents valid for a maximum of six months
   and may be updated, replaced, or obsoleted by other documents at any
   time.  It is inappropriate to use Internet- Drafts as reference
   material or to cite them other than as "work in progress."

   The list of current Internet-Drafts can be accessed at
   http://www.ietf.org/1id-abstracts.html

   The list of Internet-Draft Shadow Directories can be accessed at
   http://www.ietf.org/shadow.html

   Distribution of this document is unlimited.

Copyright Notice

   Copyright (C) The Internet Society (2003).  All Rights Reserved.


Abstract

   This memo documents the methods used by the current peer-to-peer
   (P2P) applications to communicate in the presence of middleboxes
   such as firewalls and network address translators (NAT). In
   addition, the memo suggests guidelines to application designers
   and middlebox implementers on the measures they could take to
   enable immediate, wide deployment of P2P applications with or
   without requiring the use of special proxy, relay or midcom
   protocols.  




Ford, Srisuresh & Kegel                                         [Page 1]

Internet-Draft     P2P applications across middleboxes      October 2003



Table of Contents

   1.  Introduction .................................................
   2.  Terminology ..................................................
   3.  Techniques for P2P communication over middleboxes ............
       3.1.  Relaying ...............................................
       3.2.  Connection reversal ....................................
       3.3.  UDP Hole Punching ......................................
             3.3.1.  Peers behind different NATs ..................
             3.3.2.  Peers behind the same NAT ....................
             3.3.3.  Peers separated by multiple NATs ...............
             3.3.4.  Consistent port bindings .......................
       3.4.  UDP Port number prediction .............................
       3.5.  Simultaneous TCP open ..................................
   4.  Application design guidelines ................................
       4.1. What works with P2P middleboxes .........................
       4.2. Applications behind the same NAT ........................
       4.3. Peer discovery ..........................................
       4.4. TCP P2P applications ....................................
       4.5. Use of midcom protocol ..................................
   5.  NAT design guidelines ........................................
       5.1. Deprecate the use of symmetric NATs .....................
       5.2. Add incremental Cone-NAT support to symmetric NAT devices
       5.3. Maintaining consistent port bindings for UDP ports .....
             5.3.1.  Preserving Port Numbers ........................
       5.4. Maintaining consistent port bindings for TCP ports .....
       5.5. Large timeout for P2P applications ......................
   6.  Security considerations ......................................


1. Introduction

   Present-day Internet has seen ubiquitous deployment of
   "middleboxes" such as network address translators(NAT), driven
   primarily by the ongoing depletion of the IPv4 address space.  The
   asymmetric addressing and connectivity regimes established by these
   middleboxes, however, have created unique problems for peer-to-peer
   (P2P) applications and  protocols, such as teleconferencing and
   multiplayer on-line gaming. These issues are likely to persist even
   into the IPv6 world, where NAT is often used as an IPv4 compatibility
   mechanism [NAT-PT], and firewalls will still be commonplace even 
   after NAT is no longer required.

   Currently deployed middleboxes are designed primarily around the
   client/server paradigm, in which relatively anonymous client machines
   actively initiate connections to well-connected servers having stable
   IP addresses and DNS names.  Most middleboxes implement an asymmetric



Ford, Srisuresh & Kegel                                         [Page 2]

Internet-Draft     P2P applications across middleboxes      October 2003


   communication model in which hosts on the private internal network
   can initiate outgoing connections to hosts on the public network, but
   external hosts cannot initiate connections to internal hosts except
   as specifically configured by the middlebox's administrator. In the
   common case of NAPT, a client on the internal network does not have
   a unique IP address on the public Internet, but instead must share
   a single public IP address, managed by the NAPT, with other hosts
   on the same private network.  The anonymity and inaccessibility of
   the internal hosts behind a middlebox is not a problem for client
   software such as web browsers, which only need to initiate outgoing
   connections. This inaccessibility is sometimes seen as a privacy
   benefit.

   In the peer-to-peer paradigm, however, Internet hosts that would
   normally be considered "clients" need to establish communication
   sessions directly with each other. The initiator and the responder
   might lie behind different middleboxes with neither endpoint 
   having any permanent IP address or other form of public network
   presence. A common on-line gaming architecture, for example,
   is for the participating application hosts to contact a well-known
   server for initialization and administration purposes. Subsequent
   to this, the hosts establish direct connections with each other
   for fast and efficient propagation of updates during game play. 
   Similarly, a file sharing application might contact a well-known
   server for resource discovery or searching, but establish direct
   connections with peer hosts for data transfer. Middleboxes create
   problems for peer-to-peer connections because hosts behind a
   middlebox normally have no permanently usable public ports on the
   Internet to which incoming TCP or UDP connections from other peers
   can be directed.  RFC 3235 [NAT-APPL] briefly addresses this issue,
   but does not offer any general solutions.

   In this document we address the P2P/middlebox problem in two ways.
   First, we summarize known methods by which P2P applications can
   work around the presence of middleboxes. Second, we provide a set
   of application design guidelines based on these practices to make
   P2P applications operate more robustly over currently-deployed
   middleboxes. Further, we provide design guidelines for future
   middleboxes to allow them to support P2P applications more
   effectively. Our focus is to enable immediate and wide deployment
   of P2P applications requiring to traverse middleboxes.

2. Terminology

In this section we first summarize some middlebox terms. We focus here
on the two kinds of middleboxes that commonly cause problems for P2P
applications.




Ford, Srisuresh & Kegel                                         [Page 3]

Internet-Draft     P2P applications across middleboxes      October 2003


   Firewall
      A firewall restricts communication between a private internal
      network and the public Internet, typically by dropping packets
      that are deemed unauthorized.  A firewall examines but does
      not modify the IP address and TCP/UDP port information in
      packets crossing the boundary.

   Network Address Translator (NAT)
      A network address translator not only examines but also modifies
      the header information in packets flowing across the boundary,
      allowing many hosts behind the NAT to share the use of a smaller
      number of public IP addresses (often one).

   Network address translators in turn have two main varieties:

   Basic NAT
      A Basic NAT maps an internal host's private IP address to a
      public IP address without changing the TCP/UDP port
      numbers in packets crossing the boundary.  Basic NAT is generally
      only useful when the NAT has a pool of public IP addresses from
      which to make address bindings on behalf of internal hosts.

   Network Address/Port Translator (NAPT)
      By far the most common, a Network Address/Port Translator examines
      and modifies both the IP address and the TCP/UDP port number
      fields of packets crossing the boundary, allowing multiple
      internal hosts to share a single public IP address simultaneously.

   Refer to [NAT-TRAD] and [NAT-TERM] for more general information on
   NAT taxonomy and terminology. Additional terms that further classify
   NAPT are defined in more recent work [STUN]. When an internal host
   opens an outgoing TCP or UDP session through a network address/port
   translator, the NAPT assigns the session a public IP address and
   port number so that subsequent response packets from the external
   endpoint can be received by the NAPT, translated, and forwarded
   to the internal host. The effect is that the NAPT establishes a 
   port binding between (private IP address, private port number) and
   (public IP address, public port number). The port binding
   defines the address translation the NAPT will perform for the
   duration of the session.  An issue of relevance to P2P
   applications is how the NAT behaves when an internal host initiates
   multiple simultaneous sessions from a single (private IP, private
   port) pair to multiple distinct endpoints on the external network.

   Cone NAT
      After establishing a port binding between a (private IP, private
      port) tuple and a (public IP, public port) tuple, a cone NAT will 
      re-use this port binding for subsequent sessions the



Ford, Srisuresh & Kegel                                         [Page 4]

Internet-Draft     P2P applications across middleboxes      October 2003


      application may initiate from the same private IP address and
      port number, for as long as at least one session using the port
      binding remains active.

      For example, suppose Client A in the diagram below initiates two
      simultaneous outgoing sessions through a cone NAT, from the same
      internal endpoint (10.0.0.1:1234) to two different
      external servers, S1 and S2.  The cone NAT assigns just one public
      endpoint tuple, 155.99.25.11:62000, to both of these sessions,
      ensuring that the "identity" of the client's port is maintained
      across address translation. Since Basic NATs and firewalls do 
      not modify port numbers as packets flow across
      the middlebox, these types of middleboxes can be viewed as a
      degenerate form of Cone NAT.



           Server S1                                     Server S2
        18.181.0.31:1235                              138.76.29.7:1235
               |                                             |
               |                                             |
               +----------------------+----------------------+
                                      |
          ^  Session 1 (A-S1)  ^      |      ^  Session 2 (A-S2)  ^
          |  18.181.0.31:1235  |      |      |  138.76.29.7:1235  |
          v 155.99.25.11:62000 v      |      v 155.99.25.11:62000 v
                                      |
                                   Cone NAT
                                 155.99.25.11
                                      |
          ^  Session 1 (A-S1)  ^      |      ^  Session 2 (A-S2)  ^
          |  18.181.0.31:1235  |      |      |  138.76.29.7:1235  |
          v   10.0.0.1:1234    v      |      v   10.0.0.1:1234    v
                                      |
                                   Client A
                                10.0.0.1:1234















Ford, Srisuresh & Kegel                                         [Page 5]

Internet-Draft     P2P applications across middleboxes      October 2003


   Symmetric NAT
      A symmetric NAT, in contrast, does not maintain a consistent
      port binding  between (private IP, private port) and (public IP,
      public port) across all sessions. Instead, it assigns a new
      public port to each new session.  For example, suppose Client A
      initiates two outgoing sessions from the same port as above, one
      with S1 and one with S2.  A symmetric NAT might allocate the
      public endpoint 155.99.25.11:62000 to session 1, and then allocate
      a different public endpoint 155.99.25.11:62001, when the
      application initiates session 2.  The NAT is able to differentiate
      between the two sessions for translation purposes because the
      external endpoints involved in the sessions (those of S1
      and S2) differ, even as the endpoint identity of the client 
      application is lost across the address translation boundary.



           Server S1                                     Server S2
        18.181.0.31:1235                              138.76.29.7:1235
               |                                             |
               |                                             |
               +----------------------+----------------------+
                                      |
          ^  Session 1 (A-S1)  ^      |      ^  Session 2 (A-S2)  ^
          |  18.181.0.31:1235  |      |      |  138.76.29.7:1235  |
          v 155.99.25.11:62000 v      |      v 155.99.25.11:62001 v
                                      |
                                 Symmetric NAT
                                 155.99.25.11
                                      |
          ^  Session 1 (A-S1)  ^      |      ^  Session 2 (A-S2)  ^
          |  18.181.0.31:1235  |      |      |  138.76.29.7:1235  |
          v   10.0.0.1:1234    v      |      v   10.0.0.1:1234    v
                                      |
                                   Client A
                                10.0.0.1:1234

   The issue of cone versus symmetric NAT behavior applies equally 
   to TCP and UDP traffic. 

   Cone NAT is further classified according to how liberally the NAT
   accepts incoming traffic directed to an already-established (public
   IP, public port) pair.  This classification generally applies only to
   UDP traffic, since NATs and firewalls reject incoming TCP
   connection attempts unconditionally unless specifically configured to
   do otherwise.

   Full Cone NAT



Ford, Srisuresh & Kegel                                         [Page 6]

Internet-Draft     P2P applications across middleboxes      October 2003


      After establishing a public/private port binding for a new
      outgoing session, a full cone NAT will subsequently accept
      incoming traffic to the corresponding public port from ANY
      external endpoint on the public network.  Full cone NAT is
      also sometimes called "promiscuous" NAT.

   Restricted Cone NAT
      A restricted cone NAT only forwards an incoming packet directed to
      a public port if its external (source) IP address matches the
      address of a node to which the internal host has previously sent
      one or more outgoing packets.  A restricted cone NAT effectively
      refines the firewall principle of rejecting unsolicited incoming
      traffic, by restricting incoming traffic to a set of "known" 
      external IP addresses.

   Port-Restricted Cone NAT
      A port-restricted cone NAT, in turn, only forwards an incoming
      packet if its external IP address AND port number match those of
      an external endpoint to which the internal host has previously
      sent outgoing packets.  A port-restricted cone NAT provides 
      internal nodes the same level of protection against unsolicited
      incoming traffic that a symmetric NAT does, while maintaining a
      private port's identity across translation.

   Finally, in this document we define new terms for classifying
   the P2P-relevant behavior of middleboxes:

   P2P-Application
      P2P-application as used in this document is an application in
      which each P2P participant registers with a public
      registration server, and subsequently uses either its 
      private endpoint, or public endpoint, or both, to establish
      peering sessions.

   P2P-Middlebox
      A P2P-Middlebox is middlebox that permits the traversal of
      P2P applications. 

   P2P-firewall
      A P2P-firewall is a P2P-Middlebox that provides firewall
      functionality but performs no address translation.

   P2P-NAT
      A P2P-NAT is a P2P-Middlebox that provides NAT functionality, and
      may also provide firewall functionality. At minimum, a
      P2P-Middlebox must implement Cone NAT behavior for UDP traffic,
      allowing applications to establish robust P2P connectivity using
      the UDP hole punching technique.



Ford, Srisuresh & Kegel                                         [Page 7]

Internet-Draft     P2P applications across middleboxes      October 2003



   Loopback translation
      When a host in the private domain of a NAT device attempts to
      connect with another host behind the same NAT device using
      the public address of the host, the NAT device performs the 
      equivalent of a "Twice-nat" translation on the packet as
      follows. The originating host's private endpoint is translated
      into its assigned public endpoint, and the target host's public
      endpoint is translated into its private endpoint, before 
      the packet is forwarded to the target host. We refer the above
      translation performed by a NAT device as "Loopback translation".
  
3. Techniques for P2P Communication over middleboxes

   This section reviews in detail the currently known techniques for
   implementing peer-to-peer communication over existing middleboxes,
   from the perspective of the application or protocol designer.

3.1. Relaying

   The most reliable, but least efficient, method of implementing peer-
   to-peer communication in the presence of a middlebox is to make the
   peer-to-peer communication look to the network like client/server
   communication through relaying.  For example, suppose two client
   hosts, A and B, have each initiated TCP or UDP connections with a
   well-known server S having a permanent IP address.  The clients
   reside on separate private networks, however, and their respective
   middleboxes prevent either client from directly initiating a
   connection to the other.

                                Server S
                                   |
                                   |
            +----------------------+----------------------+
            |                                             |
          NAT A                                         NAT B
            |                                             |
            |                                             |
         Client A                                      Client B

   Instead of attempting a direct connection, the two clients can simply
   use the server S to relay messages between them.  For example, to
   send a message to client B, client A simply sends the message to
   server S along its already-established client/server connection, and
   server S then sends the message on to client B using its existing
   client/server connection with B.

   This method has the advantage that it will always work as long as



Ford, Srisuresh & Kegel                                         [Page 8]

Internet-Draft     P2P applications across middleboxes      October 2003


   both clients have connectivity to the server.  Its obvious
   disadvantages are that it consumes the server's processing power and
   network bandwidth unnecessarily, and communication latency between
   the two clients is likely to be increased even if the server is well-
   connected.  The TURN protocol [TURN] defines a method of implementing
   relaying in a relatively secure fashion.













































Ford, Srisuresh & Kegel                                         [Page 9]

Internet-Draft     P2P applications across middleboxes      October 2003


3.2. Connection reversal

   The second technique works if only one of the clients is behind a
   middlebox.  For example, suppose client A is behind a NAT but client
   B has a globally routable IP address, as in the following diagram:

                                Server S
                            18.181.0.31:1235
                                   |
                                   |
            +----------------------+----------------------+
            |                                             |
          NAT A                                           |
    155.99.25.11:62000                                    |
            |                                             |
            |                                             |
         Client A                                      Client B
      10.0.0.1:1234                               138.76.29.7:1234

   Client A has private IP address 10.0.0.1, and the application is
   using TCP port 1234.  This client has established a connection with
   server S at public IP address 18.181.0.31 and port 1235.  NAT A has
   assigned TCP port 62000, at its own public IP address 155.99.25.11,
   to serve as the temporary public endpoint address for A's session
   with S: therefore, server S believes that client A is at IP address
   155.99.25.11 using port 62000.  Client B, however, has its own
   permanent IP address, 138.76.29.7, and the peer-to-peer application
   on B is accepting TCP connections at port 1234.

   Now suppose client B would like to initiate a peer-to-peer
   communication session with client A.  B might first attempt to
   contact client A either at the address client A believes itself to
   have, namely 10.0.0.1:1234, or at the address of A as observed by
   server S, namely 155.99.25.11:62000.  In either case, however, the
   connection will fail.  In the first case, traffic directed to IP
   address 10.0.0.1 will simply be dropped by the network because
   10.0.0.1 is not a publicly routable IP address.  In the second case,
   the TCP SYN request from B will arrive at NAT A directed to port
   62000, but NAT A will reject the connection request because only
   outgoing connections are allowed.

   After attempting and failing to establish a direct connection to A,
   client B can use server S to relay a request to client A to initiate
   a "reversed" connection to client B.  Client A, upon receiving this
   relayed request through S, opens a TCP connection to client B at B's
   public IP address and port number.  NAT A allows the connection to
   proceed because it is originating inside the firewall, and client B
   can receive the connection because it is not behind a middlebox.



Ford, Srisuresh & Kegel                                        [Page 10]

Internet-Draft     P2P applications across middleboxes      October 2003



   A variety of current peer-to-peer systems implement this technique.
   Its main limitation, of course, is that it only works as long as only
   one of the communicating peers is behind a NAT: in the increasingly
   common case where both peers are behind NATs, the method fails.  
   Because connection reversal is not a general solution to the problem,
   it is NOT recommended as a primary strategy.  Applications may choose
   to attempt connection reversal, but should be able to fall back
   automatically on another mechanism such as relaying if neither a
   "forward" nor a "reverse" connection can be established.

3.3. UDP hole punching

   The third technique, and the one of primary interest in this
   document, is widely known as "UDP Hole Punching."  UDP hole punching
   relies on the properties of common firewalls and cone NATs to allow
   appropriately designed peer-to-peer applications to "punch holes"
   through the middlebox and establish direct connectivity with each
   other, even when both communicating hosts may lie behind middleboxes.
   This technique was mentioned briefly in section 5.1 of RFC 3027 [NAT-
   PROT], and has been informally described elsewhere on the Internet
   [KEGEL] and used in some recent protocols [TEREDO, ICE].  As the name
   implies, unfortunately, this technique works reliably only with UDP.

   We will consider two specific scenarios, and how applications can be
   designed to handle both of them gracefully.  In the first situation,
   representing the common case, two clients desiring direct peer-to-
   peer communication reside behind two different NATs.  In the second,
   the two clients actually reside behind the same NAT, but do not
   necessarily know that they do.

3.3.1. Peers behind different NATs

   Suppose clients A and B both have private IP addresses and lie behind
   different network address translators.  The peer-to-peer application
   running on clients A and B and on server S each use UDP port 1234.  A
   and B have each initiated UDP communication sessions with server S,
   causing NAT A to assign its own public UDP port 62000 for A's session
   with S, and causing NAT B to assign its port 31000 to B's session
   with S, respectively.

                                Server S
                            18.181.0.31:1234
                                   |
                                   |
            +----------------------+----------------------+
            |                                             |
          NAT A                                         NAT B



Ford, Srisuresh & Kegel                                        [Page 11]

Internet-Draft     P2P applications across middleboxes      October 2003


    155.99.25.11:62000                            138.76.29.7:31000
            |                                             |
            |                                             |
         Client A                                      Client B
      10.0.0.1:1234                                 10.1.1.3:1234

   Now suppose that client A wants to establish a UDP communication
   session directly with client B.  If A simply starts sending UDP
   messages to B's public address, 138.76.29.7:31000, then NAT B will
   typically discard these incoming messages (unless it is a full cone
   NAT), because the source address and port number does not match those
   of S, with which the original outgoing session was established.
   Similarly, if B simply starts sending UDP messages to A's public
   address, then NAT A will typically discard these messages.

   Suppose A starts sending UDP messages to B's public address, however,
   and simultaneously relays a request through server S to B, asking B
   to start sending UDP messages to A's public address.  A's outgoing
   messages directed to B's public address (138.76.29.7:31000) cause NAT
   A to open up a new communication session between A's private address
   and B's public address.  At the same time, B's messages to A's public
   address (155.99.25.11:62000) cause NAT B to open up a new
   communication session between B's private address and A's public
   address.  Once the new UDP sessions have been opened up in each
   direction, client A and B can communicate with each other directly
   without further burden on the "introduction" server S.

   The UDP hole punching technique has several useful properties.  Once
   a direct peer-to-peer UDP connection has been established between two
   clients behind middleboxes, either party on that connection can in
   turn take over the role of "introducer" and help the other party
   establish peer-to-peer connections with additional peers, minimizing
   the load on the initial introduction server S.  The application does
   not need to attempt to detect explicitly what kind of middlebox it is
   behind, if any [STUN], since the procedure above will establish peer-
   to-peer communication channels equally well if either or both clients
   do not happen to be behind a middlebox.  The hole punching technique
   even works automatically with multiple NATs, where one or both
   clients are removed from the public Internet via two or more levels
   of address translation.

3.3.2. Peers behind the same NAT

   Now consider the scenario in which the two clients (probably
   unknowingly) happen to reside behind the same NAT, and are therefore
   located in the same private IP address space.  Client A has
   established a UDP session with server S, to which the common NAT has
   assigned public port number 62000.  Client B has similarly



Ford, Srisuresh & Kegel                                        [Page 12]

Internet-Draft     P2P applications across middleboxes      October 2003


   established a session with S, to which the NAT has assigned public
   port number 62001.

                                Server S
                            18.181.0.31:1234
                                   |
                                   |
                                  NAT
                         A-S 155.99.25.11:62000
                         B-S 155.99.25.11:62001
                                   |
            +----------------------+----------------------+
            |                                             |
         Client A                                      Client B
      10.0.0.1:1234                                 10.1.1.3:1234

   Suppose that A and B use the UDP hole punching technique as outlined
   above to establish a communication channel using server S as an
   introducer.  Then A and B will learn each other's public IP addresses
   and port numbers as observed by server S, and start sending each
   other messages at those public addresses.  The two clients will be
   able to communicate with each other this way as long as the NAT
   allows hosts on the internal network to open translated UDP sessions
   with other internal hosts and not just with external hosts. We refer
   to this situation as "loopback translation," because packets arriving
   at the NAT from the private network are translated and then "looped
   back" to the private network rather than being passed through to the
   public network.  For example, when A sends a UDP packet to B's public
   address, the packet initially has a source IP address and port number
   of 10.0.0.1:124 and a destination of 155.99.25.11:62001.  The NAT
   receives this packet, translates it to have a source of
   155.99.25.11:62000 (A's public address) and a destination of
   10.1.1.3:1234, and then forwards it on to B.  Even if loopback
   translation is supported by the NAT, this translation and forwarding
   step is obviously unnecessary in this situation, and is likely to add
   latency to the dialog between A and B as well as burdening the NAT.

   The solution to this problem is straightforward, however.  When A and
   B initially exchange address information through server S, they
   should include their own IP addresses and port numbers as "observed"
   by themselves, as well as their addresses as observed by S.  The
   clients then simultaneously start sending packets to each other at
   each of the alternative addresses they know about, and use the first
   address that leads to successful communication.  If the two clients
   are behind the same NAT, then the packets directed to their private
   addresses are likely to arrive first, resulting in a direct
   communication channel not involving the NAT.  If the two clients are
   behind different NATs, then the packets directed to their private



Ford, Srisuresh & Kegel                                        [Page 13]

Internet-Draft     P2P applications across middleboxes      October 2003


   addresses will fail to reach each other at all, but the clients will
   hopefully establish connectivity using their respective public
   addresses.  It is important that these packets be authenticated in
   some way, however, since in the case of different NATs it is entirely
   possible for A's messages directed at B's private address to reach
   some other, unrelated node on A's private network, or vice versa.

3.3.3. Peers separated by multiple NATs 

   In some topologies involving multiple NAT devices, it is not
   possible for two clients to establish an "optimal" P2P route between
   them without specific knowledge of the topology.  Consider for
   example the following situation.


                                Server S
                            18.181.0.31:1234
                                   |
                                   |
                                 NAT X
                         A-S 155.99.25.11:62000
                         B-S 155.99.25.11:62001
                                   |
                                   |
            +----------------------+----------------------+
            |                                             |
          NAT A                                         NAT B
    192.168.1.1:30000                             192.168.1.2:31000
            |                                             |
            |                                             |
         Client A                                      Client B
      10.0.0.1:1234                                 10.1.1.3:1234

   Suppose NAT X is a large industrial NAT deployed by an internet
   service provider (ISP) to multiplex many customers onto a few public
   IP addresses, and NATs A and B are small consumer NAT gateways
   deployed independently by two of the ISP's customers to multiplex
   their private home networks onto their respective ISP-provided IP
   addresses.  Only server S and NAT X have globally routable IP
   addresses; the "public" IP addresses used by NAT A and NAT B are
   actually private to the ISP's addressing realm, while client A's and
   B's addresses in turn are private to the addressing realms of NAT A
   and B, respectively.  Each client initiates an outgoing connection to
   server S as before, causing NATs A and B each to create a single
   public/private translation, and causing NAT X to establish a
   public/private translation for each session.

   Now suppose clients A and B attempt to establish a direct peer-to-



Ford, Srisuresh & Kegel                                        [Page 14]

Internet-Draft     P2P applications across middleboxes      October 2003


   peer UDP connection.  The optimal method would be for client A to
   send messages to client B's public address at NAT B,
   192.168.1.2:31000 in the ISP's addressing realm, and for client B to
   send messages to A's public address at NAT B, namely
   192.168.1.1:30000.  Unfortunately, A and B have no way to learn these
   addresses, because server S only sees the "global" public addresses
   of the clients, 155.99.25.11:62000 and 155.99.25.11:62001.  Even if A
   and B had some way to learn these addresses, there is still no
   guarantee that they would be usable because the address assignments
   in the ISP's private addressing realm might conflict with unrelated
   address assignments in the clients' private realms.  The clients
   therefore have no choice but to use their global public addresses as
   seen by S for their P2P communication, and rely on NAT X to provide
   loopback translation.

3.3.4. Consistent port bindings

   The hole punching technique has one main caveat: it works only if
   both NATs are cone NATs (or non-NAT firewalls), which maintain a
   consistent port binding between a given (private IP, private UDP)
   pair and a (public IP, public UDP) pair for as long as that UDP port
   is in use.  Assigning a new public port for each new session, as a
   symmetric NAT does, makes it impossible for a UDP application to
   reuse an already-established translation for communication with
   different external destinations.  Since cone NATs are the most
   widespread, the UDP hole punching technique is fairly broadly
   applicable; nevertheless a substantial fraction of deployed NATs are
   symmetric and do not support the technique.

3.4. UDP port number prediction

   A variant of the UDP hole punching technique discussed above exists
   that allows peer-to-peer UDP sessions to be created in the presence
   of some symmetric NATs.  This method is sometimes called the "N+1"
   technique [BIDIR] and is explored in detail by Takeda [SYM-STUN].
   The method works by analyzing the behavior of the NAT and attempting
   to predict the public port numbers it will assign to future sessions.
   Consider again the situation in which two clients, A and B, each
   behind a separate NAT, have each established UDP connections with a
   permanently addressable server S:

                                  Server S
                              18.181.0.31:1234
                                     |
                                     |
              +----------------------+----------------------+
              |                                             |
       Symmetric NAT A                               Symmetric NAT B



Ford, Srisuresh & Kegel                                        [Page 15]

Internet-Draft     P2P applications across middleboxes      October 2003


   A-S 155.99.25.11:62000                        B-S 138.76.29.7:31000
              |                                             |
              |                                             |
           Client A                                      Client B
        10.0.0.1:1234                                 10.1.1.3:1234

   NAT A has assigned its own UDP port 62000 to the communication
   session between A and S, and NAT B has assigned its port 31000 to the
   session between B and S.  By communicating through server S, A and B
   learn each other's public IP addresses and port numbers as observed
   by S.  Client A now starts sending UDP messages to port 31001 at
   address 138.76.29.7 (note the port number increment), and client B
   simultaneously starts sending messages to port 62001 at address
   155.99.25.11.  If NATs A and B assign port numbers to new sessions
   sequentially, and if not much time has passed since the A-S and B-S
   sessions were initiated, then a working bi-directional communication
   channel between A and B should result.  A's messages to B cause NAT A
   to open up a new session, to which NAT A will (hopefully) assign
   public port number 62001, because 62001 is next in sequence after the
   port number 62000 it previously assigned to the session between A and
   S.  Similarly, B's messages to A will cause NAT B to open a new
   session, to which it will (hopefully) assign port number 31001.  If
   both clients have correctly guessed the port numbers each NAT assigns
   to the new sessions, then a bi-directional UDP communication channel
   will have been established as shown below.

                                  Server S
                              18.181.0.31:1234
                                     |
                                     |
              +----------------------+----------------------+
              |                                             |
            NAT A                                         NAT B
   A-S 155.99.25.11:62000                        B-S 138.76.29.7:31000
   A-B 155.99.25.11:62001                        B-A 138.76.29.7:31001
              |                                             |
              |                                             |
           Client A                                      Client B
        10.0.0.1:1234                                 10.1.1.3:1234

   Obviously there are many things that can cause this trick to fail.
   If the predicted port number at either NAT already happens to be in
   use by an unrelated session, then the NAT will skip over that port
   number and the connection attempt will fail.  If either NAT sometimes
   or always chooses port numbers non-sequentially, then the trick will
   fail.  If a different client behind NAT A (or B respectively) opens
   up a new outgoing UDP connection to any external destination after A
   (B) establishes its connection with S but before sending its first



Ford, Srisuresh & Kegel                                        [Page 16]

Internet-Draft     P2P applications across middleboxes      October 2003


   message to B (A), then the unrelated client will inadvertently
   "steal" the desired port number.  This trick is therefore much less
   likely to work when either NAT involved is under load.

   Since in practice a P2P application implementing this trick would
   still need to work if the NATs are cone NATs, or if one is a cone NAT
   and the other is a symmetric NAT, the application would need to
   detect beforehand what kind of NAT is involved on either end [STUN]
   and modify its behavior accordingly, increasing the complexity of the
   algorithm and the general brittleness of the network.  Finally, port
   number prediction has no chance of working if either client is behind
   two or more levels of NAT and the NAT(s) closest to the client are
   symmetric.  For all of these reasons, it is NOT recommended that new
   applications implement this trick; it is mentioned here for
   historical and informational purposes.

3.5. Simultaneous TCP open

   There is a method that can be used in some cases to establish direct
   peer-to-peer TCP connections between a pair of nodes that are both
   behind existing middleboxes.  Most TCP sessions start with one
   endpoint sending a SYN packet, to which the other party responds with
   a SYN-ACK packet.  It is possible and legal, however, for two
   endpoints to start a TCP session by simultaneously sending each other
   SYN packets, to which each party subsequently responds with a
   separate ACK.  This procedure is known as a "simultaneous open."

   If a middlebox receives a TCP SYN packet from outside the private
   network attempting to initiate an incoming TCP connection, the
   middlebox will normally reject the connection attempt by either
   dropping the SYN packet or sending back a TCP RST (connection reset)
   packet.  If, however, the SYN packet arrives with source and
   destination addresses and port numbers that correspond to a TCP
   session that the middlebox believes is already active, then the
   middlebox will allow the packet to pass through.  In particular, if
   the middlebox has just recently seen and transmitted an outgoing SYN
   packet with the same addresses and port numbers, then it will
   consider the session active and allow the incoming SYN through.  If
   clients A and B can each correctly predict the public port number
   that its respective middlebox will assign the next outgoing TCP
   connection, and if each client initiates an outgoing TCP connection
   with the other client timed so that each client's outgoing SYN passes
   through its local middlebox before either SYN reaches the opposite
   middlebox, then a working peer-to-peer TCP connection will result.

   Unfortunately, this trick may be even more fragile and timing-
   sensitive than the UDP port number prediction trick described above.
   First, unless both middleboxes are simple firewalls or implement cone



Ford, Srisuresh & Kegel                                        [Page 17]

Internet-Draft     P2P applications across middleboxes      October 2003


   NAT behavior on their TCP traffic, all the same things can go wrong
   with each side's attempt to predict the public port numbers that the
   respective NATs will assign to the new sessions.  In addition, if
   either client's SYN arrives at the opposite middlebox too quickly,
   then the remote middlebox may reject the SYN with a RST packet,
   causing the local middlebox in turn to close the new session and make
   future SYN retransmission attempts using the same port numbers
   futile.  Finally, even though support for simultaneous open is
   technically a mandatory part of the TCP specification [TCP], it is
   not implemented correctly in some common operating systems.  For this
   reason, this trick is likewise mentioned here only for historical
   reasons; it is NOT recommended for use by applications.  Applications
   that require efficient, direct peer-to-peer communication over
   existing NATs should use UDP.


4. Application design guidelines

4.1. What works with P2P middleboxes

  Since UDP hole punching is the most efficient existing method of
  establishing direct peer-to-peer communication between two nodes
  that are both behind NATs, and it works with a wide variety of
  existing NATs, it is recommended that applications use this
  technique if efficient peer-to-peer communication is required, 
  but be prepared to fall back on simple relaying when direct
  communication cannot be established. 

4.2. Peers behind the same NAT

  In practice there may be a fairly large number of users who 
  have not two IP addresses, but three or more. In these cases,
  it is hard or impossible to tell which addresses to send to 
  the registration server. The applications should send all its
  addresses, in such a case.

4.3. Peer discovery
 
  Applications sending packets to several addresses to discover
  which one is best to use for a given peer may become a 
  significant source of 'space junk' littering the net, as the
  peer may have chosen to use routable addresses improperly as
  an internal LAN (e.g. 11.0.1.1, which is assigned to the DOD).
  Thus applications should exercise caution when sending the
  speculative hello packets.

4.4. TCP P2P applications




Ford, Srisuresh & Kegel                                        [Page 18]

Internet-Draft     P2P applications across middleboxes      October 2003


  The sockets API, used widely by application developers, is 
  designed with client-server applications in mind. In its 
  native form, only a single socket can bind to a TCP or UDP
  port. An application is not allowed to have multiple
  sockets binding to the same port (TCP or UDP) to initiate 
  simultaneous sessions with multiple external nodes (or) 
  use one socket to listen on the port and the other sockets
  to initiate outgoing sessions.

  The above single-socket-to-port bind restriction is not a
  problem however with UDP, because UDP is a datagram based
  protocol. UDP P2P application designers could use a single
  socket to send as well as receive datagrams from multiple
  peers using recvfrom() and sendto() calls.

  This is not the case with TCP. With TCP, each incoming and
  outgoing connection is to be associated with a separate
  socket. Linux sockets API addresses this problem with the
  aid of SO_REUSEADDR option. On FreeBSD and NetBSD, this 
  option does not seem to work; but, changing it to use the
  BSD-specific SetReuseAddress call (which Linux doesn't
  have and isn't in the Single Unix Standard) seems to work.
  Win32 API offers an equivalent SetReuseAddress call. 
  Using any of the above mentioned options, an application
  could use multiple sockets to reuse a TCP port. Say, open
  two TCP stream sockets bound to the same port, do a 
  listen() on one and a connect() from the other.
 
4.5. Use of midcom protocol

  If the applications know the middleboxes they would be 
  traversing and these middleboxes implement the midcom
  protocol, applications could use the midcom protocol to
  ease their way through the middleboxes. 

  For example, P2P applications require that NAT middleboxes
  preserve end-point port bindings. If midcom is supported on
  the middleboxes, P2P applications can exercise control over
  port binding (or address binding) parameters such as lifetime,
  maxidletime, and directionality so the applications can both
  connect to external peers as well as receive connections from
  external peers; and do not need to send periodic keep-alives to
  keep the port binding alive. When the application no longer needs
  the binding, the application could simply dismantle the binding, 
  also using the midcom protocol. 


5. NAT Design Guidelines



Ford, Srisuresh & Kegel                                        [Page 19]

Internet-Draft     P2P applications across middleboxes      October 2003



   This section discusses considerations in the design of network
   address translators, as they affect peer-to-peer applications.   

5.1. Deprecate the use of symmetric NATs

   Symmetric NATs gained popularity with client-server
   applications such as web browsers, which only need to initiate
   outgoing connections. However, in the recent times, P2P 
   applications such as Instant messaging and audio conferencing
   have been in wide use. Symmetric NATs do not support the
   concept of retaining endpoint identity and are not suitable
   for P2P applications. Deprecating symmetric NATs is 
   recommended to support P2P applications. 

   A P2P-middlebox must implement Cone NAT behavior for UDP
   traffic, allowing applications to establish robust P2P
   connectivity using the UDP hole punching technique.  
   Ideally, a P2P-middlebox should also allow applications to
   make P2P connections via both TCP and UDP.

5.2. Add incremental cone-NAT support to symmetric NAT devices

   One way for a symmetric NAT device to extend support to P2P
   applications would be to divide its assignable port
   namespace, reserving a portion of its ports for one-to-one
   sessions and a different set of ports for one-to-many
   sessions.

   Further, a NAT device may be explicitly configured with
   applications and hosts that need the P2P feature, so the
   NAT device can auto magically assign a P2P port from the
   right port block. 

5.3. Maintain consistent port bindings for UDP ports

   The primary and most important recommendation of this document for
   NAT designers is that the NAT maintain a consistent and stable
   port binding between a given (internal IP address, internal UDP
   port) pair and a corresponding (public IP address, public UDP 
   port) pair for as long as any active sessions exist using that
   port binding. The NAT may filter incoming traffic on a 
   per-session basis, by examining both the source and destination
   IP addresses and port numbers in each packet. When a node on the
   private network initiates connection to a new external
   destination, using the same source IP address and UDP port as an
   existing translated UDP session, the NAT should ensure that the
   new UDP session is given the same public IP address and UDP port



Ford, Srisuresh & Kegel                                        [Page 20]

Internet-Draft     P2P applications across middleboxes      October 2003


   numbers as the existing session.   

5.3.1. Preserving port numbers

   Some NATs, when establishing a new UDP session, attempt to assign the
   same public port number as the corresponding private port number, if
   that port number happens to be available.  For example, if client A
   at address 10.0.0.1 initiates an outgoing UDP session with a datagram
   from port number 1234, and the NAT's public port number 1234 happens
   to be available, then the NAT uses port number 1234 at the NAT's
   public IP address as the translated endpoint address for the session.
   This behavior might be beneficial to some legacy UDP applications
   that expect to communicate only using specific UDP port numbers, but
   it is not recommended that applications depend on this behavior since
   it is only possible for a NAT to preserve the port number if at most
   one node on the internal network is using that port number.

   In addition, a NAT should NOT try to preserve the port number in a
   new session if doing so would conflict with the goal of maintaining a
   consistent binding between public and private endpoint addresses.
   For example, suppose client A at internal port 1234 has established a
   session with external server S, and NAT A has assigned public port
   62000 to this session because port number 1234 on the NAT was not
   available at the time.  Now suppose port number 1234 on the NAT
   subsequently becomes available, and while the session between A and S
   is still active, client A initiates a new session from its same
   internal port (1234) to a different external node B.  In this case,
   because a port binding has already been established between client
   A's port 1234 and the NAT's public port 62000, this binding should be
   maintained and the new session should also use port 62000 as the
   public port corresponding to client A's port 1234.  The NAT should
   NOT assign public port 1234 to this new session just because port
   1234 has become available: that behavior would not be likely to
   benefit the application in any way since the application has already
   been operating with a translated port number, and it would break any
   attempts the application might make to establish peer-to-peer
   connections using the UDP hole punching technique.

5.4. Maintaining consistent port bindings for TCP ports

   For consistency with the behavior of UDP translation, cone NAT
   implementers should also maintain a consistent binding between
   private and public (IP address, TCP port number) pairs for TCP
   connections, in the same way as described above for UDP.  
   Maintaining TCP endpoint bindings consistently will increase
   the NAT's compatibility with P2P TCP applications that initiate
   multiple TCP connections from the same source port. 




Ford, Srisuresh & Kegel                                        [Page 21]

Internet-Draft     P2P applications across middleboxes      October 2003


5.5. Large timeout for P2P applications

   We recommend the middlebox implementers to use a minimum timeout
   of, say, 5 minutes (300 seconds) for P2P applications, i.e., 
   configure the middlebox with this idle-timeout for the port 
   bindings for the ports set aside for P2P use. Middlebox
   implementers are often tempted to use a shorter one, as they are
   accustomed to doing currently. But, short timeouts are
   problematic. Consider a P2P application that involved 16 peers.
   They will flood the network with keepalive packets every 10
   seconds to avoid NAT timeouts.  This is so because one might 
   send them 5 times as often as the middlebox's timeout just in 
   case the keepalives are dropped in the network.

5.6. Support loopback translation

   We strongly recommend that middlebox implementers support
   loopback translation, allowing hosts behind a middlebox to
   communicate with other hosts behind the same middlebox through
   their public, possibly translated endpoints. Support for
   loopback translation is particularly important in the case
   of large-capacity NATs that are likely to be deployed as the
   first level of a multi-level NAT scenario. As described in
   section 3.3.3, hosts behind the same first-level NAT but
   different second-level NATs have no way to communicate with
   each other by UDP hole punching, even if all the middleboxes
   preserve endpoint identities, unless the first-level NAT
   also supports loopback translation.


6. Security Considerations

   Following the recommendations in this document should not
   inherently create new security issues, for either the 
   applications or the middleboxes. Nevertheless, new security
   risks may be created if the techniques described here are
   not adhered to with sufficient care. This section describes
   security risks the applications could inadvertently create
   in attempting to support P2P communication across middleboxes,
   and implications for the security policies of P2P-friendly
   middleboxes.

6.1. IP address aliasing

   P2P applications must use appropriate authentication mechanisms
   to protect their P2P connections from accidental confusion with
   other P2P connections as well as from malicious connection
   hijacking or denial-of-service attacks. NAT-friendly P2P 



Ford, Srisuresh & Kegel                                        [Page 22]

Internet-Draft     P2P applications across middleboxes      October 2003


   applications effectively must interact with multiple distinct 
   IP address domains, but are not generally aware of the exact
   topology or administrative policies defining these address
   domains.  While attempting to establish P2P connections via
   UDP hole punching, applications send packets that may frequently
   arrive at an entirely different host than the intended one.

   For example, many consumer-level NAT devices provide DHCP
   services that are configured by default to hand out site-local
   IP addresses in a particular address range. Say, a particular
   consumer NAT device, by default, hands out IP addresses starting
   with 192.168.1.100. Most private home networks using that NAT 
   device will have a host with that IP address, and many of these
   networks will probably have a host at address 192.168.1.101 as
   well. If host A at address 192.168.1.101 on one private network
   attempts to establish a connection by UDP hole punching with 
   host B at 192.168.1.100 on a different private network, then as
   part of this process host A will send discovery packets to 
   address 192.168.1.100 on its local network, and host B will send
   discovery packets to address 192.168.1.101 on its network. Clearly,
   these discovery packets will not reach the intended machine since 
   the two hosts are on different private networks, but they are very
   likely to reach SOME machine on these respective networks at the
   standard UDP port numbers used by this application, potentially
   causing confusion. especially if the application is also running
   on those other machines and does not properly authenticate its
   messages.

   This risk due to aliasing is therefore present even without a
   malicious attacker. If one endpoint, say host A, is actually
   malicious, then without proper authentication the attacker could
   cause host B to connect and interact in unintended ways with
   another host on its private network having the same IP address
   as the attacker's (purported) private address. Since the two
   endpoint hosts A and B presumably discovered each other through
   a public server S, and neither S nor B has any means to verify
   A's reported private address, all P2P applications must assume
   that any IP address they find to be suspect until they successfully
   establish authenticated two-way communication.

6.2. Denial-of-service attacks

   P2P applications and the public servers that support them must
   protect themselves against denial-of-service attacks, and ensure
   that they cannot be used by an attacker to mount denial-of-service
   attacks against other targets. To protect themselves, P2P 
   applications and servers must avoid taking any action requiring
   significant local processing or storage resources until



Ford, Srisuresh & Kegel                                        [Page 23]

Internet-Draft     P2P applications across middleboxes      October 2003


   authenticated two-way communication is established. To avoid being
   used as a tool for denial-of-service attacks, P2P applications and
   servers must minimize the amount and rate of traffic they send to
   any newly-discovered IP address until after authenticated two-way
   communication is established with the intended target.

   For example, P2P applications that register with a public rendezvous
   server can claim to have any private IP address, or perhaps multiple
   IP addresses. A well-connected host or group of hosts that can
   collectively attract a substantial volume of P2P connection attempts
   (e.g., by offering to serve popular content) could mount a
   denial-of-service attack on a target host C simply by including C's
   IP address in their own list of IP addresses they register with the
   rendezvous server. There is no way the rendezvous server can verify
   the IP addresses, since they could well be legitimate private
   network addresses useful to other hosts for establishing
   network-local communication. The P2P application protocol must
   therefore be designed to size- and rate-limit traffic to unverified
   IP addresses in order to avoid the potential damage such a 
   concentration effect could cause.

6.3. Man-in-the-middle attacks

   Any network device on the path between a P2P client and a
   rendezvous server can mount a variety of man-in-the-middle
   attacks by pretending to be a NAT.  For example, suppose 
   host A attempts to register with rendezvous server S, but a 
   network-snooping attacker is able to observe this registration
   request. The attacker could then flood server S with requests
   that are identical to the client's original request except with
   a modified source IP address, such as the IP address of the
   attacker itself.  If the attacker can convince the server to
   register the client using the attacker's IP address, then the
   attacker can make itself an active component on the path of all
   future traffic from the server AND other P2P hosts to the
   original client, even if the attacker was originally only able
   to snoop the path from the client to the server.

   The client cannot protect itself from this attack by
   authenticating its source IP address to the rendezvous server,
   because in order to be NAT-friendly the application MUST allow
   intervening NATs to change the source address silently.  This
   appears to be an inherent security weakness of the NAT paradigm.
   The only defense against such an attack is for the client to
   authenticate and potentially encrypt the actual content of its
   communication using appropriate higher-level identities, so that
   the interposed attacker is not able to take advantage of its
   position.  Even if all application-level communication is



Ford, Srisuresh & Kegel                                        [Page 24]

Internet-Draft     P2P applications across middleboxes      October 2003


   authenticated and encrypted, however, this attack could still be
   used as a traffic analysis tool for observing who the client is
   communicating with.

6.4. Impact on middlebox security

   Designing middleboxes to preserve endpoint identities does not
   weaken the security provided by the middlebox. For example, a 
   Port-Restricted Cone NAT is inherently no more "promiscuous" 
   than a Symmetric NAT in its policies for allowing either
   incoming or outgoing traffic to pass through the middlebox.
   As long as outgoing UDP sessions are enabled and the middlebox
   maintains consistent binding between internal and external
   UDP ports, the middlebox will filter out any incoming UDP packets
   that do not match the active sessions initiated from within the
   enclave. Filtering incoming traffic aggressively while maintaining
   consistent port bindings thus allows a middlebox to be
   "peer-to-peer friendly" without compromising the principle of
   rejecting unsolicited incoming traffic.

   Maintaining consistent port binding could arguably increase the
   predictability of traffic emerging from the middlebox, by revealing
   the relationships between different UDP sessions and hence about
   the behavior of applications running within the enclave. This
   predictability could conceivably be useful to an attacker in
   exploiting other network or application level vulnerabilities.
   If the security requirements of a particular deployment scenario
   are so critical that such subtle information channels are of
   concern, however, then the middlebox almost certainly should not be
   configured to allow unrestricted outgoing UDP traffic in the
   first place. Such a middlebox should only allow communication
   originating from specific applications at specific ports, or
   via tightly-controlled application-level gateways.  In this
   situation there is no hope of generic, transparent peer-to-peer
   connectivity across the middlebox (or transparent client/server
   connectivity for that matter); the middlebox must either
   implement appropriate application-specific behavior or disallow
   communication entirely.

7. Acknowledgments

   The authors wish to thank Henrik, Dave, and Christian Huitema
   for their valuable feedback.

8. References

8.1. Normative references




Ford, Srisuresh & Kegel                                        [Page 25]

Internet-Draft     P2P applications across middleboxes      October 2003


[BIDIR]    Peer-to-Peer Working Group, NAT/Firewall Working Committee,
           "Bidirectional Peer-to-Peer Communication with Interposing
           Firewalls and NATs", August 2001.
           http://www.peer-to-peerwg.org/tech/nat/

[KEGEL]    Dan Kegel, "NAT and Peer-to-Peer Networking", July 1999.
           http://www.alumni.caltech.edu/~dank/peer-nat.html

[MIDCOM]   P. Srisuresh, J. Kuthan, J. Rosenberg, A. Molitor, and
           A. Rayhan, "Middlebox communication architecture and
           framework", RFC 3303, August 2002.

[NAT-APPL] D. Senie, "Network Address Translator (NAT)-Friendly
           Application Design Guidelines", RFC 3235, January 2002.

[NAT-PROT] M. Holdrege and P. Srisuresh, "Protocol Complications
           with the IP Network Address Translator", RFC 3027,
           January 2001.

[NAT-PT]   G. Tsirtsis and P. Srisuresh, "Network Address
           Translation - Protocol Translation (NAT-PT)", RFC 2766,
           February 2000.

[NAT-TERM] P. Srisuresh and M. Holdrege, "IP Network Address
           Translator (NAT) Terminology and Considerations", RFC
           2663, August 1999.

[NAT-TRAD] P. Srisuresh and K. Egevang, "Traditional IP Network
           Address Translator (Traditional NAT)", RFC 3022,
           January 2001.

[STUN]     J. Rosenberg, J. Weinberger, C. Huitema, and R. Mahy,
           "STUN - Simple Traversal of User Datagram Protocol (UDP)
           Through Network Address Translators (NATs)", RFC 3489,
           March 2003.

8.2. Informational references

[ICE]      J. Rosenberg, "Interactive Connectivity Establishment (ICE):
           A Methodology for Network Address Translator (NAT) Traversal
           for the Session Initiation Protocol (SIP)",
           draft-rosenberg-sipping-ice-00 (Work In Progress),
           February 2003.

[RSIP]     M. Borella, J. Lo, D. Grabelsky, and G. Montenegro,
           "Realm Specific IP: Framework", RFC 3102, October 2001.

[SOCKS]    M. Leech, M. Ganis, Y. Lee, R. Kuris, D. Koblas, and



Ford, Srisuresh & Kegel                                        [Page 26]

Internet-Draft     P2P applications across middleboxes      October 2003


           L. Jones, "SOCKS Protocol Version 5", RFC 1928, March 1996.

[SYM-STUN] Y. Takeda, "Symmetric NAT Traversal using STUN",
           draft-takeda-symmetric-nat-traversal-00.txt (Work In
           Progress), June 2003.

[TCP]      "Transmission Control Protocol", RFC 793, September 1981.

[TEREDO]   C. Huitema, "Teredo: Tunneling IPv6 over UDP through NATs",
           draft-ietf-ngtrans-shipworm-08.txt (Work In Progress),
           September 2002.

[TURN]     J. Rosenberg, J. Weinberger, R. Mahy, and C. Huitema,
           "Traversal Using Relay NAT (TURN)",
           draft-rosenberg-midcom-turn-01 (Work In Progress),
           March 2003.

[UPNP]     UPnP Forum, "Internet Gateway Device (IGD) Standardized
           Device Control Protocol V 1.0", November 2001.
           http://www.upnp.org/standardizeddcps/igd.asp

9. Author's Address

   Bryan Ford
   Laboratory for Computer Science
   Massachusetts Institute of Technology
   77 Massachusetts Ave.
   Cambridge, MA 02139
   Phone: (617) 253-5261
   E-mail: baford@mit.edu
   Web: http://www.brynosaurus.com/


   Pyda Srisuresh
   Caymas Systems, Inc.
   11799-A North McDowell Blvd.
   Petaluma, CA 94954
   Phone: (707) 283-5063
   E-mail: srisuresh@yahoo.com

   Dan Kegel
   Kegel.com
   901 S. Sycamore Ave.
   Los Angeles, CA 90036
   Phone: 323 931-6717    
   Email: dank@kegel.com
   Web: http://www.kegel.com/




Ford, Srisuresh & Kegel                                        [Page 27]

Internet-Draft     P2P applications across middleboxes      October 2003



Full Copyright Statement

   Copyright (C) The Internet Society (2003).  All Rights Reserved.

   This document and translations of it may be copied and furnished to
   others, and derivative works that comment on or otherwise explain it
   or assist in its implementation may be prepared, copied, published
   and distributed, in whole or in part, without restriction of any
   kind, provided that the above copyright notice and this paragraph are
   included on all such copies and derivative works.  However, this
   document itself may not be modified in any way, such as by removing
   the copyright notice or references to the Internet Society or other
   Internet organizations, except as needed for the purpose of
   developing Internet standards in which case the procedures for
   copyrights defined in the Internet Standards process must be
   followed, or as required to translate it into languages other than
   English.

   The limited permissions granted above are perpetual and will not be
   revoked by the Internet Society or its successors or assigns.

   This document and the information contained herein is provided on an
   "AS IS" basis and THE INTERNET SOCIETY AND THE INTERNET ENGINEERING
   TASK FORCE DISCLAIMS ALL WARRANTIES, EXPRESS OR IMPLIED, INCLUDING
   BUT NOT LIMITED TO ANY WARRANTY THAT THE USE OF THE INFORMATION
   HEREIN WILL NOT INFRINGE ANY RIGHTS OR ANY IMPLIED WARRANTIES OF
   MERCHANTABILITY OR FITNESS FOR A PARTICULAR PURPOSE.























Ford, Srisuresh & Kegel                                        [Page 28]

