# Project Write-up: Deploying and Hardening subnetlab.dev on AWS

This document serves as a standalone infrastructure project review. It tracks the design, deployment, and security hardening of a public, internet-facing platform utility built completely from scratch.

**Live Architecture:** [https://subnetlab.dev](https://subnetlab.dev)

## System Overview

The utility is a browser-native IPv4 subnetting and VLSM breakdown engine. It takes input strings in CIDR block notation and computes comprehensive mathematical breakdowns—including netmasks, network bounds, broadcast vectors, and host allocation pools—completely client-side via vanilla JavaScript. 

The application frontend (HTML/CSS/JS compiled as a static artifact) was intentionally generated using artificial intelligence pipelines. The interface code was never the engineering objective of this sprint; rather, the frontend serves as a real-world static asset used to validate my system deployment skills—architecting the virtual machine layer, configuring server runtimes, setting up cryptographic transport policies, managing public DNS zones, and locking down the host operating system.

## The Infrastructure Stack

The layout below outlines the complete deployment pipeline used to take a bare virtual compute node and harden it into a secure, production-grade web endpoint:

### 1. Cloud Compute & Virtualization (AWS EC2)
- **Node Allocation:** Provisioned an AWS EC2 `t3.micro` compute instance running inside the AWS `eu-north-1` (Stockholm) availability zone. 
- **Operating System Baseline:** Deployed Amazon Linux 2023 (AL2023) as the minimal, performance-optimized operating system footprint.
- **Access Management:** Terminated generic administrative access paths; configured highly restricted SSH authentication keys coupled with hardened system policies.

### 2. Edge Routing, Firewalls & Security Groups
- **Network Access Control Lists:** Configured a strict external AWS Security Group acting as the stateless firewall perimeter. 
- **Ingress Policies:** Opened access strictly to public web traffic ports (TCP `80` for HTTP validation, TCP `443` for HTTPS transport), and locked down administrative SSH access (TCP `22`) to limited, verified source networks.
- **Egress Policies:** Outbound network connections are restricted purely to trusted update mirrors to pull package dependencies securely, minimizing data exfiltration vectors.

### 3. Server Engineering & Reverse Proxy Optimization (Nginx)
- **Web Runtime Selection:** Deployed Nginx as a high-throughput, resource-efficient engine designed to parse and serve static web assets.
- **Config Customization:** Stripped out generic default server configuration footprints to prevent fingerprinting. Engineered a highly customized server block structure to isolate site tracking.

### 4. Transport Encryption & SSL/TLS Management (Let's Encrypt)
- **Cert Provisioning:** Implemented Let's Encrypt automated public-key certificate infrastructure using the `certbot` daemon engine.
- **Automated Renewals:** Configured an asynchronous local system cron schedule to execute automated validation checks, eliminating certificate expiration risks.
- **Encryption Profile Tuning:** Hardened the server block parameters to enforce cryptographic protocols (disabling legacy TLS 1.0/1.1; mandating TLS 1.2 and TLS 1.3 execution contexts).

### 5. Perimeter Protection & Edge Proxy Integration (Cloudflare)
- **DNS Zone Management:** Migrated authoritative name server controls directly into the Cloudflare global edge network.
- **Proxy Acceleration:** Enabled full edge proxying (orange-cloud routing) to mask the underlying AWS EC2 origin IP address entirely from public visibility, neutralizing raw direct-to-origin DDoS paths.
- **Transport Strictness:** Enforced Full Strict SSL configuration profiles, requiring validated cryptographic validation exchanges along the entire transit track (Client → Cloudflare Edge → AWS Origin Node).

### 6. Edge Security Hardening & Defensive Headers
- **Security Headers Injected:** Engineered custom server parameters to inject strict protection mechanisms directly into all outgoing HTTP headers:
  * `Strict-Transport-Security` (HSTS) to mandate long-term local browser HTTPS rewriting.
  * `X-Content-Type-Options: nosniff` to eliminate MIME-type sniffing vulnerabilities.
  * `X-Frame-Options: DENY` to completely block clickjacking iframe injection exploits.
  * Hardened Content Security Policies (CSP) to restrict JavaScript execution strictly to the browser's local sandbox scope.

## Key Architectural Takeaways

This infrastructure sprint successfully documents my ability to handle end-to-end web deployment pipelines:
*   Transitioning from an abstract domain purchase to a highly optimized public web application.
*   Enforcing multi-tier network boundaries using Cloudflare edge proxies combined with nested AWS Security Groups.
*   Hardening Linux host environments, eliminating fingerprinting paths, and configuring secure transport parameters to prioritize performance and visibility under deep real-world inspection.
