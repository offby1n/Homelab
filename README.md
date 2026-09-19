# Homelab

Infrastructure I run at home and on a public VPS, documented one layer of the stack at a time.

## What this is

A working record of building platform infrastructure the way it actually gets built — one layer at a time, each running on real hardware with real services on top. The systems are live: a Dell R630 on Proxmox, an OPNsense firewall in front of the house network, a public internet-facing VPS, and services other people use. Nothing here is a lab exercise that gets deleted afterwards.

Each layer gets one write-up that grows as the work happens — what I built, what broke, how I found the cause, and what I'd do differently.

## Where things stand

The write-ups in `writeups/` are the status: a file exists for a layer once it has work done, and the highest-numbered one is what I'm on now. That folder is the source of truth, so this README doesn't repeat it.

## Network

![Home network topology](diagrams/homelab.map.preview.png)

The current network map. An OPNsense VM on the Dell R630 routes and firewalls the home network. Addressing, topology detail and the reasoning live in the Layer 01 write-up; this diagram is replaced as the network changes.

## Layers

The stack, in the order I build it — each opens a write-up in `writeups/` when I start it:

1. Networking
2. Linux ops
3. Containers
4. Kubernetes
5. Observability
6. Automation & IaC (Ansible, Terraform, AWS)
7. GPU & model serving
8. Storage & backups
9. CI/CD & GitOps
10. Hardening

After layer 10, the list repeats from the top, one notch deeper.

## Repo layout

    homelab/
    ├── README.md
    ├── writeups/     one file per layer, added as each layer opens
    ├── diagrams/     the network map
    └── network/      sanitised configs and notes for the current layer

## Notes

- Everything committed here is sanitised: no public addresses, keys, credentials or hardware identifiers. Private RFC1918 addressing is left in because the topology is meaningless without it.
- Python tooling written alongside this work — exporters, health checkers, load generators, CLIs — lives in its own repositories, not here.
