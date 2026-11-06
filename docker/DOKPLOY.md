# Deploying Dify on Dokploy

This guide shows how to deploy Dify on a [Dokploy](https://dokploy.com) host
using the dedicated compose file in this repository.

The setup ships PostgreSQL, Redis, Weaviate, Sandbox, SSRF proxy, Plugin
Daemon, the Agent v2 runtime (agent backend, local sandbox and its own SSRF
proxy) and an internal Nginx — all wired up so that Dokploy's Traefik can
front the stack on a domain you choose.

Current versions: Dify 1.19.0, plugin daemon 0.6.14, Weaviate 1.39.13.

## Files

| File | Purpose |
| ---- | ------- |
| `docker/docker-compose.dokploy.yaml` | Dokploy-ready compose (no `container_name`, named volumes, Traefik labels). |
| `docker/.env.dokploy.example` | Minimum env vars to set in the Dokploy UI. |
| `docker/postgres-init/01-create-plugin-db.sh` | First-boot script that creates the `dify_plugin` database (plugin_daemon expects it to exist). |

## Why a separate compose file?

The default `docker/docker-compose.yaml` is tuned for a self-managed host
with its own ports + certbot. Dokploy already provides Traefik with
Let's Encrypt and treats certain compose features as anti-patterns. The
Dokploy file fixes the common pitfalls:

1. **No `container_name`** — Dokploy attaches its logging/metrics by the
   compose-generated names; explicit `container_name` breaks both.
2. **Named volumes only** — Dokploy re-clones the repository on every
   deploy, so any bind mount under `./volumes/` would be wiped. Switching
   to named volumes also enables Dokploy's Volume Backups feature.
3. **No published host ports** — port 80/443 are owned by Dokploy's
   Traefik. Internal nginx listens on port 80 only inside the compose
   network; Traefik forwards to it via the external `dokploy-network`.
4. **External `dokploy-network`** — required so Traefik can reach the
   container at all. The `traefik.docker.network=dokploy-network` label
   tells Traefik which network to use when there are multiple.
5. **Internal nginx kept** — Dify needs path-based routing
   (`/console`, `/api`, `/socket.io`, `/e/`, `/`) onto three different
   backends. Letting Dify's nginx do that and pointing Traefik at the
   single nginx service is far simpler than recreating each route as
   Traefik labels.
6. **Certbot removed** — Dokploy already issues and renews Let's Encrypt
   certificates via Traefik.
7. **Postgres init script bundled** — `docker/postgres-init/` is
   bind-mounted into `/docker-entrypoint-initdb.d` so the `dify_plugin`
   database (required by plugin_daemon) is created on first boot. Postgres
   only runs this when the data volume is empty, so subsequent deploys
   are no-ops.

## Step-by-step

### 1. Prepare DNS

Point an `A` (or `AAAA`) record from your domain to the Dokploy host.
Do this **before** adding the domain in Dokploy so Let's Encrypt's
HTTP-01 challenge can succeed on first deploy.

### 2. Create a Compose service in Dokploy

* New Service → **Compose**
* Source: this Git repository, branch of your choice
* **Compose Path**: `docker/docker-compose.dokploy.yaml`
* Compose Type: `docker-compose`

### 3. Set environment variables

Open the **Environment** tab and paste the contents of
`docker/.env.dokploy.example`. Replace the `CHANGE_ME_*` values and the
`dify.example.com` placeholders with your real domain. Save.

Generate strong values with:

```
openssl rand -base64 42
```

at least for `DB_PASSWORD`, `REDIS_PASSWORD`, `SANDBOX_API_KEY`,
`PLUGIN_DAEMON_KEY`, `PLUGIN_DIFY_INNER_API_KEY`, and `WEAVIATE_API_KEY`.

### 4. Attach the domain

Domains tab → **Add Domain**:

* Host: `dify.example.com`
* Service: `nginx`
* Container Port: `80`
* HTTPS: ✅
* Certificate Provider: Let's Encrypt
* Path: `/`

Dokploy will write a Traefik router for the `nginx` service.

### 5. Deploy

Hit Deploy. First build pulls about 4–5 GB of images. After the
containers report healthy:

1. Open `https://dify.example.com`.
2. Complete the bootstrap wizard with the email of your choice and the
   `INIT_PASSWORD` you set in the env tab.

## Common gotchas

* **Bad Gateway / 404 for ~30s after deploy** — Traefik needs about
  10 seconds to attach to the new container, then nginx needs to wait
  for `api` and `web` to become healthy. Wait it out.
* **`dokploy-network not found`** — only happens if you copy this
  compose to a host without Dokploy. Create the network manually:
  `docker network create dokploy-network`.
* **Certificate not issued** — confirm the DNS record resolves to the
  Dokploy host before adding the domain; redeploy after fixing DNS.
* **WebSocket disconnects** — make sure `NEXT_PUBLIC_SOCKET_URL` uses
  `wss://` (not `https://`) and points to the same host as the rest of
  the URLs.
* **Lost data on redeploy** — verify volumes are listed under the
  service's Volumes tab in Dokploy. If you see bind mounts pointing to
  `./volumes/...`, you are deploying the wrong compose file.
* **Plugins can't reach api** — `PLUGIN_DIFY_INNER_API_URL` must stay
  `http://api:5001` (the internal compose hostname), not the public URL.
* **Forgot `INIT_PASSWORD`** — bring the stack down, set a new value in
  the env tab, scale `db_postgres` volume down/up, and redeploy. Easier:
  set it once on the very first deploy.

## Switching the vector store

The default file uses Weaviate. To use Qdrant/Milvus/pgvector instead:

1. Replace the `weaviate` service block with the desired service from
   `docker/docker-compose.yaml` (and add its named volume to the
   `volumes:` section at the bottom).
2. Override `VECTOR_STORE` and the relevant endpoint vars in the env tab
   (see `docker/envs/vectorstores/*.env.example` for the full list).
3. Redeploy.

## Scaling tips

* `SERVER_WORKER_AMOUNT` (api Gunicorn workers) and
  `CELERY_WORKER_AMOUNT` (worker concurrency) are the two main knobs.
  Start at `2`/`2` on a 4 vCPU box.
* `NGINX_CLIENT_MAX_BODY_SIZE` defaults to `100M`. Raise it if you allow
  larger document uploads.
* Run Dokploy's Volume Backup on `db_postgres_data`, `app_storage`,
  `plugin_daemon_storage`, `weaviate_data`, and
  `agent_local_sandbox_workspace`.

## Updating Dify

Pin the image tags inside `docker-compose.dokploy.yaml`
(`dify-api`, `dify-web`, `dify-agent-backend`, `dify-agent-local-sandbox`,
`dify-sandbox`, `dify-plugin-daemon`) and redeploy. Dokploy will pull the
new tags and rebuild only the affected containers; named volumes are
preserved. Database migrations run automatically on `api` startup.

Before each upgrade, compare against the upstream compose
(`docker/docker-compose.yaml`). It is the source of truth for new
services, mounted templates and required variables. The Dokploy file
mounts nginx and squid templates from the repo, so a template change
upstream can require a matching mount or env var here.

### Upgrading from 1.14.x to 1.19.0

1. **Back up first.** Run a Volume Backup of `db_postgres_data` and
   `weaviate_data`. Weaviate jumps from 1.27 to 1.39 and the API runs
   database migrations on first start.
2. **Add the new secrets** in the Environment tab (see
   `.env.dokploy.example`, section "Agent v2"):
   `DIFY_AGENT_LOCAL_SANDBOX_AUTH_TOKEN` and `DIFY_AGENT_API_TOKEN`.
   `DIFY_AGENT_SERVER_SECRET_KEY` is optional but has a strict format:
   unpadded base64url that decodes to 32 bytes. Generate it with
   `openssl rand 32 | base64 | tr '+/' '-_' | tr -d '=\n'`.
3. **Redeploy.** Three new services start: `agent_backend`,
   `local_sandbox` and `agent_ssrf_proxy`.

What changed in the stack:

* `ssrf_proxy` now also mounts `squid-common.conf.template`. Without it the
  new entrypoint fails and Squid does not start.
* New optional `SSRF_PROXY_ALLOW_PRIVATE_IPS` /
  `SSRF_PROXY_ALLOW_PRIVATE_DOMAINS` let HTTP nodes, tools and plugins reach
  private addresses.
* `RESPECT_XFORWARD_HEADERS_ENABLED=true` matches the new nginx templates,
  which forward the external host and port to the API.
* The Celery broker URL now follows `REDIS_PASSWORD`. Earlier versions of
  this file kept the default password in it, which broke the worker when
  the Redis password was changed.
