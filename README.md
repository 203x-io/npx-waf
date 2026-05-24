# npx-waf

Drop-in nginx with inline WAF enforcement built into the binary.

npx-waf gives teams enforcement-grade protection at the nginx layer they already run.

It blocks SQL injection, XSS, RCE, scanner reconnaissance, and named-CVE exploits in the same nginx ACCESS phase that already handles your rewrite and proxy rules. Keep your nginx config, run the image, point traffic at port 8080, and the WAF inspects every request inline at about 3 microseconds of median overhead in the benchmark below.

The current release scores 100% in the GoTestWAF v0.5 run below and includes an SBOM and Trivy scan as inspectable artifacts.

No sidecar required. No managed proxy between your users and your app. No control plane to operate unless you choose centralized fleet management later. Just nginx with attack filtering built in.

[![GoTestWAF](https://img.shields.io/badge/GoTestWAF-100%25-brightgreen.svg)](#performance--accuracy)
[![CVE Scan](https://img.shields.io/badge/Trivy-0%20HIGH%2FCRITICAL-brightgreen.svg)](#security-posture)
[![Image Size](https://img.shields.io/badge/size-under%2050%20MB-blue.svg)](#quick-reference)
[![Multi-arch](https://img.shields.io/badge/arch-amd64%20%2B%20arm64-blue.svg)](#quick-reference)
[![Distroless](https://img.shields.io/badge/base-distroless-blue.svg)](#security-posture)
[![SBOM](https://img.shields.io/badge/SBOM-SPDX%202.3-blue.svg)](#security-posture)
[![License](https://img.shields.io/badge/license-BUSL--1.1-orange.svg)](#license)

---

## Quick reference

- **Where to get help:** see the Support section at the bottom
- **Built on:** nginx 1.31.1 (mainline) — full nginx core feature set available
- **Supported architectures:** `linux/amd64`, `linux/arm64` — single multi-arch manifest list, Docker pulls the right one for your host
- **Tag strategy:** `latest` always points to the newest stable release; semver tags (`MAJOR.MINOR.PATCH`, `MAJOR.MINOR`, `MAJOR`) pin to specific versions. Current tag list: <https://github.com/203x-io/npx-waf/pkgs/container/npx-waf>
- **Source license:** BUSL-1.1 → Apache-2.0 on the Change Date (January 1, 2030)
- **Production use:** free for internal and self-hosted deployments protecting the operator's own applications and infrastructure; service-provider, hosted, or multi-tenant deployments serving third parties are subject to commercial terms

---

## Who it is for

npx-waf is built for platform, DevOps, AppSec, and infrastructure teams that already run nginx at the edge and want WAF enforcement inside that same layer. It is a strong fit for self-hosted SaaS, internal platforms, API gateways, staging edges, and regulated environments where traffic should stay on infrastructure you control.

If you need a vendor-operated global edge, pair this with your CDN/WAF strategy. If you operate nginx yourself, this is the direct path — replace plain nginx with a hardened nginx image that blocks attacks before they reach your upstreams.

---

## How it sits in your stack

```
   Internet                  npx-waf inspection                 Result
   ────────────────────      ──────────────────────────────     ────────
   GET /?q=hello         →   [scan 4,050 patterns]          →   200  →  backend
   GET /?q=[XSS]         →   [hit: XSS, score=8, ~3 µs]     →   403  ✕  backend
   GET /?id=[SQLi]       →   [hit: SQLi, score=8, ~3 µs]    →   403  ✕  backend
   POST [NoSQLi body]    →   [hit: NoSQLi, score=8, ~5 µs]  →   403  ✕  backend
   POST [tokeniser]      →   [libinjection match]           →   403  ✕  backend
                             ↓                                   ↓
                       Prometheus /metrics             JSON access log
                       (counters + histograms)         (waf_action, waf_category,
                                                        waf_severity, waf_reason)
                             ↓                                   ↓
                       Alertmanager / Grafana          SIEM (Elastic / Loki /
                                                        Splunk / Datadog / SigNoz)
```

That's the whole architecture. Inspection happens inside nginx, not beside it. Your
existing nginx config keeps working; the WAF adds an inline decision step before the
request reaches the backend. The compiled signature DB lives in a binary bundle that
`mmap`-loads once at startup and is shared across every worker via fork-COW.

---

## Try it in 30 seconds

```bash
docker run -d --name npx-waf -p 8080:8080 ghcr.io/203x-io/npx-waf:latest

# Benign request passes through
curl -i http://127.0.0.1:8080/?q=hello
# HTTP/1.1 200 OK
# X-WAF-Action: PASS

# Any classic injection probe (XSS / SQLi / RCE / LFI payload of your choice)
# is blocked at the edge:
curl -i 'http://127.0.0.1:8080/?q=...test-payload-here...'
# HTTP/1.1 403 Forbidden
# X-WAF-Action: BLOCK
# X-WAF-Reason: xss-block      (or sqli-block, rce-block, lfi-block, ...)
```

That's it — one command, working WAF. Mount your own `nginx.conf` (full example below) when you're ready to point traffic at a real backend instead of the demo stub.

---

## Why teams choose npx-waf

|  | npx-waf | Common WAF deployment pattern |
|---|---|---|
| Runtime model | **inside the nginx binary** | sidecar, external proxy, module stack, or managed edge |
| Per-request overhead (median) | **about 3 µs** in the benchmark below | depends heavily on engine, ruleset, and tuning |
| Detection accuracy | **100% in the GoTestWAF v0.5 run below** | must be measured against the deployed ruleset |
| Attack signatures | **4,653**, compiled into a SIMD scan engine | rules often evaluated on the request path |
| Named-CVE virtual-patches | **88** (16 on the CISA KEV list — actively exploited) | varies by vendor, ruleset, and subscription tier |
| Container CVE scan | **0 fixable HIGH/CRITICAL** in Trivy at release | image-dependent |
| Base image | distroless, **no shell**, non-root UID 65532 | frequently general-purpose Linux userland |
| Update flow | **image tag swap** — graceful drain, rollback in seconds | rule reload, config rollout, or vendor-side change |
| Rule update cost per request | **0** (bundle is `mmap`-loaded) | depends on cache, JIT, and reload strategy |
| SBOM | **SPDX 2.3 attached via cosign** | project-dependent |

### Why this matters

- **Enforcement without months of tuning.** GoTestWAF gives you a concrete baseline
  for attack coverage and false-positive rate before production traffic ever touches
  the image. Start in `learn` mode for sensitive legacy routes, or move straight to
  `enforce` where the traffic profile is known.

- **The fast path stays fast.** The ruleset is compiled ahead of time and loaded once.
  There is no per-request rule compilation, no regex hot-reload state, and no
  first-request warmup penalty.

- **Security events are easy to consume.** Every decision lands in your SIEM as a
  top-level JSON field (`waf_action`, `waf_category`, `waf_severity`). No Grok
  patterns to write, no parser pipelines to maintain, no brittle log regex that
  needs babysitting on every release.

- **Integration is operationally boring.** Pull the image, mount your `nginx.conf`,
  point traffic at port 8080. You're swapping plain nginx for nginx with inline WAF
  inspection, not introducing a new request path for the team to operate.

- **Audit data ships with the image.** The release includes an SPDX 2.3 SBOM
  (generated by syft) and a Trivy CVE scan report as inspectable artifacts. Your
  AppSec team gets evidence they can verify, not just a security claim in a README.

Built for teams that already trust nginx at the edge and want the WAF in that same
execution path.

---

## Production deployment

A `docker-compose.yml` for plain HTTP on port 80 (TLS lives in your mounted `nginx.conf`):

```yaml
services:
  waf:
    image: ghcr.io/203x-io/npx-waf:latest
    restart: unless-stopped
    container_name: npx-waf
    ports:
      - "80:8080"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    read_only: true
    tmpfs:
      - /var/cache/nginx:size=64m,uid=65532,gid=65532
      - /run:size=8m,uid=65532,gid=65532
    cap_drop: [ALL]
    cap_add:  [NET_BIND_SERVICE]
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        # Reservations are scheduler hints (k8s / Swarm).  Limits are
        # deliberately omitted — see the Sizing section below and pick
        # numbers that match your worker_processes count and traffic
        # profile.
        reservations:
          cpus:   '0.5'
          memory: 256M
    stop_signal: SIGQUIT
    stop_grace_period: 30s
```

Defense-in-depth posture: read‑only root filesystem, capabilities dropped to one (`NET_BIND_SERVICE`), `no-new-privileges`, and a graceful SIGQUIT shutdown. The container has no shell to spawn and no writable root to drop a payload onto — properties that limit common post-exploitation options if an RCE-class bug ever surfaces inside the WAF.

### Sizing — pick CPU and memory limits for your traffic

Resource usage scales linearly with `worker_processes` (defaults to `auto`, which equals the host's vCPU count).  Measured against the production image, per-worker steady-state:

|  | Per-worker | Notes |
|---|---|---|
| RSS steady-state (idle + light traffic) | **about 30 MB** | Scan-engine scratch (about 400 KB) + nginx worker (about 2 MB) + COW-shared bundle pages |
| RSS peak under heavy POST body inspection (50 KB JSON) | **about 60 MB** | Adds body buffer + decoded-staging + per-MIME scratch |
| Idle CPU | near 0% | epoll waits, no busy loop |
| Per-request CPU (clean GET) | about 3 µs | SIMD single-pass scan |
| Per-request CPU (POST JSON body) | about 5 µs | Adds body decode + multipart parsing |

Plus per-container fixed overhead:

|  | Size |
|---|---|
| nginx master process | about 10 MB |
| Compiled bundle (read-only, COW-shared across all workers) | about 8 MB |
| Shared-memory blocklist zone (sized in `nginx.conf` — default `16m` in our example) | configurable |
| Shared-memory reputation zone (default `8m`) | configurable |

**RAM sizing formula:**

```
peak_RAM_MB = 50 + (worker_processes × 60) + blocklist_zone + reputation_zone
```

Recommended sizing for production:

| Deployment | Workers | RAM limit (peak + headroom) | CPU limit |
|---|---|---|---|
| Small VPS / development | 4 | 512 MB | 2 cores |
| Mid-tier (single-host edge) | 8 | 1 GB | 4 cores |
| Standard production edge | 16 | 2 GB | 8 cores |
| High-throughput edge | 32 | 4 GB | 16 cores |
| Dual-socket / large instance | 64 | 8 GB | 32 cores |

Add `limits` to the compose `deploy.resources` block once you've picked the row for your shape:

```yaml
    deploy:
      resources:
        limits:
          cpus:   '8'        # match your worker_processes
          memory: 2G
        reservations:
          cpus:   '0.5'
          memory: 256M
```

Note on measurements: The numbers above come from the published production image (full nginx module set, default bundle). Custom bundles with fewer patterns lower the COW‑shared baseline; bundles with significantly more patterns raise it — each additional pattern adds about 1.3 KB to the compiled bundle on average.

### A minimal `nginx.conf` showing WAF directives

```nginx
worker_processes  auto;
events { worker_connections 16384; }

http {
    # Load the pre-compiled WAF bundle baked into the image
    waf_serialized_db /var/lib/npx-waf/bundles/baseline/;

    # Shared-memory state — required for blocklist + per-IP reputation
    waf_zone zone=blocklist:16m;
    waf_zone zone=reputation:8m;

    log_format waf_jsonl escape=json
        '{"ts":"$time_iso8601","ip":"$remote_addr","method":"$request_method",'
         '"uri":"$request_uri","status":$status,'
         '"waf_action":"$waf_action","waf_category":"$waf_category",'
         '"waf_severity":"$waf_severity","waf_reason":"$waf_reason"}';

    server {
        listen 8080 default_server;
        access_log /dev/stdout waf_jsonl;

        # Turn on inspection (off by default per-location)
        waf               on;
        waf_mode          enforce;     # alt: learn / shadow (log-only, no block)
        waf_threshold     8;           # 8 = single high-confidence signal blocks
        waf_inspect_body  on;          # scan POST bodies up to client_max_body_size

        # Expose WAF decision to downstream / SIEM
        add_header X-WAF-Action $waf_action always;
        add_header X-WAF-Reason $waf_reason always;

        # Health endpoints — bypass the WAF for kubelet probes
        location = /healthz { waf off; access_log off; return 200 "ok\n"; }
        location = /readyz  { waf off; access_log off; return 200 "ready\n"; }

        # Your real backend
        location / {
            proxy_pass http://your-upstream:3000;
        }
    }
}
```

Mount this at `/etc/nginx/nginx.conf` (as in the compose file above) and the image picks it up on start. Reload with `docker kill --signal=SIGHUP npx-waf` after edits — no restart, no dropped connections.

---

## Migration guide

### Coming from ModSecurity / Coraza

- **A familiar severity model.** `waf_severity` uses CRITICAL / HIGH / MEDIUM / LOW,
  mapped from the anomaly score (CRITICAL ≥ 8, HIGH ≥ 5, MEDIUM ≥ 3, LOW ≥ 1).
  If you're coming from CRS's CRITICAL / ERROR / WARNING / NOTICE labels, the
  mapping is simple: CRITICAL stays CRITICAL, ERROR → HIGH, WARNING → MEDIUM,
  NOTICE → LOW.

- **CRS-style anomaly scoring.** Each match contributes a fixed weight (SQLi=8,
  XSS=8, RCE=8, …). The request blocks once the cumulative score crosses
  `waf_threshold`. You tune the threshold at the nginx location level instead of
  carrying a long list of one-off rule exceptions.

- **Fewer rule-by-rule chores.** Tune by category at the location block
  (`waf_threshold` per location), or bypass known-safe URIs and source ranges with
  explicit nginx directives. The goal is to keep policy readable at the edge instead
  of spreading fragile exceptions across thousands of individual signatures.

### Coming from AWS WAF / Cloudflare WAF

- **The same primitive, on your infrastructure.** Block at the edge, log structured
  JSON, and alert through your existing Prometheus or SIEM stack. The difference is
  placement: npx-waf runs inside your nginx layer instead of upstream of it.

- **Predictable cost model.** Updates are image-tag swaps. You spend on the compute
  you already use for your edge, not on per-request inspection events.

- **CVE virtual-patches in the base image.** The release ships with curated named-CVE
  virtual-patches, including actively-exploited entries on the CISA KEV list. Coverage
  is inspectable in the rule files shipped with the image.

### Coming from F5 / NGINX Plus App Protect

- **Same nginx operating model.** Same config language, same deployment habits. You
  replace the image and keep your existing `nginx.conf`; WAF directives are additive
  (`waf on;` per location).

- **Source-available and self-hosted.** BUSL-1.1 applies to the source, with free
  internal and self-hosted production use for the operator's own applications and
  infrastructure. On the Change Date (January 1, 2030), the source re-licenses to
  Apache 2.0 automatically.

---

## Configuration recipes

Copy-paste building blocks for the patterns operators ask about most. All snippets go inside your mounted `nginx.conf`.

### Real client IP behind a CDN (Cloudflare / AWS CloudFront / Fastly)

```nginx
http {
    # Trust X-Forwarded-For only from your CDN's published IP ranges.
    # Example: Cloudflare — list from https://www.cloudflare.com/ips/
    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 173.245.48.0/20;
    # ...remaining Cloudflare ranges...
    real_ip_header   X-Forwarded-For;
    real_ip_recursive on;
    # All blocklist / reputation / SIEM-log IP fields now use the
    # resolved client IP, not the CDN edge.
}
```

### Whitelist by URI prefix

```nginx
# Bypass WAF inspection for these URI prefixes — directive is
# multi-instance, repeat for each path.  Scope: main / server / location.
waf_whitelist_uri /api/internal/sql-allowed;
waf_whitelist_uri /admin/legacy-route;
waf_whitelist_uri /healthz;
waf_whitelist_uri /readyz;
```

Observed behavior: a SQLi test payload sent to `/api/internal/sql-allowed?id=...payload...` reaches the backend and returns the upstream response with `X-WAF-Action: PASS` — inspection genuinely skipped.

### Whitelist by source IP / CIDR

```nginx
# CIDR ACL.  Skips WAF inspection for requests originating from
# these IP ranges (after the realip module resolved the client IP).
waf_whitelist_ip 10.0.0.0/8;          # internal RFC 1918
waf_whitelist_ip 192.168.0.0/16;
waf_whitelist_ip 203.0.113.7/32;      # specific monitoring host
```

### Blacklist by source IP / CIDR

```nginx
# Pre-emptive block — request returns 403 (or your `waf_status`) before
# any inspection.  CIDR matches the resolved client IP.
waf_blacklist_ip 198.51.100.0/24;     # known-hostile CIDR
waf_blacklist_ip 45.33.32.0/24;
```

Observed behavior: with `waf_blacklist_ip 198.51.100.0/24` in place, a request carrying `X-Forwarded-For: 198.51.100.42` gets HTTP 403 and `X-WAF-Action: BLOCK`. The same payload from an IP outside the range returns HTTP 200.

### Honeypot URIs — catch reconnaissance scanners

```nginx
# REQUIRED for the IP-ban side effect: a shared-memory blocklist
# zone.  Without `waf_zone zone=blocklist:...` the honeypot directive
# still returns 403 on the fake path, but the offending IP is NOT
# added to the dynamic blocklist — subsequent requests from the same
# IP get inspected normally instead of short-circuiting at 403.
waf_zone zone=blocklist:16m;

# Fake-but-tempting paths.  Any request hitting them is, by
# construction, scanner traffic.  Two effects, both at the WAF layer:
#   1. The honeypot request itself is rejected with 403.
#   2. The client IP is inserted into the blocklist zone above and
#      every subsequent request from that IP (any path, any payload)
#      is short-circuited to 403 until the ban TTL expires.
# Pair with a `Disallow:` entry in robots.txt so legitimate crawlers
# skip the path; only scanners ever trip it.
#
# Pick paths that scanners probe for (commonly: exposed VCS metadata,
# environment-file backups, PHP info dumps, well-known admin
# installation URLs).  The four below are placeholders — swap them
# for whatever traps best match the scanner traffic you see in logs.
waf_honeypot_uri /trap/vcs-metadata;
waf_honeypot_uri /trap/env-backup;
waf_honeypot_uri /trap/info-dump;
waf_honeypot_uri /trap/admin-installer;
```

Observed behavior on the live image: a request from IP `A` to any honeypot path returns 403. The very next request from IP `A` — even to a benign path like `/api/users` or `/healthz` — also returns 403, short-circuited by the blocklist. Meanwhile a different IP `B` hitting the same benign path gets 200. The IP-level ban is immediate and cross-worker: the honeypot handler writes to shared memory, and every nginx worker sees the new entry on the next request.

**TTL caveat.** The configured ban TTL is 24 hours, controlled by the `chal_window` and `bl_expire` settings on your `waf_zone` directive. Verify how long the ban actually persists on your own deployment before relying on the 24-hour figure as a hard guarantee.

### Disable an entire category on a noisy endpoint

```nginx
# WordPress wp-admin legitimately uses SQL keywords in admin actions.
# Raise the threshold so a single SQLi pattern alone doesn't block —
# require a second signal (e.g., XSS + SQLi together) to reach
# threshold=16.
location /wp-admin/ {
    waf on;
    waf_threshold 16;
    proxy_pass http://wordpress-backend;
}
```

### Disable a single specific rule (offline bundle recompile)

The image ships with the raw `.rules` / `.data` source files at
`/var/lib/npx-waf/rules/` and the offline compiler at
`/usr/local/sbin/npx-waf-compile`.

```bash
# 1. Extract the rule sources from the image
docker create --name extract ghcr.io/203x-io/npx-waf:latest
docker cp extract:/var/lib/npx-waf/rules ./rules
docker rm extract

# 2. Find the rule line you want disabled and remove (or comment with #) it.
#    Rule files are line-delimited: `SEVERITY:regex` (one rule per line).
#    Example — disable a single CVE-2026-1234 virtual-patch:
grep -n "CVE-2026-1234" ./rules/*.rules
sed -i '/CVE-2026-1234/d' ./rules/npx-vpatch-cve.rules

# 3. Recompile using the offline tool inside the image
mkdir -p bundle
docker run --rm \
    -v "$PWD/rules:/rules:ro" \
    -v "$PWD/bundle:/bundle" \
    --entrypoint /usr/local/sbin/npx-waf-compile \
    ghcr.io/203x-io/npx-waf:latest \
    --rules /rules \
    --output /bundle \
    --name custom \
    --version "$(date -u +%Y%m%d-%H%M%S)"

# 4. Mount the custom bundle into the running container
docker run -d --name npx-waf \
    -v "$PWD/bundle:/var/lib/npx-waf/bundles/baseline:ro" \
    -v "$PWD/nginx.conf:/etc/nginx/nginx.conf:ro" \
    -p 80:8080 \
    ghcr.io/203x-io/npx-waf:latest
```

### Add your own virtual-patch for a new CVE

```bash
# 1. Create your custom-rule file (same format as the bundled .rules)
cat > rules/my-cve-vpatches.rules <<'EOF'
# CVE-2026-XYZ123 — your vendor's vulnerable endpoint signature.
# Severity prefix is freeform (CRITICAL / ERROR / WARNING) — affects
# log severity only, not the score.  Score comes from the filename
# prefix (see scoring model section above).
CRITICAL:(?i)/api/v1/your-vulnerable-endpoint\?action=exploit
CRITICAL:(?i)/cgi-bin/your-vendor/[encoded-traversal-pattern][sensitive-file-name]
EOF

# 2. Recompile the bundle (same as above) — your new file is
#    auto-discovered next to the bundled .rules
# 3. Mount the bundle and reload
```

For categories that map to the CRS scoring model (SQLI=8, XSS=8, RCE=8, …), name your file with a recognised prefix: `npx-sqli-custom.rules`, `npx-rce-custom.rules`, etc. Files starting with `npx-vpatch-` map to the CUSTOM category at score 8 — suitable for one-off CVE patches.

### Run in log-only mode for the first week

```nginx
server {
    waf      on;
    waf_mode learn;          # log decisions, never block — use `enforce` to block
    # ... rest of the config
}
```

In `learn` mode the WAF still runs the full inspection chain — emits `$waf_action`, `$waf_category`, `$waf_severity` log fields and the `X-WAF-Action` response header — but it won't return 403. Observed behavior: an XSS probe under `waf_mode learn;` returns HTTP 200 with an `X-WAF-Action: BLOCK` header and a JSON log line carrying `"action":"BLOCK","category":"XSS","status":200`. You see exactly what would have been blocked, without blocking anything.

Run for a week against your production traffic. Watch `waf_action_total{action="block"}` and the `waf_score` histogram in Prometheus. Adjust `waf_threshold` based on your false-positive rate, then switch to `waf_mode enforce;`.

Note: `shadow` is currently a synonym for `learn` — both are non-blocking log-only modes.

---

## What it blocks

The OWASP Top 10, plus newer attack patterns common in modern API and web stacks:

- **Injection.** SQLi, NoSQLi, LDAP, command injection, code injection in PHP / Python / Java / Node deserialisation, server-side template injection (SSTI), LFI, RFI, XXE, SSI, CRLF.
- **XSS.** Tokeniser-grade detection — survives base64, `%uXXXX`, UTF-8 overlong, mixed-case obfuscation, and DOM-clobbering payloads that defeat naive regex matching.
- **SSRF.** Cloud-metadata endpoints for AWS, GCP, Azure, DigitalOcean, Alibaba, Oracle; localhost and link-local probes; URL-parser confusion bypasses.
- **Scanner reconnaissance.** Over 50 signature families for User-Agent, Referer, and probing-path fingerprints — nuclei, sqlmap, nikto, masscan, and the rest of the noisy bunch.
- **Web shells.** PHP, ASP, and JSP signatures across the common shell families.
- **Named-CVE virtual-patches.** Vendor-specific signatures going back to 2014, including actively-exploited entries on the CISA KEV list — Ivanti EPM, Citrix NetScaler, Apache ActiveMQ, Fortinet, MetInfo, SmarterMail, Langflow RCE, MCPJam Inspector, and more.
- **Modern XSS vectors.** ES6 dynamic `import()`, Shadow DOM injection, importmap abuse, Trusted Types bypasses, JavaScript prototype pollution — techniques drawn from the recent nuclei-templates corpus.

### Scoring model

CRS-style anomaly threshold. Each category contributes a fixed weight on first match:

```
SQLi=8   XSS=8   RCE=8   LFI=8   SSRF=7
Scanner=3  Webshell=8  Upload=4  Disclosure=3
```

Default `waf_threshold = 8` — a single high-confidence hit blocks; lower-confidence signals (e.g. scanner User-Agent alone) require a second signal. Tunable per-location.

---

## Operations

### Health checks

```bash
curl http://127.0.0.1:8080/healthz   # 200 once the worker is up — kubelet liveness
curl http://127.0.0.1:8080/readyz    # 200 once the WAF bundle has loaded — readiness
```

The image declares a Docker-native `HEALTHCHECK` (interval 10s, retries 3) that runs `nginx -t` — the config-test re-deserialises the WAF bundle from disk on every invocation, so a corrupted or version-incompatible bundle fails the check. After three consecutive failures (about 30 seconds) Docker marks the container `unhealthy`; an orchestrator with `condition: service_healthy` (docker-compose) or a `livenessProbe` (Kubernetes) restarts it or pulls it from rotation. Running workers keep serving from the bundle they already loaded — a failed reload doesn't take live traffic offline.

### Prometheus metrics

`/metrics` is bound to `127.0.0.1:9113` **inside the container netns** — not reachable through `-p 9113:9113`. The split is deliberate: the endpoint exposes sensitive operational data (blocklist size, per-category counters, latency histograms) that should never be reachable from outside the pod. Two supported scrape patterns:

```bash
# Kubernetes — Prometheus sidecar in the same Pod (shares localhost):
- name: prometheus
  image: prom/prometheus:v3.0.1
  # Scrape config: targets: ['localhost:9113']

# docker-compose — scraper attached to the WAF's network namespace:
prometheus:
  image: prom/prometheus:v3.0.1
  network_mode: service:waf

# Ad-hoc curl from any container in the same netns:
docker run --rm --network=container:npx-waf curlimages/curl:latest \
    -s http://127.0.0.1:9113/metrics | grep waf_
```

**Exported families:**

```
waf_requests_total                                  counter
waf_action_total{action="pass|challenge|block"}     counter
waf_score                                           histogram
waf_category_hits_total{category="XSS|SQLI|…"}      counter
waf_inspect_duration_us                             histogram
waf_blocklist_hits_total                            counter
waf_rep_penalty_applied_total                       counter
waf_rep_slots_claimed_total                         counter
```

### Structured access log

Every request emits one JSON line on stdout — pipe directly into your SIEM without log parsers:

```json
{"ts":"2026-05-12T00:47:08+00:00","ip":"203.0.113.7","method":"GET",
 "uri":"/?q=...payload-redacted...","status":403,
 "waf_action":"BLOCK","waf_score":"8","waf_category":"XSS",
 "waf_severity":"CRITICAL","waf_reason":"xss-block"}
```

`waf_action` and `waf_category` are first-class JSON fields — SIEM rules filter on them directly.

---

## SIEM integration

The structured access log is built for direct ingestion into modern SIEM stacks. **No regex parsers, no Grok patterns, no field-extraction pipelines** — every field a SOC analyst needs is already a top-level JSON key.

### Ready-to-use shippers

**Vector → Elasticsearch / OpenSearch**

```yaml
# vector.yaml
sources:
  npx_waf_logs:
    type: docker_logs
    include_containers: [npx-waf]

transforms:
  parse_json:
    type: remap
    inputs: [npx_waf_logs]
    source: |
      parsed = parse_json!(.message)
      . = merge!(., parsed)
      .timestamp = parse_timestamp!(.ts, "%+")
      del(.ts)

sinks:
  elasticsearch:
    type: elasticsearch
    inputs: [parse_json]
    endpoints: [http://elasticsearch:9200]
    mode: bulk
    bulk:
      index: 'waf-logs-%Y.%m.%d'
```

**Filebeat → Elasticsearch** — set `json.keys_under_root: true` and Filebeat parses the log lines automatically.

**Promtail → Loki**

```yaml
scrape_configs:
  - job_name: npx-waf
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        filters: [{ name: name, values: [npx-waf] }]
    pipeline_stages:
      - json:
          expressions:
            waf_action: waf_action
            waf_category: waf_category
            waf_severity: waf_severity
            status: status
      - labels: { waf_action: ~, waf_category: ~, waf_severity: ~, status: ~ }
```

**Datadog Agent** — autodiscovery via container label `com.datadoghq.ad.logs='[{"source":"nginx","service":"waf"}]'`; Datadog parses the JSON natively.

**Splunk Universal Forwarder** — `sourcetype = _json`, every `waf_*` field becomes a Splunk field at index time.

### High-signal SIEM rules out of the box

| Detection | KQL / Lucene / SPL |
|---|---|
| All blocked attacks | `waf_action: "BLOCK"` |
| CRITICAL-severity incidents only | `waf_severity: "CRITICAL"` |
| Specific attack class | `waf_category: "RCE"` |
| Score ≥ threshold (high-confidence hits) | `waf_score >= "8"` |
| First-time attacker (no prior blocklist) | `waf_action: "BLOCK" AND waf_reason: "xss-block"` (specific, not generic blocklist short-circuit) |
| Persistent scanner (10+ blocks / IP / hour) | `waf_action: "BLOCK"` grouped by `ip`, count > 10 |
| Named-CVE attempt | `waf_reason: "block"` with no `waf_score` — indicates blocklist or CVE-specific path hit |

### What this means for your SOC

- **Zero parsing engineering.** Detection rules look like `waf_action: "BLOCK"`, not `_grok_match: "(?P<verdict>\\w+).*\\b(?:block|pass)\\b"`. You write queries; you don't maintain log regex.

- **Stable schema across versions.** Image releases follow semver. Field names don't get renamed between minor versions, so the dashboards and alerts you build today keep working when you pull the next tag.

- **Correlation-ready.** The WAF logs nginx's `$request_id` on every decision. Add `proxy_set_header X-Request-ID $request_id;` to your upstream block and the same identifier flows from edge to backend — one query joins the WAF decision with the backend trace on a single key.

- **Anomaly-score-derived severity.** CRITICAL ≥ 8, HIGH ≥ 5, MEDIUM ≥ 3, LOW ≥ 1. Different label set from OWASP CRS's CRITICAL / ERROR / WARNING / NOTICE — if you're migrating rules from a ModSecurity or Coraza deployment, the remap is straightforward: CRITICAL stays, ERROR → HIGH, WARNING → MEDIUM, NOTICE → LOW.

---

## Performance & accuracy

Numbers below come from this image running on AMD Ryzen 9 7950X (Zen 4, 16 cores, sustained around 5.5 GHz with the performance governor on), nginx with 32 workers and `reuseport`, a 3-byte static `index.html` as the backend so upstream latency stays out of the picture. Two server blocks of the same image — port 28102 with `waf on;`, port 28103 with `waf off;` — so the delta below is purely the cost of WAF inspection, not a different TLS stack or event-loop configuration. Numbers are medians of three 5-second `oha` runs.

**Single connection** (`-c 1`, no queueing) — per-request WAF cost:

| Scenario | WAF rps · p50 · p99 | noWAF rps · p50 · p99 | WAF overhead p50 |
|---|---|---|---|
| GET `/` clean | 32,901 · 27 µs · 41 µs | 36,566 · 25 µs · 36 µs | **+2 µs** |
| GET `/?q=`(2 KiB benign args) | 24,854 · 35 µs · 52 µs | 31,177 · 28 µs · 40 µs | +7 µs |
| POST JSON 250 B body | 40,783 · 22 µs · 34 µs | 47,963 · 19 µs · 26 µs | **+3 µs** |
| Attack (XSS → 403, early-exit) | 44,270 · 20 µs · 30 µs | (noWAF→200) 36,590 · 25 µs · 35 µs | **−5 µs** ¹ |

**100 concurrent** (`-c 100`, production-shape edge load):

| Scenario | WAF rps · p50 · p99 | noWAF rps · p50 · p99 | Δ rps |
|---|---|---|---|
| GET `/` | 483,062 · 156 µs · 790 µs | 499,487 · 149 µs · 783 µs | **−3.3%** |
| POST JSON 250 B body | **661,064** · 125 µs · 475 µs | 817,781 · 95 µs · 407 µs | −19.2% |

¹ Attack-path is *faster* than the no-WAF path in this run. This is a benchmark artifact — the `403` early-exit skips nginx's static-file pipeline, while the no-WAF path still has to serve the 3-byte file through it.

**Detection accuracy** — measured against the public Wallarm GoTestWAF v0.5 corpus
(674 attack payloads + 141 benign samples):

| Metric | Result | What it means |
|---|---|---|
| True-Positive (App Security) | 674 / 674 — **100.00%** | Every attack payload blocked |
| True-Positive (API Security) | 14 / 14 — **100.00%** | Every API-shaped attack blocked |
| True-Negative (FP-freedom) | 141 / 141 — **100.00%** | No false positives in this corpus |
| **Overall Score** | **100.00%** | |

Reproducible in 2 minutes — point gotestwaf at any running instance:

```bash
docker run --rm --network=host wallarm/gotestwaf --url=http://127.0.0.1:8080 --noEmailReport
```

Re-run the benchmark yourself before any production rollout. The important number is
not the badge — it is the delta between `waf on;` and `waf off;` on hardware that
looks like yours.

---

## Under the hood

Three layers, all running inside a single nginx ACCESS-phase handler:

1. **Per-field decoder chain.** HTML entities → URL `%XX` and JSON `\uXXXX` (two-pass) → base64 — applied to the URI path, every query argument, body content (per MIME type), every header, every cookie, and Referer. Encoded attacks get decoded before inspection, not after.

2. **Tokeniser-grade SQLi and XSS detection** via an established open-source token engine — catches the semantic injection that pure regex misses (SQL comment splitting, JavaScript event-handler concatenation, polyglot payloads).

3. **SIMD regex scan** of the compiled signature DB in a single pass over a NUL-separated concatenation of every decoded field. AVX2 and AVX-512 on x86, NEON on arm64, with branchless strip-quotes and base64-decode primitives running in 64-byte SIMD lanes.

Rules compile into a binary bundle at image build time and `mmap()` into worker memory. No per-request regex compilation, no JIT warmup, no cold cache. Updates are image-tag swaps with rollback in seconds.

---

## Security posture

- **Distroless base.** Built on `gcr.io/distroless/cc-debian13:nonroot` — no shell,
  no package manager, no busybox. These omissions remove the usual post-exploitation
  tools from the runtime image and keep the container surface deliberately small.

- **Non-root runtime.** Non-root UID 65532, capabilities dropped to `NET_BIND_SERVICE` (or zero if you map ports externally), `no-new-privileges` enforced. The root filesystem is read-only; the only writable paths are `tmpfs` mounts for `/var/cache/nginx` and `/run`.

- **Zero fixable HIGH/CRITICAL CVEs at release.** The release pipeline gates on Trivy
  results before publishing. Bring your own scanner and verify against the attached
  SBOM; both numbers should agree.

- **Hardened binaries.** `strip --strip-all`, debug sections removed (`.comment`, `.note.gnu.build-id`, `.note.ABI-tag`), compiled with `-fvisibility=hidden` and `-Wl,--build-id=none`. The runtime image carries no debug info, no GNU build IDs, no source paths — a smaller surface for fingerprinting and a harder anchor for reverse engineering.

- **Inspectable supply chain.** Every release is signed via cosign keyless OIDC and the signature event is recorded in the public Rekor transparency log. SPDX 2.3 SBOM and a Trivy CVE scan report ship alongside the release. Verify a pulled image with:

```bash
cosign verify ghcr.io/203x-io/npx-waf:<tag> \
  --certificate-identity antonybizov@gmail.com \
  --certificate-oidc-issuer https://accounts.google.com
```

- **Graceful shutdown.** `STOPSIGNAL SIGQUIT` — `docker stop` drains in-flight requests instead of killing them mid-response. Rolling deploys don't drop traffic.

- **Zero phone-home.** The image makes no outbound calls on its own. The only "usage signal" is what GitHub Container Registry itself records when you `docker pull` — no analytics beacons, no license check-ins, no anonymous telemetry from inside the container.

---

## FAQ

**Will this slow down my application?**
Measure it against your own traffic profile, but in our benchmarks the overhead was small: +2 µs at p50 for a clean GET and +3 µs at p50 for a 250 B JSON POST in a single-connection test.

At 100 concurrent connections, throughput for clean GET requests was 3.3% lower than with `waf off;`. Throughput for JSON POST requests was 19.2% lower because decoding and inspecting the request body does add real work. For most API edge workloads, though, that overhead is still well below the latency introduced by the upstream application.

**Can I run it in log-only mode without blocking traffic?**
Yes. Set `waf_mode learn;` per location; `waf_mode shadow;` is currently just an alias. Decisions are still logged with the full `waf_*` metadata, and the `X-WAF-Action` header is still added, but requests continue to pass through to the backend. Run it on production traffic for a week, watch the score distribution in Prometheus, and switch to `waf_mode enforce;` once you're comfortable with the threshold.

**What happens during a rule update?**
Rule updates are shipped as new image tags. Pull the new tag and run `docker compose up -d`; Docker will replace the container normally. `SIGQUIT` gives the old container time to drain in-flight requests while the new one starts with the updated ruleset. There's no regex hot-reload state to coordinate, and rollback is the same image swap in reverse.

**Does it work behind Cloudflare / AWS ALB / CloudFront?**
Yes. Add `real_ip_from <cdn-cidr>;` and `real_ip_header X-Forwarded-For;` to `nginx.conf`. The WAF reads the resolved client IP from `r->connection->sockaddr` after nginx's realip module populates it. Blocklist, reputation, and SIEM log entries will all use the resolved client IP, not the CDN edge IP.

**How much memory does it use?**
The WAF bundle loads once into the nginx master process, and workers inherit it via copy-on-write. Pages remain shared since the bundle is read-only. In the benchmarks above, steady-state RSS is ~30 MB per worker under light traffic, peaking at ~60 MB during POST body inspection.

**Can I add my own custom rules?**
Yes — patterns compile into a binary bundle, so adding a rule requires recompiling the bundle. The offline compiler `/usr/local/sbin/npx-waf-compile` ships in the image; no separate toolchain needed. See the full workflow with copy-pasteable commands in [Configuration recipes — Disable a single specific rule](#disable-a-single-specific-rule-offline-bundle-recompile) and [Add your own virtual-patch for a new CVE](#add-your-own-virtual-patch-for-a-new-cve) above. The same steps work for adding or removing any rule: extract the `.rules` source directory from the image, edit/add files, recompile with the built-in tool, and mount the new bundle as a volume.

**What's BUSL-1.1 — can I use it in production?**
Yes, for internal/self-hosted deployments protecting your own apps and infrastructure. Service providers, hosted, or multi-tenant deployments serving third parties (SaaS, hosted WAF, managed edge filtering) require commercial terms. On January 1, 2030 (the Change Date), it automatically re-licenses to Apache 2.0. See the `LICENSE` file for exact terms.

**Why aren't `/metrics` and `/healthz` on different ports?**
They are. `/metrics` is on port `9113` (bound to `127.0.0.1` in the container netns); `/healthz` is on the public `8080`. The split is deliberate: kubelet probes need an unauthenticated public endpoint, while `/metrics` exposes sensitive operational data (blocklist size, latency histograms, per-category counters) that should never be reachable outside the pod.

**Can I see the full rule set?**
Yes. The raw `.rules` and `.data` source files are included in the image at `/var/lib/npx-waf/rules/` — read-only and fully inspectable.

---

## License

License: **BUSL-1.1**. Free for internal/self-hosted deployments protecting your own apps and infrastructure. Converts to Apache 2.0 on January 1, 2030.

Service-provider, hosted, or multi-tenant deployments serving third parties (SaaS, hosted WAF, managed edge filtering) require commercial terms. See the included `LICENSE` file for details.

---

## Support

General questions, feature requests, configuration help, and operational discussion: open a thread at **<https://github.com/203x-io/npx-waf/discussions>**.

Responsible-disclosure security reports and commercial inquiries: contact the maintainer privately. Please don't file security issues in the public Discussions board — pre-disclosure exposure could let attackers weaponise the bug before a patched image ships.

For larger fleets, an optional hosted control plane is available — automated rule-update rollout and multi-tenant management, with security-analyst escalation under separate terms. The image runs standalone; the control plane is an add-on for teams managing many edges centrally.
