# OSPF Basics

| Version | Edited  | Author | Notes |
| --- | --- | --- | --- |
| 1.0 | 2024/06/29 | Jeremiah Wolfe | Initial Release |   

---

## Description

This document is intended to guide one through the **basic** configuration of OSPF v2 and v3 on Cisco hardware.

This document is NOT intended to teach theory (a vast subject) but rather to simply to act as a configuration quick reference.

---

## Table of Contents


1. OSPF Fundamentals  
[1.1 Network Types](#network-types)  
[1.2 Area Types](#area-types)  
[1.3 Designated Router Election](#dr-election) 
2. [The OSPF Process](#ospf-process) 
3. [Advertising a Network](#advertising-network)
4. [Filtering](#filtering) 
5. [Summarization](#summarization) 
6. [Default Route Injection](#default-route-injection) 
7. [Cost](#cost)
8. [Virtual Links](#virtual-links)
9. [BESTPATH Determination](#bestpath)

---

## 1. OSPF Fundamentals

<!-- TOC --><a name="network-types"></a>

### 1.1 Network Types

| Type | DR/BDR  | Hello | Dead | Manual Neighbor | Description |
| --- | --- | --- | --- | --- | --- |
| broadcast | Yes | 10 | 40 | No |multiaccess LAN segments |
| non-broadcast | Yes* | 30 | 120 | Yes | multiaccess without multicast support |
| point-to-point | No | 10 | 40 | No |point-to-point networks (2 routers only) |
| point-to-multipoint | No | 30 | 120 | No | Multipoint networks (e.g. frame-relay)|
| point-to-multipoint non-broadcast | No | 30 | 120 | Yes | Multipoint w/o multicast support.

**In partial mesh configurations the spoke routers must be configured with priority 0 so that only the hub can become DR.*