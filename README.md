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
- Secrets path:
  - `${BASE_STACK_DATA_PATH}/secrets` (bind-mounted read-only into containers at `/run/authelia-secrets`)
- Redis is intentionally non-persistent (session loss after Redis/container restart is accepted).

## Container user

- Authelia runs as non-root user via `AUTHELIA_UID`/`AUTHELIA_GID`.
- Default values in `.env.example`: `1007:1007`.
- Ensure the Authelia data path ownership matches this user, e.g.:
  - `chown -R 1007:1007 /opt/docker/authelia/authelia`

## Integration links

- Reverse proxy: https://github.com/sidey79/caddy-rproxy
- SMTP relay: https://github.com/sidey79/smtp-relay-docker

## Quick start

```bash
cp .env.example .env
# adapt .env and config/authelia/configuration.yml for your domain
mkdir -p /opt/docker/authelia/authelia /opt/docker/authelia/postgresql
mkdir -p /opt/docker/authelia/secrets
openssl rand -hex 32 > /opt/docker/authelia/secrets/reset_password_jwt_secret
openssl rand -hex 32 > /opt/docker/authelia/secrets/session_secret
openssl rand -hex 32 > /opt/docker/authelia/secrets/storage_encryption_key
openssl rand -hex 32 > /opt/docker/authelia/secrets/postgres_password
chown -R root:1007 /opt/docker/authelia/secrets
chmod 750 /opt/docker/authelia/secrets
chmod 640 /opt/docker/authelia/secrets/*
docker compose pull
docker compose up -d
```

## Security notes

- Never commit `.env`.
- Replace all placeholder values in `.env`.
- Keep `${BASE_STACK_DATA_PATH}/secrets` readable only for privileged users.
- Restrict access to services on `network_backend_net`.
