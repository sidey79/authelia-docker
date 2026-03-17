# authelia-docker

Authelia SSO/OIDC stack for internal services.

## Included

- `docker-compose.yml` with:
  - `authelia`
  - `postgresql`
  - `redis`
- `config/authelia/configuration.yml` baseline config
- `config/authelia/users_database.yml` file backend placeholder
- `.env.example` as secret-free template
- `.gitignore` to avoid committing local secrets

## Networking model

- No host ports are published.
- `authelia` is attached to:
  - `authelia_internal` (internal app network)
  - external proxy network `network_backend_net`
- Ensure `network_backend_net` exists before starting this stack.

## Persistence model

- Persistent data is bind-mounted from host paths via `BASE_STACK_DATA_PATH`.
- Default in `.env.example`: `/opt/docker/authelia`
- Effective paths:
  - `${BASE_STACK_DATA_PATH}/authelia`
  - `${BASE_STACK_DATA_PATH}/postgresql`
- Redis is intentionally non-persistent (session loss after Redis/container restart is accepted).

## Integration links

- Reverse proxy: https://github.com/sidey79/caddy-rproxy
- SMTP relay: https://github.com/sidey79/smtp-relay-docker

## Quick start

```bash
cp .env.example .env
# adapt .env and config/authelia/configuration.yml for your domain
mkdir -p /opt/docker/authelia/authelia /opt/docker/authelia/postgresql
docker compose pull
docker compose up -d
```

## Security notes

- Never commit `.env`.
- Replace all placeholder values in `.env`.
- Restrict access to services on `network_backend_net`.
