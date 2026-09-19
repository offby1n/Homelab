# Homelab

Infrastructure running across my physical hardware and public cloud instances, documented one layer of the stack at a time.

## What this is

A live working record of building platform infrastructure the way it actually gets deployed — incrementally, on real hardware, managing persistent network traffic. My core environment runs on a physical Dell R630 hypervisor using Proxmox VE, isolated behind a dedicated OPNsense firewall, and bridged to public, internet-facing cloud nodes hosting live playground domains like subnetlab.dev. 

Nothing here is a temporary sandbox setup designed to be blown away after an exercise. These systems are meant to stay up, evolve, and take live traffic.

Instead of hiding mistakes, each layer contains an engineering log tracking exactly what was built, what broke under load, how the root cause was identified, and the architectural lessons learned along the way.

## Where things stand

The engineering logs inside `writeups/` represent the literal status of the architecture. A dedicated file is committed as soon as a technical layer is initialized. The highest-numbered file reflects the stack layer I am actively profiling and hardening right now. 

## Layers & Architecture Roadmap

The platform stack is constructed systematically from the bare metal up. The roadmap outlines my long-term engineering track:

- [/] Layer 01: Networking & Edge Routing (OPNsense, Topology Routing)
- [ ] Layer 02: Linux Operations & Compute Node Hardening (Arch/Debian Systems)
- [ ] Layer 03: Container Runtimes & Orchestration Isolation
- [ ] Layer 04: Local Kubernetes Deployments & Cluster Networking
- [ ] Layer 05: Observability Platforms (Metrics Exporters, Logs, Telemetry)
- [ ] Layer 06: Infrastructure as Code & Automation (Ansible, AWS Systems)
- [ ] Layer 07: GPU Compute & Model Serving Pipelines
- [ ] Layer 08: Storage Engineering & Redundant Backups
- [ ] Layer 09: Continuous Integration & GitOps Delivery
- [ ] Layer 10: Deep Platform Hardening & Access Control

Once the base roadmap is fully deployed, the cycle loops back to Layer 01 to refactor, scale, and optimize infrastructure depth.

## Repo Layout

```text
homelab/
├── README.md
├── writeups/     # Engineering logs committed per structural layer
├── diagrams/     # Live network topology maps and state diagrams
└── network/      # Sanitized infrastructure configuration files
```

## Repository Guardrails

*   **Sanitization First:** All committed configurations are strictly sanitized. Public IP ranges, cryptographic keys, personal access tokens, and unique physical MAC addresses are stripped out entirely. Private RFC1918 internal routing architecture is intentionally retained to preserve topological accuracy.
*   **Decoupled Code:** Any internal programmatic utilities built alongside this infrastructure—such as custom metrics exporters, health checkers, traffic generators, or system CLIs—are maintained in distinct code repositories, not inside this configuration registry.

