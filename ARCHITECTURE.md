# ARCHITECTURE.md

## Riree

Riree is a small-footprint relay system with **three roles**:

* **Bridge**: public ingress inside constrained networks; accepts client connections and forwards traffic to exits.
* **Exit**: egress outside constrained networks; dials destinations and relays back.
* **Manager**: optional-to-required (by phase) control plane for provisioning, policy, node coordination, and automation.

Design intent: **operators can stand up a working system with minimal moving parts** (single binary + systemd + SQLite), and later scale into a managed fleet without rewriting the data plane.

---

## 0) Scope, goals, constraints

### Goals

* **Fast setup**: download one binary, run `riree install`, enable systemd.
* **Low dependencies**: no Docker, no external DB required.
* **Client compatibility**: users connect using common V2Ray-family clients (VLESS).
* **Operational safety**: basic abuse controls, accounting, revocation.
* **Progressive complexity**: Phase A works without Manager; later phases introduce Manager and automation.

### Non-goals (v1–v2)

* UDP relay
* Complex routing rules (split tunneling, geo rules)
* A public user portal (future: web portal / Telegram bot as separate components)

### Hard constraints

* System-wide install (root at install time; runtime as unprivileged user).
* Operators choose listening ports (not hardcoded to 443).
* **Bridge↔Exit uses self-issued mTLS** (no public CA).
* Public TLS for Client↔Bridge is **preferred**; fallback modes exist because operators vary.

---

## 1) Phased delivery plan (realistic ladder)

### Phase A — “Two nodes, one config, works today”

**Target operator**: solo admin, 1 bridge + 1 exit.

* Commands: `riree bridge`, `riree exit`, `riree user add|revoke|list`, `riree pki init|issue`
* Bridge accepts VLESS and forwards to a single exit.
* Bridge↔Exit: persistent mTLS connection + multiplexed streams.
* Storage: one SQLite DB on Bridge (users + local policy). Exit can be stateless.
* No Manager required.

### Phase B — “Resilience without a control plane”

**Target operator**: needs uptime.

* Bridge supports multiple exits (priority list + health checks + fast failover).
* Drain/maintenance modes for exits.
* Better accounting and rate limits.
* Still no Manager required.

### Phase C — “Manager optional, enables sane operations”

**Target operator**: multiple bridges/exits, needs centralized provisioning.

* Manager issues user identities, pushes policy bundles to bridges.
* Node enrollment and cert issuance for Bridge↔Exit (still self-issued CA, but managed).
* Bridges cache bundles and keep working if Manager is down.

### Phase D — “Fleet mode: Manager required”

**Target operator**: serious deployments.

* Automated exit assignment, health scoring, rotation, audit logs.
* Rolling updates / policy rollout.
* Multi-bridge + multi-exit orchestration.

### Phase E — “UX layer”

* Web portal + Telegram bot (separate services; never put them inside Bridge/Exit).
* These talk only to Manager.

---

## 2) High-level architecture

### Data plane (traffic path)

```
Client (VLESS) ──TLS/TCP──> Bridge ──mTLS + MUX──> Exit ──TCP──> Destination
                                     ^ streams ^
```

### Control plane (by phase)

* Phase A/B: local-only control (SQLite on Bridge + CLI).
* Phase C+: Manager becomes source of truth for users/policy/nodes.

---

## 3) Components and responsibilities

### 3.1 Bridge

**Responsibilities**

* Accept VLESS client sessions.
* Enforce **admission control** (limits, bans).
* Convert each client connection into a **stream** inside the Bridge↔Exit tunnel.
* Maintain a small set of **long-lived tunnels** to exits; multiplex many client streams.
* Report health/metrics (Phase C+: to Manager).

**Implementation direction**

* Embed Xray-core as a library for VLESS parsing/compatibility.
* Implement a custom outbound in-process that sends traffic over Riree tunnels.

**No separate Xray process**: Riree runs it in-process to keep setup simple.

### 3.2 Exit

**Responsibilities**

* Accept Bridge tunnels (mTLS required).
* For each tunnel stream, dial the destination and relay bytes.
* Enforce egress policy (basic port policy, rate limits).
* Track per-user usage (Phase B+: required to survive abuse).

Exit does **not** accept direct client connections.

### 3.3 Manager (Phase C+)

**Responsibilities**

* Node registry and enrollment.
* Issue/rotate node certs and push policy bundles.
* User provisioning (create/revoke/limits).
* Exit assignment logic (Phase D).
* Audit logging and reporting.

Manager is **not a public internet-facing service** in normal deployments. If a portal exists later, it is separate and talks to Manager privately.

---

## 4) Trust and security model

### 4.1 Trust zones

* Client devices/networks: untrusted.
* Bridge: exposed to hostile input and load; treat as semi-trusted.
* Exit: exposed to abuse; treat as semi-trusted.
* Manager: most trusted.

### 4.2 Identity model

* **User identity**: UUID (VLESS user id).
* **Node identity**: cert-based identity for Bridge/Exit tunnels.

### 4.3 TLS & cert strategy

#### Bridge↔Exit (mandatory mTLS, self-issued)

* A Riree CA is generated once (`riree pki init`).
* Bridge and Exit get leaf certs (`riree pki issue --role bridge|exit`).
* Tunnel server (Exit) requires client cert signed by Riree CA.
* Bridge pins the Exit CA (or full chain).

This is free and reliable; no external dependencies.

#### Client↔Bridge (preferred public TLS)

Riree supports multiple modes; operators choose:

1. **ACME mode (recommended)**: Bridge obtains cert automatically.

   * Requires domain DNS pointing to Bridge IP and inbound reachability on chosen ACME challenge ports.
   * Uses Let's Encrypt or any ACME-compatible CA.

2. **Manual mode**: operator provides cert/key files.

   * Works for custom CAs or enterprise environments.

3. **Terminated mode**: TLS terminated by an external reverse proxy (nginx/caddy).

   * Riree listens on localhost/plain TCP.
   * This sacrifices “single binary does everything” but keeps security and flexibility.

4. **Disabled mode** (allowed only as last resort): plaintext VLESS.

   * Supported for lab/testing; discouraged for real use.

**Rule**: bridge must document which TLS mode is used and how clients should configure it.

### 4.4 Revocation and abuse containment

Minimum viable controls:

* per-user **max concurrent connections**
* per-user **new connections per minute**
* per-user **bandwidth caps** (optional but strongly recommended)
* server-side **denylist** (revoked users)
* exit-side **egress policy** (block known abuse ports if desired)

Revocation must be enforced at Bridge first (drop early), and at Exit as a backstop.

---

## 5) Riree tunnel design (Bridge↔Exit)

### 5.1 Goals

* Few long-lived connections.
* Many independent bidirectional streams.
* Simple reconnection and failover.
* Small, testable framing.

### 5.2 Transport

* Single TCP connection wrapped in mTLS.
* Stream multiplexing library (yamux-style) to provide:

  * stream open/close
  * backpressure
  * independent flow control at stream-level

### 5.3 Stream types (v1)

* **CONNECT** stream: represents one proxied TCP connection.

  * metadata: user UUID, destination host, destination port, optional timeouts
  * payload: raw TCP bytes

### 5.4 Failover (Phase B+)

* Bridge maintains ordered list of exits.
* If tunnel fails:

  * reconnect to current exit (quick retry)
  * otherwise switch to next exit and re-establish tunnels
* Existing streams are lost on hard fail (acceptable for v1 TCP), but reconnection should be fast.

---

## 6) Data plane integration: VLESS inbound → tunnel outbound

### 6.1 Inbound

* VLESS inbound listener in Bridge (in-process).
* Each accepted session yields:

  * authenticated user UUID
  * destination address and port (from VLESS request)
  * a bidirectional byte stream

### 6.2 Outbound

Bridge outbound does:

1. check user policy (revoked? limits?)
2. open a tunnel stream to Exit with CONNECT metadata
3. relay bytes both directions until EOF/error
4. update counters (bytes, duration)

No external processes, no local SOCKS hop, no extra configuration.

---

## 7) Storage model (SQLite)

### 7.1 Why SQLite

* single file, atomic updates
* supports migrations
* good fit for “systemd + one binary”
* later: Manager can also use SQLite initially, and move to Postgres if needed

### 7.2 Files

* DB file: `/var/lib/riree/riree.db`
* migrations are embedded in the binary and applied on startup.

### 7.3 Suggested schema (Phase A/B)

**users**

* `user_uuid TEXT PRIMARY KEY`
* `status INTEGER` (1=active, 2=revoked)
* `created_at INTEGER`
* `revoked_at INTEGER NULL`
* `limits_json TEXT` (or separate columns for simplicity)

**usage_daily** (optional Phase B, recommended)

* `user_uuid TEXT`
* `day INTEGER` (YYYYMMDD)
* `bytes_up INTEGER`
* `bytes_down INTEGER`
* `conns INTEGER`
* PRIMARY KEY (`user_uuid`, `day`)

**exits** (Phase B)

* `exit_id TEXT PRIMARY KEY`
* `host TEXT`
* `port INTEGER`
* `priority INTEGER`
* `enabled INTEGER`
* `last_ok_at INTEGER`

**local_policy**

* `key TEXT PRIMARY KEY`
* `value TEXT`

Manager adds tables later (nodes, certs, assignments).

---

## 8) CLI and commands

### 8.1 Philosophy

* CLI first (copy/paste).
* No TUI until Manager fleet mode is real.
* Every operation should be doable non-interactively.

### 8.2 Command surface (minimum)

**Runtime**

* `riree bridge --config /etc/riree/riree.yaml`
* `riree exit --config /etc/riree/riree.yaml`
* `riree manager --config /etc/riree/riree.yaml` (Phase C+)

**Lifecycle**

* `riree install --role bridge|exit|manager`
* `riree uninstall [--purge]`
* `riree status`
* `riree config check`

**Users (Bridge local in Phase A/B; Manager in Phase C+)**

* `riree user add [--limit ...]` → prints UUID + VLESS link
* `riree user revoke <uuid>`
* `riree user list`

**PKI (Bridge↔Exit)**

* `riree pki init` (creates CA)
* `riree pki issue --role bridge|exit --out /etc/riree/pki/`
* `riree pki rotate` (Phase C+ or advanced)

**TLS for Client↔Bridge**

* `riree tls acme enable --domain example.com` (optional)
* `riree tls status`

**Ops**

* `riree exit drain` (Phase B)
* `riree bridge exits list|test` (Phase B)

### 8.3 Reload behavior

* SIGHUP triggers:

  * reload config file
  * reload user policy from SQLite
* Should not require restarts for common changes.

---

## 9) Configuration

### 9.1 Location (system-wide)

* `/etc/riree/riree.yaml`
* `/etc/riree/pki/` (cert material)
* `/var/lib/riree/` (DB)

### 9.2 Config structure (example)

```yaml
common:
  role: bridge              # bridge|exit|manager
  node_id: "bridge-01"
  data_dir: "/var/lib/riree"
  log_level: "info"

bridge:
  listen:
    host: "0.0.0.0"
    port: 443               # operator chooses
  vless:
    enabled: true
  tls:
    mode: "acme"            # acme|manual|terminated|disabled
    domain: "bridge.example.com"
    cert_file: ""           # for manual mode
    key_file: ""            # for manual mode
  exits:
    - id: "exit-01"
      host: "exit.example.net"
      port: 443
      priority: 10
  tunnel:
    mtls:
      ca_file: "/etc/riree/pki/ca.crt"
      cert_file: "/etc/riree/pki/bridge.crt"
      key_file: "/etc/riree/pki/bridge.key"
  limits:
    max_conns_per_user: 4
    max_new_conns_per_min: 30

exit:
  listen:
    host: "0.0.0.0"
    port: 443
  tunnel:
    mtls:
      ca_file: "/etc/riree/pki/ca.crt"
      cert_file: "/etc/riree/pki/exit.crt"
      key_file: "/etc/riree/pki/exit.key"
  egress_policy:
    blocked_ports: [25, 465, 587]   # optional

manager:
  enabled: false
  listen:
    host: "127.0.0.1"
    port: 9443
  db:
    path: "/var/lib/riree/riree.db"
```

---

## 10) systemd integration

### 10.1 Runtime user

* Create a system user/group: `riree:riree`
* Service runs unprivileged with capability to bind low ports:

  * `AmbientCapabilities=CAP_NET_BIND_SERVICE`
  * `NoNewPrivileges=true`

### 10.2 Unit file (template)

```ini
[Unit]
Description=Riree (%i)
After=network-online.target
Wants=network-online.target

[Service]
User=riree
Group=riree
ExecStart=/usr/local/bin/riree %i --config /etc/riree/riree.yaml
Restart=always
RestartSec=2
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
LimitNOFILE=1048576

# Hardening (adjust as needed)
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/etc/riree /var/lib/riree /run/riree
CapabilityBoundingSet=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

Operators enable role instances:

* `systemctl enable --now riree@bridge`
* `systemctl enable --now riree@exit`

---

## 11) Install/uninstall rules

### 11.1 `riree install`

Must:

* install binary to `/usr/local/bin/riree` (or verify it exists)
* create `riree` user/group
* create directories with correct permissions:

  * `/etc/riree` (root:root, 0755)
  * `/etc/riree/pki` (root:riree, 0750)
  * `/var/lib/riree` (riree:riree, 0750)
  * `/run/riree` (riree:riree)
* write a minimal config template if none exists
* install systemd unit template `riree@.service`
* enable/start the requested role (optional flag)

### 11.2 `riree uninstall`

Default behavior is conservative:

* stop/disable units
* remove unit files
* optionally remove binary
* keep `/etc/riree` and `/var/lib/riree` by default

`riree uninstall --purge` removes:

* `/etc/riree/*`
* `/var/lib/riree/*`
* system user/group (optional flag; safer as `--purge --remove-user`)

---

## 12) Observability

### Metrics

Expose Prometheus-style metrics on localhost (configurable):

* active connections (total, per user)
* new connections per minute
* bytes up/down (total, per user)
* tunnel state: connected, reconnects, RTT estimates
* exit dial errors/timeouts

### Logs

Structured logs to journald:

* connection open/close with:

  * user id (maybe hashed), dst host/port, bytes, duration, error reason
* do not log payload

---

## 13) Operational playbooks (minimum)

### Add a user

* `riree user add`
* distribute VLESS link to user
* user connects to bridge domain/IP and port

### Revoke a user

* `riree user revoke <uuid>`
* bridge enforces immediately (reload if needed)

### Rotate an exit

* Phase A: edit config to point to new exit, reload bridge
* Phase B+: `riree exit drain` then remove/replace

### Cert failures

* Bridge↔Exit: re-issue certs from same CA and reload services
* Client↔Bridge: switch TLS mode to manual or terminated if ACME isn’t feasible

---

## 14) Repository structure

```
/cmd/riree/                 # main binary (subcommands)
/internal/
  bridge/                   # bridge runtime
  exit/                     # exit runtime
  manager/                  # manager runtime (phase C+)
  tunnel/                   # mTLS + mux + framing
  vless/                    # glue to embedded xray-core
  store/                    # sqlite, migrations
  pki/                      # CA + cert issuance
  tls/                      # acme/manual/terminated helpers
  policy/                   # limits, revocations
  obs/                      # logging, metrics
/docs/
  ARCHITECTURE.md
  OPERATIONS.md
  CONFIG.md
/scripts/
  install.sh (optional wrapper)
```

---

## 15) Compatibility and upgrade rules

* Config is backward compatible across patch releases.
* DB migrations are automatic; binary must refuse to start if migration fails.
* Bridge must be able to run with older Exit for at least one minor version (Phase B+ goal).

---

## 16) Roadmap notes (what NOT to bake in early)

* Don’t build portals/bots into Bridge/Exit.
* Don’t hardcode “443”; keep ports configurable.
* Don’t add “clever” behaviors; keep the system observable and debuggable.
* Add Manager only when you need centralized provisioning and multi-node automation.

---

## 17) Safety and operational responsibility

Operators are responsible for complying with applicable laws and running the system responsibly (abuse controls, revocation, and auditability are mandatory in practice).
