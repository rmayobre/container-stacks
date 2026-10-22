# Management Stack (Komodo)

[Komodo](https://komo.do) for managing Docker hosts, stacks, and deployments from a web UI.

| File | Contents |
|---|---|
| `compose.core.yaml` | MongoDB, Komodo Core (web UI/API, exposed via Traefik and on port `9120`), and a local Komodo Periphery agent. |
| `compose.peer.yaml` | Periphery agent only, for **additional servers** you want Core to manage. |

## Prerequisites

- Docker with the Compose plugin on every managed host.
- For `compose.core.yaml`: the **PWN stack** (`../pwn`) running, as it provides the external `proxy-network`, Traefik, and DNS; plus a DNS record for `KOMODO_HOSTNAME`.
- Host directories:
  ```
  sudo mkdir -p /opt/komodo/{keys,backups,data,config,periphery}
  ```
  (`MONGO_DB_DATA`, `MONGO_DB_CONFIG`, `COMPOSE_KOMODO_BACKUPS_PATH`, and `PERIPHERY_ROOT_DIRECTORY` in `template.env` can point elsewhere.)
- Generated secrets, e.g. `openssl rand -hex 32` for `KOMODO_PASSKEY`, `KOMODO_WEBHOOK_SECRET`, and `KOMODO_JWT_SECRET`.

## Core setup

1. `cp template.env .env` and fill in at least:
   - `KOMODO_HOSTNAME`: hostname (no scheme) for Traefik; `KOMODO_HOST`: the full URL, e.g. `https://komodo.example.com`.
   - `INTERNAL_DNS_IP`: IP of your internal DNS server.
   - `KOMODO_DB_PASSWORD` (and optionally `KOMODO_DB_USERNAME`).
   - `KOMODO_PASSKEY`, `KOMODO_WEBHOOK_SECRET`, `KOMODO_JWT_SECRET`.
   - `KOMODO_INIT_ADMIN_PASSWORD`: initial admin password (or comment out the initial-admin variables and sign up in the UI).
   - Review the remaining options (timezone, OAuth/OIDC, polling intervals); all are documented inline.
2. `docker compose -f compose.core.yaml up -d`

## Follow-up configuration

- Log in at `KOMODO_HOST` with the initial admin credentials, then change the password. After creating accounts, consider `KOMODO_DISABLE_USER_REGISTRATION=true`.
- The local Periphery is registered automatically as the first server (`KOMODO_FIRST_SERVER`; set `KOMODO_FIRST_SERVER_NAME` to name it).
- **SSO with Pocket ID** (optional): create an OIDC client in Pocket ID with callback `KOMODO_HOST/auth/oidc/callback`, then set `KOMODO_OIDC_ENABLED=true`, `KOMODO_OIDC_PROVIDER`, `KOMODO_OIDC_CLIENT_ID`, and `KOMODO_OIDC_CLIENT_SECRET`, and recreate Core.
- Back up the `MONGO_DB_*` directories and `/opt/komodo/keys`. Core writes dated database backups to `COMPOSE_KOMODO_BACKUPS_PATH`.

## Adding a peer server

On each additional host:

1. Create the directories above (at least `/opt/komodo/keys` and `/opt/komodo/periphery`) and copy `compose.peer.yaml` there.
2. Create a `.env` next to it containing `PERIPHERY_ONBOARDING_KEY=...`. Generate a key and register it in Komodo Core (see the Komodo docs for onboarding keys in your version).
3. `docker compose -f compose.peer.yaml up -d`
4. In Core, add a server with address `https://<peer-host>:8120`. Port `8120` must be reachable from Core, and only from Core.

## Security notes

- Periphery mounts the Docker socket and `/proc`; it is root-equivalent on the host. Keep port `8120` firewalled.
- `compose.core.yaml` publishes `9120` directly as well as through Traefik. Remove the `ports` entry (and uncomment `expose`) if you only want Traefik access.
- Never commit your `.env`.

## Official documentation

- Komodo: https://komo.do/docs (backups: https://komo.do/docs/setup/backup)
- Komodo Core config reference: https://github.com/moghtech/komodo/blob/main/config/core.config.toml
- Komodo Periphery config reference: https://github.com/moghtech/komodo/blob/main/config/periphery.config.toml
- MongoDB: https://www.mongodb.com/docs/
