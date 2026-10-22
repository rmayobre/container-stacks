# Authentication Stack

Single sign-on and user directory.

| Service | Image | Purpose |
|---|---|---|
| `pocket-id` | `ghcr.io/pocket-id/pocket-id:v2` | Passkey-based OIDC identity provider. Port `1411`, exposed through Traefik at `POCKET_ID_HOST`. |
| `lldap` | `lldap/lldap:stable` | Lightweight LDAP server with web UI (`17170`) and LDAP (`3890`, published through Traefik on the `ldap` entrypoint). |

## Prerequisites

- The **PWN stack** (`../pwn`) running: it creates the external `proxy-network` Docker network and provides Traefik (including the `websecure` and `ldap` entrypoints and the `letsencrypt` resolver) and internal DNS.
- DNS records for `POCKET_ID_HOST` and `LLDAP_HOST` pointing at the Docker host.
- Data directories: `sudo mkdir -p /opt/pocketid/data /opt/lldap/data`
- Pocket ID runs with `PUID`/`PGID` 1000 by default; make sure that user can write to `/opt/pocketid/data`.
- Generated secrets: `openssl rand -base64 32` for the Pocket ID encryption key, and `openssl rand -hex 32` for `LLDAP_JWT_SECRET` and `LLDAP_KEY_SEED`.

## Setup

1. `cp template.env .env` and fill in:
   - `POCKET_ID_HOST`, `LLDAP_HOST`: hostnames (no scheme) for each service.
   - `INTERNAL_DNS_IP`: IP of your internal DNS server.
   - `POCKET_ID_ENCRYPTION_KEY`: the Pocket ID encryption key. Back it up; losing it makes the stored data unreadable.
   - `APP_URL`: full public URL of Pocket ID, e.g. `https://auth.example.com`.
   - `TRUST_PROXY=true` (leave it, since Traefik fronts the service), optionally `MAXMIND_LICENSE_KEY`, `PUID`, `PGID`.
   - `TZ`, `LLDAP_JWT_SECRET`, `LLDAP_KEY_SEED`, `LLDAP_LDAP_BASE_DN` (e.g. `dc=example,dc=com`), and `LLDAP_LDAP_USER_PASS` (password for the LLDAP `admin` user).
2. `docker compose up -d`

## Follow-up configuration

- **Pocket ID**: open `APP_URL/setup` to create the first admin account and register a passkey right away. Then create OIDC clients for your apps (e.g. Komodo; see the management stack's OIDC variables) and note each client ID/secret and callback URL.
- **LLDAP**: log in at `LLDAP_HOST` as `admin` with `LLDAP_LDAP_USER_PASS`, then create users and groups. Its data (database and generated config) lives in `/opt/lldap/data`; back it up along with `LLDAP_KEY_SEED` and `LLDAP_JWT_SECRET`. `LLDAP_LDAP_USER_PASS` only sets the admin password on first start; change it afterwards in the UI. See the [LLDAP docs](https://github.com/lldap/lldap) for more options (LDAPS, SMTP, etc.).
- To use LDAP in Pocket ID, configure it under Pocket ID's settings with the LLDAP bind DN and base DN. Pocket ID can reach LLDAP at `lldap:3890` over the shared `proxy-network`.
- LLDAP uses its default SQLite database in `/data`. To use MySQL/MariaDB/PostgreSQL instead, set `LLDAP_DATABASE_URL`.

## Notes

- The raw LDAP route uses `HostSNI(`*`)` on the `ldap` entrypoint (port 389 on Traefik) with no TLS; restrict it to your internal network.
- `ldap-net` (`10.10.0.0/16`) is a bridge network that only `lldap` currently joins; other containers that need LDAP can attach to it. Adjust the subnet if it conflicts with your network.

## Official documentation

- Pocket ID: https://pocket-id.org/docs
- LLDAP: https://github.com/lldap/lldap
- Traefik (reverse proxy and TCP routing): https://doc.traefik.io/traefik/
