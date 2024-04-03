
# VXLAN EVPN on NX-OS (Basic)

| Version | Edited  | Author | Notes |
| --- | --- | --- | --- |
| 1.0 | 2024/04/03 | Jeremiah Wolfe | Initial Release |   

---

## Description

This document is intended to guide one through the **basic** configuration of VXLAN eVPN on cisco Nexus devices. While some explanation will be provided, this is not a teaching tool, but rather a quick reference.

For best experience view this file in a Markdown viewer. In addition to dedicated editor/viewers, most IDEs include markdown support. Additionally, web browser extensions are available.

> ⚠️ NOTE: The below syntax examples should work with **Nexus 7k** and **Nexus 9k** devices as of **NX-OS 10.2.7**.
>Nexus 5600 requires some additional syntax which will be noted inline.

---

## Table of Contents

1. UNDERLAY CONFIG  
[1.1 About ECMP and MTU](#ecmp-and-mtu)  
[1.1 IGP Routing Configuration](#igp-routing-config)  
[1.2 Multicast Configuration](#multicast-configuration)  
[1.4 MP-BGP Configuration](#mpbgp-configuration)  
2. L2 BRIDGING CONFIG  
[2.1 VLAN Configuration and VNI Mapping](#vlan-vni-configuration)  
[2.2 Downstream Port Configuration](#downstream-port-configuration)  
[2.3 NVE Interface Configuration](#nve-configuration)  
[2.4 Define L2 VNI under EVPN Config Mode](#define-l2-vni)
3. L3 OVERLAY CONFIG  
[3.1 Configure VRF(s) and Define L3 VNI](#vrf-l3-vni-configuration)  
[3.2 Configure VRF-shared VLAN Associate with L3 VNI](#shared-vrf-vni-configuration)  
[3.3 Distributed Anycast Gateway MAC Configuration](#anycast-mac-configuration)  
[3.4 Configure all required SVIs](#svi-configuration)  
[3.5 Associate L3 VNI Under NVE Interfae](#l3-vni-nve-configuration)  
[3.6 Redistribute SVI Subnets into BGP](#redistribute-svi-subnets-bgp)  
4. DHCP over VXLAN EVPN  
[4.1 Scennario 1: Client on Tenant VRF - Server on Default VRF](#dhcp-scenario-one)  
[4.2 Scenario 2: Client & Server on Same Tenant VRF](#dhcp-scenario-two)  
[4.3 Additional DHCP Scenarios](#dhcp-additional)
5. Verification and Troubleshooting  
[5.1 Helpful Commands](#helpful-commands)  
6. Complete VXLAN EVPN Example  
[6.1 VXLAN EVPN Basic Topology](#basic-topology)  
[6.2 NX-01 Config](#nx-01-config)  
[6.2 NX-02 Config](#nx-02-config)  
[6.2 NX-03 Config](#nx-03-config)  
[6.2 NX-04 Config](#nx-04-config)  
[6.2 NX-05 Config](#nx-05-config)

---
---

## UNDERLAY CONFIG

<!-- TOC --><a name="ecmp-and-mtu"></a>

### 1.1 ECMP and MTU

Due to MTU requirements in OSPF, this section is presented first so that the associated config will make sense in later examples.

+ Equal Cost Multi-Path is achieved by using a hash for souce UDP port.
  + OSPF allows equal cost paths by default (# of paths depends on platform). Can increase with "maximum-paths" command.
+ 50-byte overhead is added. Path MTU must reflect this. Fragmentation is **NOT** allowed.

```text
1.1 Example Syntax MTU & ECMP
=============================

! Configure MTU on all Spine-VTEP links.

int eth1/4, eth1/5
 mtu 9150

-==OPTIONAL==-
! Configure multipath if IGP default is not enough.

router ospf UNDERLAY
 maximum-path 16
```

---

<!-- TOC --><a name="igp-routing-config"></a>

### 1.2 IGP Routing Configuration

An underlay routing mechanism must be configured.  
Any IGP or even Static routes can be used.

**_GOAL:_** *VTEP to VTEP routability.*

+ VTEP IP must be /32
+ Loopback is best.  

```text
1.2 Example Syntax for VTEP and Spine.
=====================================

feature ospf
router ospf UNDERLAY

! Configure loopback interface
! VTEP IP Address on LEAF
! BGP Address on SPINE

int lo0
 ip address 10.0.0.11/32
 ip router ospf UNDERLAY area 0

! Configure all Spine-Life links.
! Example /29 for 2 spines (4 addresses)
! OSPF point-to-point to remove delay from DR election

interface eth1/5
 mtu 9150
 ip address 10.11.14.1/29
 ip ospf network point-to-point
 ip router ospf UNDERLAY area 0
```

---
<!-- TOC --><a name="multicast-configuration"></a>

### 1.3 Multicast Configuration

Multicast routing is configured using PM.  
Auto RP, BSR, or manual RP may be use.

**_GOAL:_** Allow for forwarding of BUM Traffic (Broadcast, Unknown Unicast and Multicast)  
Since BUM cannot be advertised in BGP (no specific VTEP to point to) Multicast is used to "flood" the traffic to VTEPs configured with appropriate vni mcast group.

+ RP role configured on spines.
+ Bidir PIM can be used for scalability.
+ Anycast RP in ASM or phantom RP in Bidir for redundancy.

```text
1.3a Example Syntax for VTEP
STATIC Rendezvous Point - Easiest - Recommended 
===============================================

feature pim

! Static RP.

ip pim rp-address 10.4.5.10 group-list 224.0.0.0/4

! Enable PIM sparse mode on Loopback and all Spine-VTEP interfaces.

int lo0
 ip pim sparse-mode

int eth1/4
 ip pim sparse-mode
```

```text
1.3b Example Syntax for Spine
STATIC Rendezvous Point - Easiest - Recommended 
===============================================

feature pim

! Static RP.

ip pim rp-address 10.4.5.10 group-list 224.0.0.0/4

! Configure anycast RP referencing the Spine Loopback IPs

ip pim rp-anycast 10.4.5.10 10.0.0.14
ip pim rp-anycast 10.4.5.10 10.0.0.15

! Create loopback with anycast IP Address
! Enable PIM sparse mode.
! Advertise in IGP

int l01
 ip address 10.4.5.10/32
  ip pim sparse-mode
  ip router ospf UNDERLAY area 0

! Enable PIM sparse mode on Loopback and ALL Spine-VTEP interfaces.

int lo0
 ip pim sparse-mode

int eth1/5, eth1/6
 ip pim sparse-mode 
```

```test
-==OPTIONAL==-
1.3c Example Syntax for Bidir PIM & Phantom RP
==============================================

! On ALL SWITCHES - Modify rp statement with "bidir"

ip pim rp-address 10.4.5.10 group-list 224.0.0.0/4 bidir

! On Spine switches actings as RP
! Loopback IP is in same subnet as anycast address, but NOT actual address.
! Each switch uses different subnet mask.
! RP subnet is advertised in IGP.
! Switch with most specific subnet becomes PRIMARY, etc...

SW-4:
----
int lo1
 ip address 10.4.5.9/30
 ip pim sparse-mode
 ip router ospf UNDERLAY area 0
 ip ospf network point-to-point
```

---
<!-- TOC --><a name="mpbgp-configuration"></a>

### 1.4 MP-BGP Configuration

L2VPN EVPN neighborship must be established between each Spine and VTEP.

+ VTEP <==> Spine - YES
+ VTEP <==> VTEP  - **NO!**

**_GOAL:_** Advertise end-to-end VXLAN L2 & L3 VIN information.

```text
1.4a Example Syntax VTEP MP-BGP
==============================

feature bgp
feature nv overlay
nv overlay evpn

! Configure iBGP with neighbor to EACH SPINE
! Using separate router ID than Lo0 can make troubleshooting easier.

router bgp 65000
 router-id 1.1.1.1

 neighbor 10.0.0.14
  remote-as 65000
  update-source lo0
  address-family l2vpn evpn
   send-community extended

 neighbor 10.0.0.15
  remote-as 65000
  update-source lo0
  address-family l2vpn evpn
   send-community extended
```

```text
1.4B Example Syntax Spine MP-BGP
================================

feature bgp
feature nv overlay
nv overlay evpn

! Neighbor prefix can be used to reduce configuration.
! Configure at least 2 spines as route reflector.

router bgp 65000
 router-id 4.4.4.4
 neighbor 10.1.1.0/24
  remote-as 65000
  update-so lo0
  address-family l2vpn evpn
   send-community extended
   route-reflector-client

```

---

---

## L2 BRIDGING CONFIG

<!-- TOC --><a name="vlan-vni-configuration"></a>

### 2.1 VLAN Configuration and VNI Mapping

Configure the appropriate VLANs per switch then enable VNI mapping.  
Only those VLANs that will terminate at the VTEP should be configured.  
VLAN numbers are locally significant to the switch. They can be duplicated across switches but it's not necessary.  
VNI number must be unique fabric-wide.  
It's similar to the relationship between Route Distinquishers and Route Targets.

```text
2.1 Example Syntax VLAN Config and VNI Mapping on VTEP
======================================================

feature vn-segment-vlan-based

vlan 10
 vn-segment 100010
vlan 20
 vn-segment 100020
```

---

<!-- TOC --><a name="downstream-port-configuration"></a>

### 2.2 Downstream Port Configuration

Standard port configuration for downstream (client side) ports. Just like in a basic network.

```text
2.2 Example Syntax Downstream Port Configuration on VTEP
========================================================

interface e1/7-8
 switchport
 switchport mode trunk

interface e1/9
 switchport
 switchport mode access
 switchport access vlan 10

interface e1/10
 switchport
 switchport mode access
 switchport access vlan 20
```

<!-- TOC --><a name="nve-configuration"></a>

### 2.3 NVE Interface Configuration

Network Virtualization Edge Interface (NVE) is used to encapsulate/decapsulate traffic.

```text
2.3 Example Syntax NVE Interface Configuration on VTEP
======================================================

! Reachability protocol tells system we're using BGP rather than flood and learn.
! Source interface is VTEP loopback. This interface will be source of multicast for BUM traffic.
! NVE is always "1" (i.e. interface nve1). There is no option for 2, 3, etc...

interface nve1
 no shutdown
 host-reachability protocol bgp
 source-interface loopback0
 member vni 100010
  mcast-group 239.0.0.10
  suppress-arp
 member vni 100020
  mcast-group 239.0.0.20
  suppress-arp
```

---

<!-- TOC --><a name="define-l2-vni"></a>

### 2.4 Define L2 VNI under EVPN Config Mode

```text
2.3 Example Syntax Define L2 VNI under EVPN on VTEP
===================================================

feature fabric forwarding
evpn
 vni 100010 l2
  rd auto
  route-target import auto
  route-target export auto
 vni 100020 l2
  rd auto
  route-target both auto
```

```test
-== ADDITIONAL COMMANDS FOR NEXUS 5600 ONLY==-

install feature-set fabric
feature-set fabric
hardware ethernet store-and-forward-switching

! Save configuration.
! RELOAD switch.
```

---

---

## L3 OVERLAY CONFIG

<!-- TOC --><a name="vrf-l3-vni-configuration"></a>

### 3.1 Configure VRF(s) and Define L3 VNI

Create VRF for each Tenant.

```text
3.1 Example Syntax VRF adn L2 VNI Configuration for all VTEPs
=============================================================

! Notice both route-target statements. 
! The normal statement is for IPv4 routing outside of the fabric. 
! The evpn statement is for routing within the fabric.

vrf context TENANT1
 vni 50000
 rd auto
 address-family ipv4 unicast
  route-target both auto
  route-target both auto evpn
```

---

<!-- TOC --><a name="shared-vrf-vni-configuration"></a>

### 3.2 Configure VRF-shared VLAN Associate with L3 VNI

Create new vlan and vlan to vxlan mapping.  
Can be any vlan not otherwise in use, then map it to VNI configured under VRF.

```text
3.2 Example Syntax VRF-Shared VLAN and VNI Mapping for all VTEPs
================================================================

vlan 500
 vn-segment 50000th auto evpn
```

---

<!-- TOC --><a name="anycast-mac-configuration"></a>

### 3.3 Distributed Anycast Gateway MAC Configuration

Define anycast gateway virtual MAC address.  
Can be any MAC, but must be unique fabric-wide.

See document [IEEE 802 ec-17-0174-00-00EC](https://mentor.ieee.org/802-ec/dcn/17/ec-17-0174-00-00EC-ieee-802-tutorial-of-2017-11-06-local-mac-addresses-in-the-overview-and-architecture-based-on-ieee-std-802c.pdf) for explanation of the Locally Administered MAC ranges.

"Legal" Locally Administered MAC Addresses should begin with 02.

| SLAP | Y bit | Z bit | X bit | M bit | SLAP Type | SLAP Identifier |
| --- | --- | --- | --- | --- | --- | --- |
| 00 | 0 | 0 | 1 | 0 | Admin Assigned | AAI ||

Example: **02**-00-00-11-22-33

```text
3.3 Example Syntax Define Anycast GW MAC for all VTEPs
======================================================

fabric forwarding anycast-gateway-mac 0200.1234.5678
```

---

<!-- TOC --><a name="svi-configuration"></a>

### 3.4 Configure all required SVIs

Only needed for the VLANs downstream of the switch.

```text
3.4 Example Syntax SVI Configuration for all VTEPs
======================================================

! MTU must match underlay MTU.
! The IP Address will be the anycast GW address.
! Add an arbitrary tag value to the ip address to simplify redistribution into bgp.

feature interface-vlan
interface vlan10
 mtu 9150
 vrf member TENANT1
 ip address 192.168.10.254/24 tag 67696
 ip pim sparse-mode
 fabric forwarding mode anycast-gateway
 no shutdown

interface vlan20
 mtu 9150
 vrf member TENANT1
 ip address 192.168.20.254/24 tag 67696
 ip pim sparse-mode
 fabric forwarding mode anycast-gateway
 no shutdown

 interface vlan500
  vrf member TENANT1
  ip forward
  no shutdown
```

---

<!-- TOC --><a name="l3-vni-nve-configuration"></a>

### 3.5 Associate L3 VNI Under NVE Interface

```text
3.5 Example Syntax Associate L3 VNI to NVE Interface for all VTEPs
==================================================================

! The NVE interface was added in a step 2.3.
! Here we're adding just one additional command.

interface nve1
 member vni 50000 associate-vrf
```

---

<!-- TOC --><a name="redistribute-svi-subnets-bgp"></a>

### 3.6 Redistribute SVI Subnets into BGP

In the below example we use a route map matching the previously defined tag value, but other methods (e.g prefix-list) will work.

```text
3.6 Example Syntax Redistribute SVI Subnets into BGP for all VTEPs
==================================================================

! BGP was previously configured in step 1.3a.
! Here we are adding the following additional commands.

router bgp 65000
 vrf TENANT1
  address-family ipv4 unicast
   advertise l2vpn evpn <<< Nexus 6500 ONLY! 
   redistribute direct route-map Overlay_Subnets
!
route-map Overlay_Subnets permit 10
 match tag 67696
```

---

---

## DHCP over VXLAN EVPN

>---
>
>### ⚠️ Please Note ⚠️
>
>When performing DHCP relay over VXLAN EVPN additional DHCP options are used. The DHCP server must support these options. Even then, additional configuration will likely be required.
>
>While testing these configurations I was UNABLE to a Cisco router based DHCP server to respond to the DHCP Discover messages dispite the examples in documentation indicating that it should work.
>
>Cisco documentation does include detailed configuration examples for setting up a Windows server as a VXLAN EVPN compatible DHCP server.
>
>#### Additional Resources:
>
>[NX-OS VXLAN Configuration Guide - Chapter: DHCP Relay in VXLAN BGP EVPN](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/102x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-102x/m_configuring_dhcp_relay.html)
>
>[Configure DHCP in IOS XE EVPN/VXLAN](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-9300-series-switches/217366-configure-dhcp-in-ios-xe-evpn-vxlan.html)
>
> ---

<!-- TOC --><a name="dhcp-scenario-one"></a>

### Scennario 1: Client on Tenant VRF - Server on Default VRF

+ Likely the easiest configuration.
+ The DHCP server must be reachable through the underlay.

```text
4.1a Example Syntax DHCP Server in Default VRF Server Connected Switch
======================================================================

! This config may or may not be necessary depending on the setup of your underlay network.
! Here we connect the DHCP server (10.5.10.2/30) to the underlay network in VLAN 510.
! VLAN 510 is NOT part of the VXLAN.

vlan 510

interface vlan510
 ip address 10.5.10.1/30
 ip router ospf UNDERLAY area 0
 ip ospf passive-interface
 no shut

! Now we configure the DHCP server itself. 
! NOTE: This is the configuration recommended by Cisco, but I could not get it to work.

DHCP-ROUTER#
ip vrf WOLFECO
!
ip dhcp excluded-address vrf WOLFECO 192.168.10.0 192.168.10.99
ip dhcp excluded-address vrf WOLFECO 192.168.20.0 192.168.20.99
!
ip dhcp pool VLAN10
 vrf WOLFECO
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.254 
!
ip dhcp pool VLAN20
 vrf WOLFECO
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.254 

```

```text
4.1b Example Syntax DHCP Server in Default VRF ALL VTEPs
========================================================

! On ALL VTEPs which will act as DHCP relay.
! In this example the DHCP server's address is 10.5.10.2.
! We will be adding DHCP relay to the EXISTING tenant SVIs.
! SVI's config, including vrf membership, should already exist, but is show here for clarity.

feature dhcp

service dhcp
ip dhcp relay
ip dhcp relay information option
ip dhcp relay information option vpn
ipv6 dhcp relay

interface Vlan10
  no shutdown
  mtu 9150
  vrf member WOLFECO
  ip address 192.168.10.254/24 tag 67696
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway
  ip dhcp relay address 10.5.10.2 use-vrf default

interface Vlan20
  no shutdown
  mtu 9150
  vrf member WOLFECO
  ip address 192.168.20.254/24 tag 67696
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway
  ip dhcp relay address 10.5.10.2 use-vrf default

```

---

<!-- TOC --><a name="dhcp-scenario-two"></a>

### 4.2 Scenario 2: Client & Server in Same Tenant VRF

+ Due to the use of anycast gateway for the L3 EVPN, OFFER messages from the DHCP sever may go to the wrong VTEP and be black holed.
+ To resolve this issue, each VTEP (whith DHCP clients downstream) must be configured with an additional loopback interface to act as the source of DHCP relay packets.

```text
4.2a Example Syntax DHCP Server Config
======================================

! NOTE: This is the configuration recommended by Cisco, but I could not get it to work.

DHCP-ROUTER#
ip vrf WOLFECO
!
ip dhcp excluded-address vrf WOLFECO 192.168.10.0 192.168.10.99
ip dhcp excluded-address vrf WOLFECO 192.168.20.0 192.168.20.99
!
ip dhcp pool VLAN10
 vrf WOLFECO
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.254 
!
ip dhcp pool VLAN20
 vrf WOLFECO
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.254 


```

```text
4.2b Example Syntax Client and Server in Same VRF All VTEPs
===========================================================

! Configure a NEW loopback to act as the DHCP Relay source.
! The IP is tagged so that it is advertised in BGP.
! In this example, the DHCP server's IP is 192.168.40.2 and resided in vlan 40.
! Vlan 40 is just another l3 tenant vlan. We could have used the previously 
! configured vlan 10 or vlan 20.

interface loopback1
 vrf member WOLFECO
 ip address 192.168.99.1/32 tag 67696

interface Vlan10
  ip dhcp relay address 192.168.40.2 
  ip dhcp relay source-interface loopback1 
 
interface Vlan20
  ip dhcp relay address 192.168.40.2 
  ip dhcp relay source-interface loopback1 
 
```

---

<!-- TOC --><a name="dhcp-additional"></a>

### 4.3 Additional DHCP Scenarios

Here: [NX-OS VXLAN Configuration Guide - Chapter: DHCP Relay in VXLAN BGP EVPN](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/102x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-102x/m_configuring_dhcp_relay.html)

The following additional scenarios can be found in the official Cisco NX-OS documentation referenced in the preceding link:
+ Client on Tenant VRF (VRF X) and Server on Different Tenant VRF (VRF Y)
+ Client on Tenant VRF and Server on Non-Default Non-VXLAN VRF
+ Configuring vPC Peers Example
+ vPC VTEP DHCP Relay Configuration Example

---

---

## Verification and Troubleshooting

<!-- TOC --><a name="helpful-commands"></a>

### 5.1 Helpful Commands

```text
5.1a General CLI Commands
=========================

NX-01# checkpoint CC1022 description Prior to CC#2024-1022

NX-01# show checkpoint summary

NX-01# show checkpoint CC1022

NX-01# show diff rollback-patch checkpoint CC1022 running-config

NX-01# show cli history unformatted last 10

```

```text
5.1b Verify Underlay Routing
============================

NX-01# show ip ospf neighbors

NX-01# ping 10.0.0.3 source-interface lo0

NX-01# sh ip ospf int brie

NX-01# sh ip route 10.0.0.3

```

```text
5.1c Verify PIM Configuration
=============================

NX-01# show ip pim rp

NX-01# show ip pim neighbor 
```

```text
5.1d Verify BGP Configuration & Overlay Routing
===============================================

NX-01# show run bgp

NX-01# show bgp l2vpn evpn

NX-01# show bgp l2vpn evpn summary

NX-01# show ip route vrf WOLFECO
```

```text
5.1e Verify VXLAN Configuration
=============================

NX-01# show vxlan

NX-01# show nve interface

NX-01# show nve peers  ! Requires interesting traffic to populate.

NX-01# show nve vni

NX-01# show mac address-table

NX-01# show mac address-table dynamic
```

---
---

## Complete VXLAN EVPN Example

<!-- TOC --><a name="basic-topology"></a>

#### Basic VXLAN EVPN Topology:

![Basic VXLAN EVPN Topology](https://jeremiahwolfe.github.io/Networking-Docs/VXLAN_EVPN_NXOS/VXLAN-EVPN-BASIC.drawio.png)

<!-- TOC --><a name="nx-01-config"></a>

```text
6.2 NX-01 Config
================

NX-01#

hostname NX-01

nv overlay evpn
feature ospf
feature bgp
feature pim
feature fabric forwarding
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay

fabric forwarding anycast-gateway-mac 0020.0000.1234
ip pim rp-address 10.4.5.10 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8
vlan 1,10,500
vlan 10
  vn-segment 100010
vlan 500
  vn-segment 50000

route-map REDIST permit 10
  match tag 67696 

vrf context WOLFECO
  vni 50000
  ip pim ssm range 232.0.0.0/8
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn

interface Vlan10
  no shutdown
  mtu 9150
  vrf member WOLFECO
  ip address 192.168.10.254/24 tag 67696
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway

interface Vlan500
  no shutdown
  vrf member WOLFECO
  ip forward

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 50000 associate-vrf
  member vni 100010 mcast-group 239.0.0.10

interface Ethernet1/1
  description Link to VLAN10 Client
  switchport access vlan 10

interface Ethernet1/2
  description Link to VLAN10 Client
  switchport access vlan 10

interface Ethernet1/3
  shutdown

interface Ethernet1/4
  description Uplink to NX-4 Eth1/1
  no switchport
  mtu 9150
  ip address 10.1.14.1/29
  ip ospf cost 4
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown

interface Ethernet1/5
  description Uplink to NX-5 Eth1/1
  no switchport
  mtu 9150
  ip address 10.1.15.1/29
  ip ospf cost 4
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown


interface loopback0
  description Loopback for Control Plane Communication
  ip address 10.0.0.1/32
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode

cli alias name wr copy run start

router ospf UNDERLAY
router bgp 65000
  router-id 1.1.1.1
  neighbor 10.0.0.4
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.0.0.5
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf WOLFECO
    address-family ipv4 unicast
      redistribute direct route-map REDIST
evpn
  vni 100010 l2
    rd auto
    route-target import auto
    route-target export auto

NX-01#
```

<!-- TOC --><a name="nx-02-config"></a>

```text
6.2 NX-02 Config
================

NX-02#

hostname NX-02

nv overlay evpn
feature ospf
feature bgp
feature pim
feature fabric forwarding
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay

fabric forwarding anycast-gateway-mac 0020.0000.1234
ip pim rp-address 10.4.5.10 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8
vlan 1,10,20,40,500
vlan 10
  vn-segment 100010
vlan 20
  vn-segment 100020
vlan 40
  vn-segment 100040
vlan 500
  vn-segment 50000

route-map REDIST permit 10
  match tag 67696 
vrf context WOLFECO
  vni 50000
  ip pim ssm range 232.0.0.0/8
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn

interface Vlan10
  no shutdown
  mtu 9150
  vrf member WOLFECO
  ip address 192.168.10.254/24 tag 67696
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway

interface Vlan20
  no shutdown
  mtu 9150
  vrf member WOLFECO
  ip address 192.168.20.254/24 tag 67696
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway

interface Vlan40
  no shutdown
  mtu 9150
  vrf member WOLFECO
  ip address 192.168.40.254/24 tag 67696
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway

interface Vlan500
  no shutdown
  vrf member WOLFECO
  no ip redirects
  ip forward

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 50000 associate-vrf
  member vni 100010 mcast-group 239.0.0.10
  member vni 100020 mcast-group 239.0.0.20
  member vni 100040 mcast-group 239.0.0.40

interface Ethernet1/1
  description Link to VLAN10 Client
  switchport access vlan 10

interface Ethernet1/2
  description Link to VLAN20 Client
  switchport access vlan 20

interface Ethernet1/3
  description Link to Downstream Switch with Multiple Clients
  switchport mode trunk

interface Ethernet1/4
  description Link to NX-04 Eth1/2
  no switchport
  mtu 9150
  ip address 10.1.24.2/29
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown

interface Ethernet1/5
  description Link to NX-05 Eth1/2
  no switchport
  mtu 9150
  ip address 10.1.25.2/29
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown

interface loopback0
  description Loopback for Control Plane Communication
  ip address 10.0.0.2/32
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
icam monitor scale

cli alias name wr copy run start

router ospf UNDERLAY
router bgp 65000
  router-id 2.2.2.2
  neighbor 10.0.0.4
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.0.0.5
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf WOLFECO
    address-family ipv4 unicast
      redistribute direct route-map REDIST
evpn
  vni 100010 l2
    rd auto
    route-target import auto
    route-target export auto
  vni 100020 l2
    rd auto
    route-target import auto
    route-target export auto
  vni 100040 l2
    rd auto
    route-target import auto
    route-target export auto


NX-02#      
```

<!-- TOC --><a name="nx-03-config"></a>

```text
6.2 NX-03 Config
================

NX-03#

hostname NX-03

nv overlay evpn
feature ospf
feature bgp
feature pim
feature fabric forwarding
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay

fabric forwarding anycast-gateway-mac 0020.0000.1234
ip pim rp-address 10.4.5.10 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8
vlan 1,20,500,510
vlan 20
  vn-segment 100020
vlan 500
  vn-segment 50000

route-map REDIST permit 10
  match tag 67696 
vrf context WOLFECO
  vni 50000
  ip pim ssm range 232.0.0.0/8
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn

interface Vlan20
  no shutdown
  mtu 9150
  vrf member WOLFECO
  ip address 192.168.20.254/24 tag 67696
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway

interface Vlan500
  no shutdown
  vrf member WOLFECO
  no ip redirects
  ip forward

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 50000 associate-vrf
  member vni 100020 mcast-group 239.0.0.20

interface Ethernet1/1
  description Link to VLAN20 Client
  switchport access vlan 20

interface Ethernet1/4
  description Link to NX-04 Eth1/3
  no switchport
  mtu 9150
  ip address 10.1.34.3/29
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown

interface Ethernet1/5
  description Link to NX-05 Eth1/3
  no switchport
  mtu 9150
  ip address 10.1.35.3/29
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown

interface loopback0
  description Loopback for Control Plane Communication
  ip address 10.0.0.3/32
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode

cli alias name wr copy run start

router ospf UNDERLAY
router bgp 65000
  router-id 3.3.3.3
  neighbor 10.0.0.4
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.0.0.5
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf WOLFECO
    address-family ipv4 unicast
      redistribute direct route-map REDIST
evpn
  vni 100020 l2
    rd auto
    route-target import auto
    route-target export auto


NX-03#  
```

<!-- TOC --><a name="nx-04-config"></a>

```text
6.2 NX-04 Config
================

NX-04#

hostname NX-04

nv overlay evpn
feature ospf
feature bgp
feature pim
feature nv overlay

ip pim rp-address 10.4.5.10 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8
ip pim anycast-rp 10.4.5.10 10.0.0.4
ip pim anycast-rp 10.4.5.10 10.0.0.5

interface Ethernet1/1
  description Link to NX-01 Eth1/4
  no switchport
  mtu 9150
  ip address 10.1.14.4/29
  ip ospf cost 4
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown

interface Ethernet1/2
  description Link to NX-02 Eth1/4
  no switchport
  mtu 9150
  ip address 10.1.24.4/29
  ip ospf cost 4
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown

interface Ethernet1/3
  description Link to NX-03 Eth1/4
  no switchport
  mtu 9150
  ip address 10.1.34.4/29
  ip ospf cost 4
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown


interface loopback0
  description Loopback for Control Plane Communication
  ip address 10.0.0.4/32
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode

interface loopback1
  description Loopback for PIM RP
  ip address 10.4.5.10/32
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode


cli alias name wr copy run start

router ospf UNDERLAY
router bgp 65000
  router-id 4.4.4.4
  neighbor 10.0.0.0/16
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
      route-reflector-client


NX-04#          
```

<!-- TOC --><a name="nx-05-config"></a>

```text
6.2 NX-05 Config
================

NX-05#

hostname NX-05

nv overlay evpn
feature ospf
feature bgp
feature pim
feature nv overlay

ip pim rp-address 10.4.5.10 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8
ip pim anycast-rp 10.4.5.10 10.0.0.4
ip pim anycast-rp 10.4.5.10 10.0.0.5

vrf context management

interface Ethernet1/1
  description Link to NX-01 Eth1/5
  no switchport
  mtu 9150
  ip address 10.1.15.5/29
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown

interface Ethernet1/2
  description Link to NX-02 Eth1/5
  no switchport
  mtu 9150
  ip address 10.1.25.5/29
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown

interface Ethernet1/3
  description Link to NX-03 Eth1/5
  no switchport
  mtu 9150
  ip address 10.1.35.5/29
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown

interface loopback0
  description Loopback for Control Plane Communication
  ip address 10.0.0.5/32
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode

interface loopback1
  description Loopback for PIM RP
  ip address 10.4.5.10/32
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
 

cli alias name wr copy run start

router ospf UNDERLAY
router bgp 65000
  router-id 4.4.4.4
  neighbor 10.0.0.0/16
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
      route-reflector-client
  neighbor 10.1.0.0/24
    remote-as 65000
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
      route-reflector-client


NX-05#             
```

### End of Document