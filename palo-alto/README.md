# Palo Alto HA Firewall Lab

## Topology

![Palo Alto Topology](./topology.png)

Dual-ISP edge (R1/R2) with VRRP-based gateway redundancy (VRRP IP, priority 120, 
preempt), feeding an Active/Passive Palo Alto (PA-1/PA-2) HA pair. Internal network 
segmented into VLAN 10 (IT), VLAN 20 (HR), VLAN 30 (Finance), and a dedicated DMZ 
zone, with a separated management network.

## HA Configuration

- Dedicated control link (eth1/3) and backup control link (eth1/5)
- Dedicated data link (eth1/4) and backup data link (eth1/6)
- Priority-based election with preempt enabled

## Routing

Static routing configured on the virtual router to reach internal subnets 
(e.g. 10.1.10.0/24 via next-hop 172.16.150.101).

## What I Tested

- Confirmed HA failover between PA-1 and PA-2
- Verified inter-VLAN traffic flow and DMZ isolation
