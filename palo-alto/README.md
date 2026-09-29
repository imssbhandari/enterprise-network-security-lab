# Palo Alto HA Firewall Lab

**Platform:** PAN-OS 10.2 (Palo Alto VM), deployed in EVE-NG

## Topology

![Palo Alto Topology](./topology.png)

This lab implements a **three-tier hierarchical enterprise network design** (collapsed core), with a redundant dual-ISP edge and a firewall-based security perimeter:

- **Edge layer:** Two routers (R1, R2), each connected to a separate ISP, providing WAN redundancy.
- **Perimeter layer:** An Active/Passive Palo Alto HA pair (PA-1/PA-2) acting as the security boundary between the edge and the internal network, with a dedicated DMZ segment for externally-facing services.
- **Core/access layer:** A core switch distributing to three access switches, each hosting a dedicated VLAN (IT, HR, Finance) — a star topology fanning out from a single collapsed core.
- **Management plane:** A separate, isolated management network for out-of-band administrative access to the firewalls.

## WAN Edge Redundancy (VRRP)

R1 and R2 each connect to a different ISP and share a virtual gateway IP via VRRP, so internal traffic always has a reachable default gateway even if one router or ISP link fails.

    interface e0/0
      ip address 172.16.50.1 255.255.255.248
      vrrp 10 ip 172.16.50.200
      vrrp 10 priority 120
      vrrp 10 preempt

- Virtual/shared gateway IP: **172.16.50.200**
- R1: 172.16.50.1/24 (priority 120, preempt enabled)
- R2: 172.16.50.5/24 (standby peer)

## HA Configuration (Active/Passive)

The Palo Alto pair uses dedicated, isolated links for HA control and data synchronization, separate from production traffic:

| Link | Interface | Purpose |
|---|---|---|
| Control link (primary) | eth1/3 | HA1 — 192.168.1.1/2 |
| Control link (backup) | eth1/5 | HA2 backup — 192.168.2.1/2 |
| Data link (primary) | eth1/4 | Session sync |
| Data link (backup) | eth1/6 | Session sync (backup) |

**Setup steps followed:**
1. Dedicate interfaces eth1/3–eth1/6 on both firewalls exclusively to HA (no production traffic on these links).
2. Enable HA under **Device > High Availability > General**, and configure the peer IP for each HA interface (e.g. 192.168.1.2 for eth1/3, 192.168.2.2 for eth1/5).
3. Configure election settings under **Device > High Availability > Election** — the firewall with the **lower priority number becomes the active/root** peer; preempt is enabled so the designated primary reclaims active status after recovery.
4. Configure the control and data links under **Device > High Availability > HA Communications**, assigning each interface its role (control/data, primary/backup) as shown above.

**Note on Shared Interface IPs (HA Behavior)**
Data-plane interfaces (e.g., eth1/1) are configured with identical IP addresses on both PA-1 and PA-2. This is expected Active/Passive HA behavior: only the Active unit forwards traffic on that IP at any given time, while the Passive unit holds the same configuration in sync via the HA data link. On failover, the newly Active unit assumes forwarding on that same IP, so the network (e.g., the core switch) requires no reconfiguration.

## Management

Each firewall has a dedicated management interface on an isolated management subnet, separate from production and HA traffic:

    set deviceconfig system type static
    set deviceconfig system ip-address 172.16.100.1 netmask 255.255.255.0
    set deviceconfig system hostname active

- PA-1 management: 172.16.100.1
- PA-2 management: 172.16.100.2

## DMZ

A dedicated DMZ zone (192.168.20.0/24) hosts externally-reachable services (a Windows server and an additional server), isolated from internal VLANs and reachable only through explicit firewall policy.

## Security Zones

Three security zones are configured on the firewall pair:

| Zone | Interface | Facing |
|---|---|---|
| Untrust | eth1/1 (172.16.50.2/24) | Dual-ISP edge (R1/R2) via Switch2 |
| Trust | eth1/7 (172.16.150.100) | Internal core switch / VLANs (IT, HR, Finance) |
| DMZ | eth1/2 (192.168.20.1/24) | DMZ switch (web server, internal server) |

*Interface-to-zone mapping shown above is based on the lab topology; verify this matches the actual zone assignment configured on the firewall.*

## Security Policies

- **DMZ → Untrust (Web Server Publishing):** Explicit allow rule permitting the DMZ web server to reach the Untrust zone for required web services.
- **Untrust → Trust / Untrust → DMZ:** Denied by default; only traffic explicitly permitted by policy is allowed inbound from the Untrust zone.
- **Trust → Untrust (Internet Access):** Explicit allow rule permitting internal (Trust zone) users to reach the internet.

## NAT Configuration

- **Source NAT (SNAT):** Applied for Trust-zone traffic exiting to the Untrust zone, translating internal VLAN addresses to the firewall's Untrust-facing IP for internet access.
- **Destination NAT (DNAT):** Applied to publish the DMZ web server, translating traffic destined for a public/Untrust-facing address to the internal DMZ server address.
- **U-Turn NAT:** Applied so that internal (Trust zone) users can reach the DMZ web server using the same public-facing address used by external users, with NAT correctly redirecting the session back through the firewall.

## Internal Network Segmentation

| VLAN | Segment | Subnet |
|---|---|---|
| VLAN 10 | IT | 10.1.10.0/24 |
| VLAN 20 | HR | 10.1.20.0/24 |
| VLAN 30 | Finance | 10.1.30.0/24 |

Each VLAN connects through its own access switch (ASW-1/2/3) into the core switch, which uplinks to both firewalls (172.16.150.100 / .101) for HA-aware routing.

## Routing

Static routes are configured on the virtual router to reach internal subnets through the core switch:

    set network virtual-router default routing-table ip static-route to-10.1.10.0 destination 10.1.10.0/24 nexthop ip-address 172.16.150.101

## Verification Commands Used

    show session all filter source application ping
    show session id 143

## What I Tested

**HA Failover**
Shut down the active port on PA-1 (simulating a link/device failure) and monitored the HA dashboard on both firewalls. PA-2 correctly transitioned to the active role, and traffic continued passing through the firewall pair without loss of connectivity.

**VLAN Connectivity**
Pinged between endpoints on each VLAN (IT, HR, Finance) to verify reachability. Traffic was permitted as expected, and firewall traffic logs were reviewed to confirm the sessions were seen and processed correctly by the firewall.

**Note:** Inter-VLAN traffic was permitted in this lab configuration to validate routing and connectivity end-to-end. A production deployment would typically apply granular security policies to restrict inter-VLAN traffic based on business need (e.g., Finance isolated from general IT/HR access).
