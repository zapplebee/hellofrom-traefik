# HelloFrom Traefik Gateway

This repo contains the **internet-facing Traefik gateway** for HelloFrom.

It:

- Terminates TLS for services using **Let’s Encrypt (HTTP-01)**.
- Redirects all HTTP → HTTPS.
- Discovers backends via **Docker labels** (Docker provider).
- Exposes the Traefik dashboard/API **only on localhost** (`127.0.0.1:8080`).
- Defines a reusable **localhost-only allowlist middleware** (`hf-localonly`) that other stacks can reference.

---

## What’s in here

- `docker-compose.yml` – runs Traefik with:

  - entrypoints:

    - `web` on `:80`
    - `websecure` on `:443`
    - `traefik` (dashboard/API) on `:8080` (localhost only)

  - ACME storage at `./traefik-letsencrypt/acme.json`
  - Docker provider with `exposedByDefault=false` (nothing routes unless explicitly labeled)

- `./traefik-letsencrypt/` – persistent Let’s Encrypt state (do **not** delete unless you want to re-issue certs)

---

## Prereqs

- Docker + Docker Compose
- Ports **80** and **443** open to the internet on this host
- A DNS record pointing at this host (example: `auth.hellofrom.app`)
- A valid email configured for Let’s Encrypt

---

## Quick start

1. Create the ACME storage file with correct permissions:

```bash
mkdir -p traefik-letsencrypt
touch traefik-letsencrypt/acme.json
chmod 600 traefik-letsencrypt/acme.json
```

2. Update the Let’s Encrypt email in `docker-compose.yml`:

```yaml
- --certificatesresolvers.le.acme.email=you@hellofrom.app
```

3. Start Traefik:

```bash
docker compose up -d
```

4. Watch logs:

```bash
docker compose logs -f traefik
```

---

## Dashboard

The dashboard and API are available **only from the host**:

- `http://localhost:8080/dashboard/`

Because we bind `127.0.0.1:8080:8080`, nothing on your LAN (or the internet) can hit it directly.

If you want to view it from another machine, use SSH port forwarding:

```bash
ssh -L 8080:127.0.0.1:8080 <host>
```

Then open `http://localhost:8080/dashboard/` on your laptop.

---

## Networks

This compose file creates two Docker networks:

- `hf_edge`
  Public-facing “edge” network. Traefik lives here.

- `hf_svc_keycloak` (**internal**)
  Private service network for Traefik ↔ Keycloak traffic.

Other services can follow the same pattern:

- Create a dedicated `hf_svc_<service>` internal network
- Attach Traefik + that service to it
- Keep DBs on separate `hf_db_<service>` internal networks (Traefik should not join DB networks)

---

## How other services attach

Other stacks should:

1. Join the appropriate Traefik service network (example: `hf_svc_keycloak`)
2. Add Traefik labels to their service container(s)
3. Set `traefik.enable=true`
4. Set `traefik.docker.network=hf_svc_keycloak` when the container is on multiple networks

Example (labels on a backend container):

```yaml
labels:
  - traefik.enable=true
  - traefik.docker.network=hf_svc_keycloak
  - traefik.http.routers.myservice.rule=Host(`auth.hellofrom.app`)
  - traefik.http.routers.myservice.entrypoints=websecure
  - traefik.http.routers.myservice.tls=true
  - traefik.http.routers.myservice.tls.certresolver=le
  - traefik.http.services.myservice.loadbalancer.server.port=8080
```

---

## Localhost-only middleware

This compose defines a reusable middleware:

- `hf-localonly` = allow only `127.0.0.1` / `::1`

Other stacks can use it like:

```yaml
labels:
  - traefik.http.routers.some-private-thing.middlewares=hf-localonly@docker
```

This is handy for “admin-only” paths that you access via SSH tunnel.

---

## Troubleshooting

### Certs not issuing

- Confirm ports 80/443 are reachable from the internet.
- Confirm DNS points to this host.
- Check logs:

```bash
docker compose logs -f traefik
```

### Dashboard returns 404

- Make sure you include the trailing slash: `/dashboard/`
- Confirm you’re hitting `http://localhost:8080`, not `https://...`

### Permissions error for ACME storage

- Ensure `traefik-letsencrypt/acme.json` exists and is `chmod 600`.

---

## Notes / security posture

- Docker provider is `exposedByDefault=false`, so nothing is exposed unless labeled.
- Dashboard is bound to localhost and also protected by `hf-localonly` middleware.
- ACME state is persisted on disk in `./traefik-letsencrypt` so restarts don’t re-issue certificates.
