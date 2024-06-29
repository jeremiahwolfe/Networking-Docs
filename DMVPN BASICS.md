# DMVPN Basics

| Version | Edited  | Author | Notes |
| --- | --- | --- | --- |
| 1.0 | 2024/06/29 | Jeremiah Wolfe | Initial Release |   

---

## Description

This document is intended to guide one through the **basic** configuration of DMVPN on Cisco hardware.

Some older aspects will be omitted as they are long obsolete.

---

## Table of Contents


1. [Indtrouction](#introduction)  
2. [Single Hub Diagram](#1h-diagram)
3. Phase Two - Single Hub  
[3.1 Initial Configuration](#p2-1h-initial)  
[3.2 OSPF Configuration](#p2-1h-ospf-configuration)  
[3.3 EIGRP Configuration](#p2-1h-eigrp-configuration)  
[3.4 iBGP Configuration](#p2-1h-ibgp-configuration)  
[3.5 eBGP Configuration](#p2-1h-ebgp-configuration)  
4. Phase Three - Single Hub  
[4.1 Initial Configuration](#p3-1h-initial)  
[4.2 OSPF Configuration](#p3-1h-ospf-configuration)  
[4.3 EIGRP Configuration](#p3-1h-eigrp-configuration)  
[4.4 iBGP Configuration](#p3-1h-ibgp-configuration)  
[4.5 eBGP Configuration](#p3-1h-ebgp-configuration)  

5. Phase Three - Single Hub Dual Cloud

6. Phase Three - Dual Hub

7. Phase Three - Dual Hub Dual Cloud

8. NHS Clustering & Interesting Traffic

9. DMVPN & DHCP

10. DMVPN over MPLS

11. DMVPN IPSec with IKEv1

12. DMVPN IPsec with IKEv2

13. DMVPN Front Door VRF

<!-- TOC --><a name="introduction"></a>
## 1. Introduction
DMVPN uses an UNDERLAY network (such as the public Internet) to build an OVERLAY network by creating a series of tunnels.

**GRE** is the most common tunneling protocol used, with **IPSec** added for security.

By default, GRE can only create point-to-point links, which would become overly burdensome for any network with more than a few sites. As such **gre multipoint** can be used to allow a one-to-many tunneling topology.

To support the one-to-many scheme an additional protocol is used, **Next-hop Resolution Protocol (NHRP)**. (See RFC 3223 - NBMA Next Hop Resolution Protocol)

DMVPN has three "flavors," Phase One, Phase Two, and Phase Three.  

**Phase One - Hub-and-Spoke** - Mimics traditional hub-and-spoke nework where the hub router creates an individual tunnel (using mGRE) to each of the spoke routers. All traffic flows through the hub. As such the spoke's routing table can be highly optimized (it sends everything to the hub).

**Phase Two - Full Mesh | Partial Mesh** - Allows spokes to build temporary spoke-to-spoke tunnels between each other as needed. For this to work the spokes need to run gre multipoint like the hub in phase one. Further an NHRP map is required for each spoke that a connection may form. Spoke routers must query the hub router to learn NBMA-to-overlay mapping information. (Static mappings may also be configured on spokes, but this may become burdensome.)

In modern Phase Two implementations, the requesting spoke querrys the hub. The hub forwards the request to the target spoke. The target spoke then replys to the requesting spoke. The reduces the load on the hub somewhat.

For spoke-to-spoke communication to function, each spoke must have routing information for all networks behind the other spokes. This can lead to memory issues on the spoke routers, which is why Phase Three DMVPN was introduced.

**Phase Three - Hib-initiated Spoke-to-Spoke Tunnels** - In Phase Three DMVPN routing is determined by the hub router. Here NHRP is used to communicate the necessary routes on an as-needed basis then override the existing routing table.

<!-- TOC --><a name="1h-diagram"></a>  

## 2.0 Single Hub Configurations

![Basic VXLAN EVPN Topology](https://jeremiahwolfe.github.io/Networking-Docs/DMVPN/DMVPN_Single_Hub.png)

## 3.0 Phase Two - Single Hub  

<!-- TOC --><a name="p2-1h-initial"></a>  

### 3.1 Initial Configuration   

```
3.1 Example Initial Configuration
=================================

Notes:
Tunnel Interface address is OVERLAY address.
NHRP network-id is LOCALLY SIGNIFICANT only, but it is best practice for all devices to share the same id.

HUB ROUTER
----------
interface tunnel 10
 ip address 100.1.1.2 255.255.255.0
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 tunnel source GigabitEthernet0/5
 tunnel mode gre multipoint

SPOKE-1 ROUTER
------------
interface tunnel 10
 ip address 100.1.1.2 255.255.255.0
 ip nhrp network-id 100
 tunnel source 25.1.1.1
 tunnel mode tru multipoint
 ip nhrp nhs 100.1.1.1 nbma 15.1.1.1 multicast 

SPOKE-2 Router
--------------
interface tunnel 10
 ip address 100.1.1.3 255.255.255.0
 ip nhrp network-id 100
 tunnel source 35.1.1.1
 tunnel mode tru multipoint
 ip nhrp nhs 100.1.1.1 nbma 15.1.1.1 multicast

```

```
3.1.1 Useful Commands
=====================

show dmvp
sh ip nhrp
sh ip nhrp multicast
sh ip cef

```

<!-- TOC --><a name="p2-1h-ospf-configuration"></a>  

### 3.2 OSPF Configuration   

```
3.2 Example OSPF Configuration
=================================
Notes:
All non-broadcast network types will be ignored and cannot be used.
Spoke routers should be set to OSPF Priority 0 so that they do not participate in designated router elections.

HUB Router
----------
int tunnel 10
 ip ospf network broadcast
 ip ospf 100 area 0

int g0/7
 ip ospf 100 area 0

router ospf 100
 passive-interface g0/7

SPOKE-1 Router
--------------
int tunnel 10
 ip ospf network broadcast
 ip ospf 100 area 0
 ip ospf priority 0

int g0/4
 ip ospf 100 area 0

router ospf 100
 passive-interface g0/4

SPOKE-2 Router
--------------
int tunnel 10
 ip ospf network broadcast
 ip ospf 100 area 0
 ip ospf priority 0

int g0/4
 ip ospf 100 area 0

router ospf 100
 passive-interface g0/4

```

```
3.2.1 Useful Commands
=====================

sh ip ospf neighbor
sh ip route ospf
sh ip ospf int brie

```

<!-- TOC --><a name="p2-1h-eigrp-configuration"></a>  

### 3.3 EIGRP Configuration   

```
3.3 Example EIGRP Configuration
=================================
Notes:
Split horizon must be disabled on hub router.
Also the hub must not set itself as the next hop for routes passing through it.

HUB Router
----------
router eigrp 100
 network 17.1.1.1 0.0.0.0
 network 100.1.1.1 0.0.0.0
 passive-interface g0/7

int tunnel 10
 no ip next-hop-self eigrp 100
 no ip split-horizon eigrp 100

Spoke-1 Router
--------------
router eigrp 100
 network 24.1.1.1 0.0.0.0
 network 100.1.1.2 0.0.0.0
 passive-interface g0/4

Spoke-2 Router
--------------
router eigrp 100
 network 36.1.1.1 0.0.0.0
 network 100.1.1.2 0.0.0.0
 passive-interface g0/4

```
```
3.3.1 Useful Commands
=====================
sh ip eigrp 100 neighbor
sh ip route eigrp 100

```

<!-- TOC --><a name="p2-1h-ibgp-configuration"></a>

### 3.4 iBGP Configuration   

```
3.4 Example iBGP Configuration
=================================
Notes:
The hub router is configured as a route reflector with dynamic peering.

HUB Router
----------
router bgp 100
 neighbor spokes peer-group
 neighbor spokes remote-as 100
 bgp listen range 100.1.1.0/24 peer-group spokes
!
 address-family ipv4
  neighbor spokes active
  neighbor spokes route-reflector-client
  network 17.1.1.0 mask 255.255.255.0

Spoke Router-1
--------------
router bgp 100
 neighbor 100.1.1.1 remote-as 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  network 24.1.1.0 mask 255.255.255.0

Spoke Router-2
--------------
router bgp 100
 neighbor 100.1.1.1 remote-as 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  network 36.1.1.0 mask 255.255.255.0

```
```
3.4.1 Useful Commands
=====================
sh ip bgp
sh ip route bgp

```

<!-- TOC --><a name="p2-1h-ebgp-configuration"></a>

### 3.5 eBGP Configuration   

```
3.5.1 Example eBGP Configuration - Spokes Same AS
=================================
Notes:
Spokes must use: allow-as in 
Since they will be learning routes from other sites with the same AS through eBGP with the hub.

HUB Router
----------
router bgp 100
 neighbor spokes peer-group
 neighbor spokes remote-as 230
 bgp listen range 100.1.1.0/24 peer-group spokes
 !
 address-family ipv4
  neighbor spokes activate
  network 17.1.1.0 mask 255.255.255.0

Spoke Router-1
--------------
router bgp 230
 neighbor 100.1.1.1 remote 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  neighbor 100.1.1.1 allowas-in
  network 24.1.1.0 mask 255.255.255.0

Spoke Router-2
--------------
router bgp 230
 neighbor 100.1.1.1 remote 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  neighbor 100.1.1.1 allowas-in
  network 36.1.1.0 mask 255.255.255.0

```
```
3.5.2 Example eBGP Configuration - Spokes Different AS
=================================
Notes:

HUB Router
----------
router bgp 100
 neighbor spokes peer-group
 neighbor spokes remote-as 200 alternate-as 300
 bgp listen range 100.1.1.0/24 peer-group spokes
 !
 address-family ipv4
  neighbor spokes activate
  network 17.1.1.0 mask 255.255.255.0

Spoke Router-1
--------------
router bgp 200
 neighbor 100.1.1.1 remote 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  network 24.1.1.0 mask 255.255.255.0

Spoke Router-2
--------------
router bgp 300
 neighbor 100.1.1.1 remote 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  network 36.1.1.0 mask 255.255.255.0

```
```
3.5.3 Useful Commands
=====================
show ip bgp

```

## 4.0 Phase Three - Single Hub  

![Basic VXLAN EVPN Topology](https://jeremiahwolfe.github.io/Networking-Docs/DMVPN/DMVPN_Single_Hub.png)

<!-- TOC --><a name="p3-1h-initial"></a>  

### 4.1 Initial Configuration   

```
4.1 Example Initial Configuration
=================================

Notes:
Tunnel Interface address is OVERLAY address.
NHRP network-id is LOCALLY SIGNIFICANT only, but it is best practice for all devices to share the same id.

HUB ROUTER
----------
interface tunnel 10
 ip address 100.1.1.2 255.255.255.0
 ip nhrp map multicast dynamic
 ip nhrp redirect
 ip nhrp network-id 100
 tunnel source GigabitEthernet0/5
 tunnel mode gre multipoint

SPOKE-1 ROUTER
------------
interface tunnel 10
 ip address 100.1.1.2 255.255.255.0
 ip nhrp network-id 100
 ip nhrp shortcut
 ip nhrp nhs 100.1.1.1 nbma 15.1.1.1 multicast 
 tunnel source 25.1.1.1
 tunnel mode tru multipoint
 

SPOKE-2 Router
--------------
interface tunnel 10
 ip address 100.1.1.3 255.255.255.0
 ip nhrp network-id 10
 ip nhrp shortcut
 ip nhrp nhs 100.1.1.1 nbma 15.1.1.1 multicast
 tunnel source 35.1.1.1
 tunnel mode tru multipoint

```

```
3.1.1 Useful Commands
=====================

show dmvp
sh ip nhrp
sh ip nhrp multicast
sh ip cef

```

<!-- TOC --><a name="p3-1h-ospf-configuration"></a>  

### 4.2 OSPF Configuration   

The OSPF broadcast network type does not provide true Phase 3 behavior.
The next hop is preserved with the use of broadcast network type. As such the spoke self-triggers a nhrp resolution request for the next hop.

In true Phase 3 behavior the nhrp resolution request should be triggered through the hub.

Under this configuration, the spoke will perform 2 resolutions. One for the remote spoke's overlay address (caused by an incomplete CEF adjancency), and another for the target network.

Due to this, the Phase 3 resolution is redundant and may lead to no change in the routing table.

Therefor, OSPF's broadcast network and NBMA network types do NOT allow for true Phase 3 behavior.

This leaves P2P and P2MP OSPF network types to achieve true Phase 3 behavior. 

```
4.2 Example OSPF Configuration
=================================

Notes:
The Hub router is configured with a P2MP network type as it has to allow it to form OSPF adjacencies with multiple spokes.

Spoke routers use P2P network type in order to form OSPF adjancency with the Hub.

The hello timer on the Hub is modified to match the hello timer of the P2P interface on the Spokes.


HUB Router
----------
int tunnel 10
 ip ospf network point-to-multipoint
 ip ospf 100 area 0
 ip ospf hello-interval 10
 ip ospf dead-interval 40

int g0/7
 ip ospf 100 area 0

router ospf 100
 passive-interface g0/7

SPOKE-1 Router
--------------
int tunnel 10
 ip ospf network point-to-point
 ip ospf 100 area 0
 ip ospf priority 0

int g0/4
 ip ospf 100 area 0

router ospf 100
 passive-interface g0/4

SPOKE-2 Router
--------------
int tunnel 10
 ip ospf network point-to-point
 ip ospf 100 area 0
 ip ospf priority 0

int g0/4
 ip ospf 100 area 0

router ospf 100
 passive-interface g0/4

```

```
3.2.1 Useful Commands
=====================

sh ip ospf neighbor
sh ip route ospf
sh ip ospf int *G0/1*
sh ip ospf int brie

```

<!-- TOC --><a name="p3-1h-eigrp-configuration"></a>  

### 4.3 EIGRP Configuration   

```
4.3 Example EIGRP Configuration
=================================
Notes:
The configuration is similiar to Phase 2. With the exception that the hub router now advertises a summary route to the spokes.

HUB Router
----------
router eigrp 100
 network 17.1.1.1 0.0.0.0
 network 100.1.1.1 0.0.0.0
 passive-interface g0/7

int tunnel 10
 ip summary-address eigrp 100 0.0.0.0 0.0.0.0


Spoke-1 Router
--------------
router eigrp 100
 network 24.1.1.1 0.0.0.0
 network 100.1.1.2 0.0.0.0
 passive-interface g0/4

Spoke-2 Router
--------------
router eigrp 100
 network 36.1.1.1 0.0.0.0
 network 100.1.1.2 0.0.0.0
 passive-interface g0/4

```
```
4.3.1 Useful Commands
=====================
sh ip eigrp 100 neighbor
sh ip route eigrp 100

```

<!-- TOC --><a name="p3-1h-ibgp-configuration"></a>

### 4.4 iBGP Configuration   

```
4.4 Example iBGP Configuration
=================================
Notes:
The hub router is configured as a route reflector with dynamic peering.
Hub is configured to send a default route to the spokes.

HUB Router
----------
ip prefix-list DMVPN permit 0.0.0.0/0

route-map default per 10
 match ip addr prefix DMVPN

router bgp 100
 neighbor spokes peer-group
 neighbor spokes remote-as 100
 bgp listen range 100.1.1.0/24 peer-group spokes
!
 address-family ipv4
  neighbor spokes active
  neighbor spokes route-reflector-client
  neighbor spokes default-originate
  neighbor spokes route-map DMVPN out
  network 17.1.1.0 mask 255.255.255.0

Spoke Router-1
--------------
router bgp 100
 neighbor 100.1.1.1 remote-as 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  network 24.1.1.0 mask 255.255.255.0

Spoke Router-2
--------------
router bgp 100
 neighbor 100.1.1.1 remote-as 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  network 36.1.1.0 mask 255.255.255.0

```
```
3.4.1 Useful Commands
=====================
sh ip bgp
sh ip route bgp

```

<!-- TOC --><a name="p2-1h-ebgp-configuration"></a>

### 4.5 eBGP Configuration   

```
4.5.1 Example eBGP Configuration - Spokes Same AS
=================================
Notes:
Hub will advertise a default route.

HUB Router
----------
ip prefix-list DMVPN permit 0.0.0.0/0

route-map default per 10
 match ip address prefix-list DMVPN

router bgp 100
 neighbor spokes peer-group
 neighbor spokes remote-as 230
 bgp listen range 100.1.1.0/24 peer-group spokes
 !
 address-family ipv4
  neighbor spokes activate
  neighbor spokes default-originate
  neighbor spokes route-map default-out
  network 17.1.1.0 mask 255.255.255.0

Spoke Router-1
--------------
router bgp 230
 neighbor 100.1.1.1 remote 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  network 24.1.1.0 mask 255.255.255.0

Spoke Router-2
--------------
router bgp 230
 neighbor 100.1.1.1 remote 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  network 36.1.1.0 mask 255.255.255.0

```
```
4.5.2 Example eBGP Configuration - Spokes Different AS
=================================
Notes:

HUB Router
----------
router bgp 100
 neighbor spokes peer-group
 neighbor spokes remote-as 200 alternate-as 300
 bgp listen range 100.1.1.0/24 peer-group spokes
 !
 address-family ipv4
  neighbor spokes activate
  network 17.1.1.0 mask 255.255.255.0

Spoke Router-1
--------------
router bgp 200
 neighbor 100.1.1.1 remote 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  network 24.1.1.0 mask 255.255.255.0

Spoke Router-2
--------------
router bgp 300
 neighbor 100.1.1.1 remote 100
 !
 address-family ipv4
  neighbor 100.1.1.1 activate
  network 36.1.1.0 mask 255.255.255.0

```
```
3.5.3 Useful Commands
=====================
show ip bgp

```

## 5. Phase Three - Single Hub Dual Cloud

## 6. Phase Three - Dual Hub

## 7. Phase Three - Dual Hub Dual Cloud

## 8. NHS Clustering & Interesting Traffic

## 9. DMVPN & DHCP

## 10. DMVPN over MPLS

## 11. DMVPN IPSec with IKEv1

```
11.1 Example DMVPN IPSec with IKEv1
===================================


ISAKMP Policy
----------
crypto isakmp policy <#>
hash <hash>
encryption <type>
group <diffie-hellman-group>
authentication pre-share

Create Pre-Shared Key
---------------------
crypto isakmp key <KEY> address <prefix>

Configure IPsec Transform Set
-----------------------------
crypto ipsec transform-set <TS_NAME> <encryption-alg> <auth-algs>
  mode transport

Configure IPsec Profile
-----------------------
crypto ipsec profile <PROF_NAME>
  set transform-set <TS_NAME>

Apply to DMVPN Tunnel Interface
-------------------------------
interface Tunnel1
  tunnel protection ipsec profile <PROF_NAME>

```  

```
11.2 Complete Config Example DMVPN IPSec with IKEv1
===================================================
crypto isakmp policy 10
  authentication pre-share
  encryption 3des
  hash md5
  group 2
!
crypto isakmp key DMVPN_KEY address 0.0.0.0 0.0.0.0
!
crypto ipsec transform-set 3DES_MD5 esp-3des esp-md5-hmac
  mode transport
!
crypto ipsec profile DMVPN
  set transform-set 3DES_MD5
!
interface Tunnel1
  ip address 10.0.0.1 255.255.255.0
  ip mtu 1400
  ip tcp adjust-mss 1360
  tunnel source Gig0/1
  tunnel mode gre multipoint
  ip nhrp network-id 100
  ip nhrp authentication CISCO
  ip nhrp map multicast dynamic
  ip nhrp redirect
  tunnel protection ipsec profile DMVPN

```
## 12. DMVPN IPsec with IKEv2

```
12.1 Example DMVPN IPSec with IKEv2
===================================
NOTE: This configuration relys on Cisco "Smart Defaults" for IKEv2.
See Cisco's website for more info on Smart Deafults.
(Google: Cisco ikev2 smart defaults)

Configure IKEv2 KeyRing
-----------------------
crypto ikev2 keyring <KR_NAME>
 peer <NAME>
  address 0.0.0.0 0.0.0.0
  pre-shared-key <KEY>

Configure IKEv2 Profile
-----------------------
crypto ikev2 profile <IKE_PROF_NAME>
 keyring <KR_NAME>
 authentication local pre-share
 authentication remote pre-share
 match address local 0.0.0.0
 match identity remote address 0.0.0.0 0.0.0.0

Configure IPsec profile
-----------------------
crypto ipsec profile <IPSEC_PROF>
 set ikev2-profile <IKE_PROF_NAME>

Apply To the Tunnel Interface
-----------------------------
interface Tunnel1
 tunnel protection ipsec profile <IPSEC_PROF>

```

```
12.2 Complete Config Example DMVPN IPSec with IKEv2
===================================================
crypto ikev2 keyring IKEV2-KEYRING
 peer dmvpn-node
  address 0.0.0.0 0.0.0.0
  pre-shared-key CISCO123
!
crypto ikev2 profile IKEV2-PROF
 keyring IKEV2-KEYRING
 authentication local pre-share
 authentication remote pre-share
 match address local 0.0.0.0
 match identity remote address 0.0.0.0 0.0.0.0
!
crypto ipsec profile IPSEC-IKEV2
 set ikev2-profile IKEV2-PROF
!
interface Tunnel1
 ip address 10.0.0.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Gig0/1
 tunnel mode gre multipoint
 ip nhrp network-id 100
 ip nhrp authentication CISCO
 ip nhrp map multicast dynamic
 ip nhrp redirect
 tunnel protection ipsec profile IPSEC-IKEV2
```

```
12.3 Useful Commands

show crypto ikev2 policy
show crypto ikev2 proposal
show crypto ikev2 profile
show crypto ikev2 sa detailed

```

## 13. DMVPN IPSec with Front Door VRF