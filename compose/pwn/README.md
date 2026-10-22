# Private Wireguard Network (PWN)

The foundation stack. It provides the VPN, the reverse proxy, and internal DNS that the other stacks depend on.

| Service | Image | Purpose |
|---|---|---|
| `wgdashboard` | `ghcr.io/wgdashboard/wgdashboard` | Web UI for managing WireGuard / AmneziaWG servers and clients (UI on `10086`, VPN on `51820/udp`). |
| `socket-proxy` | `tecnativa/docker-socket-proxy` | Read-only, filtered Docker API access for Traefik, so Traefik doesn't mount the raw Docker socket. |
| `traefik` | `traefik:v3` | Reverse proxy with automatic wildcard Let's Encrypt certificates via the Linode DNS challenge. Ports `80`, `443`, and `389` (LDAP entrypoint for the authentication stack). |
| `dns-server` | `technitium/dns-server` | Internal DNS server (port `53`), with optional ad blocking, so internal hostnames resolve to your services. |

This stack **creates** the `proxy-network` Docker network (`10.2.0.0/24`). The other stacks join it as an external network, so start this stack first.
Static addresses used on that network: `10.2.0.2` DNS, `10.2.0.3` Traefik, `10.2.0.4` Pocket ID, `10.2.0.5` Komodo, `10.2.0.6` LLDAP.

## Prerequisites

- Docker with the Compose plugin.
- A domain whose DNS is hosted at **Linode** (the Traefik DNS challenge is configured for the Linode provider; to use another provider, change `dnschallenge.provider` and the token env var; see the [Traefik ACME docs](https://doc.traefik.io/traefik/https/acme/#providers)).
- A Linode personal access token with read/write access to Domains.
- Host directories (the compose file bind-mounts these):
  ```
  sudo mkdir -p /opt/wgdashboard/{aconf,conf,data} /opt/traefik/{certs,logs,sconfig,endpoints} /opt/technitium/dns
  ```
- `/opt/traefik/sconfig/config.yml` must exist (it is mounted read-only). An empty file is fine to start with.
- Port 53 on the host must be free. On many Linux distros `systemd-resolved` listens on it; disable its stub listener (`DNSStubListener=no` in `/etc/systemd/resolved.conf`) first.
- Ports 80/443 forwarded to the host if you want it reachable from outside, and `51820/udp` forwarded for WireGuard.

## Setup

1. `cp template.env .env` and fill in:
   - `DOMAIN`: base domain; the certificate covers it and `*.DOMAIN`.
   - `ACME_EMAIL`: Let's Encrypt contact email.
   - `TRAEFIK_DASHBOARD_HOST`: hostname for the Traefik dashboard.
   - `TRAEFIK_LINODE_TOKEN`: the Linode API token.
   - `DNS_SERVER_HOST`: hostname for the DNS server (also used as its container hostname).
   - `DNS_ADMIN_PASSWORD`: initial Technitium admin password.
   - Image tag variables (`WGDASHBOARD_IMAGE_TAG`, `TRAEFIK_IMAGE_TAG`, `SOCKET_PROXY_IMAGE_TAG`, `TECHNITIUM_IMAGE_TAG`) are pre-filled with pinned versions. Upgrade by changing them and running `docker compose pull && docker compose up -d`.
   - Blank required secrets make `docker compose` stop with a message naming the missing variable.
2. `docker compose up -d`

## Follow-up configuration

- **Traefik**: the first certificate request can take a minute or two. Watch `/opt/traefik/logs/traefik.log`. The `acme.json` is stored in `/opt/traefik/certs`. The dashboard is only reachable at `TRAEFIK_DASHBOARD_HOST`, so create a DNS record for it (see below).
- **Technitium DNS** (`https://DNS_SERVER_HOST`, or temporarily publish port `5380`):
  - Log in as `admin` with `DNS_ADMIN_PASSWORD` and change it if it was left as the default.
  - Create a primary zone for your domain and add A records pointing the hostnames used by your stacks (dashboard, dns, auth, account, komodo, etc.) at the Docker host's LAN IP.
  - Set upstream forwarders (e.g. `9.9.9.9`, `1.1.1.1`) under Settings, and optionally enable blocking and add block-list URLs.
  - Point your router / clients at this server. Other stacks take its IP as `INTERNAL_DNS_IP`.
  - Many more options are available as commented environment variables in `compose.yaml`.
- **WGDashboard** (`http://HOST:10086`): the default login is `admin` / `admin`; change it immediately. Create a WireGuard configuration listening on `51820`, then add peers. To serve the dashboard through Traefik, add a file to `/opt/traefik/endpoints` (the dynamic file provider directory).

## Host firewall (recommended)

This setup is intended to be reachable only through the WireGuard VPN or the local network. Only `51820/udp` (WireGuard) needs to be reachable from the internet. Everything else (`80`, `443`, `389`, `53`, `10086`, and `9120` / `8120` from the management stack) should be restricted to the VPN subnet and LAN.

> **Docker bypasses `ufw` and `firewalld` rules for published ports.** Docker adds its own iptables rules, so `ufw deny 443` has no effect on a published container port. Filter in the `DOCKER-USER` chain (traffic to containers), or bind published ports to a specific interface IP (e.g. `192.168.1.10:443:443`).

Example (replace `eth0` with your WAN-facing interface; rules in `DOCKER-USER` are evaluated top to bottom, and inserting places each rule first, so these are listed in reverse):

```
# 3) drop all other new traffic arriving from the internet to containers
sudo iptables -I DOCKER-USER -i eth0 -j DROP
# 2) allow WireGuard
sudo iptables -I DOCKER-USER -i eth0 -p udp -m conntrack --ctorigdstport 51820 -j ACCEPT
# 1) allow replies to connections the containers started (ACME, image pulls, ...)
sudo iptables -I DOCKER-USER -i eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

Notes:
- Make the rules persistent (e.g. `iptables-persistent` / `netfilter-persistent`, or a systemd unit) and repeat them with `ip6tables` if the host has public IPv6.
- Traffic from the WireGuard interface and the LAN is not affected by the rules above.
- Verify from outside your network (e.g. `nmap` against the public IP): only `51820/udp` should respond.
- Let's Encrypt uses the DNS challenge, so ports 80/443 do **not** need to be open to the internet for certificates.
- The Traefik dashboard has no authentication middleware; the firewall limits who can reach it, but consider adding basic auth or forward auth for defense in depth against a compromised VPN peer.

## Security notes

- Traefik reads container labels through `socket-proxy` (on an internal-only network) rather than mounting the Docker socket. The proxy itself mounts the socket read-only and only allows `containers` and `networks` GET requests. Not every container is protected this way: Komodo Periphery (management stack) still mounts the raw socket and is root-equivalent on the host.
- Containers have log rotation (10 MB x 3 files) and Traefik has a healthcheck (`--ping`).
- `wgdashboard` sets `net.ipv4.ip_forward=1` and `net.ipv4.conf.all.src_valid_mark=1` so it can route VPN traffic.
- Review `socket-proxy` permissions if you add Traefik features that need more of the Docker API.
- Never commit your `.env`.

## Official documentation

- WGDashboard: https://docs.wgdashboard.dev/
- Traefik: https://doc.traefik.io/traefik/ (ACME / DNS challenge providers: https://doc.traefik.io/traefik/https/acme/)
- Technitium DNS Server: https://technitium.com/dns/ (Docker environment variables: https://github.com/TechnitiumSoftware/DNS/blob/master/DockerEnvironmentVariables.md)
- WireGuard: https://www.wireguard.com/
