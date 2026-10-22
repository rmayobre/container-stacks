# container-stacks

Docker Compose stacks for a self-hosted setup. Each stack has its own README, a `compose.yaml`, and a `template.env` to copy to `.env` and fill in.

| Stack | Description |
|---|---|
| [`pwn`](compose/pwn/) | WireGuard dashboard, Traefik reverse proxy, and Technitium DNS. **Start this first.** |
| [`authentication`](compose/authentication/) | Pocket ID (OIDC) and LLDAP. |
| [`management`](compose/management/) | Komodo for managing Docker hosts. |

Secrets, domains, and addresses are supplied through `.env` files, which should never be committed.

See [AGENTS.md](AGENTS.md) for contribution and documentation conventions.
