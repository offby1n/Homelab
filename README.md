# Homelab

The infrastructure I run on my own hardware and in the cloud, and the write-ups from building it. It covers whatever I'm working on: networking, Linux, security, containers, AI infrastructure and more.

## What's running

- **Home:** a Dell R630 on Proxmox VE, behind an OPNsense firewall, with Debian VMs for a database and testing.
- **Cloud:** an AWS EC2 node hosting [subnetlab.dev](https://subnetlab.dev), behind Cloudflare, with nginx and Let's Encrypt. It only accepts web traffic from Cloudflare.
- **Automation:** a self-hosted Hermes Agent team that runs health checks and small admin jobs on the lab.

## Write-ups

Each build gets a write-up in `writeups/`: what I built, what went wrong, how I found the cause, and what I'd do differently.

- [Hermes agent team](writeups/hermes-agent-team.md)

## Repo layout

```
homelab/
├── README.md
├── writeups/     # One write-up per build
├── diagrams/     # Network topology and system diagrams
└── network/      # Sanitized configuration files
```

## Guardrails

- **Sanitized:** public IPs, keys, tokens and MAC addresses are removed before anything is committed. Private RFC1918 addressing is kept so the diagrams and configs stay accurate.
- **Code lives elsewhere:** tools built alongside this lab (health checkers, exporters, CLIs) have their own repos. See my [profile](https://github.com/offby1n).
