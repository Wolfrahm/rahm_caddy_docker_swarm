# caddy-swarm

A container image of [Caddy](https://caddyserver.com) bundling two extra modules so a fleet of identical replicas can serve as an HA reverse proxy on Docker Swarm:

- **[`caddy-docker-proxy`](https://github.com/lucaslorentz/caddy-docker-proxy)** — generates the Caddyfile dynamically from `caddy.*` labels on Swarm services. Watches the Docker API and reloads on every service change.
- **[`certmagic-s3`](https://github.com/ss098/certmagic-s3)** — replaces Caddy's default filesystem storage with an S3-backed one. All ACME state (account key, issued certs, in-flight challenge tokens, distributed locks) lives in a shared bucket.

Published as **`ghcr.io/wolfrahm/caddy-swarm`** by the GitHub Actions workflow in this repo. Multi-arch: `linux/amd64` + `linux/arm64`.

## Why this is the right shape for HA on Docker Swarm

Stock Caddy already auto-issues TLS for every hostname it routes, but only as a single instance. The moment you want **multiple Caddy replicas across a Swarm**, three problems appear, and these two plugins solve all of them.

### 1. Service discovery without a leader

Each Caddy replica has to know about every Swarm service, not just the containers on its own node. `caddy-docker-proxy` does this by watching the Docker API (typically a TCP-exposed socket proxy on a manager) and rebuilding the Caddyfile in-memory whenever services come and go. **Every replica generates the same config independently** — no controller/follower split, no admin-API push between replicas.

Adding or removing a Caddy replica is just `docker service scale caddy=N`. Each new container connects to the socket proxy, builds its config, starts serving.

### 2. Certificate coordination

If N replicas independently tried to issue a cert for the same hostname, you'd get duplicate orders, ACME rate-limit pain, and divergent state. `certmagic-s3` solves this by giving every replica the same backing store and a distributed lock primitive:

- Replica A receives a request for an unissued hostname → checks S3, no cert → tries to acquire a lock object in S3.
- Whichever replica wins runs the ACME flow.
- Winner writes cert + key + updated account to S3 and releases the lock.
- All other replicas pick up the cert from S3 on the next config refresh.

The replica count becomes irrelevant to issuance correctness: there's always exactly one issuance in flight per hostname.

### 3. HTTP-01 challenge response from any replica

This is the subtle one. During HTTP-01 validation, Let's Encrypt GETs `http://example.com/.well-known/acme-challenge/<token>`. Whichever replica receives that request must respond with the right token — but with N replicas behind round-robin DNS or a host-mode port, the request can land on any one of them.

`certmagic-s3` writes the challenge token into S3 **before** telling LE to validate. When any replica receives the GET, it reads the token from S3 and responds. The replica that issued the order doesn't have to be the one that serves the challenge. No internal forwarding, no "issuer node" pinning, no synchronized in-memory state.

### Operational consequence

A traditional HA-Traefik-on-Swarm setup typically requires:

- A pinned "ACME node" running a single-replica Traefik that owns issuance.
- A cert-sync sidecar that distributes PEMs from that node to all other proxy nodes.
- Internal routing rules so HTTP-01 challenges reach the issuer.

With this image, **none of that exists**. The full stack is a global `caddy` service plus a socket proxy on managers. Every replica is identical, equally privileged, and stateless beyond the shared S3 bucket. Node death is uneventful — Swarm reschedules, the new container reads state from S3, joins the rotation.

## Usage

Pair this image with the [`rahm_terraform_docker_swarm_caddy`](https://github.com/Wolfrahm/rahm_terraform_docker_swarm_caddy) Terraform module — it deploys this image as a global Swarm service plus a `tecnativa/docker-socket-proxy` for Docker API exposure.

Standalone, the image accepts the same environment and Caddyfile a stock Caddy build does, plus the `caddy-docker-proxy` invocation as its default `CMD`:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v ./Caddyfile:/etc/caddy/Caddyfile:ro \
  -e CADDY_DOCKER_CADDYFILE_PATH=/etc/caddy/Caddyfile \
  -e ACME_EMAIL=admin@example.com \
  -e S3_HOST=s3.eu-west-1.amazonaws.com \
  -e S3_BUCKET=my-bucket \
  -e S3_PREFIX=caddy \
  -e AWS_ACCESS_KEY_ID=... \
  -e AWS_SECRET_ACCESS_KEY=... \
  ghcr.io/wolfrahm/caddy-swarm:latest
```

Minimal `Caddyfile` to wire up the storage adapter:

```caddyfile
{
  email {$ACME_EMAIL}
  storage s3 {
    host {$S3_HOST}
    bucket {$S3_BUCKET}
    prefix {$S3_PREFIX}
    access_id {$AWS_ACCESS_KEY_ID}
    secret_key {$AWS_SECRET_ACCESS_KEY}
  }
}
```

Apps in the swarm declare routing with two labels:

```yaml
labels:
  caddy: myapp.example.com
  caddy.reverse_proxy: "{{upstreams 8080}}"
```

That's enough for auto-HTTPS.

## Tags

| Tag | What it points at |
|---|---|
| `2.11`, `2.11.2`, …  | Specific Caddy version |
| `latest` | Most recent build on `main` |
| `sha-<short>` | Specific commit |

Pin to a major.minor in production; `latest` is fine for sandboxes.

## Building locally

```bash
docker buildx build --platform linux/arm64 -t caddy-swarm:dev --load .
```

For multi-arch local builds you need a `docker buildx` setup with QEMU. The CI workflow takes care of this for the published image.

## Licenses

This image is a thin assembly of Apache-2.0 / MIT components. See the upstream repos for full license text:

- [caddyserver/caddy](https://github.com/caddyserver/caddy) — Apache-2.0
- [lucaslorentz/caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy) — MIT
- [ss098/certmagic-s3](https://github.com/ss098/certmagic-s3) — Apache-2.0
