# AGENTS.md

Docker Compose stacks intended for a **public** repository. Each stack lives in `compose/<stack>/`.

## Stack layout

Every stack directory must contain:

- `compose.yaml` (or `compose.<role>.yaml` when a stack has several files)
- `template.env`: every variable the compose files read, with blank values
- `README.md`: documentation for the stack (see below)

## No private information

- Never commit secrets, tokens, passwords, keys, real domain names, emails, or private IPs/hostnames. `.env` files must never be committed (keep `.env` in `.gitignore`).
- Compose files must reference such values as `${VARIABLE}`, and each variable must be declared in that stack's `template.env` with a blank value and a comment saying what it is and how to generate it.
- Use `example.com` for example domains in docs and comments. Do not paste real credentials, even as "examples"; for generated values, document the command (e.g. `openssl rand -hex 32`).
- Before finishing any change, grep the repo for leaked values (domains, IPs, passwords, tokens) and confirm every `${VAR}` used in a compose file exists in `template.env`.
- Validate changes with `cp template.env .env && docker compose config -q`, then delete the `.env`.

## README rules (per stack)

Each stack README must include, in this order:

1. **Title and overview**: what the stack does, with a table of services (name, image, purpose).
2. **Prerequisites**: dependencies on other stacks, external Docker networks, host directories to create, required DNS records, ports/firewall, accounts or API tokens, and generated secrets.
3. **Setup**: copying `template.env` to `.env`, what each variable means, and the exact `docker compose` command(s) to start the stack.
4. **Follow-up configuration**: manual steps after the containers are up (first login, initial admin creation, integrations, changing default passwords).
5. **Security notes**: Docker socket mounts, exposed ports, and similar caveats, where relevant.
6. **Official documentation**: links to the upstream docs for every service in the stack.

Additional rules:

- Keep the README in sync with the compose files and `template.env`; any change to a service, variable, port, network, or volume path updates the README in the same change.
- Document start-order dependencies between stacks explicitly (e.g. `pwn` creates `proxy-network` and must start first).
- Document known gaps or caveats honestly rather than leaving them implicit.
- Prefer upstream official documentation links; add a new service's docs link when adding the service.
- The root `README.md` lists every stack with a one-line description and a link; update it when adding or removing a stack.
