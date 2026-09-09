# Layer 01 — Networking

The network everything else in the lab sits on: routing, firewalling, addressing and remote access. The Dell R630 is the edge — the ISP feed lands on it, and the firewall runs as a virtual machine on that same host.

Status: in progress.

## Scope

The layer closes when I can do each of these on my own lab without a guide open, and explain it in plain English:

- Subnetting by hand — CIDR, masks, network and broadcast addresses, usable host ranges, splitting a /24
- DHCP with reservations
- DNS records, and recursive versus authoritative resolution
- NAT and port forwarding
- VLANs and inter-VLAN rules
- Stateful firewalling — default deny, and reading the block log
- WireGuard road-warrior access
- TCP/IP fundamentals — the handshake, TCP versus UDP, ARP, RST, MTU
- Tooling — `ip`, `ss`, `dig`, `nmap`, `iperf3`, `tcpdump`, and reading a capture in Wireshark
- A network diagram committed to this repo

## Current topology

![Home network topology](../diagrams/homelab.map.preview.png)

    ISP — upstream
    └── Dell R630 — Proxmox, network edge
        └── OPNsense VM
            WAN  vtnet0 → ISP uplink, address by DHCP
            LAN  vtnet1 → 192.168.10.1/24, DHCP via Dnsmasq (.100–.199)
            └── switch — L2 core
                ├── WAP — wireless clients, DHCP from OPNsense
                ├── iDRAC — static, out-of-band management
                └── Proxmox host — 192.168.10.10/24 static on vmbr0

Physical NICs on the R630: `nic0` WAN and the ISP handoff, `nic2` LAN and management (`vmbr0`), `nic1` isolated lab island (`vmbr9`), `nic3` spare.

Addressing today is one flat `/24`. Every device — servers, wireless clients, management interfaces — shares the same broadcast domain and the same firewall rules. That is deliberate for now, not finished: VLAN segmentation is part of this layer, and when it lands it gets its own entry below and this section is rewritten to match.

## Open items

- VLAN segmentation — management, trusted, wireless and lab, plus the inter-VLAN rules between them
- WireGuard road-warrior configuration
- Firewall rule documentation
- Subnetting notes

---

## 1. OPNsense moved off a dying laptop onto the R630

September 2026

### Why

The firewall was running on an old laptop that was failing. That put unreliable consumer hardware at the edge of the network, in front of every device in the house, with no snapshots, no out-of-band console and no way to recover it quickly if it died mid-day. The R630 was already running Proxmox, so moving OPNsense into a VM puts the firewall on server hardware with iDRAC, snapshot rollback and a defined boot order behind it.

### What I did

- Exported `config.xml` from the laptop installation.
- Built VM 102 on Proxmox with two virtio NICs — WAN on `vtnet0`, LAN on `vtnet1`.
- Installed OPNsense fresh from the ISO onto UFS, then imported `config.xml` and remapped the interface assignments onto the new `vtnet` names.
- Moved the ISP handoff onto the R630's WAN NIC, so the OPNsense WAN interface takes its address straight from the ISP over DHCP and the host is the network edge.
- Set the VM to start on host boot, gave it first position in the boot order so the network is up before anything that depends on it, and enabled automatic restart on crash.
- Took a snapshot once it was verified working, as the rollback point.

### What cost me time

The install kept looking like it had failed, for two separate reasons:

- The installer only runs if you log into the live environment as `installer`. Logging in as `root` drops you into the live system with no installer available, and rebooting from there just boots the ISO again — so it looks like the install silently did nothing.
- After the install completes, the VM has to be powered down and the ISO detached before booting again. A reset with the ISO still attached boots the installer a second time instead of the installed system.

### Result

Every device on the LAN now routes through the OPNsense VM, and the firewall survives a host reboot without manual intervention. The Proxmox host sits on the LAN behind a firewall it is itself hosting — its default gateway is one of its own guests. iDRAC is the way back in if that guest won't come up.

### Known constraints

- The R630 is the edge. There is nothing upstream filtering anything, so the WAN ruleset is the only thing between the internet and the LAN. Default deny inbound and the block log are load-bearing, not decoration.
- The host depends on a guest for its route off the LAN. Out-of-band access via iDRAC is the recovery path, and it needs to stay reachable and tested.
- No segmentation yet. A compromised wireless client sits in the same broadcast domain as iDRAC and the Proxmox host. VLANs are the fix, and they are the next thing in this layer.

---

