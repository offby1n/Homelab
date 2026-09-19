# Layer 01 — Networking & Core Routing

The structural network baseline everything else in the lab builds upon: edge routing, stateful firewalling, CIDR addressing, and encrypted remote access. The physical Dell R630 hypervisor hosts the core gateway environment—running OPNsense as a top-priority virtual machine that processes all local area network (LAN) traffic. Upstream, the ISP perimeter router (a CosmOTE Speedport) sits between the server and the public internet fabric.

**Status:** In Progress (Active Focus)

## Operational Scope

This foundational infrastructure layer is considered closed when I can execute each of the following tasks on my local lab environment without documentation open, and explain the underlying engineering concepts in plain English:

- **Subnetting Mechanics:** Manual binary calculation of CIDR blocks, network masks, network/broadcast boundaries, usable host pools, and VLSM splitting of a /24 allocation.
- **Stateful DHCP & Local DNS:** Stateful DHCP space management with static IP reservations and recursive vs. authoritative local DNS resolution.
- **Network Address Translation:** Configuring incoming port forwards and understanding Layer 3 outbound NAT mechanics.
- **VLAN Segmentation:** Constructing 802.1Q tagged interfaces and enforcing strict inter-VLAN firewall rulesets.
- **Stateful Packet Inspection:** Default-deny firewall logic, state tracking, and analyzing real-time block logs.
- **Secure Remote Access:** Engineering a resilient WireGuard road-warrior tunnel topology.
- **TCP/IP Fundamentals:** Analyzing the 3-way handshake, TCP vs. UDP performance profiles, ARP binding, TCP resets (RST), and handling MTU boundaries.
- **Network Observability:** Mastering CLI network diagnostics using `ip`, `ss`, `dig`, `nmap`, `iperf3`, `tcpdump`, and parsing packet captures (`pcap`) inside Wireshark.
- **Topological Documentation:** Committing a live, verifiable network diagram to this repository.

## Current Network Topology

```text
ISP Gateway (Dynamic Public IPv4, No CGNAT)
└── CosmOTE Speedport (Edge Router, 192.168.1.1 — Holds Public IP)
    └── Dell PowerEdge R630 (Proxmox VE Host, WAN Interface Fixed at 192.168.1.11 via Speedport DHCP Reservation)
        └── OPNsense VM (Core Firewall Engine)
            ├── WAN (vtnet0) → 192.168.1.11 (Double-NAT Boundary)
            └── LAN (vtnet1) → 10.10.10.1/24 (DHCP Scope: .100–.200 via Unbound/DHCP Core)
                └── Layer 2 Core Switch
                    ├── Wireless Access Point (WAP Clients — DHCP Segment)
                    ├── iDRAC Management Interface (10.10.10.x Static Out-of-Band Node)
                    └── Proxmox VE Hypervisor (10.10.10.10/24 Static Interface on vmbr0)
```

### Physical Handoff (R630 Interface Bindings)
*   **nic0:** Dedicated WAN link to the CosmOTE Speedport LAN handoff.
*   **nic2:** Shared LAN and host management infrastructure interface (`vmbr0`).
*   **nic1:** Completely isolated local laboratory segment (`vmbr9`).
*   **nic3:** Unallocated hot-spare interface.

*Architectural Note:* Current addressing relies on a single, flat `10.10.10.0/24` broadcast domain. All endpoints, hypervisors, management platforms, and wireless devices temporarily share the same security posture. This is a deliberate, transitional phase. Hardened 802.1Q VLAN segmentation is the current development sprint and will replace this section upon completion.

## Active Engineering Backlog
- [ ] Implement isolated VLAN trunks (Management, Trusted LAN, Untrusted Wireless, Lab Sandbox).
- [ ] Enforce zero-trust default-deny rules between VLAN interfaces.
- [ ] Compile comprehensive documentation for stateful firewall policies.

---

## 1. Migrating OPNsense Core Router to Virtualized Infrastructure

**Date:** September 2026

### Architectural Motivation
The edge firewall was previously running on a legacy bare-metal laptop. Deploying consumer hardware at the network perimeter introduced single points of failure across the entire domestic topology. The system lacked hardware redundancy, storage snapshots, automated failover capabilities, or out-of-band serial consoles for rapid recovery. Migrating OPNsense to a high-availability virtual machine on the enterprise Dell R630 leverages severe server-grade advantages: reliable power paths, iDRAC out-of-band outlays, atomic ZFS boot snapshots, and explicit hypervisor boot prioritizations.

### Execution Logs
1.  **Backup & Export:** Extracted a master cryptographic `config.xml` profile from the legacy bare-metal edge laptop.
2.  **VM Provisioning:** Built VM ID `102` on Proxmox VE utilizing two high-performance `virtio-net` network interfaces mapped to independent hardware bridges (`vtnet0` for WAN, `vtnet1` for LAN).
3.  **Bootstrap & Restore:** Deployed a fresh installation of OPNsense from the official ISO image onto a native UFS file structure. Imported the `config.xml` backup and remapped the target interface abstractions to the newly instantiated `virtio` naming conventions.
4.  **Edge Integration:** Connected the R630's assigned WAN NIC directly into the LAN side of the CosmOTE Speedport. Configured the OPNsense WAN interface to pull `192.168.1.11` via an upstream DHCP reservation. The resulting double-NAT environment is an accepted engineering tradeoff for this project phase to bypass putting the ISP residential gateway into fragile bridge states.
5.  **Perimeter Availability:** Enforced strict hypervisor policies: modified VM boot ordering to position `102` first on host initialization to guarantee internet availability before dependent guest workloads boot. Enabled automated kernel panic restart tracking.
6.  **State Capture:** Committed an atomic Proxmox snapshot post-verification to establish a permanent rollback recovery baseline.

### Root-Cause Diagnostics (Time Reductions)
*   **Installer Environment Trap:** The FreeBSD-based installer payload requires explicit console execution via the `installer` daemon context. Authenticating as generic `root` drops the shell into a standard live volatile memory landscape without triggering partition routines, resulting in silent installation loops on reboot.
*   **ISO Boot Priority:** Failure to unmount or detach the loopback virtual optical media device prior to initial host reboots causes the system firmware to cycle directly back into the installation media sequence, rather than falling back to the local storage target blocks.

### Engineering Outcomes & System Constraints
The local environment routes cleanly through the virtualized OPNsense platform. The edge gateway effortlessly survives host power cycles without human intervention. The hypervisor now sits on the LAN behind a stateful environment it is actively computing. 
*   **Recovery Vectors:** Because the hypervisor depends on a guest machine for external routing, the local out-of-band iDRAC configuration must remain accessible on a hardened static configuration as the primary emergency fallback.
*   **NAT Layers:** Upstream port forwards must be synchronously mirrored on both the CosmOTE Speedport and the local OPNsense gateway to maintain public reachability.
*   **Lack of Micro-segmentation:** Until the VLAN roadmap item is resolved, a compromise on a wireless device grants immediate local discovery vectors to internal hypervisor management layers.

---

## 2. Engineering Secure WireGuard Remote Access via Dynamic DNS

**Date:** September 2026

### Architectural Motivation
The infrastructure workspace requires reliable, low-latency, encrypted access from remote clients without exposing standard management planes to the public internet. Because the residential ISP path utilizes a dynamic public IPv4 allocation without Carrier-Grade NAT (CGNAT), a mechanism was required to dynamically track edge routing changes and map client configurations cleanly.

### Execution Logs
1.  **DDNS Synchronization:** Provisioned a custom domain and configured `os-ddclient` on OPNsense using a scoped Cloudflare API token constrained strictly to zone DNS edits. Established a non-proxied (grey-cloud) `vpn` record to ensure native UDP traffic passes cleanly without getting intercepted or terminated by Cloudflare's Layer 7 reverse proxy engines.
2.  **Tunnel Configuration:** Provisioned a virtual WireGuard interface engine (`100.100.100.1/24`, binding to UDP port `51820`). Configured strict firewall interface rules: allowing traffic originating from the `WireGuard` network block to talk explicitly to specified `LAN` resources, enforcing a hard block on everything else.
3.  **NAT Traversal Routing:** Engineered a synchronous nested port-forward mapping: forwarding public traffic on UDP port `51820` from the CosmOTE Speedport down to the OPNsense WAN interface (`192.168.1.11`), which then transparently passes packets directly into the internal WireGuard listening daemon.
4.  **Endpoint Provisioning:** Generated independent asymmetric keypair peers (Mobile Endpoint: `100.100.100.2/32`, Remote Workstation: `100.100.100.3/32`). Enforced highly specific split-tunnel parameters (`AllowedIPs = 10.10.10.0/24`) to guarantee non-lab web traffic bypasses the tunnel. Configured the workstation tunnel sequence as a persistent native `systemd` unit (`wg-quick@homelan`) to survive client-side restarts.

### Root-Cause Diagnostics (Time Reductions)
