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

## Integration links

- Reverse proxy: https://github.com/sidey79/caddy-rproxy
- SMTP relay: https://github.com/sidey79/smtp-relay-docker

## Quick start

```bash
cp .env.example .env
mkdir -p secrets

# create required secret files
openssl rand -hex 32 > secrets/authelia_jwt_secret.txt
openssl rand -hex 32 > secrets/authelia_session_secret.txt
openssl rand -hex 32 > secrets/authelia_storage_encryption_key.txt
openssl rand -hex 32 > secrets/postgres_password.txt

# adapt config/authelia/configuration.yml for your domain
docker compose pull
docker compose up -d
```

## Security notes

- Never commit `.env` or `secrets/*.txt`.
- Replace all placeholder values in `config/authelia/configuration.yml`.
- Restrict access to services on `network_backend_net`.
