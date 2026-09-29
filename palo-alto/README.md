# Palo Alto HA Firewall Lab

**Platform:** PAN-OS 10.2 (Palo Alto VM), deployed in EVE-NG

## Topology

![Palo Alto Topology](./topology.png)

This lab implements a **three-tier hierarchical enterprise network design** (collapsed 
core), with a redundant dual-ISP edge and a firewall-based security perimeter:

- **Edge layer:** Two routers (R1, R2), each connected to a separate ISP, providing 
  WAN redundancy.
- **Perimeter layer:** An Active/Passive Palo Alto HA pair (PA-1/PA-2) acting as the 
  security boundary between the edge and the internal network, with a dedicated DMZ 
  segment for externally-facing services.
- **Core/access layer:** A core switch distributing to three access switches, each 
  hosting a dedicated VLAN (IT, HR, Finance) — a star topology fanning out from a 
  single collapsed core.
- **Management plane:** A separate, isolated management network for out-of-band 
  administrative access to the firewalls.

## WAN Edge Redundancy (VRRP)

R1 and R2 each connect to a different ISP and share a virtual gateway IP via VRRP, 
so internal traffic always has a reachable default gateway even if one router or 
ISP link fails.
