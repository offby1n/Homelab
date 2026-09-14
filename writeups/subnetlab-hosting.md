# Hosting subnetlab.dev — a public tool on AWS from scratch

A standalone project write-up. Not part of the numbered layer track — this was
a self-contained "stand something up on the public internet and do the hosting
properly" exercise.

Live at: https://subnetlab.dev

## What it is

A public IPv4 subnet / CIDR calculator. You enter an address in CIDR notation
and it breaks down the network, broadcast, usable host range, masks, and
binary. It also does VLSM subnet splitting and an "is this IP in this subnet"
check. Everything runs client-side in the browser — no backend, no data leaves
the page.

The calculator front end (the single-file HTML/CSS/JS) was AI-generated. That
was deliberate: the app was never the point. I wanted a real, non-trivial
static site to host so that the hosting itself was the project — the server,
the network path, DNS, TLS, and locking the whole thing down. That part I did
by hand, and that's what this write-up is about.

## The stack I built

The full chain, from bare instance to a locked-down public site:

- AWS EC2 (t3.micro, Amazon Linux 2023, eu-north-1) running nginx as a static
  file server
- A dedicated admin user with key-only login and password-required sudo,
  keeping the default cloud user as a tested break-glass fallback
- An Elastic IP so the address survives stop/start
- nginx enabled as a systemd service so it comes back on every boot
- A security group acting as the firewall: SSH restricted to my own IP,
  HTTP/HTTPS open only where they needed to be
- DNS on Cloudflare, domain registered there too
- TLS via Let's Encrypt (certbot), with auto-renewal armed and dry-run tested
- Cloudflare put in front as a reverse proxy (DDoS protection, CDN, origin IP
  hidden)
- The EC2 firewall then locked so ports 80/443 accept traffic only from
  Cloudflare's IP ranges — the origin can't be hit directly anymore

## What actually challenged me

The app was handed to me finished. Every real problem was in the hosting,
which is exactly what I wanted.

SSH key permissions. First connection was refused outright — the private key
file was world-readable (0644) and OpenSSH won't touch a key other users can
read. `chmod 600` and it worked. Small, but a clean reminder that the client
side of key auth has rules too.

certbot failing on a "firewall problem." certbot's challenge timed out trying
to reach the box over port 80. The error said "likely firewall problem" and it
was right — I'd never actually opened HTTP/HTTPS in the security group. The
launch defaults only had SSH. Added the rules, confirmed port 80 answered from
outside, and the challenge passed.

certbot got the cert but couldn't install it. Second run issued the
certificate fine but failed to wire it into nginx: "could not find a matching
server block." The default nginx config used a catch-all `server_name _;`
instead of naming the domain, so certbot had nowhere to attach the cert. Set
`server_name subnetlab.dev www.subnetlab.dev`, reloaded, ran `certbot install`,
done.

The dynamic-IP SSH lockout. My home connection doesn't have a static IP, and
the security group's SSH rule was pinned to one address. When my IP rotated,
port 22 started timing out while the website (443) kept serving fine. That
split — site up, SSH dead — is what pointed straight at the firewall rather
than the box. Re-pinning the rule fixes it each time — a manual annoyance
rather than a solved problem. I considered SSM to remove the public SSH port
entirely but chose to keep hardened key-only SSH for now.

Locking the origin to Cloudflare — and locking myself out doing it. The goal
was to only allow Cloudflare to reach the origin. My first attempt was wrong: I
deleted the HTTP/HTTPS rules instead of scoping them, so nothing could reach
the box — Cloudflare included — and the site went down behind a Cloudflare
error page. That was the useful lesson: in a security group, no rule means no
access; there's no implicit allow. The correct approach was a Managed Prefix
List holding Cloudflare's 15 IPv4 ranges, then pointing the 80/443 rules at
that list. I added the scoped rules first, confirmed the site still loaded, and
only then removed the open ones — add-then-remove, never the reverse.

## How I proved the lockdown worked

The clean test is two commands that must give opposite results:

    # through Cloudflare — should work
    curl -I https://subnetlab.dev

    # direct to the origin IP, bypassing Cloudflare — should now time out
    curl -I --connect-timeout 10 https://<elastic-ip> -k

The first returns 200. The second hangs. Site up for real users, origin sealed
to everyone but Cloudflare. Confirming the direct path is dead mattered as much
as confirming the proxied path is alive — a lockdown you didn't test isn't a
lockdown.

## One number

The Let's Encrypt certificate is valid for 90 days, and auto-renewal is armed
and dry-run tested — so the TLS side is genuinely set-and-forget, not a thing
I'll have to remember in December.

## What I'd do differently

Scope the firewall to Cloudflare from the start with a prefix list, instead of
opening everything to the world and closing it later. The open-then-close path
is how I locked myself out twice. Building the allow-list first would have been
fewer steps and zero downtime.

## Hardening the box

Done as a follow-up session once the site was live. Every change was tested
before it was trusted — on a live public box you prove a lockdown works, you
don't assume it.

- SSH, root login off. The box was already key-only with password auth
  disabled; I turned off root SSH entirely (PermitRootLogin no). root is the
  one username every bot tries first, so removing it from the login surface
  costs nothing — I log in as my own user and sudo — and deletes the
  highest-value target.

- fail2ban. Watches SSH auth events in the systemd journal and bans an IP via
  nftables after 5 failed attempts in 10 minutes. With key-only auth nobody
  brute-forces in anyway, so here it mostly auto-drops persistent scanners and
  keeps the auth log readable. Gotcha caught on the way: the package pulled in
  firewalld and enabled it, which would have started on the next reboot and
  blocked 80/443 — I disabled it so the site stays reachable.

- nginx security headers. Five headers added via a drop-in: HSTS (force HTTPS
  on future visits), X-Frame-Options DENY (no clickjacking via iframe),
  X-Content-Type-Options nosniff (no MIME-sniffing), Referrer-Policy (don't
  leak full URLs off-site), and a Content-Security-Policy. The CSP uses
  'unsafe-inline' because the app is a single file with inline JS/CSS — a
  strict policy would refuse to run it. For a static tool with no login or user
  data that trade-off is fine; the honest note is it weakens CSP's main XSS
  protection.

- Config cleanup. Fixed a duplicated "server_name server_name" left over from
  the certbot edits — cosmetic, but wrong.

