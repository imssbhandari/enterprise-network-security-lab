# Enterprise Network Security Lab — Palo Alto HA Firewall

A complete, self-built enterprise network design implemented and tested in EVE-NG on Palo Alto firewalls — dual-ISP redundancy, Active/Passive HA clustering, zone-based security policies, NAT, and VLAN-segmented internal networking.

## Full write-up

See [`/palo-alto/README.md`](./palo-alto) for the complete architecture, configuration, and testing documentation.

## Skills Demonstrated

- Firewall HA clustering (Active/Passive) — link/interface planning, priority-based failover
- VRRP gateway redundancy across dual ISPs
- Zone-based security policy design (Trust / Untrust / DMZ)
- NAT (Source NAT, Destination NAT, U-Turn NAT)
- VLAN design and segmentation (IT / HR / Finance / DMZ)
- Static routing and virtual router configuration
