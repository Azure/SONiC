# DHCPv6 Relay Agent

# High Level Design Document

# Table of Contents
* [Scope](#scope)
* [Definition](#definition)
* [Overview](#overview)
* [DHCPv6](#dhcpv6)
    - [Why DHCPv6 relay agent](#why-dhcpv6-relay-agent)
    - [DHCPv6 Relay messages](#dhcpv6-relay-messages)
    - [DHCPv6 Packet Forwarding](#dhcpv6-packet-forwarding)
    - [Relay Agent Behavior](#relay-agent-behavior)
* [Requirements](#requirements)
* [Topology](#topology)
* [Design](#design)
  - [CLI and Usage](#cli-and-usage)
  - [DHCPRELAY counter](#dhcprelay-counter)
  - [CONFIG DB schema](#config-db-schema)
  - [YANG Model schema](#yang-model-schema)
  - [Option 79 for client link-layer address](#option-79-for-client-link-layer-address)
  - [Option for Dual ToR](#option-for-dual-tor)
  - [Feature table](#feature-table)
  - [RADV modification](#radv-modification)
  - [CoPP manager](#copp-manager)
  - [Source IP](#source-ip)
* [Performance](#performance)
* [Testing](#testing)

# Scope

This document describes high level design details of SONiC's DHCPv6 relay agent.

# Definition

DHCP: Dynamic Host Configuration Protocol

DUID: DHCP Unique Identifier (Each DHCPv6 client and server has a DUID. DHCPv6 servers use it to identify clients for the selection of configuration parameters with clients. DHCPv6 clients use it to identify a server in messages where a server needs to be identified.)

# Overview

SONiC currently supports DHCPv4 Relay via the use of open source ISC DHCP package. However, DHCPv6 specification does not define a way to communicate client link-layer address to the DHCP server where DHCP server is not connected to the same network link as DHCP client. DHCPv6 requires all clients prepare and send a DUID as the client identifier in all DHCPv6 message exchanges. However, these methods do not provide a simple way to extract a client's link-layer address. Providing option 79 in DHCPv6 Relay-Forward messages will help carry the client link-layer address explicitly. The server needs to know the client's MAC address to allow DHCP Reservation, which provides pre-set IP address to specific client based on its physical MAC address. The DHCPv6 relay agent is able to read the source MAC address of DHCPv6 messages that it received from client, and encapsulate these messages within a DHCPv6 Relay-Forward message, inserting the client MAC address as option 79 in the Relay-Forward header sent to the server.

With heterogenous DHCP client implementation across the network, DUIDs could not resolve IP resource tracking issue. The two types of DUIDs, DUID-LL and DUID-LLT used to facilitate resource tracking both have link layer addresses embedded. The current client link-layer address option in DHCPv6 specification limits the DHCPv6 Relay to first hop to provide the client link layer address, which are relay agents that are connected to the same link as the client, and that limits SONiC DHCPv6 deployment to ToR/MoR switches for early stages. One solution would be to provide SONiC's own DHCPv6 relay agent feature.

# DHCPv6

DHCP is a network protocol used to assign IP addresses and provide configuration for devices to communicate on a network.

- DHCP server: receives clients' requests and replies to them
- DHCP client: send configuration requests to the server
- DHCP relay agent: forwards DHCP packets between clients and servers that do not reside on a shared physical subnet

1. Solicit: DHCPv6 client sends a SOLICIT message to locate DHCPv6 servers to the All\_DHCP\_Relay\_Agents\_and\_Servers multicast address.
2. Advertise: DHCPv6 server sends an ADVERTISE message to indicate that it is available for DHCP service, in response to the SOLICIT message
3. Request, Renew, Rebind: DHCPv6 client sends a REQUEST message to request configuration parameters(IP address or delegated prefixes) from the DHCPv6 server
4. Reply: DHCPv6 server sends a REPLY message containing assigned addresses and configuration parameters in response to a CONFIRM message that confirms or denies that the addresses assigned to the client are appropriate to the link to which the client is connected. REPLY message acknowledges receipt of a RELEASE or DECLINE message.

![image](https://user-images.githubusercontent.com/42761586/117859723-3adcc800-b244-11eb-9dd4-dbde609185a1.png)

# Why DHCPv6 relay agent

Generally, the DHCPv6 clients get IP by multicasting the DHCP packets in the LAN, and the server will respond to clients' request. In this case, it would be necessary to keep the DHCPv6 server and clients in the same LAN. DHCPv6 relay agent is used to transmit different subnets' DHCPv6 packets, so that all subnets can share DHCPv6 server, and DHCPv6 server is not required on every LAN.

A DHCPv6 client sends most messages using a reserved, link-scoped multicast destination address so that the client need not be configured with the address or addresses of DHCP servers.

![image](https://user-images.githubusercontent.com/42761586/117859791-4b8d3e00-b244-11eb-88f1-59bfd5baa1d6.png)

In a Relay-forward message, the received message is relayed to the next relay agent or server; in a Relay-reply message, the message is to be copied and relayed to the relay agent or client whose address is in the peer-address field of the Relay-reply message.

# DHCPv6 Relay messages

**Relay-Forward Message**

hop-count: Number of relay agents that have relayed this message.

link-address: A global or site-local address that will be used by the server to identify the link on which the client is located.

peer-address: The address of the client or relay agent from which the message to be relayed was received.

options: include a &quot;Relay Message option&quot; and other options included by relay agent

**Relay-Reply Message**

hop-count: Copied from the Relay-forward message

link-address: Copied from the Relay-forward message

peer-address: Copied from the Relay-forward message

options: include a &quot;Relay Message option&quot;

# DHCPv6 Packet Forwarding

The DHCPv6 relay agent on the routing switch forwards DHCPv6 client packets to all DHCPv6 servers that are configured in the table administrated for each VLAN.

A DHCPv6 client locates a DHCPv6 server using a reserved, link-scoped multicast address.

The packets are forwarded to configurable IPv6 helpers addresses.

# Relay Agent Behavior

1. DHCPv6 client sends multicast SOLICIT message to ALL\_DHCP\_Relay\_Agents\_and\_Servers. Message received by relay agent.
  - Relay agent at default uses ALL\_DHCP\_Servers multicast address. It may be configured to use unicast addresses, or other addresses selected by the network administrator.
2. DHCPv6 relay agent constructs a Relay-forward message copies the source address from header of the IP datagram to the peer-address field of the Relay-forward message and received DHCP message into Relay Message option, and relays this Relay-forward message to the DHCPv6 server in RELAY\_FORWARD message
  - DHCPv6 relay agent also places a global or site-scope address with a prefix assigned to the link on which the client should be assigned an address in the link-address field. (will be used by server to determine the link from which the client should be assigned an address)
  - Hop-count in Relay-forward message is set to 0.
  - If Relay Agent were to relay a message from a relay agent, it checks if the hop-count in the message is greater than or equal to HOP\_COUNT\_LIMIT, and discard if so. Else, hop\_count is incremented by 1.
3. DHCPv6 server received the SOLICIT message, refers to the Relay Agent IP and select an IP address to allocate to the DHCPv6 client.
4. The DHCPv6 server constructs a RELAY-REPLY message that embeds the ADVERTISE messages, and sends it to the DHCPv6 relay agent.
5. DHCPv6 relay agent extracts ADVERTISE message from RELAY-REPLY message and forwards it to the client.
6. DHCPv6 client receives ADVERTISE message and relays a REQUEST message to the DHCPv6 relay agent.
7. DHCPv6 relay agent constructs REQUEST message into a RELAY-FORWARD message, and relays to DHCPv6 server.
8. DHCPv6 server receives the REQUEST message, and sends a REPLY message to the relay agent. Server creates a Relay-reply message that includes a Relay Message option containing the the REPLY message and sends it to the relay agent.
9. DHCPv6 relay agent extracts message and relays the message to the address contained in the peer-address field of the Relay-reply message.
10. DHCPv6 client receives the REPLY message that contains the desired IP address.

![image](https://user-images.githubusercontent.com/42761586/117859842-5fd13b00-b244-11eb-9297-c2674d128dd9.png)

# Requirements

- Configured and running DHCPv6 client and server
- Connectivity between the relay agent and DHCPv6 server
- Configure one or more IP helper addresses for specified VLANs to forward DHCPv6 requests to DHCPv6 servers on other subnets.
- Client UDP port:546
- Server and Relay Agent UDP port: 547

# Topology

![image](https://user-images.githubusercontent.com/42761586/117859824-58aa2d00-b244-11eb-991f-7ad8759f6612.png)

# Design

# CLI and Usage

-show dhcp6relay_counters

-sonic-clear dhcprelay_counters

-enable/Disable option 79

-enable/Disable use-loopback-address (for dual tor)

-show/config ip helpers

# DHCPRELAY counter

Keeps count of all relay Messages:
SOLICIT
ADVERTISE
REQUEST
CONFIRM
RENEW
REBIND
REPLY
RELEASE
DECLINE
RELAY-FORWARD
RELAY-REPLY

# CONFIG DB schema

<pre>
DHCP|intf-i|dhcpv6\_servers: [&quot;dhcp-server-0&quot;, &quot;dhcp-server-1&quot;, ...., &quot;dhcp-server-n-1&quot;]

DHCP|intf-i|dhcpv6\_options: [&quot;79&quot;]
</pre>

# YANG Model schema

sonic-dhcpv6-relay.yang
<pre>
module DHCP  
    container DHCP {  	
        list VLAN_LIST {
    		key name;
   		    leaf name {
    			type string;
  		    }
   		    leaf dhcpv6_servers {
     		    type inet6:ip-address;
  		    }
		    leaf options {
			    type uint16;
		    }
        }
    }
}
</pre>

# Option 79 for client link-layer address

Option 79 should be enabled by default and can be disabled through command line.

# Option for Dual ToR

Relayed DHCPv6 packet from ToR may have the response routed to the peer ToR that has the link as standby. Since the originating client is not active on this ToR, the peer ToR won't be able to relay the response. Peer ToR will not receive the packets as the originating client is not active on this ToR. Instead of using Vlan SVI IP address, relay agent source address needs to be set to listen on the loopback address. When DHCP server responses are received by relay agent on the peer ToR, DHCP relay agent would then forward the packet to the peer ToR using its loopback IP interface.

# Feature table

Adding to existing DHCP relay container. No new feature table added

# RADV modification

Router sends an Router Advertisement message that indicates to nodes on the network that they should use DHCPv6 as their method of dynamic address configuration. RA message contains A, M, O, L bits. The routers can use two flags in RA messages to tell the attached end hosts which method to use:

- Managed-Config-Flag(M-bit) tells the end-host to use DHCPv6 exclusively;
- Other-Config-Flag(O-bit) tells the end-host to use SLAAC to get IPv6 address and DHCPv6 to get other parameters such as DNS server address.
- Absence of both flags tells the end-host to use only SLAAC.

# CoPP manager

Control Plane Policing manager is currently configured to only trap DHCPv6 packets when DHCPv6 is enabled.

# Source IP

VLAN SVI IP

Configurable option to use loopback address for dual ToR

# Performance

SONiC DHCP relay agent is currently not relaying many DHCP requests. Frequency arrival rate of DHCP packets is not high so it is not going to affect performance.

# Testing

Use counter to check if DHCP messages are forwarded successfully using DHCPv6 relay agent

Check validity of DHCP message content

Validate control plane behavior when DHCPv6 is enabled/disabled

Configuration validation
