---
title: Glossary
description: Definitions for 23 key terms used throughout 44Mesh documentation including BGP, ZeroTier, and networking concepts.
topics: [glossary, definitions, terminology, bgp, zerotier, networking]
related:
  - docs/overview/what-is-44mesh.md
  - docs/overview/architecture.md
---

# Glossary

---

**AMPRNet**
The Amateur Radio IP network, administered by the ARDC (Amateur Radio Digital Communications) foundation. Manages the 44/8 IPv4 block for use by amateur radio operators.

---

**AS (Autonomous System)**
A collection of IP networks and routers under the control of a single organization that presents a common routing policy to the Internet. Each AS has a unique AS number (ASN).

---

**ASN (Autonomous System Number)**
A unique identifier assigned to an Autonomous System by a Regional Internet Registry (RIR). Used in BGP to identify routing participants.

---

**BGP (Border Gateway Protocol)**
The routing protocol used to exchange routing information between autonomous systems on the Internet. Defined in RFC 4271.

---

**BIRD2**
An open-source Internet routing daemon that implements BGP, OSPF, RIP, and other routing protocols. Used in 44Mesh as the BGP daemon on the border router.

---

**Border Router**
The network device or host that connects the 44Mesh overlay to the public Internet. Runs BGP to announce the mesh's IP prefix and acts as the ingress node for all inbound traffic.

---

**Docker Compose**
A tool for defining and running multi-container Docker applications using a YAML configuration file.

---

**eBGP (External BGP)**
A BGP session between routers in different autonomous systems. Contrasted with iBGP (Internal BGP), which runs within a single AS.

---

**Ingress Node**
In 44Mesh, the border router node through which all inbound Internet traffic enters the mesh. Configured via the `ingressNodeV4` ZeroTier network parameter. All mesh nodes route Internet-bound traffic through the ingress node.

---

**ingressNodeV4**
A custom field added to the ZeroTier network configuration by the AlterMundi fork. Specifies the ZeroTier-assigned IP of the border router. When nodes join the network, the fork automatically installs source routing rules pointing to this address.

---

**IXP (Internet Exchange Point)**
A physical infrastructure where multiple networks interconnect and exchange traffic directly. IXPs can be used as BGP peering points in lieu of or in addition to a transit ISP.

---

**Mesh Node**
Any host that runs the ZeroTier client, joins the 44Mesh overlay, and receives a public IP from the controller's address pool.

---

**NAT (Network Address Translation)**
A method of mapping one IP address space to another. Commonly used by ISPs and home routers to allow many devices to share a single public IP. 44Mesh eliminates NAT for mesh nodes by giving each node a real public IP.

---

**Observation**
A single sensor reading in the 44Mesh data model. Includes a timestamp, location, sensor type, value, unit, and confidence score.

---

**Overlay Network**
A virtual network built on top of an existing physical or logical network. ZeroTier is an overlay that runs over the existing Internet.

---

**Policy Routing**
A Linux networking feature that allows different routing tables to be selected based on packet attributes (source address, destination address, etc.). Used in 44Mesh to implement symmetric routing.

---

**Prefix**
A range of IP addresses expressed in CIDR notation (e.g., `44.30.127.0/24`). BGP operates by exchanging prefixes.

---

**RIR (Regional Internet Registry)**
Organizations that manage the allocation and registration of IP addresses and ASNs within a geographic region. The five RIRs are ARIN (North America), LACNIC (Latin America), RIPE NCC (Europe/Middle East), APNIC (Asia-Pacific), and AFRINIC (Africa).

---

**Routing Table**
A data structure used by the OS or a routing daemon to determine where to forward packets. Linux supports multiple routing tables, identified by number.

---

**Source Routing**
Routing decisions based on the source IP address of a packet. Used in 44Mesh to ensure packets sourced from public mesh IPs are forwarded through the border router.

---

**ZeroTier**
An open-source overlay networking platform that creates virtual Ethernet networks over the Internet. Provides NAT traversal, peer discovery, and encrypted transport.

---

**ZeroTier Controller**
The network management component in ZeroTier. Controls membership, authorizes nodes, assigns IP addresses, and distributes network configuration. In 44Mesh, the controller runs on the border router.

---

**ZeroTier Fork (AlterMundi)**
A custom build of ZeroTierOne (`github.com/AlterMundi/ZeroTierOne`, branch `feature/ingress-node`) that adds the `ingressNodeV4` network configuration field and automatic source routing installation on node authorization.
