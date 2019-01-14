---
title: DNS Reverse IP AMT Discovery
abbrev: DRIAD
docname: draft-jholland-mboned-driad-amt-discovery-03
date: 2019-01-13
category: std

ipr: trust200902
area: Ops
workgroup: Mboned
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: J. Holland
    name: Jake Holland
    org: Akamai Technologies, Inc.
    street: 150 Broadway
    city: Cambridge, MA 02144
    country: United States of America
    email: jakeholland.net@gmail.com

normative:
  RFC1034:
  RFC1035:
  RFC2119:
  RFC2181:
  RFC3376:
  RFC3596:
  RFC3597:
  RFC3810:
  RFC4604:
  RFC4607:
  RFC6763:
  RFC7450:
  RFC8085:
  RFC8174:

informative:
  RFC3550:
  RFC4025:
  RFC4033:
  RFC5110:
  RFC5507:
  RFC6726:
  RFC6895:
  RFC7359:
  RFC7761:
  RFC8126:
  RFC8313:
  RFC8499:

--- abstract

This document defines a new DNS resource record (RR) used to advertise
addresses for Automatic Multicast Tunneling (AMT) relays capable of receiving
multicast traffic from the owner of the RR. The new AMTRELAY RR makes
possible a source-specific method for AMT gateways to discover appropriate
AMT relays, in order to ingest traffic for source-specific multicast channels
into multicast-capable receiving networks when no multicast connectivity
is directly available between the sending and receiving networks.

--- middle

#Introduction        {#intro}

This document defines DNS Reverse IP AMT Discovery (DRIAD), a mechanism for
AMT gateways to discover AMT relays which are capable of forwarding multicast
traffic from a known source IP address.

AMT (Automatic Multicast Tunneling) is defined in {{RFC7450}}, and provides
a method to transport multicast traffic over a unicast tunnel, in order to
traverse non-multicast-capable network segments.

Section 4.1.5 of {{RFC7450}} explains that relay selection might need to
depend on the source of the multicast traffic, since a relay must be able
to receive multicast traffic from the desired source in order to forward it.

That section suggests DNS-based queries as a possible solution.  DRIAD
is a DNS-based solution, as suggested there.  This solution also addresses
the relay discovery issues in the "Disadvantages" lists in Section 3.3 of
{{RFC8313}} and Section 3.4 of {{RFC8313}}.

The goal for DRIAD is to enable multicast connectivity between separate
multicast-enabled networks when neither the sending nor the receiving network
is connected to a multicast-enabled backbone, without pre-configuring any
peering arrangement between the networks.

##Background

The reader is assumed to be familiar with the basic DNS concepts
described in {{RFC1034}}, {{RFC1035}}, and the subsequent documents that
update them, particularly {{RFC2181}}.

The reader is also assumed to be familiar with the concepts and terminology
regarding source-specific multicast as described in {{RFC4607}} and the
use of IGMPv3 {{RFC3376}} and MLDv2 {{RFC3810}} for group management of
source-specific multicast channels, as described in {{RFC4604}}.

The reader should also be familiar with AMT, particularly the terminology
listed in Section 3.2 of {{RFC7450}} and Section 3.3 of {{RFC7450}}.

##Terminology

###Relays and Gateways

When reading this document, it's especially helpful to recall that once
an AMT tunnel is established, the relay receives native multicast traffic
and sends unicast tunnel-encapsulated traffic to the gateway, and the gateway
receives the tunnel-encapsulated packets, decapsulates them, and forwards
them as native multicast packets, as illustrated in {{figtunnel}}.

~~~

 Multicast  +-----------+  Unicast  +-------------+  Multicast
>---------> | AMT relay | >=======> | AMT gateway | >--------->
            +-----------+           +-------------+
~~~
{: #figtunnel title="AMT Tunnel Illustration"}

###Definitions

   Term | Definition
   ----:|:----------
  (S,G) | A source-specific multicast channel, as described in {{RFC4607}}. A pair of IP addresses with a source host IP and destination group IP.
downstream | Further from the source of traffic.
   FQDN | Fully Qualified Domain Name, as described in {{RFC8499}}
gateway | An AMT gateway, as described in {{RFC7450}}
  relay | An AMT relay, as described in {{RFC7450}}
    RPF | Reverse Path Forwarding, as described in {{RFC5110}}
     RR | A DNS Resource Record, as described in {{RFC1034}}
 RRType | A DNS Resource Record Type, as described in {{RFC1034}}
    SSM | Source-specific multicast, as described in {{RFC4607}}
upstream | Closer to the source of traffic.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
{{RFC2119}} and {{RFC8174}} when, and only when, they appear in all
capitals, as shown here.

#Relay Discovery Operation

##Overview

The AMTRELAY resource record (RR) defined in this document is used to
publish the IP address or domain name of an AMT relay that can receive,
encapsulate, and forward multicast traffic from a particular sender.

The sender is the owner of the RR, and configures the RR so that it contains
the address or domain name of an AMT relay that can receive multicast IP
traffic from that sender.

This enables AMT gateways in remote networks to discover an AMT relay that
is capable of forwarding traffic from the sender.  This in turn enables those
AMT gateways to receive the multicast traffic tunneled over a unicast AMT
tunnel from those relays, and then to pass the multicast packets into
networks or applications that are using the gateway to subscribe to traffic
from that sender.

This mechanism only works for source-specific multicast (SSM) channels.  The
source address of the (S,G) is reversed and used as an index into one of the
reverse mapping trees (in-addr.arpa for IPv4, as described in Section 3.5 of
{{RFC1035}}, or ip6.arpa for IPv6, as described in Section 2.5 of {{RFC3596}}).

Some detailed example use cases are provided in {{exampledeployments}}, and
other applicable example topologies appear in Section 3.3 of {{RFC8313}},
Section 3.4 of {{RFC8313}}, and Section 3.5 of {{RFC8313}}.

##Signaling and Discovery

This section describes a typical example of the end-to-end process for
signaling a receiver's join of a SSM channel that relies on an AMTRELAY
RR.

The example in {{figmessaging}} contains 2 multicast-enabled
networks that are both connected to the internet with non-multicast-capable
links, and which have no direct association with each other.

A content provider operates a sender, which is a source of multicast traffic
inside a multicast-capable network.

An end user who is a customer of the content provider has a multicast-capable
internet service provider, which operates a receiving network that uses an
AMT gateway.  The AMT gateway is DRIAD-capable.

The content provider provides the user with a receiving application that
tries to subscribe to at least one (S,G).  This receiving application could
for example be a file transfer system using FLUTE {{RFC6726}} or a live
video stream using RTP {{RFC3550}}, or any other application that might
subscribe to a SSM channel.

~~~
                 +---------------+
                 |    Sender     |
  |    |         | 198.51.100.15 |
  |    |         +---------------+
  |Data|                 |
  |Flow|      Multicast  |
 \|    |/      Network   |
  \    /                 |        5: Propagate RPF for Join(S,G)
   \  /          +---------------+
    \/           |   AMT Relay   |
                 | 203.0.113.15  |
                 +---------------+
                         |        4: Gateway connects to Relay,
                                     sends Join(S,G) over tunnel
                         |
                Unicast
                 Tunnel  |

     ^                   |        3: --> DNS Query: type=AMTRELAY,
     |                           /         15.100.51.198.in-addr.arpa.
     |                   |      /    <-- Response:
 Join/Leave       +-------------+          AMTRELAY=203.0.113.15
  Signals         | AMT gateway |
     |            +-------------+
     |                   |        2: Propagate RPF for Join(S,G)
     |        Multicast  |
               Network   |
                         |        1: Join(S=198.51.100.15, G)
                  +-------------+
                  |   Receiver  |
                  |  (end user) |
                  +-------------+
~~~
{: #figmessaging title="DRIAD Messaging"}

In this simple example, the sender IP is 198.51.100.15, and the relay IP is
203.0.113.15.

The content provider has previously configured the DNS zone that
contains the domain name "15.100.51.198.in-addr.arpa.", which is the
reverse lookup domain name for his sender.  The zone file contains an
AMTRELAY RR with the Relay's IP address.  (See {{rpformat}} for
details about the AMTRELAY RR format and semantics.)

The sequence of events depicted in {{figmessaging}} is as follows:

 1. The end user starts the app, which issues a join to the (S,G):
    (198.51.100.15, 232.252.0.2).
 2. The join propagates with RPF through the multicast-enabled network
    with PIM {{RFC7761}} or another multicast routing mechanism, until the AMT
    gateway receives a signal to join the (S,G).
 3. The AMT gateway performs a reverse DNS lookup for the AMTRELAY RRType,
    by sending an AMTRELAY RRType query for the FQDN
    "15.100.51.198.in-addr.arpa.", using the reverse IP domain name for the
    sender's source IP address (the S from the (S,G)), as described in
    Section 3.5 of {{RFC1035}}.

    The DNS resolver for the AMT gateway uses ordinary DNS recursive
    resolution until it has the authoritative result that the content
    provider configured, which informs the AMT gateway that the relay address
    is 203.0.113.15.
 4. The AMT gateway performs AMT handshakes with the AMT relay as described
    in Section 4 of {{RFC7450}}, then forwards a Membership report to the
    relay indicating subscription to the (S,G).
 5. The relay propagates the join through its network toward the sender,
    then forwards the appropriate AMT-encapsulated traffic to the
    gateway, which decapsulates and forwards it as native multicast through
    its downstream network to the end user.

##Optimal Relay Selection

The reverse source IP DNS query of an AMTRELAY RR is a good way for a gateway
to discover a relay that is known to the sender.

However, it is NOT necessarily a good way to discover the best relay for that
gateway to use, because the RR IP will only provide information about relays
known to the source.

If there is an upstream relay in a network that is more local to the gateway
and able to receive and forward multicast traffic from the sender, that relay
is better for the gateway to use, since more of the network path uses native
multicast, allowing more chances for packet replication.  But since that relay
is not known to the sender, it won't be advertised in the sender's reverse IP
DNS record.  An example network with this scenario is outlined in {{exoffice}}.

It's only appropriate for an AMT gateway to discover an AMT relay by querying
an AMTRELAY RR owned by a sender when all of these conditions are met:

 1. The gateway needs to propagate a join of an (S,G) over AMT, because in
    the gateway's network, no RPF next hop toward the source can
    propagate a native multicast join of the (S,G); and
 2. The gateway is not already connected to a relay that forwards multicast
    traffic from the source of the (S,G); and
 3. The gateway is not configured to use a particular IP address for AMT
    discovery, or a relay discovered with that IP is not able to forward
    traffic from the source of the (S,G); and
 4. The gateway is not able to find an upstream AMT relay with DNS-SD
    {{RFC6763}}, using "_amt._udp" as the Service section of the queries, or a
    relay discovered this way is not able to forward traffic from the source of
    the (S,G)

When the above conditions are met, the gateway has no path within its local
network that can receive multicast traffic from the source IP of the (S,G).

In this situation, the best way to find a relay that can forward the
required traffic is to use information that comes from the operator of the
sender.  When the sender has configured the AMTRELAY RR defined in this
document, gateways can use the DRIAD mechanism defined in this document to
discover the relay information provided by the sender.

##DNS Configuration

Often an AMT gateway will only have access to the source and group IP addresses
of the desired traffic, and will not know any other name for the source of the
traffic. Because of this, typically the best way of looking up AMTRELAY RRs
will be by using the source IP address as an index into one of the reverse
mapping trees (in-addr.arpa for IPv4, as described in Section 3.5 of
{{RFC1035}}, or ip6.arpa for IPv6, as described in Section 2.5 of {{RFC3596}}).

Therefore, it is RECOMMENDED that AMTRELAY RRs be added to reverse IP
zones as appropriate. AMTRELAY records MAY also appear in other zones, but
the primary intended use case requires a reverse IP mapping for the source
from an (S,G) in order to be useful to most AMT gateways.

\<TBD\>

Please can a DNS expert review the following paragraph and perhaps help
construct an equivalent and more clear explanation?

I borrowed the language from https://tools.ietf.org/html/rfc4025#section-1.2,
but I'm not actually sure what "the fashion usual for PTR records" means,
precisely...

PTR gives a domain name, and then we do what, an A/AAAA record lookup, and then
a AMTRELAY lookup on the final name that has a valid A/AAAA after any
CNAME/DNAME chain?
- jake 2019-01-13

\</TBD\>

When the reverse IP mapping has no AMTRELAY RR but does have a PTR record,
the lookup is done in the fashion usual for PTR records.  The IP
address' octets (IPv4) or nibbles (IPv6) are reversed and looked up
with the appropriate suffix.  Any CNAMEs or DNAMEs found MUST be
followed, and finally the AMTRELAY RR is queried with the resulting domain
name.

See {{rrdef}} and {{rpformat}} for a detailed explanation of the contents
for a DNS Zone file.

#Example Deployments {#exampledeployments}

##Example Receiving Networks {#exrx}

###Tier 3 ISP {#exrxisp}

One example of a receiving network is an ISP that offers multicast ingest
services to its subscribers, illustrated in {{figrxisp}}.

In the example network below, subscribers can join (S,G)s with MLDv2 or
IGMPv3 as described in {{RFC4604}}, and the AMT gateway in this
ISP can receive and forward multicast traffic from one of the example sending
networks in {{extx}} by discovering the appropriate AMT relays with a DNS
lookup for the AMTRELAY RR with the reverse IP of the source in the (S,G).

~~~
                    Internet
                 ^            ^      Multicast-enabled
                 |            |      Receiving Network
          +------|------------|-------------------------+
          |      |            |                         |
          |  +--------+   +--------+    +=========+     |
          |  | Border |---| Border |    |   AMT   |     |
          |  | Router |   | Router |    | gateway |     |
          |  +--------+   +--------+    +=========+     |
          |      |            |              |          |
          |      +-----+------+-----------+--+          |
          |            |                  |             |
          |      +-------------+    +-------------+     |
          |      | Agg Routers | .. | Agg Routers |     |
          |      +-------------+    +-------------+     |
          |            /     \ \     /         \        |
          | +---------------+         +---------------+ |
          | |Access Systems | ....... |Access Systems | |
          | |(CMTS/OLT/etc.)|         |(CMTS/OLT/etc.)| |
          | +---------------+         +---------------+ |
          |        |                        |           |
          +--------|------------------------|-----------+
                   |                        |
             +---+-+-+---+---+        +---+-+-+---+---+
             |   |   |   |   |        |   |   |   |   |
            /-\ /-\ /-\ /-\ /-\      /-\ /-\ /-\ /-\ /-\
            |_| |_| |_| |_| |_|      |_| |_| |_| |_| |_|

                           Subscribers
~~~
{: #figrxisp title="Receiving ISP Example"}

###Small Office {#exoffice}

Another example receiving network is a small branch office that regularly
accesses some multicast content, illustrated in {{figrxofficenm}}.

This office has desktop devices that need to receive some multicast traffic,
so an AMT gateway runs on a LAN with these devices, to pull traffic in
through a non-multicast next-hop.

The office also hosts some mobile devices that have AMT gateway instances
embedded inside apps, in order to receive multicast traffic over their
non-multicast wireless LAN.  (Note that the "Legacy Router" is a
simplification that's meant to describe a variety of possible conditions--
for example it could be a device providing a split-tunnel VPN as described
in {{RFC7359}}, deliberately excluding multicast traffic for a VPN
tunnel, rather than a device which is incapable of multicast forwarding.)

~~~
                 Internet
              (non-multicast)
                     ^
                     |                  Office Network
          +----------|----------------------------------+
          |          |                                  |
          |    +---------------+ (Wifi)   Mobile apps   |
          |    | Modem+ | Wifi | - - - -  w/ embedded   |
          |    | Router |  AP  |          AMT gateways  |
          |    +---------------+                        |
          |          |                                  |
          |          |                                  |
          |     +----------------+                      |
          |     | Legacy Router  |                      |
          |     |   (unicast)    |                      |
          |     +----------------+                      |
          |      /        |      \                      |
          |     /         |       \                     |
          | +--------+ +--------+ +--------+=========+  |
          | | Phones | | ConfRm | | Desks  |   AMT   |  |
          | | subnet | | subnet | | subnet | gateway |  |
          | +--------+ +--------+ +--------+=========+  |
          |                                             |
          +---------------------------------------------+
~~~
{: #figrxofficenm title="Small Office (no multicast up)" :}

By adding an AMT relay to this office network as in {{figrxoffice}}, it's
possible to make use of multicast services from the example multicast-capable
ISP in {{exrxisp}}.

~~~
           Multicast-capable ISP
                     ^
                     |                  Office Network
          +----------|----------------------------------+
          |          |                                  |
          |    +---------------+ (Wifi)   Mobile apps   |
          |    | Modem+ | Wifi | - - - -  w/ embedded   |
          |    | Router |  AP  |          AMT gateways  |
          |    +---------------+                        |
          |          |               +=======+          |
          |          +---Wired LAN---|  AMT  |          |
          |          |               | relay |          |
          |     +----------------+   +=======+          |
          |     | Legacy Router  |                      |
          |     |   (unicast)    |                      |
          |     +----------------+                      |
          |      /        |      \                      |
          |     /         |       \                     |
          | +--------+ +--------+ +--------+=========+  |
          | | Phones | | ConfRm | | Desks  |   AMT   |  |
          | | subnet | | subnet | | subnet | gateway |  |
          | +--------+ +--------+ +--------+=========+  |
          |                                             |
          +---------------------------------------------+
~~~
{: #figrxoffice title="Small Office Example"}

When multicast-capable networks are chained like this, with a network like
the one in {{figrxoffice}} receiving internet services from a
multicast-capable network like the one in {{figrxisp}}, it's important for
AMT gateways to reach the more local AMT relay, in order to avoid
accidentally tunneling multicast traffic from a more distant AMT relay with
unicast, and failing to utilize the multicast transport capabilities of the
network in {{figrxisp}}.

For this reason, it's RECOMMENDED that AMT gateways by default perform
service discovery using DNS Service Discovery (DNS-SD) {{RFC6763}} for
_amt._udp.\<domain\> (with \<domain\> chosen as described in Section 11 of
{{RFC6763}}) and use the AMT relays discovered that way in preference to
AMT relays discoverable via the mechanism defined in this document (DRIAD).

It's also RECOMMENDED that when the well-known anycast IP addresses defined
in Section 7 of {{RFC7450}} are suitable for discovering an AMT relay that
can forward traffic from the source, that a DNS record with the AMTRELAY
RRType be published for those IP addresses along with any other appropriate
AMTRELAY RRs to indicate the best relative precedences for
receiving the source traffic.

Accordingly, AMT gateways SHOULD by default discover the most-preferred relay
first by DNS-SD, then by DRIAD as described in this document (in precedence
order, as described in {{rrdef-precedence}}), then with the anycast
addresses defined in Section 7 of {{RFC7450}} (namely: 192.52.193.1 and
2001:3::1) if those IPs weren't listed in the AMTRELAY RRs.
This default behavior MAY be overridden by administrative configuration where
other behavior is more appropriate for the gateway within its network.

The discovery and connection process for multiple relays MAY operate in
parallel, but when forwarding multicast group membership reports with new
joins from an AMT gateway, membership reports SHOULD be forwarded to the
most-preferred relays first, falling back to less preferred relays only after
failing to receive traffic for an appropriate timeout, and only after reporting
a leave to any more- preferred connected relays that have failed to subscribe
to the traffic.

It is RECOMMENDED that the default timeout be no less than 3 seconds, but the
value MAY be overridden by administrative configuration, where known groups or
channels need a different timeout for successful application performance.

##Example Sending Networks {#extx}

###Sender-controlled Relays

When a sender network is also operating AMT relays to distribute multicast
traffic, as in {{figtxrelay}}, each address could appear as an AMTRELAY RR
for the reverse IP of the sender, or one or more domain names could appear
in AMTRELAY RRs, and the AMT relay addresses can be discovered by finding
an A or AAAA record from those domain names.

~~~
                                      Sender Network
                +-----------------------------------+
                |                                   |
                | +--------+   +=======+  +=======+ |
                | | Sender |   |  AMT  |  |  AMT  | |
                | +--------+   | relay |  | relay | |
                |     |        +=======+  +=======+ |
                |     |            |          |     |
                |     +-----+------+----------+     |
                |           |                       |
                +-----------|-----------------------+
                            v
                         Internet
                      (non-multicast)
~~~
{: #figtxrelay title="Small Office Example"}

###Provider-controlled Relays

When an ISP offers a service to transmit outbound multicast traffic through
a forwarding network, it might also offer AMT relays in order to reach
receivers without multicast connectivity to the forwarding network, as in
{{figtxisp}}. In this case it's RECOMMENDED that the ISP also provide a domain
name for the AMT relays for use with the discovery process defined in this
document.

When the sender wishes to use the relays provided by the ISP for
forwarding multicast traffic, an AMTRELAY RR should be configured to use
the domain name provided by the ISP, to allow for address reassignment of the
relays without forcing the sender to reconfigure the corresponding AMTRELAY
RRs.

~~~
                  +--------+
                  | Sender |
                  +---+----+        Multicast-enabled
                      |              Sending Network
          +-----------|-------------------------------+
          |           v                               |
          |    +------------+     +=======+ +=======+ |
          |    | Agg Router |     |  AMT  | |  AMT  | |
          |    +------------+     | relay | | relay | |
          |           |           +=======+ +=======+ |
          |           |               |         |     |
          |     +-----+------+--------+---------+     |
          |     |            |                        |
          | +--------+   +--------+                   |
          | | Border |---| Border |                   |
          | | Router |   | Router |                   |
          | +--------+   +--------+                   |
          +-----|------------|------------------------+
                |            |
                v            v
                   Internet
                (non-multicast)
~~~
{: #figtxisp title="Sending ISP Example"}

#AMTRELAY Resource Record Definition {#rrdef}

##AMTRELAY RRType

The AMTRELAY RRType has the mnemonic AMTRELAY and type code TBD1 (decimal).

##AMTRELAY RData Format

The AMTRELAY RData consists of a 8-bit precedence field, a 1-bit
"Discovery Optional" field, a 7-bit type field, and a variable
length relay field.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   precedence  |D|    type     |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
~                            relay                              ~
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

###RData Format - Precedence {#rrdef-precedence}

This is an 8-bit precedence for this record.  It is interpreted in
the same way as the PREFERENCE field described in Section 3.3.9 of
{{RFC1035}}.

Relays listed in AMTRELAY records with a lower value for precedence are to be
attempted first.

Where there is a tie in precedence, the default choice of relay MUST be
non-deterministic, to support load balancing. The AMT gateway operator
MAY override this default choice with explicit configuration when it's
necessary for administrative purposes.

For example, one network might prefer to tunnel IPv6 multicast traffic over
IPv6 AMT and IPv4 multicast traffic over IPv4 AMT to avoid routeability
problems in IPv6 from affecting IPv4 traffic and vice versa, while another
network might prefer to tunnel both kinds of traffic over IPv6 to reduce the
IPv4 space used by its AMT gateways. In this example scenario or other cases
where there is an administrative preference that requires explicit
configuration, a receiving network MAY make systematically different
precedence choices among records with the same precedence value.

###RData Format - Discovery Optional (D-bit) {#rrdef-dbit}

The D bit is a "Discovery Optional" flag.

If the D bit is set to 0, a gateway using this RR MUST perform AMT relay
discovery as described in Section 4.2.1.1 of {{RFC7450}}, rather than directly
sending an AMT request message to the relay.

That is, the gateway MUST receive an AMT relay advertisement message (Section
5.1.2 of {{RFC7450}}) for an address before sending an AMT request message
(Section 5.1.3 of {{RFC7450}}) to that address. Before receiving the relay
advertisement message, this record has only indicated that the address can be
used for AMT relay discovery, not for a request message.  This is necessary for
devices that are not fully functional AMT relays, but rather load balancers or
brokers, as mentioned in Section 4.2.1.1 of {{RFC7450}}.

If the D bit is set to 1, the gateway MAY send an AMT request message directly
to the discovered relay address without first sending an AMT discovery message.

This bit should be set according to advice from the AMT relay operator. The
D bit MUST be set to zero when no information is available from the AMT relay
operator about its suitability.

###RData Format - Type {#rtype}

The type field indicates the format of the information that
is stored in the relay field.

The following values are defined:

 * type = 0:
   The relay field is empty (0 bytes).
 * type = 1:
   The relay field contains a 4-octet IPv4 address.
 * type = 2:
   The relay field contains a 16-octet IPv6 address.
 * type = 3:
   The relay field contains a wire-encoded domain name. The wire-encoded
   format is self-describing, so the length is implicit. The domain name
   MUST NOT be compressed.  (See Section 3.3 of {{RFC1035}} and Section 4 of
   {{RFC3597}}.)

###RData Format - Relay {#rdformat}

The relay field is the address or domain name of the AMT relay. It is
formatted according to the type field.

When the type field is 0, the length of the relay field is 0, and it
indicates that no AMT relay should be used for multicast traffic from this
source.

When the type field is 1, the length of the relay field is 4 octets, and a
32-bit IPv4 address is present. This is an IPv4 address as described in
Section 3.4.1 of {{RFC1035}}. This is a 32-bit number in network byte
order.

When the type field is 2, the length of the relay field is 16 octets, and
a 128-bit IPv6 address is present. This is an IPv6 address as described in
Section 2.2 of {{RFC3596}}. This is a 128-bit number in network byte order.

When the type field is 3, the relay field is a normal wire-encoded domain
name, as described in Section 3.3 of {{RFC1035}}. Compression MUST NOT be
used, for the reasons given in Section 4 of {{RFC3597}}.

##AMTRELAY Record Presentation Format {#rpformat}

###Representation of AMTRELAY RRs

AMTRELAY RRs may appear in a zone data master file. The precedence, D-bit,
relay type, and relay fields are REQUIRED.

If the relay type field is 0, the relay field MUST be ".".

The presentation for the record is as follows:

      IN AMTRELAY precedence D-bit type relay

###Examples

In a DNS resolver that understands the AMTRELAY type, the zone might
contain a set of entries like this:

~~~
    $ORIGIN 100.51.198.in-addr.arpa.
    10     IN AMTRELAY  10 0 1 203.0.113.15
    10     IN AMTRELAY  10 0 2 2001:DB8::15
    10     IN AMTRELAY 128 1 3 amtrelays.example.com.
~~~

This configuration advertises an IPv4 discovery address, an IPv6
discovery address, and a domain name for AMT relays which can receive
traffic from the source 198.51.100.10.  The IPv4 and IPv6 addresses
are configured with a D-bit of 0 (meaning discovery is mandatory, as
described in {{rrdef-dbit}}), and a precedence 10 (meaning they're
preferred ahead of the last entry, which has precedence 128).

For zone files in resolvers that don't support the AMTRELAY RRType
natively, it's possible to use the format for unknown RR types, as
described in {{RFC3597}}.  This approach would replace the AMTRELAY
entries in the example above with the entries below:

  \[To be removed (TBD): replace 65280 with the IANA-assigned value
   TBD1, here and in Appendix B. \]

~~~
    10   IN TYPE65280  \# (
           6  ; length
           0a ; precedence=10
           01 ; D=0, relay type=1, an IPv4 address
           cb00710f ) ; 203.0.113.15
    10   IN TYPE65280  \# (
           18 ; length
           0a ; precedence=10
           02 ; D=0, relay type=2, an IPv6 address
           20010db800000000000000000000000f ) ; 2001:db8::15
    10   IN TYPE65280  \# (
           24 ; length
           80 ; precedence=128
           83 ; D=1, relay type=3, a wire-encoded domain name
           616d7472656c6179732e6578616d706c652e636f6d2e ) ; domain name
~~~

See {{extranslate}} for more details.

#IANA Considerations {#iana}

This document updates the IANA Registry for DNS Resource Record Types
by assigning type TBD1 to the AMTRELAY record.

This document creates a new registry named "AMTRELAY Resource Record Parameters", with a sub-registry for the "Relay Type Field".  The initial values in the sub-registry are:

~~~
          +-------+---------------------------------------+
          | Value | Description                           |
          +-------+---------------------------------------+
          |   0   | No relay is present.                  |
          |   1   | A 4-byte IPv4 address is present      |
          |   2   | A 16-byte IPv6 address is present     |
          |   3   | A wire-encoded domain name is present |
          | 4-255 | Unassigned                            |
          +-------+---------------------------------------+
~~~

Values 0, 1, 2, and 3 are further explained in {{rtype}} and {{rdformat}}.
Relay type numbers 4 through 255 can be assigned with a policy of
Specification Required (as described in {{RFC8126}}).

#Security Considerations

\[ TBD: these 3 are just the first few most obvious issues, with just
    sketches of the problem. Explain better, and look for trickier
    issues. \]

##Record-spoofing

If AMT is used to ingest multicast traffic, providing a false AMTRELAY record
to a gateway using it for discovery can result in Denial of Service, or
artificial multicast traffic from a source under an attacker's control.

Therefore, it is important to ensure that the AMTRELAY record is authentic,
with DNSSEC {{RFC4033}} or other operational safeguards that can provide
assurance of the authenticity of the record contents.

##Local Override

The local relays, while important for overall network performance, can't be
secured by DNSSEC.

##Congestion

Multicast traffic, particularly interdomain multicast traffic, carries some
congestion risks, as described in Section 4 of {{RFC8085}}. Network operators
are advised to take precautions including monitoring of application traffic
behavior, traffic authentication, and rate-limiting of multicast traffic, in
order to ensure network health.

#Acknowledgements

This specification was inspired by the previous work of Doug Nortz,
Robert Sayko, David Segelstein, and Percy Tarapore, presented in
the MBONED working group at IETF 93.

Thanks also to Jeff Goldsmith and Lenny Giuliano for helpful reviews
and feedback.

--- back

#New RRType Request Form

This is the template for requesting a new RRType recommended in
Appendix A of {{RFC6895}}.

~~~
A. Submission Date:

B.1 Submission Type:
 [x] New RRTYPE  [ ] Modification to RRTYPE
B.2 Kind of RR:
 [x] Data RR     [ ] Meta-RR

C. Contact Information for submitter (will be publicly posted):
 Name:  Jake Holland
 Email Address: jakeholland.net@gmail.com
 International telephone number: +1-626-486-3706
 Other contact handles: jholland@akamai.com

D. Motivation for the new RRTYPE application.
 It provides a bootstrap so AMT (RFC 7450) gateways can discover
 an AMT relay that can receive multicast traffic from a specific source,
 in order to signal multicast group membership and receive multicast
 traffic over a unicast tunnel using AMT.

E. Description of the proposed RR type.
   This description can be provided in-line in the template, as an
   attachment, or with a publicly available URL.
 Please see draft-jholland-mboned-driad-amt-discovery.

F. What existing RRTYPE or RRTYPEs come closest to filling that need
   and why are they unsatisfactory?
 Some similar concepts appear in IPSECKEY, as described in
 Section 1.2 of [RFC4025]. The IPSECKEY RRType is unsatisfactory
 because it refers to IPSec Keys instead of to AMT relays, but
 the motivating considerations for using reverse IP and for
 providing a precedence are similar--an AMT gateway often
 has access to a source address for a multicast (S,G), but does
 not have access to a relay address that can receive multicast
 traffic from the source, without administrative configuration.

 Defining a format for a TXT record could serve the need for AMT
 relay discovery semantics, but Section 5 of [RFC5507] provides a
 compelling argument for requesting a new RRType instead.

G. What mnemonic is requested for the new RRTYPE (optional)?
 AMTRELAY

H. Does the requested RRTYPE make use of any existing IANA registry
   or require the creation of a new IANA subregistry in DNS
   Parameters?
 Yes, IANA is requested to create a subregistry named "AMT Relay
 Type Field" in a "AMTRELAY Resource Record Parameters" registry.
 The field values are defined in Section 4.2.3 and Section 4.2.4,
 and a summary table is given in Section 5.

I. Does the proposal require/expect any changes in DNS
   servers/resolvers that prevent the new type from being processed
   as an unknown RRTYPE (see RFC3597)?
 No.

J. Comments:
 It may be worth noting that the gateway type field from Section 2.3 of
 [RFC4025] and Section 2.5 of [RFC4025] is very similar to the
 Relay Type field in this request.  I tentatively assume that trying to
 re-use that sub-registry is a worse idea than duplicating it, but I'll
 invite others to consider the question and voice an opinion, in case
 there is a different consensus.
    https://www.ietf.org/assignments/
        ipseckey-rr-parameters/ipseckey-rr-parameters.xml
~~~

#Unknown RRType construction {#extranslate}

In a DNS resolver that understands the AMTRELAY type, the zone file might
contain this line:

      IN AMTRELAY 128 0 3 amtrelays.example.com.

In order to translate this example to appear as an unknown RRType
as defined in {{RFC3597}}, one could run the following program:

    <CODE BEGINS>
      $ cat translate.py
      #!/usr/bin/python3
      import sys
      name=sys.argv[1]
      print(len(name))
      print(''.join('%02x'%ord(x) for x in name))

      $ ./translate.py amtrelays.example.com.
      22
      616d7472656c6179732e6578616d706c652e636f6d2e
    <CODE ENDS>

The length and the hex string for the domain name "amtrelays.example.com." are
the outputs of this program, yielding a length of 22 and the above hex string.

22 is the length of the domain name, so to this we add 2 (1 for
the precedence field and 1 for the combined D-bit and relay type fields) to
get the full length of the RData.

This results in a zone file entry like this:

      IN TYPE65280  \# ( 24 ; length
              80 ; precedence = 128
              03 ; D-bit=0, relay type=3 (wire-encoded domain name)
              616d7472656c6179732e6578616d706c652e636f6d2e ) ; domain name


