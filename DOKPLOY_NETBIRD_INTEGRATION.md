# Dokploy + NetBird Domain Integration Plan

## Context

Migrating from Tailscale to NetBird. NetBird serves multiple services beyond Dokploy and cannot be removed. The goal is to automatically register Dokploy-deployed containers (including branch-based staging/dev deployments) as NetBird reverse proxy services with custom domains (e.g., `manni.inderbitzin.ch`).

---

## Architecture Overview

```
Git push → Dokploy deploys container → Docker event fires
         → Bridge service reads container labels
         → Calls NetBird API to create/update reverse proxy service
         → DNS (CNAME) already points domain to NetBird proxy cluster

Traffic flow:
  Internet → NetBird Proxy (TLS termination) → WireGuard tunnel → Docker container
```

---

## Component Analysis

### NetBird Reverse Proxy

NetBird has a full HTTP/HTTPS reverse proxy system built into its management server.

**Capabilities:**
- Custom domain support with CNAME validation
- Free auto-assigned domains (`<name>.<nonce>.<cluster>`, e.g., `myapp.abc123.eu.proxy.netbird.io`)
- Automatic TLS certificates via ACME / Let's Encrypt
- Authentication methods: PIN, password, bearer/OIDC, magic link
- Path-based routing with multiple backend targets
- CLI exposure: `netbird expose <port> --with-custom-domain myapp.example.com`

**REST API Endpoints:**

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/api/reverse-proxies/services` | List all services |
| `POST` | `/api/reverse-proxies/services` | Create service |
| `GET` | `/api/reverse-proxies/services/{id}` | Get service |
| `PUT` | `/api/reverse-proxies/services/{id}` | Update service |
| `DELETE` | `/api/reverse-proxies/services/{id}` | Delete service |
| `GET` | `/api/reverse-proxies/domains` | List domains (free + custom) |
| `POST` | `/api/reverse-proxies/domains` | Create custom domain |
| `DELETE` | `/api/reverse-proxies/domains/{id}` | Delete custom domain |
| `GET` | `/api/reverse-proxies/domains/{id}/validate` | Trigger CNAME validation |
| `GET` | `/api/reverse-proxies/clusters` | List proxy clusters |

**Authentication:** PAT token (`Authorization: Token nbp_xxx`) or JWT Bearer token.

**Create Service Example:**
```json
POST /api/reverse-proxies/services
{
  "name": "manni-app",
  "domain": "manni.inderbitzin.ch",
  "targets": [
    {
      "target_id": "<peer-id-or-resource-id>",
      "target_type": "peer",
      "protocol": "http",
      "port": 3000,
      "enabled": true
    }
  ],
  "enabled": true,
  "pass_host_header": false,
  "rewrite_redirects": false
}
```

**Target Types:**

| Type | Use Case | `host` field |
|------|----------|--------------|
| `peer` | Point to a NetBird peer by ID | Ignored (auto-resolved) |
| `host` | Point to a network resource by ID | Ignored (auto-resolved) |
| `domain` | Point to a resource by domain | Ignored (auto-resolved) |
| `subnet` | Point to a specific IP:port | **Required** — set to the IP address |

**Subnet target (direct IP) example:**
```json
{
  "target_id": "my-container",
  "target_type": "subnet",
  "protocol": "http",
  "port": 8080,
  "host": "10.0.0.50",
  "enabled": true
}
```

**Register Custom Domain Example:**
```json
POST /api/reverse-proxies/domains
{
  "domain": "manni.inderbitzin.ch",
  "target_cluster": "eu.proxy.netbird.io"
}
```

**Service Status Values:** `pending`, `active`, `tunnel_not_created`, `certificate_pending`, `certificate_failed`, `error`

---

### Dokploy

Dokploy is a self-hosting PaaS that uses **Traefik v3** internally as its reverse proxy.

**Domain handling:**
- Domains configured per application in Dokploy UI
- Traefik routes traffic to containers based on Host headers
- Preview deployments auto-generate domains: `preview-${appName}-${uniqueId}.traefik.me`
- Custom wildcard domains supported for previews (e.g., `*.dev.inderbitzin.ch`)

**Automation capabilities:**

| Feature | Status | Notes |
|---------|--------|-------|
| Git webhooks (inbound) | Supported | GitHub, GitLab, Bitbucket, Gitea, DockerHub |
| Post-deploy hooks | **Not available** | Feature requested: [issue #110](https://github.com/Dokploy/dokploy/issues/110) |
| Outbound webhooks | **Not available** | Feature requested: [issue #3016](https://github.com/Dokploy/dokploy/issues/3016) |
| REST API | Supported | JWT auth via `x-api-key` header |
| Preview deployments | Supported | Auto-created per PR, limit configurable |
| External reverse proxy | **Not supported** | Maintainers closed [issue #2528](https://github.com/Dokploy/dokploy/issues/2528) as won't fix |

**Dokploy API (relevant endpoints):**

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/api/domain/byApplicationId?applicationId=X` | Get domains for an app |
| `POST` | `/api/domain/create` | Create domain |
| `POST` | `/api/domain/delete` | Delete domain |

**Auth:** JWT token in `x-api-key` header, generated at `/settings/profile` → API/CLI section.
**Swagger:** Available at `http://<dokploy-ip>:3000/swagger`

---

## Integration Strategy

### DNS Setup

1. Register `*.inderbitzin.ch` (or specific subdomains) as a custom domain in NetBird
2. Create a CNAME record pointing to the NetBird proxy cluster:
   ```
   manni.inderbitzin.ch  CNAME  eu.proxy.netbird.io
   ```
   Or use a wildcard: `*.inderbitzin.ch CNAME eu.proxy.netbird.io`
3. Validate the domain via NetBird API: `GET /api/reverse-proxies/domains/{id}/validate`

### Bridge Service (Docker Event Listener)

A lightweight sidecar service that syncs Dokploy deployments to NetBird reverse proxy.

**How it works:**
1. Listens to Docker engine events (`container start`, `container die`)
2. Filters for Dokploy-managed containers (by label presence)
3. Reads Traefik labels to extract domain and port:
   - `traefik.http.routers.<name>.rule` → contains `Host(\`manni.inderbitzin.ch\`)`
   - Container's exposed port → target port for NetBird
4. On container start: creates NetBird reverse proxy service via API
5. On container stop/die: deletes the NetBird reverse proxy service

**Pseudocode:**
```python
import docker
import requests

NETBIRD_API = "https://api.netbird.io"
NETBIRD_TOKEN = "nbp_xxx"
PEER_ID = "<dokploy-server-peer-id>"

client = docker.from_env()

for event in client.events(decode=True, filters={"event": ["start", "die"]}):
    container_id = event["id"]
    container = client.containers.get(container_id)
    labels = container.labels

    # Extract domain from Traefik labels
    domain = extract_domain_from_traefik_labels(labels)
    port = extract_port_from_traefik_labels(labels)

    if not domain:
        continue  # Not a Dokploy app with a domain

    service_name = domain.replace(".", "-")

    if event["Action"] == "start":
        # Create NetBird reverse proxy service
        requests.post(f"{NETBIRD_API}/api/reverse-proxies/services",
            headers={"Authorization": f"Token {NETBIRD_TOKEN}"},
            json={
                "name": service_name,
                "domain": domain,
                "targets": [{
                    "target_id": PEER_ID,
                    "target_type": "peer",
                    "protocol": "http",
                    "port": port,
                    "enabled": True
                }],
                "enabled": True
            })

    elif event["Action"] == "die":
        # Find and delete the service
        services = requests.get(f"{NETBIRD_API}/api/reverse-proxies/services",
            headers={"Authorization": f"Token {NETBIRD_TOKEN}"}).json()
        for svc in services:
            if svc["domain"] == domain:
                requests.delete(
                    f"{NETBIRD_API}/api/reverse-proxies/services/{svc['id']}",
                    headers={"Authorization": f"Token {NETBIRD_TOKEN}"})
```

### Branch / Staging Domain Convention

When Dokploy creates preview deployments for PRs:

| Branch | Domain Pattern | Example |
|--------|---------------|---------|
| `main` | `<app>.inderbitzin.ch` | `manni.inderbitzin.ch` |
| `staging` | `staging-<app>.inderbitzin.ch` | `staging-manni.inderbitzin.ch` |
| `dev` | `dev-<app>.inderbitzin.ch` | `dev-manni.inderbitzin.ch` |
| PR #42 | `pr42-<app>.inderbitzin.ch` | `pr42-manni.inderbitzin.ch` |

Configure these domain patterns in Dokploy's domain settings per application. The bridge service picks them up from Traefik labels automatically.

---

## Alternative Approaches

### Option A: Poll Dokploy API
- Periodically query Dokploy's `/api/domain/byApplicationId` for all apps
- Diff against current NetBird services and sync
- **Pro:** No Docker socket access needed
- **Con:** Polling delay, more API calls, need to enumerate all apps

### Option B: GitHub Actions Post-Deploy Step
- Add a step in CI after Dokploy deploys that calls NetBird API
- **Pro:** No sidecar needed
- **Con:** Doesn't catch manual deploys or preview deployments, CI-specific

### Option C: Dokploy Docker Compose with NetBird sidecar
- Run `netbird expose` CLI inside a sidecar container in the same Docker network
- **Pro:** Uses built-in NetBird CLI
- **Con:** One sidecar per app, more resource usage

### Recommended: Docker Event Listener (described above)
- Most reliable — catches all deployment events
- Single service for all apps
- No polling, instant reaction
- Works with preview deployments

---

## Prerequisites Checklist

- [ ] NetBird installed and running on Dokploy server (peer connected)
- [ ] NetBird PAT token generated for API access
- [ ] Custom domain(s) registered in NetBird (`POST /api/reverse-proxies/domains`)
- [ ] DNS CNAME records pointing to NetBird proxy cluster
- [ ] Domain validated in NetBird (`GET /api/reverse-proxies/domains/{id}/validate`)
- [ ] Dokploy applications configured with desired domain names
- [ ] Bridge service deployed (Docker container or systemd service)

---

## Key References

- **NetBird reverse proxy API handlers:** `management/internals/modules/reverseproxy/manager/api.go`
- **NetBird REST client:** `shared/management/client/rest/reverse_proxy_services.go`
- **NetBird domain models:** `management/internals/modules/reverseproxy/reverseproxy.go`
- **NetBird OpenAPI spec:** `shared/management/http/api/openapi.yml`
- **Dokploy API docs:** `http://<dokploy-ip>:3000/swagger`
- **Dokploy preview deployments:** https://docs.dokploy.com/docs/core/applications/preview-deployments
- **Dokploy Tailscale guide (reference):** https://docs.dokploy.com/docs/core/guides/tailscale
- **Dokploy post-deploy hooks request:** https://github.com/Dokploy/dokploy/issues/110
