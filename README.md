# authelia-docker

Authelia SSO/OIDC stack for internal services.

## Included

- `docker-compose.yml` with:
  - `authelia`
  - `postgresql`
  - `redis`
- `config/authelia/configuration.yml` baseline config
- `.env.example` as secret-free template
- `.gitignore` to avoid committing local secrets

## Networking model

- No host ports are published.
- `authelia` is attached to:
  - `authelia_internal` (internal app network)
  - external proxy network `network_backend_net`
- external SMTP network `smtp_relay_net` (from `smtp-relay-docker`)
- Ensure `network_backend_net` exists before starting this stack.
- Ensure `smtp_relay_net` exists before starting this stack.

## Persistence model

- Persistent data is bind-mounted from host paths via `BASE_STACK_DATA_PATH`.
- Default in `.env.example`: `/opt/docker/authelia`
- Effective paths:
  - `${BASE_STACK_DATA_PATH}/authelia`
  - `${BASE_STACK_DATA_PATH}/postgresql`
- Secrets path:
  - `${BASE_STACK_DATA_PATH}/secrets` (bind-mounted read-only into containers at `/run/authelia-secrets`)
  - user database file: `${BASE_STACK_DATA_PATH}/secrets/users_database.yml`
  - OIDC HMAC secret: `${BASE_STACK_DATA_PATH}/secrets/oidc_hmac_secret`
  - OIDC issuer key: `${BASE_STACK_DATA_PATH}/secrets/oidc_jwks_rsa_private_key.pem`
  - Vaultwarden OIDC client secret digest: `${BASE_STACK_DATA_PATH}/secrets/vaultwarden_oidc_client_secret_digest`
- Custom trusted CAs for Authelia:
  - `${ROOT_CA_CERT_HOST_PATH}` bind-mounted read-only to `/certificates/root-ca.cert.pem`
- Redis is intentionally non-persistent (session loss after Redis/container restart is accepted).

## Container user

- Authelia runs as non-root user via `AUTHELIA_UID`/`AUTHELIA_GID`.
- Default values in `.env.example`: `1007:1007`.
- Ensure the Authelia data path ownership matches this user, e.g.:
  - `chown -R 1007:1007 /opt/docker/authelia/authelia`

## Integration links

- Reverse proxy: https://github.com/sidey79/caddy-rproxy
- SMTP relay: https://github.com/sidey79/smtp-relay-docker

## Paperless access control

- `dms.sidey.blausee.eu` is restricted to the Authelia group `paperless-users`.
- Group membership is managed in `${BASE_STACK_DATA_PATH}/secrets/users_database.yml`.
- After changing group membership, restart the Authelia container so the file-based
  user database is reloaded consistently.

## Quick start

```bash
cp .env.example .env
# adapt .env and config/authelia/configuration.yml for your domain
mkdir -p /opt/docker/authelia/authelia /opt/docker/authelia/postgresql
mkdir -p /opt/docker/authelia/secrets
openssl rand -hex 32 | tr -d '\n' > /opt/docker/authelia/secrets/reset_password_jwt_secret
openssl rand -hex 32 | tr -d '\n' > /opt/docker/authelia/secrets/session_secret
openssl rand -hex 32 | tr -d '\n' > /opt/docker/authelia/secrets/storage_encryption_key
openssl rand -hex 32 | tr -d '\n' > /opt/docker/authelia/secrets/postgres_password
openssl rand -hex 64 | tr -d '\n' > /opt/docker/authelia/secrets/oidc_hmac_secret
openssl genrsa -out /opt/docker/authelia/secrets/oidc_jwks_rsa_private_key.pem 2048
printf 'users: {}\n' > /opt/docker/authelia/secrets/users_database.yml
chown -R root:1007 /opt/docker/authelia/secrets
chmod 750 /opt/docker/authelia/secrets
chmod 640 /opt/docker/authelia/secrets/*
chown root:root "${ROOT_CA_CERT_HOST_PATH}"
chmod 644 "${ROOT_CA_CERT_HOST_PATH}"
docker compose pull
docker compose up -d
```

## Manage users

Generate a password hash with the running container:

```bash
docker exec authelia authelia crypto hash generate argon2 --password 'CHANGE_ME'
```

Then add the user entry to `${BASE_STACK_DATA_PATH}/secrets/users_database.yml` and restart Authelia.

## Vaultwarden OIDC

Generate the OIDC HMAC secret and issuer key once and keep both files stable
across deployments:

```bash
openssl rand -hex 64 | tr -d '\n' > /opt/docker/authelia/secrets/oidc_hmac_secret
openssl genrsa -out /opt/docker/authelia/secrets/oidc_jwks_rsa_private_key.pem 2048
chown root:1007 /opt/docker/authelia/secrets/oidc_hmac_secret /opt/docker/authelia/secrets/oidc_jwks_rsa_private_key.pem
chmod 640 /opt/docker/authelia/secrets/oidc_hmac_secret /opt/docker/authelia/secrets/oidc_jwks_rsa_private_key.pem
```

Do not regenerate these files during normal updates. Changing them invalidates
existing OIDC sessions and refresh tokens.

Generate the Vaultwarden OIDC client secret digest with the same plaintext secret
used as `SSO_CLIENT_SECRET` in the Vaultwarden stack:

```bash
docker exec authelia authelia crypto hash generate pbkdf2 --variant sha512 --password 'CHANGE_ME'
```

Store only the generated digest in
`${BASE_STACK_DATA_PATH}/secrets/vaultwarden_oidc_client_secret_digest`.
Users in the `vaultwarden-admins` Authelia group receive the Vaultwarden `admin`
role; users in `vaultwarden-users` receive the `user` role. Existing users can
still log in if their Vaultwarden email address matches the Authelia email claim.

## Security notes

- Never commit `.env`.
- Never commit `${BASE_STACK_DATA_PATH}/secrets/*`.
- Replace all placeholder values in `.env`.
- Keep `${BASE_STACK_DATA_PATH}/secrets` readable only for privileged users.
- Restrict access to services on `network_backend_net`.
- Keep `X_AUTHELIA_CONFIG_FILTERS=template` enabled; the OIDC key and client
  secret digest are loaded with Authelia's configuration template filter.
