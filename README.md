# Hisar API Gateway

[Türkçe](README.tr.md) · [Website](https://hisar-gateway.com) · [Releases](https://github.com/hisar-gateway/hisar-demo/releases)

The API gateway for the APIs you have — and the ones you never built. One Rust
binary fronts REST, SOAP, GraphQL, gRPC and WebSocket services, turns legacy
SOAP into plain JSON, generates CRUD APIs straight from your databases, and
puts LLM providers behind one metered endpoint. Self-hosted on your own
machines, licensed offline.

This repository is the **free edition**: the same gateway, capped at **5 routes**
and **10 requests per second per route**, with the licensed modules switched
off. No account, no time limit. Upload a licence later from the admin console
and the same installation goes to full capacity — no reinstall, no restart.

> The gateway's source code is not published here. The images come from
> `ghcr.io/hisar-gateway/apigateway-backend` and `-frontend`; this repository holds
> only the Compose file, its `.env` and this guide.

## What you get

### In the free edition

- **Multi-protocol routing** — REST, SOAP, GraphQL, gRPC and WebSocket
  behind one hierarchy: Organization → Application → API version → Service →
  Route. Deprecated versions answer with `Deprecation` / `Sunset` headers; a
  past sunset date returns `410 Gone` on its own.
- **SOAP ↔ JSON bridge** — clients send JSON; the gateway builds the SOAP
  envelope, applies action and version, and turns faults into structured JSON.
  WS-Trust / WS-Security tokens are acquired, cached and renewed by the
  gateway, so callers never touch an XML security header.
- **Caller authentication** — RS256 JWT with rotating refresh tokens and reuse
  detection, scoped API keys, per-service access mode (public, API key, JWT)
  with required roles.
- **Input validation, security headers, IP allow/deny lists**, request and
  audit logging to Elasticsearch, OpenAPI import.
- **Admin console** in Turkish and English; changes reach every replica over
  Redis Pub/Sub — no restart, no polling wait.

### Licensed modules

A licence names the modules it unlocks; these are the seventeen, all on the
same installation and switched on without a restart.

| Module | What it adds |
|---|---|
| **WAF** | Anomaly-scoring engine with 57 built-in rules across 10 attack classes (SQLi, XSS, traversal, command, LDAP, NoSQL, SSRF, SSTI, XXE…), three-layer decoding against evasion, policies at global / service / route / DB2API-table scope, Block or DetectionOnly |
| **DDoS protection** | Per-IP rate and burst caps with automatic temporary bans; in-memory reputation snapshot, so enforcement costs no Redis round-trip |
| **Geo-blocking** | MaxMind country lookups with allow and deny lists |
| **Rate limiting** | Cascade limits across up to seven tiers — user, role, route, service, API version, application, organization — with shared, per-key, per-user or per-IP buckets and sliding windows on Redis; subscription-style plans bound to API keys with per-route quotas and reserved capacity; business-hour and promotion multipliers |
| **Load balancing** | Round-robin, weighted, least-connections or random across backends; health probes take backends out and back in automatically |
| **Canary / blue-green** | Release variants by weight or pin header, tracked per variant |
| **Resilience** | Cluster-shared circuit breakers, per-host bulkheads, backoff retries for idempotent calls |
| **Response caching** | Redis-backed cache per route with TTLs, query-parameter keys and vary-by-header, restricted to public routes so nothing leaks across tenants; exact and semantic caching for AI responses |
| **Mutual TLS** | Client certificates towards upstreams and client-certificate policies for callers behind a TLS-terminating proxy |
| **CAPTCHA** | Challenge on the login form after repeated failures; provider and thresholds are runtime settings |
| **DB2API** | CRUD endpoints generated from PostgreSQL, MySQL, MariaDB, MSSQL, Oracle, MongoDB or SurrealDB — filtering, sorting, aggregation, declarative relations, row-level security. No application code |
| **AI gateway** | OpenAI, Azure OpenAI, Anthropic and custom providers behind one endpoint; prepaid token wallets per caller, per-model daily free tiers, guardrails for prompt injection and PII, streaming, fallback providers, cost metrics |
| **Data masking** | Five strategies — full, prefix, suffix, edges, stable hash — applied to responses and logs independently |
| **Advanced auth** | OIDC, LDAP / Active Directory, SAML 2.0, trusted proxy and external REST logins with just-in-time accounts and directory-driven roles; TOTP two-factor. Plain JWT / API-key authentication stays in the core |
| **Advanced observability** | OpenTelemetry traces, Prometheus `/metrics`, provisioned Grafana dashboards |
| **Notifications** | Slack, Discord, webhooks and SMTP; severity-gated alerts, licence expiry reminders, a daily security report |
| **Developer portal** | Swagger UI, per-service OpenAPI export, API catalog |

## Free edition vs. licensed

| | Free edition | With a licence |
|---|---|---|
| Routes | 5 | as licensed (0 = unlimited) |
| Requests per second | 10 per route | gateway-wide `max_rps` from the licence (0 = unlimited) |
| Licensed modules | off | the modules on your licence |
| Time limit | none | licence term, 60-day expiry warning |
| Activation | included | upload the `.lic` file in **System → License** — takes effect at once |

A request over the cap gets `429 License RPS limit exceeded (… req/s per route — restricted mode)`;
the sixth route is refused with `403 Route limit exceeded: 5/5`.

## Requirements

- Docker 24+ with Compose v2 (Linux, macOS or Windows with WSL2)
- x86-64 host (the images are built for `linux/amd64`; Apple Silicon runs them
  under emulation, which works but is slower)
- Ports 8080 (admin console) and 3000 (gateway) free, or other ports set in
  `.env` (see [Ports](#ports))
- About 2 GB of RAM for the whole stack, 1 GB less without Elasticsearch

## Start

Get the files, either way:

```bash
git clone https://github.com/hisar-gateway/hisar-demo.git && cd hisar-demo
# or: download hisar-demo.zip from the Releases page and unzip it
```

Then:

```bash
docker compose up -d
docker compose logs apigateway | grep "INITIAL ADMIN CREATED"
```

No registry login is needed — the images are public. The first start pulls
about 700 MB.

Open http://localhost:8080, sign in as `demo@hisar-gateway.com` with the password
from the log line; change it from the account menu → **Profile**.

| Component     | URL                          |
|---------------|------------------------------|
| Admin console | http://localhost:8080        |
| Gateway       | http://localhost:3000        |
| Health        | http://localhost:3000/health |

### Ports

Something else already on 8080 or 3000? Set other host ports in `.env` and
start again — the URLs above follow:

```bash
HISAR_UI_PORT=18080        # admin console  → http://localhost:18080
HISAR_GATEWAY_PORT=13000   # gateway        → http://localhost:13000
```

```bash
docker compose up -d
```

## First route

1. **Organizations → New** — the top of the hierarchy (e.g. `Demo`); every application belongs to one.
2. **Applications → New** — pick that organization; an application groups your services (slug e.g. `demo`).
3. **Services → New** — point it at an upstream, e.g. `https://httpbin.org`
   (REST) or a SOAP endpoint; pick the protocol and the auth the upstream needs.
4. **Routes → New** — slug `get`, path `/get`, backend path `/get`, method GET.
5. Call it through the gateway: `curl http://localhost:3000/demo/<service-slug>/get`.

Or import an OpenAPI file from **Services → Import**; the 5-route cap applies to
the whole import.

## Stop, reset, upgrade

```bash
docker compose down              # stop, keep data
docker compose down -v           # stop and delete all data
docker compose pull && docker compose up -d   # move to a newer image
```

`main` tracks the newest release (`HISAR_VERSION=latest`); each entry on the
Releases page carries a zip whose `.env` pins that version. The registry keeps
only recent releases, so an old pinned tag eventually fails to pull — stay on
`latest` unless you need a specific release.

## Notes

- Swagger UI (`/swagger-ui`) and `/metrics` belong to licensed modules (Developer
  Portal, Advanced Observability) and answer 404 in the free edition.
- Elasticsearch feeds the dashboard's log and security panels. Remove the
  service from `docker-compose.yml` to save memory; the gateway does not need
  it to run.
- JWT keys are generated per process: a restart of the `apigateway` container
  signs everyone out. Mount a PEM pair (see `.env`) to keep sessions.
- This layout is for evaluation. Production runs behind Traefik with separate
  control and data entrypoints and rolling replicas — the user guide covers it.

## Licence

Licences are per organization, signed offline (Ed25519) and uploaded from the
admin console; a licence unlocks the modules and capacity it names, with a
60-day expiry warning. For pricing and a licence file, see
[hisar-gateway.com](https://hisar-gateway.com).
