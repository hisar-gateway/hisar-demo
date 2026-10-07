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

**Install on [Linux](#linux) · [macOS](#macos) · [Windows](#windows)** — a few
minutes, most of them spent on the first download.

> The gateway's source code is not published here. The images come from
> `ghcr.io/hisar-gateway/apigateway-backend` and `-frontend`; this repository
> holds only the Compose file, its `.env` and this guide.

## Install

Five containers run under Docker Compose: the gateway, its admin console,
SurrealDB (configuration), Redis (counters, sessions, change events) and
Elasticsearch (request, audit and security logs).

You need:

- **Docker with the `docker compose` command** — Docker Engine 24 or later
  with the Compose plugin on Linux, Docker Desktop on macOS and Windows.
- **An x86-64 machine, or a Mac with Apple silicon.** The images are built for
  `linux/amd64` only for now; on Apple silicon Docker Desktop runs them through
  Rosetta. ARM Linux (Raspberry Pi, AWS Graviton) and Windows on Arm are not
  supported yet.
- **About 2 GB of free memory and 4 GB of disk.** The first start downloads
  about 1 GB.
- **Ports 8080 and 3000** free — or [others of your choice](#ports).

### Linux

1. Install Docker Engine and the Compose plugin —
   [instructions for each distribution](https://docs.docker.com/engine/install/),
   or Docker's convenience script:

   ```bash
   curl -fsSL https://get.docker.com | sudo sh
   sudo usermod -aG docker "$USER"     # docker without sudo: sign out and back in
   ```

   `docker compose version` should print v2 or later.

2. Download the bundle, start it and read the admin password:

   ```bash
   curl -LO https://github.com/hisar-gateway/hisar-demo/releases/latest/download/hisar-demo.zip
   unzip hisar-demo.zip && cd hisar-demo
   docker compose up -d
   docker compose logs apigateway | grep "INITIAL ADMIN"
   ```

   No `unzip`? `python3 -m zipfile -e hisar-demo.zip .` does the same.

Installing on a server that others can reach? Read
[Running on a server](#running-on-a-server) before the first start.

### macOS

1. Install [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/)
   — the Apple silicon or the Intel build, matching your Mac — open it and wait
   until it reports that the engine is running. On Apple silicon, leave
   **Settings → General → Use Rosetta for x86_64/amd64 emulation on Apple
   Silicon** switched on.

2. In Terminal:

   ```bash
   curl -LO https://github.com/hisar-gateway/hisar-demo/releases/latest/download/hisar-demo.zip
   unzip hisar-demo.zip && cd hisar-demo
   docker compose up -d
   docker compose logs apigateway | grep "INITIAL ADMIN"
   ```

### Windows

1. Install [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/)
   with the **WSL 2** backend — the installer's default; it needs Windows 10
   22H2 or Windows 11 with virtualization switched on — then start it and wait
   until it reports that the engine is running.

2. In PowerShell:

   ```powershell
   Invoke-WebRequest https://github.com/hisar-gateway/hisar-demo/releases/latest/download/hisar-demo.zip -OutFile hisar-demo.zip -UseBasicParsing
   Expand-Archive hisar-demo.zip -DestinationPath .
   cd hisar-demo
   docker compose up -d
   docker compose logs apigateway | Select-String "INITIAL ADMIN"
   ```

   Inside a WSL distribution such as Ubuntu — with Docker Desktop's WSL
   integration switched on for it — the Linux commands work as written.

Prefer git? `git clone https://github.com/hisar-gateway/hisar-demo.git` brings
the same files; its `main` branch follows the newest release.

## Sign in

The first start takes a few minutes, mostly downloading. `docker compose ps`
lists `apigateway` as `healthy` once it is ready, and the password command
prints one line:

```text
… 🔐 INITIAL ADMIN CREATED — email: demo@hisar-gateway.com  password: …
```

Open **http://localhost:8080**, sign in with that e-mail and password, then
choose your own password from the account menu → **Profile**.

The password is printed once, at the very first start, and only the log keeps
it — changing `.env` recreates the container and empties its log. Lost it
before signing in? `docker compose down -v` deletes everything, and the next
`docker compose up -d` prints a new one.

| | Address |
|---|---|
| Admin console | http://localhost:8080 |
| Gateway — your API traffic | http://localhost:3000 |
| Health check | http://localhost:3000/health |

### Ports

Something else already on 8080 or 3000? Set other host ports in `.env` and run
`docker compose up -d` again; the addresses above follow:

```bash
HISAR_UI_PORT=18080        # admin console  → http://localhost:18080
HISAR_GATEWAY_PORT=13000   # gateway        → http://localhost:13000
```

Edit `.env` with a plain-text editor: `nano .env` on Linux and macOS,
`notepad .env` on Windows. Finder does not show files whose name starts with
a dot.

### Running on a server

Both ports speak plain HTTP and by default accept connections from every
network the machine is on — Docker publishes them past host firewalls such as
`ufw`. On a machine reachable from the internet, publish them on the loopback
interface only:

```bash
HISAR_UI_PORT=127.0.0.1:8080
HISAR_GATEWAY_PORT=127.0.0.1:3000
```

and reach the console through an SSH tunnel from your own computer:

```bash
ssh -L 8080:127.0.0.1:8080 you@your-server    # then open http://localhost:8080
```

## First route

Every route is served at `http://localhost:3000/{application}/{service}/{route}`,
built from the slugs you give each of them. In the console:

1. **Catalog → Organizations → New organization** — name `Demo`, slug `demo`.
2. **Catalog → Applications → New application** — organization `Demo`, name
   `Demo`, slug `demo`.
3. **Catalog → Services → New service** — application `Demo`, name
   `Placeholder`, slug `placeholder`, service type REST, base URL
   `https://jsonplaceholder.typicode.com`, auth type None.
4. **Proxy → Routes → New route** — service `Placeholder`, name `Users`, slug
   `users`, method GET, path `/users`, backend path `/users`.
5. Call it through the gateway, in a browser or with curl:

   ```bash
   curl http://localhost:3000/demo/placeholder/users
   ```

   In Windows PowerShell, type `curl.exe` — plain `curl` is a different command
   there.

Path parameters come from the route's path: a second route with slug `user`,
path `/users/:id` and backend path `/users/:id` answers
`/demo/placeholder/user/1` with one user.

An OpenAPI or Swagger file creates a service with all of its routes at once:
**Catalog → Services → Import OpenAPI**. The 5-route cap counts the whole
import.

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
- **Admin console** in Turkish and English; changes reach every gateway
  instance over Redis Pub/Sub — no restart, no polling wait.

### Licensed modules

A licence names the modules it unlocks; these are the eighteen, all on the
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
| **Identity provider** | A user directory of your own — users, groups, two-factor sign-in — served to your applications over OpenID Connect (authorization code with PKCE, consent, rotating refresh tokens, sign-out) and over LDAPS for applications that only speak LDAP; each application admits the groups you list |
| **Advanced observability** | OpenTelemetry traces, Prometheus `/metrics`, provisioned Grafana dashboards |
| **Notifications** | Slack, Discord, webhooks and SMTP; severity-gated alerts, licence expiry reminders, a daily security report |
| **Developer portal** | Swagger UI, per-service OpenAPI export, API catalog |

## Free edition and licensed

| | Free edition | With a licence |
|---|---|---|
| Routes | 5 | as licensed (0 = unlimited) |
| Requests per second | 10 per route | gateway-wide `max_rps` from the licence (0 = unlimited) |
| Licensed modules | off | the modules on your licence |
| Time limit | none | licence term, 60-day expiry warning |
| Activation | included | upload the licence file in **Operations → License** — takes effect at once |

A request over the cap gets `429 License RPS limit exceeded (… req/s per route — restricted mode)`;
the sixth route is refused with `403 Route limit exceeded: 5/5`.

## Everyday commands

Run them in the `hisar-demo` folder.

| Command | What it does |
|---|---|
| `docker compose ps` | Shows the state of each container |
| `docker compose logs -f apigateway` | Follows the gateway log (Ctrl+C stops following) |
| `docker compose stop` | Stops everything and keeps the data |
| `docker compose start` | Starts it again |
| `docker compose down -v` | Removes the containers **and all data** |

**Upgrade.** Set `HISAR_VERSION` in `.env` to the new release — or to
`latest` — then run `docker compose pull` and `docker compose up -d`; the data
stays. The zip on each [release](https://github.com/hisar-gateway/hisar-demo/releases)
pins its own version. The registry keeps recent releases only, so an old
version eventually stops downloading.

**Uninstall.** `docker compose down -v --rmi all`, then delete the folder.

## Troubleshooting

| Symptom | What to do |
|---|---|
| `port is already allocated` or `address already in use` | Another program uses 8080 or 3000 — [choose other ports](#ports) |
| `Cannot connect to the Docker daemon` or `error during connect` | Docker is not running: start Docker Desktop, or `sudo systemctl start docker` on Linux |
| `exec format error` | An ARM machine — the images are amd64-only for now |
| No `INITIAL ADMIN` line | The gateway is still starting — wait a minute and run the command again. If the data already holds a user from an earlier start, see [Sign in](#sign-in) |
| `apigateway` stays `starting` or turns `unhealthy` | `docker compose logs apigateway` says why. It waits for SurrealDB, Redis and Elasticsearch first; on Apple silicon the first start takes longer |
| `elasticsearch` exits with code 137 | It ran out of memory — give Docker more (on macOS: Docker Desktop → Settings → Resources) |
| Elasticsearch log: `max virtual memory areas vm.max_map_count [65530] is too low` | Linux: `sudo sysctl -w vm.max_map_count=262144`, and add `vm.max_map_count=262144` to `/etc/sysctl.conf` to keep it |
| Signed out after a restart | Restarting the gateway container ends every session — sign in again |

## Notes

- Swagger UI (`/swagger-ui`) and `/metrics` belong to licensed modules (Developer
  Portal, Advanced Observability) and answer 404 in the free edition.
- This is an evaluation layout: one gateway, plain HTTP and a signing key
  created at start. Production installations run several gateways behind a TLS
  edge, with the admin console and API traffic on separate entrypoints.

## Licence

Licences are per organization, signed offline (Ed25519) and uploaded in the
admin console under **Operations → License**; a licence unlocks the modules and
capacity it names and warns 60 days before it expires. For pricing and a
licence, see [hisar-gateway.com](https://hisar-gateway.com).
