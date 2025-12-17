# hellofrom-bluegreen

A tiny, **Docker Compose + Traefik** setup for **blue/green cutovers** of `hellofrom.app` using **router priority**.

- **Blue** is the default live deployment.
- **Green** can be started temporarily and will take traffic (higher priority).
- Stopping green rolls traffic back to blue.

---

## What’s in here

- `docker-compose.yml`
  Runs:

  - `traefik` (edge proxy, Let’s Encrypt, JSON access logs)
  - `hellofrom_blue` (Bun container running `index.blue.js`)
  - `hellofrom_green` (Bun container running `index.js`)

- `index.blue.js`
  The blue variant of the app entrypoint.

- `index.js`
  The green variant of the app entrypoint.

- `./traefik-letsencrypt/`
  Local storage for Let’s Encrypt `acme.json`.

---

## How blue/green works

Traefik can have **two routers matching the same host rule** (`Host(\`hellofrom.app`)`). When both exist:

- the router with the **higher `priority` wins**
- cutover is just “start green”
- rollback is just “stop green”

**Important:** each deployment must use a **unique router name** (e.g. `hellofrom-blue` and `hellofrom-green`). Reusing the same router name on two containers is not safe.

---

## Prereqs

- A Linux host/VPS with Docker + Docker Compose v2
- Ports **80** and **443** reachable from the internet (for ACME HTTP-01)
- DNS `A` record for `hellofrom.app` → this host
- DNS `A` record for `traefik.hellofrom.app` → this host (optional, dashboard)
- A place to store `acme.json` on disk

---

## First-time setup (Let’s Encrypt storage)

Traefik needs a writable file for ACME:

```bash
mkdir -p traefik-letsencrypt
touch traefik-letsencrypt/acme.json
chmod 600 traefik-letsencrypt/acme.json
```

---

## Networks

This repo expects these Docker networks (Compose will create them if missing):

- `hf_edge` — public edge network Traefik uses to reach services
- `hf_svc_keycloak` — internal-only network (Traefik attached so it can route)
- `hf_svc_umami` — internal-only network (Traefik attached so it can route)
- `hf_svc_openobserve` — internal-only network (Traefik attached so it can route)

If you already created these networks elsewhere, keep the names consistent.

---

## Usage

### Start Traefik + Blue (default live)

```bash
docker compose --profile main up -d
```

### Cut over to Green (Green takes traffic)

```bash
docker compose --profile green up -d
```

At this point both blue and green may be running, but green should receive requests because it has a higher router priority.

### Roll back to Blue (stop Green only)

Safer than `down` (doesn’t touch Traefik/Blue):

```bash
docker compose stop hellofrom_green
docker compose rm -f hellofrom_green
```

---

## If you really want “down green” to only remove green

Compose `down` is project-scoped, not profile-scoped in the way people often assume. If you want clean isolation, run **two project names**:

```bash
# main stack (traefik + blue)
docker compose -p hf-main --profile main up -d

# green stack (just green)
docker compose -p hf-green --profile green up -d

# remove only green stack
docker compose -p hf-green down
```

Traefik will still see both containers and their labels, even across projects.

---

## Verifying which deployment is live

### Check Traefik access logs

Traefik is configured for JSON access logs and includes `RouterName`/`ServiceName`. You should see requests hitting either `hellofrom-blue@docker` or `hellofrom-green@docker` (depending on how you named routers).

### Curl the site

If you expose a simple “COLOR” response/header in your app, this is the easiest check:

```bash
curl -I https://hellofrom.app
```

---

## Traefik dashboard

The dashboard is routed at:

- `https://traefik.hellofrom.app`

It’s protected by an IP allowlist middleware (`hf-localonly`) in the compose labels. Update the allowlist to match your trusted source ranges.

---

## Troubleshooting

### Green doesn’t take traffic

Common causes:

- Green router has a lower/equal `priority` than blue
- Router names collide (both containers define the same router name)
- Green router is missing the `rule` / `entrypoints` / `tls` labels
- Service port label doesn’t match what Bun is listening on

### Let’s Encrypt isn’t issuing certs

- Confirm port **80** is reachable (HTTP-01 challenge needs it)
- Confirm `traefik-letsencrypt/acme.json` exists and is writable by the container
- Check Traefik logs:

  ```bash
  docker logs -f traefik
  ```

### Traefik can’t reach a service

- Ensure the service is attached to `hf_edge`
- Ensure `traefik.docker.network=hf_edge` is set on the service labels (so Traefik picks the right network)

---

## Suggested repo conventions

- Keep blue and green entrypoints separate (`index.blue.js` vs `index.js`)
- Always:

  - unique router name per deployment
  - same `Host()` rule on both routers
  - higher priority for green

- Treat green as ephemeral: bring it up to validate, then tear it down after cutover/rollback.
