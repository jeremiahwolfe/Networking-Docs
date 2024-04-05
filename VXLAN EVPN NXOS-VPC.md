# VXLAN EVPN with vPC on NX-OS & Integrating Non-Fabric Routes into Fabric Overlay

| Version | Edited Y/M/D | Author | Notes |
| --- | --- | --- | --- |
| 1.0 | 2024/04/05 | Jeremiah Wolfe | Initial Release |

---

## Description

This document is intended to guide one through configurations of VXLAN eVPN with vPC on cisco Nexus devices. While some explanation will be provided, this is not a teaching tool, but rather a quick reference.

For best experience view this file in a Markdown viewer. In addition to dedicated editor/viewers, most IDEs include markdown support. Additionally, web browser extensions are available.

---

## Table of Contents

1. VXLAN EVPN with VPC Overview  
[1.1 Overview of Challenges and Solutions](#vpc-overview)  
[1.2 Topology Diagram](#vpc-topology)
2. VXLAN EVPN with vPC Configuration Examples  
[2.1 Configure vPC Peer-Link Port-Channel](#peer-link-port-channel-config)  
[2.2 Configure vPC Keep-alive Interface](#keep-alive-interface-config)  
[2.3 Configure vPC Domain](#vpc-domain-config)  
[2.4 Bring vPC Peers into Consistency](#vpc-consistency)  
[2.5 Configure vPC Peer-Link](#vpc-peer-link-config)  
[2.6 Configure NVE Peer-Link VLAN](#nve-peer-link-vlan-config)  
[2.7 (OPTIONAL & RECOMMENDED) Configure L3 Routing Across NVE Peer-Link Vlan](#l3-routing-nve-peer-link-config)  
[2.8 (OPTIONAL) Configure Remaining Port-Channels](#remaining-port-channels-config)  
3. Integrating Non-Fabric Routing into Fabric Overlay  
[3.1 Configure VTEP to Non-Fabric Router Connection](#vtep-to-non-fabric-config)  
[3.2 Configure Overlay OSPF Between VTEP and Router](#ospf-vtp-router-config)  
[3.3 Extend Overlay OSPF to vPC Peer](#ospf-vtp-peers-config)  
[3.4 Configure Redistribution into OSPF](#ospf-vtp-peers-config)  

---
---

## VXLAN EVPN with vPC Overview

<!-- TOC --><a name="vpc-overview"></a>

### Overview of Challenges and Solutions

### The problem is that NVE source interface (i.e. Lo0) is the default BGP next-hop for advertised routes.

+ In a vPC setup both peers advertise the same EVPN type-2 and type-5 routes to the BGP route reflectors. As such, when trying to reach a client, the sending VTEP will have 2 choices of where to send the packet.
+ With all other attributes being equal, the VTEP with the lowest bgp Router-ID will when the path selection.
+ Implies that one vPC peer is always preferred for dual attached hosts.
+ The result is that egress traffic from VPC members is load balanced, but return traffic is polarized toward teh lower RID peer.

### The solution is to use VPC Anycast VTEP address.

+ VPC peers share duplicate IP addresses on the NVE source interface.
  + **Both vPC Peers:** interface lo0; ip address 10.1.1.100/32 secondary
+ BGP next-hop is set to the secondary IP for locally originated routes. 

### In vPC, dual-attached (and orphaned) endpoints are advertised as reachable via the vPC anycast VTEP.

+ For orphaned endpoints, a backup path is set up via the vPC peer link if the reachability to the spines is lost.
+ An underlay routing adjacency across teh vPC peer link is needed to address failures.
+ If multicast is enabled on the underlay network, then PIM should be enabled over teh peer link.

### In vPC, all ARP and MAC entries are synced between vPC peers. So both peers advertise the same set of routes over BGP EVPN.

+ Each vPC switch ignores the BGP EVPN advertisements received from its vPC peer because these advertisements are identified by a Site-Of-Origin (SoO) extended community attribute.

### Sometimes IP prefixes may be advertised by only oen fo the two vPC peers. (e.g. router hanging off of only one peer)

+ The other VPC switch is unaware of the IP prefixes and won't accept the BGP updates due to SoO, potentially black holing some traffic.
+ Layer 3 routing adjacency on a Per-Overlay-VRF basis is required to ensure routing exchange.
  + i.e. If router is hanging off Peer1 then a Layer 3 connection must be extended to Peer2 so that a routing adjacency can be formed.

---

<!-- TOC --><a name="vpc-topology"></a>

### 1.2 VXLAN EVPN with VPC Topology:

![Basic VXLAN EVPN Topology](https://jeremiahwolfe.github.io/Networking-Docs/VXLAN_EVPN_NXOS/VXLAN-EVPN-VPC.png)

---
---

## VXLAN EVPN with vPC Configuration Examples

>### ⚠️ NOTE ⚠️
>
>+ The following configuration is a continuation of the [VXLAN EVPN Basic setup document](https://jeremiahwolfe.github.io/Networking-Docs/VXLAN_EVPN_NXOS-BASIC.md).
>+ Here we are ADDING vPC to an existing VXLAN EVPN configuration.
>+ In practice it would be better to configure vPC first.
>
> &nbsp;

<!-- TOC --><a name="peer-link-port-channel-config"></a>

### 2.1 Configure vPC Peer-Link Port-Channel

```text
2.1 Configure Peer-Link Port-Channel Config on NX-01 & NX-02
============================================================

NX-01 & NX-02:
feature lacp

default int e1/1-2

interface Ethernet1/1-2
  switchport
  switchport mode trunk
  channel-group 100 mode active
  no shutdown
```

```text
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!  VERIFICATION COMMAND(S)  !!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

show port-channel summary
```

---

<!-- TOC --><a name="keep-alive-interface-config"></a>

### 2.2 Configure vPC Keep-alive Interface

```text
2.1a Configure Keep-alive interface on NX-01
============================================

NX-01:
interface loopback10
  description Used for VPC Keep-alive
  ip address 1.0.0.1/32
  ip router ospf UNDERLAY area 0.0.0.0
```

```text
2.1b Configure Keep-alive interface on NX-02
============================================

NX-02:
interface loopback10
  description Used for VPC Keep-alive
  ip address 2.0.0.2/32
  ip router ospf UNDERLAY area 0.0.0.0
```

```text
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!  VERIFICATION COMMAND(S)  !!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

NX-01# ping 2.0.0.2 source 1.0.0.1
NX-02# ping 1.0.0.1 source-int lo10
```

---

<!-- TOC --><a name="vpc-domain-config"></a>

### 2.3 Configure vPC Domain

```text
2.3a Configure vPC Domain on NX-01
==================================

NX-01:
feature vpc

vpc domain 100
  peer-switch
  role priority 1
  peer-keepalive destination 2.0.0.2 source 1.0.0.1 vrf default
  peer-gateway
```

```text
2.3b Configure vPC Domain on NX-02
==================================

NX-02:
feature vpc
  
vpc domain 100
  peer-switch
  role priority 2
  peer-keepalive destination 1.0.0.1 source 2.0.0.2 vrf default
  peer-gateway
```

```text
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!  VERIFICATION COMMAND(S)  !!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

show vpc peer-keepalive
```

---

<!-- TOC --><a name="vpc-consistency"></a>

### 2.4 Bring vPC Peers into Consistency

> ⚠️ NOTE: Because NX-01 and NX-02 are vPC peers, they must have matching VLAN/SVI/VNI/NVE configs.

```text
2.4a Configure IDENTICAL Anycast IP as SECONDARY on NX-01 & NX-02
=================================================================

NX-01 & NX-02:
interface lo0
 ip address 10.0.0.100/32 secondary
```

```text
2.4b Confirm matching VLAN, VNI and NVE Configurations on NX-01 & NX-02
============+==========================================================

NX-01 & NX-02:
vlan 1,10,20,40,500
vlan 10
  vn-segment 100010
vlan 20
  vn-segment 100020
vlan 40
  vn-segment 100040
vlan 500
  vn-segment 50000

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 50000 associate-vrf
  member vni 100010 mcast-group 239.0.0.10
  member vni 100020 mcast-group 239.0.0.20
  member vni 100040 mcast-group 239.0.0.40
```

```text
2.4c Confirm matching SVI Configurations on NX-01 & NX-02
============++===========================================

NX-01 & NX-02:
interface Vlan10
  no shutdown
  mtu 9150
  vrf member WOLFECO
  no ip redirects
  ip address 192.168.10.254/24 tag 67696
  no ipv6 redirects
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway

interface Vlan20
  no shutdown
  mtu 9150
  vrf member WOLFECO
  no ip redirects
  ip address 192.168.20.254/24 tag 67696
  no ipv6 redirects
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway

interface Vlan40
  no shutdown
  mtu 9150
  vrf member WOLFECO
  no ip redirects
  ip address 192.168.40.254/24 tag 67696
  no ipv6 redirects
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway
```

```text
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!  VERIFICATION COMMAND(S)  !!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

show vxlan
show nve interface
```

---

<!-- TOC --><a name="vpc-peer-link-config"></a>

### 2.5 Configure vPC Peer-Link

```text
2.5 Configure vPC Peer-Link on NX-01 & NX-02
============================================

NX-01 & NX-02:
interface port-channel100
  vpc peer-link
```

```text
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!  VERIFICATION COMMAND(S)  !!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

show vpc
show vpc consistency-parameters global
```

---

<!-- TOC --><a name="nve-peer-link-vlan-config"></a>

### 2.6 Configure NVE Peer-Link VLAN

```text
2.6 Configure NVE Peer-Link VLAN on NX-01 & NX-02
=================================================

NX-01 & NX-02:
vlan 1234

!!! For N5600 Only
vpc nve peer-link-vlan 1234

!!! For N9000 Only
system nve infra-vlans 1234
```

---

<!-- TOC --><a name="l3-routing-nve-peer-link-config"></a>

### 2.7 (OPTIONAL & RECOMMENDED) Configure L3 Routing Across NVE Peer-Link Vlan

+ **For Underlay routing in case VTEP loses connection to spines.**
+ **Increased OSPF interface cost to perfer spine links.**

```text
2.7a Configure L3 Routing Across NVE Peer-Link VLAN on NX-01
============================================================

NX-01:
 interface vlan1234
  mtu 9150
  ip address 10.1.12.1/30
  ip pim sparse-mode
  ip router ospf UNDERLAY area 0
  ip ospf network point-to-point
  ! Set Cost to Prefer route via spines.
  ip ospf cost 100
  no shutdown
```

```text
2.7b Configure L3 Routing Across NVE Peer-Link VLAN on NX-02
============================================================

NX-02:
 interface vlan1234
  mtu 9150
  ip address 10.1.12.2/30
  ip pim sparse-mode
  ip router ospf UNDERLAY area 0
  ip ospf network point-to-point
  ! Set Cost to Prefer route via spines.
  ip ospf cost 100
  no shutdown
```

---

<!-- TOC --><a name="remaining-port-channels-config"></a>

### 2.8 (OPTIONAL) Configure Remaining Port-Channels

```text
2.8 Configure Remaining Port-Channels on NX-01 & NX-02
======================================================

default interface eth1/6-7

int e1/6
 swi
 swi mode trunk
 swi tru all vlan 10,20
 channel-group 10 mode active

int e1/7
 swi
 swi mode trunk
 swi tru all vlan 10,20
 channel-group 11 mode active

interface port-channel10
  vpc 10

interface port-channel11
  vpc 11
```

```text
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!  VERIFICATION COMMAND(S)  !!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

show vpc
show vpc consistency-parameters global
show vpc consistency-parameters vpc 11
show port-channel summary
```

---
---

## Integrating Non-Fabric Routing into Fabric Overlay  

+ **Refer to topology shown in section 2.**
+ **R1 is connected to NX-01 and is exchanging OSPF routes.**

---

<!-- TOC --><a name="vtep-to-non-fabric-config"></a>

### 3.1 Configure VTEP to Non-Fabric Router Connection

```text
3.1a Configure Router Connection on NX-01
========================================

NX-01:
interface Ethernet1/3
  vrf member WOLFECO
  ip address 11.0.0.1/24
  no shutdown
```

```text
3.1b Configure Router Connection on R1
======================================

R1:
interface GigabitEthernet0/0
 ip address 11.0.0.2 255.255.255.0
 no shutdown
!
!!! This Loopback is used for testing purposes to inject the 172.16.0.0/24 network into OSPF.
!
interface Loopback0
 ip address 172.16.0.1 255.255.255.0
!
```

```text
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!  VERIFICATION COMMAND(S)  !!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

NX-01:
ping 11.0.0.2

R1:
ping 11.0.0.1
```

---

<!-- TOC --><a name="ospf-vtp-router-config"></a>

### 3.2 Configure OSPF Overlay Between VTEP and Router

```text
3.2a Configure Overlay OSPF on NX-01
====================================

NX-01:
router ospf OVERLAY
 vrf WOLFECO
  router-id 0.0.0.1

interface Ethernet1/3
  ip ospf network point-to-point
  ip router ospf OVERLAY area 0
```

```text
3.2b Configure Overlay OSPF on R1
=================================

R1:
router ospf 1
!
interface GigabitEthernet0/0
 ip ospf 1 area 0
!
!!! This Loopback is used for testing purposes to inject the 172.16.0.0/24 network into OSPF.
!
interface Loopback0
 ip address 172.16.0.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
```

```text
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!  VERIFICATION COMMAND(S)  !!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

NX-01:
show ip ospf OVERLAY interface brief vrf WOLFECO
show ip ospf OVERLAY neighbors vrf WOLFECO

R1:
show ip ospf int brie
show ip ospf neighbor
```

<!-- TOC --><a name="ospf-vtp-peers-config"></a>

---

### 3.2 Extend Overlay OSPF to vPC Peer

```text
3.3a vPC OSPF Overlay Configuration on NX-01
============================================

NX-01:
vlan 999
interface vlan999
 vrf member WOLFECO
 ip address 10.99.0.1/30
 ip router ospf OVERLAY area 0
 ip ospf network point-to-point
 no shutdown
```

```text
3.3b vPC OSPF Overlay Configuration on NX-02
============================================

NX-02:
router ospf OVERLAY
 vrf WOLFECO
  router-id 0.0.0.2

vlan 999
interface vlan999
 vrf member WOLFECO
 ip address 10.99.0.1/30
 ip router ospf OVERLAY area 0
 ip ospf network point-to-point
 no shutdown
```

<!-- TOC --><a name="ospf-vtp-peers-config"></a>

### 3.3 Configure Redistribution into Overlay OSPF

```text
3.3a Redistribute OSPF Overlay into BGP L2VPN EVPN
==================================================

NX-01 & NX-02:
ip prefix-list Rtr_1_Prefix seq 5 permit 172.16.0.0/24

route-map Match_Rtr_1 permit 10
  match ip address prefix-list Rtr_1_Prefix 

!! REDIST is part of base VXLAN config. Presented here as a reminder.
route-map REDIST per 10
  match tag 67696

router bgp 65000
  vrf WOLFECO
    address-family ipv4 unicast
      redistribute direct route-map REDIST
      redistribute ospf OVERLAY route-map Match_Rtr_1
```

```text
3.3b OPTION ONE: Redistribute SUBNETS (EVPN Type 5 Ip Prefixes) BGP into OSPF Overlay
=====================================================================================

NX-01:
route-map BGP_to_OSPF_Subnets per 10
  match evpn route-type 5

route-map DIRECT_to_OSPF per 10

router ospf OVERLAY
 vrf WOLFECO
  redistribute bgp 65000 route-map BGP_to_OSPF
  redistribute direct route-map DIRECT_to_OSPF
```  

```text
3.3c OPTION TWO: Redistribute ALL (EVPN Type 2 & 5) BGP into OSPF Overlay
=====================================================================================

!! NOTE: Here we match route-type internal for the bgp redistribution. This is mandatory on Nexus 9000. Other devices may allow a general match all statement.

NX-01:
route-map BGP_to_OSPF_Subnets per 10
  match route-type internal

route-map DIRECT_to_OSPF per 10

router ospf OVERLAY
 vrf WOLFECO
  redistribute bgp 65000 route-map BGP_to_OSPF
  redistribute direct route-map DIRECT_to_OSPF
```  

```text
3.3d OPTION Three: Default Route (May not be a good idea. I didn't fully test it.)
=====================================================================================

NX-01:
router ospf OVERLAY
 vrf WOLFECO
  default-information originate always
```
