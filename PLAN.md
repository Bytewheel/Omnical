# Omnical — Custom Contacts / Calendar / Tasks Server on the libreCMC Router

**Implementation Plan — v1.0**

- **Date:** 2026-09-04
- **Project:** `omnical`
- **Goal:** A self-hosted, domain-agnostic CalDAV (calendar + VTODO tasks) and CardDAV (contacts) server running on the libreCMC router at `192.168.10.1`, reachable from the public internet with real TLS via `0115d8cf.duckdns.org`, serving accounts for **any email address / any email domain**, acting as a **sync hub for accounts hosted on any provider**, with **sharing across identities** and **cross-domain invitations**.
- **Server software decision (approved):** cross-compile [RustiCal](https://github.com/lennart-k/rustical) (AGPL-3.0, Rust, single static binary) + one small custom-built component (`dav-tls`, a rustls TLS tunnel).
- **Storage decision (approved):** internal router flash + nightly backups.
- **Access/TLS decision (approved):** public hostname `0115d8cf.duckdns.org` with Let's Encrypt (DNS-01 via DuckDNS API).
- **Required clients (approved):** vdirsyncer + khal/khard, Apple Calendar/Contacts, DAVx5 + Tasks.org (Android), Thunderbird.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Target Architecture](#2-target-architecture)
3. [Environment Inventory (verified facts)](#3-environment-inventory-verified-facts)
4. [Constraints, Decisions, and Rationale](#4-constraints-decisions-and-rationale)
5. [Phase 0 — Prerequisites](#phase-0--prerequisites)
6. [Phase 1 — Build (dev machine)](#phase-1--build-dev-machine)
7. [Phase 2 — Install on Router](#phase-2--install-on-router)
8. [Phase 3 — TLS Certificate Automation](#phase-3--tls-certificate-automation)
9. [Phase 4 — Firewall & Reachability](#phase-4--firewall--reachability)
10. [Phase 5 — Users, Groups, Collections (any email domain)](#phase-5--users-groups-collections-any-email-domain)
11. [Phase 6 — Sync Hub for Any Provider (vdirsyncer)](#phase-6--sync-hub-for-any-provider-vdirsyncer)
12. [Phase 7 — End-User Clients & Cross-Domain Invitations](#phase-7--end-user-clients--cross-domain-invitations)
13. [Phase 8 — Backups & Maintenance](#phase-8--backups--maintenance)
14. [Verification Matrix](#14-verification-matrix)
15. [Rollback / Recovery](#15-rollback--recovery)
16. [Risks & Mitigations](#16-risks--mitigations)
17. [Future / Optional Enhancements](#17-future--optional-enhancements)
18. [Appendix A — RustiCal Reference Notes](#appendix-a--rustical-reference-notes)
19. [Appendix B — Key Commands & File Paths](#appendix-b--key-commands--file-paths)
20. [Appendix C — Relevant RFCs](#appendix-c--relevant-rfcs)

---

## 1. Executive Summary

The libreCMC v6.8 router (ThinkPenguin TPE-R1400, aarch64) cannot run the usual
CalDAV/CardDAV servers from packages: its opkg feeds (~2,435 packages) contain **no
python3, php, node, nginx, lighttpd, stunnel, or haproxy**, and the pre-installed socat
is compiled **without OpenSSL**. The established, proven pattern on this network is the
`router-nym` project: statically cross-compiled Rust binaries (via the zig
`aarch64-linux-musl` toolchain) pushed to the router over SCP, managed by procd init
scripts.

This plan applies that pattern to **RustiCal** — an actively developed, libre
(AGPL-3.0) CalDAV/CardDAV server in Rust (axum + sqlx + statically bundled SQLite,
embedded web frontend, WebDAV Push, group sharing, Nextcloud login flow, tested with
DAVx5/Tasks.org/Thunderbird/Apple/GNOME). RustiCal serves plain HTTP only and expects
a TLS-terminating reverse proxy; since none is installable on the router, we build
**one small custom component**: `dav-tls`, a ~100–150 line rustls TLS tunnel
(`0.0.0.0:443 → 127.0.0.1:4000`), cross-compiled with the same toolchain.

Public reachability is achieved with a Let's Encrypt certificate for
`0115d8cf.duckdns.org` (resolves to the current public IP `65.33.235.245`), issued via
**DNS-01** through the DuckDNS API (no inbound port 80 required), automated with
`acme.sh` on the dev machine, and deployed to the router on renewal. A single
port-forward (TCP 443 → `192.168.1.21:443`) is configured once on the upstream TP-Link
router.

"Works across any email domain" is delivered on four levels (all approved):

| Requirement | How it is met |
|---|---|
| **Multi-identity accounts** | RustiCal principals are arbitrary strings — one account per full email address (7 identities across 4 domains today, any future address works); auth is user-id + app token, domain-agnostic |
| **Sync hub for any provider** | vdirsyncer on the dev machine pairs Google (or any CalDAV/CardDAV provider) ↔ local vdirs ↔ Omnical, making the router the convergence point for accounts hosted anywhere |
| **Sharing across domains** | RustiCal groups + memberships; per-principal ACL sharing independent of email domain |
| **Cross-domain invitations** | Client-side iMIP/iTIP (Apple Calendar / Thunderbird email the `.ics` invites via any SMTP identity to any attendee address). RustiCal does **not** implement RFC 6638 server-side scheduling — documented limitation, optional future extension |

Total router footprint: one ~15–35 MB stripped binary + one ~2–4 MB tunnel + a SQLite
database (KBs–MBs) + TLS certs, inside the 42 MB free on the router's internal f2fs
overlay, with nightly pull-based backups to the dev machine and sysupgrade
survivability configured.

---

## 2. Target Architecture

```
                 ┌──────────────────────────────────────────────────────────┐
                 │                    CLIENTS                                │
                 │  DAVx5 + Tasks.org · Apple Calendar/Contacts ·            │
                 │  Thunderbird · vdirsyncer (dev machine sync hub)          │
                 └────────────────────────┬─────────────────────────────────┘
                                          │  HTTPS, real Let's Encrypt cert
                                          │  https://0115d8cf.duckdns.org:443
                                          ▼
                    ┌───────────────────────────────────────────┐
                    │  Upstream TP-Link router (192.168.1.1)     │
                    │  NAT port-forward: TCP 443 → 192.168.1.21  │
                    └────────────────────────┬──────────────────┘
                                          ▼
        ┌───────────────────────────────────────────────────────────────┐
        │  libreCMC v6.8 — ThinkPenguin TPE-R1400 (RK3328, 4× A53)       │
        │  WAN: 192.168.1.21 · LAN: 192.168.10.1 · ULA: fd1e:5c71:1ee8::1 │
        │                                                                 │
        │   ┌─────────────────┐      ┌──────────────────────────────┐   │
        │   │  dav-tls (NEW)  │      │  rustical (RustiCal 0.16.1)  │   │
        │   │  rustls tunnel  │─────►│  127.0.0.1:4000              │   │
        │   │  0.0.0.0:443   │      │  axum · sqlx · SQLite        │   │
        │   │  LE cert        │      │  embedded web UI + admin     │   │
        │   └─────────────────┘      └──────────────┬───────────────┘   │
        │                                            ▼                   │
        │                /usr/local/share/rustical/db.sqlite3          │
        │                (persistent overlay — NOT /var, which is      │
        │                 tmpfs and wiped every boot)                   │
        │                                                                 │
        │  procd services: rustical, dav-tls · cron · uhttpd (LuCI,     │
        │  moved to :8443) · bind authoritative DNS · wireguard tunnels  │
        └───────────────────────────────────────────────────────────────┘
                                          ▲
                                          │ nightly pull backup over SSH
                 ┌────────────────────────┴─────────────────────────┐
                 │  Dev machine (ssh alias `router`, key ~/.ssh/router) │
                 │  builds binaries (zig + rust musl), runs acme.sh,     │
                 │  runs vdirsyncer sync hub (Google ↔ vdirs ↔ Omnical)  │
                 └───────────────────────────────────────────────────────┘
```

**Data flow of the sync hub (any provider → router):**

```
Google Calendar/Contacts (any identity)  ──vdirsyncer──►  ~/.calendars, ~/.config/khard/contacts (vdirs)
                                                        │
                                                        │ vdirsyncer (caldav/carddav storages)
                                                        ▼
                                        Omnical on the router (CalDAV/CardDAV)
                                                        ▲
                                    ┌───────────────────┴──────────────────┐
                                    │  all other clients (Apple, DAVx5,   │
                                    │  Thunderbird) read/write directly    │
                                    └──────────────────────────────────────┘
```

---

## 3. Environment Inventory (verified facts)

All facts below were verified directly on 2026-09-04. They are the foundation the plan
is built on; re-verify anything that may have drifted before executing a phase.

### 3.1 Router: `router` (SSH alias → `root@192.168.10.1`, key `~/.ssh/router`)

| Item | Value |
|---|---|
| Model | ThinkPenguin TPE-R1400 (RockChip RK3328, 4× Cortex-A53, aarch64) |
| Firmware | libreCMC v6.8, revision `r0-44c488c9`, kernel 5.15.210, target `rockchip/armv8`, arch `aarch64_generic`, no taints |
| RAM | 1,011,084 KB total; ~828 MB available (127 MB used incl. nym/mullvad/bind stack) |
| Root/overlay | `/dev/root` squashfs 3.8M (read-only rom); `/dev/loop0` f2fs on overlay, 98.4M total, **42.2M free** (57% used) |
| tmpfs | `/tmp` ~494M (RAM-backed — **never put persistent data here**) |
| USB | One SuperSpeed USB device attached (the RTL8152 USB WAN NIC); `usb-storage` driver registered; a free USB host port may exist but is unconfirmed — plan does not depend on it |
| Init system | procd (+ procd-ujail, procd-seccomp available) |
| Cron | Active (`/etc/crontabs/root`): bind `srvzone` hourly at :50, root-hints update Sundays 03:30, `screech-watchdog` every 5 min |
| Web server | uhttpd (LuCI) — `listen_http` 192.168.10.1:80, 10.10.10.1:80, [fd1e:5c71:1ee8::1]:80; `listen_https` same IPs on **443** (conflicts with plan → moved to 8443); `redirect_https=1`; self-signed `/etc/uhttpd.crt` |
| DNS | bind 9.18 authoritative (OpenNIC zones, e.g. `truth.zone`), forwarding to split-dns dnsmasq instances on ports 5354–5360 (mullvad, nym1–4, direct, penguin) |
| Tunnels | WireGuard: `wg-vm1` 10.100.1.2/30, `wg-vm3` 10.100.3.2/30 (user's VMs); mullvad 10.64.70.187/32; nym exits (see nym stack from router-nym) |
| WAN | `eth0` 192.168.1.21/24 (DHCP, hostname `libreCMC`) — behind upstream router **192.168.1.1** (answers ping; TP-Link, per tplinkdns.com traces) |
| LAN | `br-lan` 192.168.10.1/24; IPv6 ULA `fd1e:5c71:1ee8::1/60`; also 10.10.10.1 in uhttpd config (address not currently on br-lan) |
| opkg feeds | `librecmc_core` (…/v6.8/targets/rockchip/armv8/packages), `librecmc_base` (…/v6.8/packages/aarch64_generic/base) — **2,435 packages listed** |
| Packages NOT available in feeds | python3, php, node, nginx, lighttpd, stunnel, haproxy, radicale — **confirmed absent** (rules out all standard DAV servers and TLS proxies) |
| Packages available & relevant | `sqlite3-cli 3410200-1` (for online DB backups), `ca-bundle 20260223-1` (`/etc/ssl/certs/ca-certificates.crt` present), `curl 8.5.0` (for duckdns update), `socat 1.7.4.4` (but compiled `#undef WITH_OPENSSL` — **cannot terminate TLS**) |
| Firewall | nftables (firewall4): zones lan/wan/vpn (+ screech_z_* zones for wg/mullvad/nym1-4/penguin); wan-input allow rules exist for DHCP/Ping/DNS etc. — pattern to follow for adding TCP 443 |

### 3.2 Dev machine (the workstation running this build)

| Item | Value |
|---|---|
| rustc | 1.97.1 — **too old for RustiCal (requires ≥ 1.98) → `rustup update stable` in Phase 0** |
| rustup targets | `aarch64-unknown-linux-musl` already installed; `x86_64-unknown-linux-gnu` for local smoke tests |
| zig | 0.16.0 (`/usr/bin/zig`) — proven as CC/AR/RANLIB/linker for musl cross-builds |
| Go | 1.26.7 (not needed for chosen approach) |
| bun | available (frontend build fallback if RustiCal's dist is not committed) |
| router-nym assets | `~/router-nym/build/bin/zig-musl-cc` wrapper scripts (CC/AR/RANLIB/NM/linker) + `scripts/build-deps.sh`, `scripts/build-rust.sh`, `deploy.sh` — **the template for this project's build/deploy** |
| SSH | `Host router` alias configured (root@192.168.10.1, IdentityFile `~/.ssh/router`, known_hosts pinned) |

### 3.3 Network / public reachability

- `0115d8cf.duckdns.org` → A record → `65.33.235.245` — matches the current public IP
  of the upstream network (verified via `dig`, `curl ifconfig.me`, and UPnP
  `GetExternalIPAddress`). **CORRECTED 2026-09-04: the hostname is NOT free** — an
  existing Deco-app forward (TCP 443 → 192.168.1.107) publicly serves the OpenNIC
  `.truth` TLD website from the dev machine's Apache under this name with a certbot
  LE cert (see Phase 0 step 3). Per user decision, **omnical uses external port 8443**
  (verified free); the `.truth` site keeps 443.
- Public exposure therefore requires one upstream port-forward (**TCP 8443 →
  192.168.1.21:8443, DONE 2026-09-04 via UPnP IGD**, see Phase 0 step 3 for the
  Deco X60 quirks and the persistence caveat) plus a libreCMC firewall input rule
  (Phase 4). DuckDNS supports TXT records via its API → **Let's Encrypt DNS-01 works
  with no inbound port 80 at all**. Note: a cert for this hostname already exists on
  the dev machine (certbot, `/etc/letsencrypt`, expires 2026-10-07) — Phase 3 must
  decide between reusing it or issuing alongside it; two ACME clients for one name
  is redundant but harmless.

### 3.4 Identity & data ecosystem (who/what this serves)

Email identities (7, across 4 domains — any future address must also work.
**CONFIRMED by user 2026-09-04**: the original 6 below plus `zero@novo-ordo.com`,
which was found in `~/.mbsyncrc` and khard (`novo-ordo-zero` addressbook) but
missing from the original plan list — added as #7; this list drives Phase 5):

1. `burningserenity@gmail.com`
2. `nfcarlton@gmail.com`
3. `nfcalaway@gmail.com`
4. `nicholas@hawksnestsoftware.com` (Google Workspace)
5. `nicholas@carltonaudio.com` (IMAP at netsol)
6. `nfcalaway@novo-ordo.com` (IMAP at novo-ordo)
7. `zero@novo-ordo.com` (IMAP at novo-ordo — khard addressbook `novo-ordo-zero`)

Existing tooling to integrate with:

- `~/.mbsyncrc` — mbsync/mbsync channels for all identities
- `~/.config/vdirsyncer/config` — pairs syncing Google Calendar/Contacts into local vdirs
  (storages: `~/.calendars/google/`, `~/.calendars/hawksnest/`, `~/.config/khard/contacts/*`;
  `.ics`/`.vcf` vdir format)
- `~/.config/khal/config` — calendars `google`, `google_hawksnest`, `local`
- `~/.config/khard/khard.conf` — 5 addressbooks (burningserenity-gmail, carltonaudio, hawksnest, nfcalaway-gmail, nfcarlton-gmail)
- `~/.calendars/local/` — an existing "local" calendar (natural home to migrate into Omnical)

### 3.5 RustiCal (server software) — verified properties

- Version pinned: **0.16.1** (repo `lennart-k/rustical`, AGPL-3.0-or-later, ~524 stars, active)
- Rust workspace (`crates/*`): axum 0.8, sqlx 0.9 with **bundled SQLite** (single
  static binary; its own Dockerfile cross-builds `aarch64-unknown-linux-musl` with
  clang/lld — same target as our zig approach), `rust-embed` embedded frontend
  (Docker builds with no node stage → frontend dist is expected to be committed;
  verify at clone time), `password-auth` (argon2 for frontend passwords, pbkdf2 for
  app tokens), `web-push` (WebDAV Push for near-instant DAVx5 sync),
  `openidconnect` (OIDC optional at config level), `provided-listeners` (TCP/Unix only)
- HTTP layer: plain HTTP, default port **4000**, `HttpConfig { bind, host, port,
  session_cookie_samesite_strict, payload_limit_mb }` — **no built-in TLS**
  (`HttpBindConfig` enum has only `Tcp(String)` and `Unix(PathBuf)`)
- Config: TOML file or `RUSTICAL_*` env vars (`rustical gen-config` prints defaults);
  db URL via `RUSTICAL_DATA_STORE__SQLITE__DB_URL` (Docker default
  `/var/lib/rustical/db.sqlite3` — **unusable on libreCMC because /var is tmpfs**)
- Endpoints: `/.well-known/caldav`, `/.well-known/carddav`, `/caldav`,
  `/caldav/principal/<user_id>`, `/caldav-compat` (for Apple Calendar, which
  mis-handles multi-home `calendar-home-set`), `/carddav`, frontend at `/`
- Auth: HTTP Basic with **user id + app token** (tokens generated in web UI / frontend
  API). User ids: arbitrary strings, forbidden chars `:` and `$` only → **full email
  addresses work, any domain**. Groups: `rustical principals --principal-type group`
  + `rustical membership`; impersonation syntax `user$group` for clients that can't
  discover group principals
- CLI: `rustical serve`, `rustical principals`, `rustical membership`,
  `rustical gen-config`, `rustical health`
- Scheduling: **RFC 6638 NOT implemented** ("not sure yet whether to implement this")
  → cross-domain invitations are client-side iMIP only
- Sync: RFC 6578 sync-token listed as relevant/partial — clients (DAVx5 etc.) fall
  back to full PROPFIND sync when absent/partial
- Runtime needs: writable `/tmp` (SQLite temp), CA bundle at `/etc/ssl/certs` —
  both already satisfied on the router

---

## 4. Constraints, Decisions, and Rationale

| # | Constraint (fact) | Decision | Rationale |
|---|---|---|---|
| C1 | No python3/php/node in libreCMC v6.8 feeds | Do not deploy Radicale/Baïkal/Nextcloud | Literally uninstallable; static-porting Python is a maintenance dead end |
| C2 | 42 MB free on f2fs overlay; internal flash only (user preference) | RustiCal stripped must fit ≤ ~35 MB + DB; size-gate at build; mitigations: `strip`, `opt-level="z"`, LTO, `panic="abort"`; last-resort `upx --lzma`; USB stick as explicit fallback | Data itself is tiny (KB–MB); the binary is the only large artifact |
| C3 | RustiCal has no TLS; no stunnel/haproxy/nginx installable; router socat lacks OpenSSL | **Build `dav-tls`**: minimal rustls TLS tunnel `0.0.0.0:443 → 127.0.0.1:4003` (raw byte-pipe post-handshake; ALPN `http/1.1` only) | ~150 lines, pure Rust, same proven toolchain; passes PROPFIND/REPORT/PUT and WebSockets (WebDAV Push) untouched; no ALPN h2 → all DAV clients happily use HTTP/1.1 |
| C4 | uhttpd occupies 443 on LAN IPs | Move uhttpd `listen_https` to **8443** (keep :80 with redirect); `dav-tls` binds `0.0.0.0:443` | Wildcard bind conflicts with uhttpd's specific binds otherwise; LuCI remains reachable on https://192.168.10.1:8443 |
| C5 | Router behind upstream TP-Link (192.168.1.1); public IP is dynamic; `0115d8cf.duckdns.org` already tracks it | One-time upstream port-forward TCP 443 → 192.168.1.21:443; ACME via **DuckDNS DNS-01**; verify what updates duckdns (add router cron if nothing does) | No port 80 exposure needed; renewal keeps working through IP changes |
| C6 | `/var` is tmpfs (wiped per boot); sysupgrade wipes non-conffile overlay files | SQLite DB at **`/usr/local/share/rustical/db.sqlite3`**; add `/etc/rustical` + `/usr/local/share/rustical` to `/etc/sysupgrade.conf`; document that `/usr/sbin` binaries need re-deploy after sysupgrade | The single most important data-safety detail of this plan |
| C7 | RustiCal is young software | Pin 0.16.1; run the full verification matrix before cutover; keep Google pairs as source of truth until verified | Risk containment |
| C8 | RFC 6638 absent | Cross-domain invitations via client-side iMIP (Apple/Thunderbird email `.ics` from any SMTP identity to any attendee domain) | Honest limitation; server-side scheduling is a future custom extension (see §17) |
| C9 | Established `router-nym` pattern exists (zig-musl-cc wrappers, procd init.d, deploy.sh, watchdog cron) | Mirror it: `~/router-dav/` project layout with `scripts/`, `out/`, `router/` overlay tree, `deploy.sh` | Consistency and re-use of working, known-good infrastructure |

---

## Phase 0 — Prerequisites

**Owner: user (unless noted). Gate for Phase 1.**

1. **Update the Rust toolchain** (dev machine) — **DONE (2026-09-04): rustc 1.97.1 → 1.98.1,
   `aarch64-unknown-linux-musl` target confirmed installed; rustup self-updated to 1.29.1.**
   ```sh
   rustup update stable        # RustiCal 0.16.1 needs rustc >= 1.98 (have 1.97.1)
   rustup target add aarch64-unknown-linux-musl   # idempotent
   ```
2. **DuckDNS token** for account containing `0115d8cf` subdomain → store in `pass`
   — **DONE (2026-09-04): stored as `secrets/duckdns/0115d8cf.duckdns.org/token`
   (user-inserted); UUID shape checked and validated against the DuckDNS update API
   (`OK` = token valid, subdomain in account).** Needed for ACME DNS-01 in Phase 3.
3. **Upstream router (TP-Link at 192.168.1.1)**: create NAT port-forward
   **TCP 443 → 192.168.1.21:443** (external 443). No port 80 forward needed.
   — **DONE (2026-09-04), AMENDED by user decision: omnical exposed on port 8443,
   not 443.** Investigation findings (all verified same day):
   - The upstream device is a **TP-Link Deco X60** (mesh; WAN holds `65.33.235.245`,
     Spectrum line, confirmed via UPnP `GetExternalIPAddress`). Its local web UI
     (green "su"/luci-stok UI at `192.168.1.1`, module list in
     `/webpages/config/navigator.json`) has **no port-forwarding page at all** —
     forwards are Deco-app-only, and no app/cloud credentials were available.
   - **Port 443 was already taken**: an existing (Deco-app-configured) static forward
     **TCP 443 → 192.168.1.107 (the dev machine)** publicly serves the OpenNIC
     **`.truth` TLD website** (Apache, certbot cert
     `/etc/letsencrypt/live/0115d8cf.duckdns.org/`, valid to 2026-10-07) at
     `https://0115d8cf.duckdns.org/` — i.e. the duckdns hostname was **not actually
     free**. Externally verified live (check-host.net: TCP + HTTP 200 from ~57
     global nodes). Re-pointing 443 to `192.168.1.21` would have taken that public
     site offline → escalated to user per plan rules; user chose **omnical on 8443**.
   - The Deco runs **MiniUPnPd 1.8** (IGD at `192.168.1.1:1900`, `/ctl/IPConn`).
     Quirks found: mapping table starts empty (app rules invisible to it); it runs
     in **secure mode** (AddPortMapping only accepted when `NewInternalClient` ==
     requesting host — so the mapping had to be sent **from the libreCMC router
     itself**, via `ssh router` + curl); mappings to **internal port 443 are silently
     not programmed** (SOAP returns success, readback works, but no kernel DNAT);
     external 443 additionally 718s (already occupied).
   - **Forward created: `TCP 8443 → 192.168.1.21:8443`** (same-port, lease 0 =
     permanent, description `omnical-dav`), added from the router. **Verified
     end-to-end**: external `65.33.235.245:8443` now elicits the libreCMC's firewall
     REJECT (fast TCP RST) from 46/57 global check-host.net nodes (all timed out
     before) — packets demonstrably reach the libreCMC WAN; the RST is expected
     until Phase 2 (dav-tls listening) + Phase 4 (allow rule) land. External 8443
     was verified free beforehand (57 nodes, 0 connects).
    - **Persistence caveat / follow-up for user**: UPnP-mapping survival across a Deco
      reboot is unverified (do not reboot the Deco to find out). Mirror the rule in
      the **Deco app** (Port Forwarding: TCP, external 8443 → `192.168.1.21`:8443,
      device "libreCMC") for a guaranteed-persistent rule, or at minimum re-check
      after the next Deco reboot; if lost, it can be re-added with one command from
      the router (same SOAP `AddPortMapping` via `curl` to `192.168.1.1:1900/ctl/IPConn`).
      — **RESOLVED (2026-09-04): user mirrored the forward in the Deco app (TCP 8443 →
      192.168.1.21:8443, device libreCMC) — a guaranteed-persistent rule now exists in
      addition to the UPnP mapping.**
   - **Required downstream amendments when executing later phases (not yet applied):**
     public URL becomes `https://0115d8cf.duckdns.org:8443`; dav-tls listens on
     **`192.168.1.21:8443`** (WAN IP — NOT `0.0.0.0`, which would collide with
     uhttpd's planned LAN `:8443` binds; with dav-tls off 443 entirely, the
     uhttpd→8443 move in 2.5/C4 becomes **unnecessary — keep uhttpd on 443 LAN**);
     Phase 4.1 firewall rule uses `dest_port='8443'`; Phase 4.3 matrix, Phase 3
     verify commands, and Phase 5–7 URL examples change 443→8443 accordingly.
4. **DuckDNS updater ownership**: confirm what currently updates the A record for
   `0115d8cf.duckdns.org` (TP-Link feature? dev-machine cron?). If nothing does,
   add a cron on the router in Phase 2 (curl to the DuckDNS update API every 5 min).
   — **INVESTIGATED (2026-09-04): no updater found — dev-machine crontab/systemd
   clean, libreCMC router cron clean, user unsure; TP-Link built-in DDNS not
   verifiable without upstream admin access. Resolution: add the router cron in
   Phase 2.7 (DuckDNS updates are idempotent, so an extra updater is harmless).**
5. **App-token storage convention**: RustiCal app tokens (per client) will be
   generated in Phase 5; store them in `pass` under `secrets/omnical/…`.
6. **Confirm the identities list** (the 6 from §3.4, or a revised list) — drives
    Phase 5. — **DONE (2026-09-04): user confirmed a revised list of 7 — the §3.4
    six plus `zero@novo-ordo.com` (present in `~/.mbsyncrc`/khard but missing from
    the original plan list). §3.4 and the §1 summary updated accordingly.**

---

## Phase 1 — Build (dev machine)

**Deliverable: two static `aarch64-unknown-linux-musl` binaries in `~/router-dav/out/`
plus a local x86_64 smoke-test pass.**

> **STATUS (2026-09-04, updated):** 1.1 skeleton **DONE** · 1.2 clone+verify
> **DONE** · **1.3 DONE — rustical cross-compiled: static aarch64 binary, 26 MiB
> stripped (within 35 MiB budget), runs under qemu-aarch64 (`--version`,
> `gen-config` OK); toolchain fix recorded under 1.3** · **1.4 DONE — dav-tls
> built: static aarch64 binary 1.22 MiB stripped (`out/dav-tls`, runs under
> qemu-aarch64) + x86_64 host build 1.4 MiB
> (`out/x86_64-unknown-linux-gnu/dav-tls`); both via `scripts/build-rust.sh`
> (clang recipe D)** · **1.5 DONE — local smoke test passed: all gates green
> (see 1.5 for results and findings)**.
> **Nothing has been deployed to the router.**

### 1.1 Project skeleton (mirror router-nym) — **DONE (2026-09-04)**

Created exactly as specified below, plus `README.md` (sysupgrade runbook), and the
zig wrappers copied into `build/bin/` (self-contained; one local patch — see
Toolchain findings). `build-deps.sh` confirmed unnecessary. `deploy.sh` written but
NOT yet run (Phase 2). Files: `scripts/build-rust.sh`, `dav-tls/` crate,
`router/etc/rustical/config.toml`, `router/etc/init.d/{rustical,dav-tls}`,
`router/etc/sysupgrade.conf.additions`, `out/tls/`.


```
~/router-dav/
├── scripts/
│   ├── build-rust.sh      # adapted from router-nym/scripts/build-rust.sh
│   └── build-deps.sh      # NOT needed (sqlx bundles SQLite; rustls is pure Rust)
├── dav-tls/               # the custom rustls tunnel crate (new code)
│   ├── Cargo.toml
│   └── src/main.rs
├── rustical/              # cloned upstream, pinned to v0.16.1 (git submodule or vendored)
├── router/                # overlay tree deployed to the router
│   ├── etc/rustical/config.toml
│   ├── etc/init.d/rustical
│   ├── etc/init.d/dav-tls
│   └── etc/sysupgrade.conf.additions
├── out/                   # cross-compiled artifacts
└── deploy.sh              # adapted from router-nym/deploy.sh
```

### 1.2 Clone & verify RustiCal — **DONE (2026-09-04)**

Cloned to `~/router-dav/rustical`, pinned `v0.16.1` (git-clean, no submodules).
**Frontend assets ARE committed** — the rust-embed folder is
`crates/frontend/public/assets/` (`js/bundle.mjs`, `style.css`, `licenses.html`),
so no JS toolchain build is needed. Upstream's build tool is **deno** (not bun —
`crates/frontend/js-components/deno.json`, `deno task build`), irrelevant since
assets ship built-in. `.sqlx/` is committed → `SQLX_OFFLINE=true` works.

```sh
git clone https://github.com/lennart-k/rustical.git ~/router-dav/rustical
git -C ~/router-dav/rustical checkout v0.16.1   # pin the release
```

- Verify `crates/frontend/dist/` (or the `rust-embed` asset folder) contains a
  pre-built frontend. The upstream Dockerfile builds with no node stage, so it should
  be committed; **if not**: `cd crates/frontend && bun install && bun run build`
  before the cargo build.
- `SQLX_OFFLINE=true` is required for the build (`.sqlx/` is committed upstream).

### 1.3 Cross-compile RustiCal — **DONE (2026-09-04): `out/rustical`, static aarch64, 26 MiB stripped**

Reuse the zig-based toolchain wrappers from `router-nym/build/bin/` (`zig-musl-cc` and
its ar/ranlib/nm symlinks) exactly as `router-nym/scripts/build-rust.sh` does:

```sh
export CC_aarch64_unknown_linux_musl="$BIN/zig-musl-cc"
export AR_aarch64_unknown_linux_musl="$BIN/zig-musl-ar"
export RANLIB_aarch64_unknown_linux_musl="$BIN/zig-musl-ranlib"
export CARGO_TARGET_AARCH64_UNKNOWN_LINUX_MUSL_LINKER="$BIN/zig-musl-cc"
export SQLX_OFFLINE=true
cargo build --release --target aarch64-unknown-linux-musl --locked
```

Notes:
- ~~Only C dependency is sqlx's bundled SQLite~~ — **WRONG (found 2026-09-04):**
  the default build pulls **three** C dependencies — bundled SQLite (`sqlx-sqlite`),
  **vendored OpenSSL** (dav_push → `ece` + `web-push`, *not* feature-gated), and
  **aws-lc-sys** (rustls' default crypto provider, via `reqwest`/`openidconnect`/
  `web-push`). Host tools available: clang 22 (`/usr/lib/llvm/22/bin`), llvm-ar,
  cmake 4.3.4, perl, make, nasm, libclang, rust-lld (in the rustup toolchain).
- Disable nothing (RustiCal has no heavyweight default features; `debug`,
  `frontend-dev`, `opentelemetry` are opt-in).
- **Size-gate:** `strip` the binary immediately. Budget: `rustical` ≤ 35 MB stripped.
  If over: add `[profile.release] opt-level = "z", lto = true, panic = "abort",
  codegen-units = 1`, rebuild, re-strip; last resort `upx --lzma` (static musl ELF);
  if still over → surface to user with the USB-stick fallback option (deferred by
  decision, not off the table).

**Toolchain findings (2026-09-04) — RESOLVED: recipe D (clang + zig musl headers) works.**

Four cross-toolchain attempts (rustc 1.98.1, `SQLX_OFFLINE=true`,
`CARGO_PROFILE_RELEASE_STRIP=true`, aarch64-unknown-linux-musl):

| # | CC (C deps) | Linker | Result |
|---|---|---|---|
| A | zig 0.16 (`build/bin/zig-musl-cc`, router-nym pattern) | zig's own ELF linker | **All C deps compile under zig cc** (vendored OpenSSL, aws-lc-sys, SQLite, ring) ✓ — but the final link fails twice: rustc now emits `-Wl,--fix-cortex-a53-843419` which zig rejects (patched our wrapper copy to strip it — router-nym binaries already run on this RK3328 without the errata workaround), then `duplicate symbol: _start` (zig's musl crt1.o collides with rust's self-contained crt1.o). |
| B | clang 22 (upstream-Dockerfile recipe) | rust-lld + `-Clink-self-contained=yes` | aws-lc-sys (jitterentropy) fails: clang has **no musl sysroot** on this host → falls back to host glibc headers (`/system/index/include`): `__float128 is not supported on this target`. |
| C | zig (for its bundled musl headers) | rust-lld + rust's self-contained musl (upstream's link recipe) | C deps compile ✓; final link fails with **empty-name undefined symbols** referenced from zig-compiled `sqlite3.o` inside `liblibsqlite3_sys` (many sites: `sqlite3CreateIndex`, `sqlite3Select`, …). **Diagnosis CONFIRMED 2026-09-04 via `readelf -sW sqlite3.o`: 17 empty-name undefined SECTION LOCAL symbols** — a zig-cc object-emission quirk strict lld rejects. Zig-CC is unusable for lld-linked builds. |
| **D (WINNER — now the default `clang` route in `scripts/build-rust.sh`)** | **clang 22 + zig's musl include dirs** (`zig libc -target aarch64-linux-musl -includes`) | rust-lld + rust's self-contained musl | **WORKS.** Recipe: `CC_aarch64_unknown_linux_musl=clang`, `AR/RANLIB=llvm-ar/llvm-ranlib`, `CFLAGS_aarch64_unknown_linux_musl="-nostdinc -isystem <clang -print-resource-dir>/include -isystem <each zig musl include dir>"` (the resource dir is required — `-nostdinc` drops clang's builtin headers, breaking `arm_neon.h` in aws-lc), `CARGO_TARGET_AARCH64_UNKNOWN_LINUX_MUSL_RUSTFLAGS="-Clink-self-contained=yes -Clinker=rust-lld"`. Validated first on a standalone bundled-rusqlite crate (0 empty-name symbols vs zig's 17), then full rustical build. |

Outcome of the D build (2026-09-04):
- `~/router-dav/out/rustical` — ELF 64-bit aarch64, **statically linked (no dynamic
  section), stripped, 27,839,344 bytes = 26 MiB** — within the 35 MiB budget, no
  size-mitigation steps (opt-level=z/LTO/upx) needed.
- Runtime check under `qemu-aarch64`: `rustical --version` → `rustical 0.16.1` ✓,
  `rustical gen-config` prints the default TOML ✓ (full smoke test is 1.5).
- Build time ~3.5 min from a clean aarch64 target dir.
- Note for 1.4: `dav-tls` should use the same recipe (`TOOLCHAIN=clang` is now the
  `build-rust.sh` default).

### 1.4 Build `dav-tls` (the one custom component) — **DONE (2026-09-04)**

Source complete in `~/router-dav/dav-tls/` and `cargo check` passes. Implementation
deviation from the spec above: **tokio + tokio-rustls (ring provider)** instead of
std::net + threads — the two-thread-with-Mutex design would block the server→client
direction during a blocking client read, stalling long-lived WebDAV-Push WebSockets;
tokio's split halves avoid that. aws-lc-rs deliberately not used in this crate.
Features implemented: repeatable `--listen`, `--upstream` (default wiring per §2.4:
`0.0.0.0:443` → `127.0.0.1:4000`), ALPN `http/1.1` only, SO_REUSEADDR (socket2),
`--user` privilege drop via libc getpwnam, graceful SIGTERM/SIGINT drain (≤5s,
1024-conn cap), TCP_NODELAY, TLS close_notify propagation. Own release profile
(opt-level="z", lto, strip). **Builds completed 2026-09-04** via the `clang`
default route of `scripts/build-rust.sh` (ring is the only C dep and compiles
under recipe D):

- `~/router-dav/out/dav-tls` — ELF 64-bit aarch64, **statically linked (no
  dynamic section), stripped, 1,279,960 bytes = 1.22 MiB** (well under the
  ~2–4 MB estimate; combined with rustical = ~27 MiB of the 42 MB overlay).
  Runtime check under `qemu-aarch64`: `--help` OK (full smoke test is 1.5).
- `~/router-dav/out/x86_64-unknown-linux-gnu/dav-tls` — host build for the
  1.5 smoke test, 1.4 MiB stripped, runs natively.
- Build times: ~42 s (aarch64, clean), ~49 s (x86_64, clean); x86_64 rustical
  rebuild from clean also completed (~6 min) as a 1.5 prerequisite.

Spec (as originally designed; implemented above with the tokio deviation):

- Args: `--listen 0.0.0.0:443` (repeatable), `--upstream 127.0.0.1:4000`,
  `--cert /etc/rustical/tls/fullchain.pem`, `--key /etc/rustical/tls/key.pem`,
  optional `--user nobody` (drop privileges after bind).
- Implementation: `rustls` 0.23 (pure Rust, no CC needed) + `tokio` (or plain
  `std::net` + threads — simpler is better here): accept TCP, perform TLS handshake
  with ALPN restricted to `http/1.1`, then **splice raw bytes** bidirectionally to
  the upstream socket. No HTTP parsing. `SO_REUSEADDR`, per-connection handling,
  graceful SIGTERM shutdown (for procd restart during cert deploys).
- Cert reload happens on service restart (renewal flow restarts the service).
- HTTP/2 note: no `h2` in ALPN → every CalDAV client falls back to HTTP/1.1. WebDAV
  Push WebSockets pass through untouched (they're just HTTP upgrades).
- Also buildable for `x86_64-unknown-linux-gnu` for local testing.
- Expected size: ~2–4 MB stripped.

### 1.5 Local smoke test (x86_64, dev machine) — **DONE (2026-09-04, all gates green)**

Executed in `/tmp/opencode/omnical-smoke/` with `out/x86_64-unknown-linux-gnu/{rustical,dav-tls}`
(rustical on `:4000`, dav-tls on `127.0.0.1:4443` with a throwaway self-signed cert; principal
`smoketest` + app token created via `rustical principals create --password` /
`rustical principals app-token create`).

**Gate results (all via dav-tls, i.e. TLS-terminated):**

| Gate | Result |
|---|---|
| `/.well-known/caldav` / `/.well-known/carddav` | 308 redirect → `/caldav` / `/carddav` ✓ |
| `OPTIONS` on calendar collection | `dav: 1, 3, access-control, calendar-access, webdav-push` ✓ |
| `OPTIONS` on addressbook collection | `dav: 1, 3, access-control, addressbook, webdav-push` ✓ |
| `PROPFIND` (Depth 0/1) | 207 multistatus ✓ |
| `PUT` test `.ics` → 201; `REPORT` calendar-query (time-range) → 207 with etag; `GET` returns byte-identical content | ✓ |
| `rustical health` | exit 0 (needs config/db env set — silent on success) ✓ |

**Findings relevant to later phases:**

- **Home-set shape:** `calendar-home-set` = `/caldav/principal/<user_id>/` itself; addressbook
  home = `/carddav/principal/<user_id>/`. (Addressbook-home-set prop 404s on the *calendar*
  principal URL — query the carddav one instead.)
- **Default collections are NOT auto-created** for a fresh principal (at least not until first
  frontend login). Collections are created fine via standard `MKCOL` with
  `resourcetype {collection, calendar}` / `{collection, addressbook}` (201 both) —
  relevant for Phase 5.4; also `MKCALENDAR` is advertised on calendars.
- **Frontend root `/` allows GET/HEAD only** (PROPFIND → 405) — DAV discovery must go via
  `/.well-known/*`, as all clients do.
- dav-tls passed PROPFIND/REPORT/PUT/GET and TLS handshakes untouched; ALPN `http/1.1`
  negotiated (curl `--http1.1` implicit via no-h2 offering).

```sh
cargo build --release --target x86_64-unknown-linux-gnu   # both crates
# run rustical with a temp db:
RUSTICAL_DATA_STORE__SQLITE__DB_URL=file:/tmp/omnical-test.db.sqlite3 \
  ./rustical serve &
# self-signed cert via openssl for dav-tls, then:
curl -vk https://127.0.0.1/.well-known/caldav -u 'smoketest:token'
curl -vk -X PROPFIND -H 'Depth: 1' https://127.0.0.1/caldav/ -u 'smoketest:token'
curl -vk -X OPTIONS https://127.0.0.1/caldav/
```

**Gate:** `OPTIONS` advertises DAV: 1, calendar-access, addressbook; `PROPFIND` returns
multistatus; `/.well-known/*` redirect correctly through dav-tls; PUT+REPORT of a test
`.ics` round-trips. Then proceed.

---

## Phase 2 — Install on Router

**Deliverable: both services installed, enabled, and running on the router
(HTTP-only, LAN-addressable; TLS wired in Phase 3.**

> **STATUS (2026-09-04): DONE — except the reboot gate, deferred by user decision.**
> Executed via `~/router-dav/deploy.sh` against a virgin router (no prior install).
> rustical 0.16.1 running + enabled at boot, healthy, listening on **127.0.0.1:4000
> only**; DB created at `/usr/local/share/rustical/db.sqlite3` (f2fs overlay —
> persistent, not /var); sysupgrade.conf entries added; `sqlite3-cli` installed;
> dav-tls pushed, proven executable on the real aarch64 CPU (`--help` exit 0), and
> **deliberately not enabled** (no certs until Phase 3); uhttpd untouched on LAN
> :443. DuckDNS updater cron added and verified (API `OK`; A record → 65.33.235.245
> via router and external resolver). Disk after deploy: 12.3 M free on the overlay
> (87 % used — the ≥ 5 M floor from 8.4 still holds).
>
> **Pre-deploy bugs found & fixed during execution review (all in `~/router-dav/`):**
> 1. `router/etc/rustical/config.toml` originally had `http.host =
>    "0115d8cf.duckdns.org"` — in RustiCal 0.16.1 `http.host` is a **deprecated bind
>    override that wins over `bind`**, so rustical would have tried to bind
>    `0115d8cf.duckdns.org:4000` (resolves to the public IP → bind failure). Key
>    removed; response URLs are generated from each request's Host header anyway
>    (clients send `0115d8cf.duckdns.org:8443` through dav-tls).
> 2. `--config-file` is a top-level clap option, **not** accepted after the
>    subcommand (`rustical serve --config-file …` → "unexpected argument"). Fixed
>    the order in `router/etc/init.d/rustical` and `deploy.sh` (`rustical
>    --config-file X serve` / `… health`).
> 3. `deploy.sh` scp'd into `/etc/rustical/` before creating it (virgin router had
>    none — scp would have failed). mkdir now precedes the config push.
> 4. `router/etc/init.d/dav-tls` amended per Phase 0.3: LISTEN `192.168.1.21:8443`
>    (WAN IP, not `0.0.0.0:443`), plus a pre-start check that refuses to start (with
>    a loud syslog message) if eth0's DHCP address ever drifts from 192.168.1.21.
>
> **2.7 finding:** the router's egress prefers IPv6 (ifconfig.me → `2603:9001:…`),
> which DuckDNS cannot use to set the A record — the cron line pins **`curl -4`**
> (verified: router v4 egress = 65.33.235.245 = current A record). Caveat: busybox
> crond logs each command line (incl. the token) to the router's root-only syslog.
>
> **Reboot gate (2.8):** deferred 2026-09-04 — verify services + DB after the next
> natural reboot, or fold into Phase 3 verification. **Reminder for Phase 3/4:**
> all verify commands and the firewall rule use port **8443** (e.g.
> `openssl s_client -connect 0115d8cf.duckdns.org:8443 …`,
> `dest_port='8443'`).

### 2.1 Files to deploy (via `deploy.sh`, modeled on router-nym) — **DONE (2026-09-04)**

| Artifact (dev) | Router target | Mode |
|---|---|---|
| `out/rustical` | `/usr/sbin/rustical` | 0755 |
| `out/dav-tls` | `/usr/sbin/dav-tls` | 0755 |
| `router/etc/rustical/config.toml` | `/etc/rustical/config.toml` | 0600 |
| `router/etc/init.d/rustical` | `/etc/init.d/rustical` | 0755 |
| `router/etc/init.d/dav-tls` | `/etc/init.d/dav-tls` | 0755 |

### 2.2 Router-side preparation (idempotent, part of deploy.sh) — **DONE**

```sh
# persistent data dir (overlay — NOT /var!)
mkdir -p /usr/local/share/rustical /etc/rustical/tls
# online-backup tool for the SQLite DB
opkg update && opkg install sqlite3-cli
```

### 2.3 `/etc/rustical/config.toml` (authoritative shape comes from
`rustical gen-config`; the fields we set:) — **DONE (host key removed — see STATUS bug 1)**

```toml
[http]
bind = "127.0.0.1:4000"          # only dav-tls talks to it
host = "0115d8cf.duckdns.org"     # used for URL generation in frontend/DAV responses
payload_limit_mb = 32

[data_store.sqlite]
db_url = "file:/usr/local/share/rustical/db.sqlite3"

[dav_push]
# leave defaults at first; enable WebSocket/WebPush transport after Phase 7 tests

[nextcloud_login]
# enabled by default — required for DAVx5 login flow
```

(Equivalent env vars if preferred: `RUSTICAL_HTTP__BIND=127.0.0.1:4000`,
`RUSTICAL_DATA_STORE__SQLITE__DB_URL=file:…`.)

### 2.4 procd init scripts — **DONE (dav-tls amended to 192.168.1.21:8443 per Phase 0.3)**

`/etc/init.d/rustical`:
- `START=95`, `STOP=10`, `USE_PROCD=1`
- `procd_set_param command /usr/sbin/rustical serve`
- `procd_set_param respawn` (threshold 3600, timeout 5, retry 0)
- `procd_set_param limits memory="256 MB"` (hard cap well under free RAM)
- `procd_set_param stdout 1` / `stderr 1` → syslog (logread)
- optional hardening later: `procd_set_param jail` (ujail) after functional verification

`/etc/init.d/dav-tls`:
- `START=96` (after rustical), similar respawn/limits
- `procd_set_param command /usr/sbin/dav-tls --listen 0.0.0.0:443 --upstream 127.0.0.1:4000 --cert /etc/rustical/tls/fullchain.pem --key /etc/rustical/tls/key.pem`
- only enabled/started in Phase 3 when certs exist (validate: `ls /etc/rustical/tls/` in
  `start_service`, refuse to start with log message if missing)

### 2.5 Free port 443 (move uhttpd HTTPS to 8443) — **SKIPPED per Phase 0.3 amendment (uhttpd stays on LAN :443; verified untouched)**

```sh
uci set uhttpd.main.listen_https='192.168.10.1:8443'
uci add_list uhttpd.main.listen_https='10.10.10.1:8443'
uci add_list uhttpd.main.listen_https='[fd1e:5c71:1ee8::1]:8443'
# remove the old :443 entries (uci delete_list), then:
uci commit uhttpd && /etc/init.d/uhttpd restart
```

LuCI remains at `https://192.168.10.1:8443` (the `:80` listener still redirects).
Document this change — it's the only visible LuCI-side modification of the whole plan.

### 2.6 Persistence across sysupgrade — **DONE**

Append to `/etc/sysupgrade.conf`:

```
/etc/rustical
/usr/local/share/rustical
```

(`/etc/init.d/*` files are preserved by default as conffiles; `/usr/sbin/rustical` and
`/usr/sbin/dav-tls` are **not** — after any sysupgrade, re-run
`~/router-dav/deploy.sh`. Document in the project README.)

### 2.7 Optional: duckdns updater cron (only if Phase 0 step 4 found nothing) — **DONE (curl -4 pinned; see STATUS finding)**

```cron
*/5 * * * * curl -s "https://www.duckdns.org/update?domains=0115d8cf&token=<TOKEN>&ip=" >/dev/null
```

(Token inserted at deploy time, chmod 600 crontab — never logged.)

### 2.8 Verify — **DONE except reboot gate (deferred by user, 2026-09-04)**

```sh
/etc/init.d/rustical enable && /etc/init.d/rustical start
/usr/sbin/rustical health
netstat -tlnp | grep 4000          # bound to 127.0.0.1:4000 only
curl -s http://127.0.0.1:4000/.well-known/caldav -o /dev/null -w '%{http_code}\n'
```

**Gate:** service healthy, DB created at
`/usr/local/share/rustical/db.sqlite3`, survives a reboot (`reboot`, re-check).

---

## Phase 3 — TLS Certificate Automation

**Deliverable: valid Let's Encrypt cert for `0115d8cf.duckdns.org` live on
`0.0.0.0:443`, with unattended renewal.**

> **STATUS (2026-09-04): DONE** — except the external end-to-end checks, which are
> gated on the Phase 4 firewall rule by design.
> - **Certbot-cert decision (§3.3):** issued **alongside** the existing certbot cert
>   ("redundant but harmless") — `/etc/letsencrypt` and the `.truth` site's lifecycle
>   untouched.
> - acme.sh installed on the dev machine (account email `burningserenity@gmail.com`,
>   the primary identity — changeable via `acme.sh --update-account -m`); default CA
>   pinned to Let's Encrypt via `--set-default-ca` (acme.sh otherwise defaults to
>   ZeroSSL).
> - Cert issued via DuckDNS DNS-01 in ~30 s: CN/SAN `0115d8cf.duckdns.org`, issuer
>   Let's Encrypt, ECDSA, valid **2026-09-04 → 2026-12-03**; next renewal auto-picked
>   from the ARI window (2026-11-04).
> - Deploy hook (saved in the domain conf → runs on every renewal): `scp -p` both
>   files → `router:/etc/rustical/tls/`, `chmod 600`, `/etc/init.d/dav-tls restart`.
>   Written **without the plan's brace-expansion idiom** — acme.sh runs reloadcmd via
>   POSIX `sh`, where `{a,b}` does not expand. DuckDNS token saved by the plugin to
>   `~/.acme.sh/account.conf`; acme.sh cron (4×/day) installed → renewals fully
>   unattended.
> - Per the Phase 0.3 amendment everything is on **:8443**: dav-tls `enable`d
>   (boot-persistent) and running, listening `192.168.1.21:8443 → 127.0.0.1:4000`,
>   privileges dropped to `nobody` after cert load (certs are read pre-drop, so 0600
>   root-owned files are correct).
> - **Verified from the router** (`curl --resolve …:8443:192.168.1.21`, full-chain
>   validation against the router's ca-bundle, `ssl_verify_result=0`, no `-k`):
>   `/.well-known/caldav` → **308** through dav-tls; `/caldav/` → 405 (plain GET on a
>   DAV root — expected); `/` → 303 (frontend login redirect). Benign first-start
>   artifact: procd `restart` on a never-started service logs `ubus … Not found` once.
> - **External probe** (check-host.net, 57 nodes): 0 connected; RST pattern intact
>   (38 refused / 17 timed out / 2 other) — matches the Phase 0 baseline: the Deco
>   forward is still alive and the libreCMC still REJECTs wan:8443 until Phase 4.1.
> - Key hygiene: 0600 on both dev (`out/tls/key.pem`) and router; `~/router-dav` is
>   not a git repo, so nothing to gitignore.
> - ~~**Deferred:** external `curl -sI https://0115d8cf.duckdns.org:8443/…` (Phase 4)~~
>   **DONE 2026-09-04 with Phase 4** (check-host.net HTTPS probe = the external
>   curl-equivalent; see Phase 4 STATUS). Still deferred: 2.8 reboot gate (next
>   natural reboot — it now also proves dav-tls starts on boot).

All on the **dev machine** (full curl/openssl tooling; router stays thin):

```sh
# 1. Install acme.sh (once)
curl https://get.acme.sh | sh -s email=<your-email>

# 2. Issue via DuckDNS DNS-01 (TXT record automation; no port 80 needed)
export DuckDNS_Token="$(pass show secrets/duckdns/0115d8cf.duckdns.org/token)"
~/.acme.sh/acme.sh --issue --dns dns_duckdns -d 0115d8cf.duckdns.org

# 3. Install-cert with a deploy hook that pushes to the router and reloads dav-tls
~/.acme.sh/acme.sh --install-cert -d 0115d8cf.duckdns.org \
  --fullchain-file ~/router-dav/out/tls/fullchain.pem \
  --key-file       ~/router-dav/out/tls/key.pem \
  --reloadcmd "scp ~/router-dav/out/tls/{fullchain.pem,key.pem} router:/etc/rustical/tls/ && \
               ssh router 'chmod 600 /etc/rustical/tls/*; /etc/init.d/dav-tls restart'"
```

Then on the router: `/etc/init.d/dav-tls enable && /etc/init.d/dav-tls start`.

Renewal is automatic (~60-day cycle, acme.sh cron). Key hygiene: key exists only on
dev machine + router, mode 0600, never in the git repo.

**Verify:**
```sh
# (ports amended 443→8443 per Phase 0.3; executed externally via check-host.net with Phase 4)
openssl s_client -connect 0115d8cf.duckdns.org:8443 -servername 0115d8cf.duckdns.org </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates -issuer     # Let's Encrypt, valid dates
curl -sI https://0115d8cf.duckdns.org:8443/.well-known/caldav   # from an EXTERNAL network
```

---

## Phase 4 — Firewall & Reachability

**Deliverable (amended per Phase 0.3): 8443 reachable from WAN through both NATs,
blocked otherwise.**

> **STATUS (2026-09-04): DONE — all gates green.**
> - **Pre-flight checks (no drift):** no pre-existing `Allow-Dav-TLS` rule;
>   dav-tls LISTEN on `192.168.1.21:8443`, rustical on `127.0.0.1:4000` only, uhttpd
>   untouched on LAN :443; A record for `0115d8cf.duckdns.org` == current public IP
>   (65.33.235.245, from dev machine and via 1.1.1.1); both services `running`.
> - **4.1 DONE** — rule added per the amendment (follows the existing
>   Allow-DHCP-Renew pattern): `src='wan', proto='tcp', dest_port='8443',
>   target='ACCEPT', family='any'` → `firewall.@rule[20]` (`cfg4492bd`), committed,
>   `/etc/init.d/firewall reload` (pre-existing benign screech_dnat reflection
>   warnings unchanged). Live in nftables as
>   `tcp dport 8443 counter … accept comment "!fw4: Allow-Dav-TLS"` with the packet
>   counter climbing — traffic demonstrably flows through the rule.
> - **4.2** — nothing to do (Phase 0): upstream forward exists twice over (Deco-app
>   mirror + UPnP), TCP 8443 → 192.168.1.21:8443.
> - **4.3 verified** (matrix below amended to the 8443 / WAN-IP-listener reality and
>   marked with results): upstream-LAN path → **308** with full LE chain validation
>   (`ssl_verify_result=0`, no `-k`); external (check-host.net, 57 global nodes):
>   **TCP 45/57 connected, HTTPS 45/57 → `308 Permanent Redirect`** on
>   `/.well-known/caldav` (remaining 12 = far-node timeouts, normal for a residential
>   line; baseline before the rule was 0 connected / 38 refused — the RST→308 flip
>   proves the firewall rule did it); port 4000 refused from outside; port 8444
>   rejected → only 8443 opened; cert seen: `CN=0115d8cf.duckdns.org`, issuer
>   Let's Encrypt (YE2), valid 2026-09-04 → 2026-12-03.
> - **Bonus finding: NAT hairpin works** — the public URL answers from *inside* the
>   network too (dev machine → `65.33.235.245:8443` → 308), so LAN clients can use
>   `https://0115d8cf.duckdns.org:8443` directly; 192.168.10.x clients can also hit
>   `192.168.1.21:8443` (lan-zone input policy is ACCEPT, verified in uci).
> - Still open from earlier phases: **2.8 reboot gate** (next natural reboot).

### 4.1 libreCMC firewall rule (follow the Allow-DNSv4 rule pattern) — **DONE (2026-09-04, dest_port 8443 per Phase 0.3)**

```sh
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Dav-TLS'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].dest_port='8443'
uci set firewall.@rule[-1].target='ACCEPT'
uci set firewall.@rule[-1].family='any'
uci commit firewall && /etc/init.d/firewall reload
```

(Also verify the IPv6 story: if a global GUA ever lands on the WAN interface,
the same rule covers it via `family='any'`; the ULA is LAN-internal only.)

### 4.2 Upstream port-forward

Done once by the user in Phase 0 (TCP 443 → 192.168.1.21:443). If it cannot be
configured, the fallback is a TCP relay on one of the existing wireguard VMs
(`wg-vm1`/`wg-vm3`) — noted here, not planned, since the port-forward path is
available.

### 4.3 Reachability matrix — **VERIFIED (2026-09-04, rows amended per Phase 0.3: dav-tls listens on the WAN IP :8443, not 0.0.0.0:443)**

| From | To | Verified |
|---|---|---|
| Upstream LAN (192.168.1.x, incl. dev machine) | https://192.168.1.21:8443 | ✓ 308 on `/.well-known/caldav`, LE chain validates (`ssl_verify_result=0`, no `-k`) |
| LAN (192.168.10.x) | https://192.168.1.21:8443 or public URL | ✓ lan-zone input = ACCEPT (uci-verified); hairpin through the Deco verified from inside — no client-side distinction |
| Internet (check-host.net, 57 nodes) | https://0115d8cf.duckdns.org:8443 | ✓ 45/57 TCP connect; 45/57 TLS handshake + `308 Permanent Redirect` (12 far-node timeouts; was 0 connected / 38 refused before the rule) |
| Any | port 4000 | ✓ refused (bound to 127.0.0.1 only) |
| Any | other WAN ports (e.g. 8444) | ✓ rejected — only 8443 was opened |

---

## Phase 5 — Users, Groups, Collections (any email domain)

**Deliverable: 7 principals + sharing groups + per-client app tokens + seeded
collections. This is where "works across any email domain" materializes.**

> **STATUS (2026-09-04): DONE — all gates green.**
> - **Pre-change safety net:** hot SQLite `.backup` pulled to
>   `~/backups/omnical/db-pre-phase5-20260904.sqlite3` (148 KB) before any change
>   (Phase 8's procedure used ad-hoc, since nightly backups don't exist yet).
> - **5.1 DONE — 7 principals** created (user ids = full email addresses,
>   displayname = the email). `principals create --password` accepts piped stdin
>   (no TTY needed; prompts once). Frontend passwords: 32-char random per
>   identity, stored in `pass` at `secrets/omnical/<identity>/frontend`.
>   Actual CLI: `create`/`remove`, not the plan's guessed `add`.
> - **5.2 DONE — group `family`** (principal-type group, displayname "Family",
>   no password — group principals are reached via member auth or impersonation
>   `<user>$family`) with **all 7 identities assigned** via
>   `principals membership assign <id> --to family` (subcommand is `assign`,
>   not `add`; `membership list <id>` shows accessible principals = self + groups).
> - **5.3 DONE — 28 app tokens** (4 per identity: `vdirsyncer`, `davx5`,
>   `thunderbird`, `apple`) via `principals app-token create --name <client>
>   <id>` (prints the bare token on stdout); all stored in `pass` at
>   `secrets/omnical/<identity>/<client>` — 7 frontend passwords + 28 tokens =
>   35 pass entries. DAV Basic auth verified with the vdirsyncer tokens.
> - **5.4 DONE — 24 collections seeded** via MKCOL through dav-tls
>   (`curl --resolve 0115d8cf.duckdns.org:8443:192.168.1.21`, LE chain validates,
>   no `-k`): per identity `personal` (calendar, component-set VEVENT+VJOURNAL),
>   `tasks` (calendar, VTODO-only component-set), `personal` (addressbook); for
>   the group: `family` (calendar "Family"), `tasks` ("Family Tasks"), `family`
>   (addressbook "Family Contacts"). **Group seeding needed no impersonation —
>   plain member auth has MKCOL rights on the group home.** MKCOL on an existing
>   collection returns 405 (matters for re-runs). Confirmed: the
>   `_birthdays_<addressbook>` auto-collection appears per principal right after
>   the addressbook exists.
> - **Verified:** `principals list` = 8 (7 INDIVIDUAL + 1 GROUP); `app-token
>   list` shows 4 tokens per identity; PROPFIND Depth 1 on every caldav + carddav
>   home = 207 with exactly the expected hrefs; all 7 memberships in `family`;
>   router overlay 12.6 M free (87 % used — ≥ 5 M floor intact).
> - **Notes for later phases:** verification-matrix row 15 now has real group
>   collections to test against; Phase 6 vdirsyncer pairs should target hrefs
>   `/caldav/principal/<id>/{personal,tasks}/` and
>   `/carddav/principal/<id>/personal/`; RustiCal PROPFIND responses use the
>   *default* (unprefixed) DAV namespace — parse `<href>`, not `<D:href>`.

### 5.1 Create principals (user id = full email address)

RustiCal user ids are domain-agnostic (only `:` and `$` are forbidden), so accounts
are literally the email addresses:

```sh
ssh router '/usr/sbin/rustical principals --help'   # confirm CLI shape on the pinned version
for u in \
  burningserenity@gmail.com \
  nfcarlton@gmail.com \
  nfcalaway@gmail.com \
  nicholas@hawksnestsoftware.com \
  nicholas@carltonaudio.com \
  nfcalaway@novo-ordo.com ; do
    ssh router "/usr/sbin/rustical principals add '$u'"   # set an argon2 frontend password when prompted
done
```

(Exact subcommand syntax per the pinned release's `principals --help`; the docs show
`rustical principals` as the interactive entry point. Any future email address on any
domain is added the same way — nothing is domain-coupled.)

### 5.2 Groups & memberships (cross-identity / cross-domain sharing)

```sh
ssh router "/usr/sbin/rustical principals add group.family --principal-type group"
ssh router "/usr/sbin/rustical membership add <owner> group.family <member>"
```

Notes from upstream docs:
- Membership = full read/write access to the group principal's collections.
- Clients that can't auto-discover member principals authenticate as
  `user$group` (principal impersonation) or add
  `/caldav-compat/principal/group.family` manually (Apple).

### 5.3 App tokens (DAV credentials)

- Log into the frontend (`https://0115d8cf.duckdns.org`) with each identity's
  frontend password; generate **one app token per client** (e.g.
  `vdirsyncer`, `davx5-nicholas`, `tb-carltonaudio`, `apple-nfcalaway`).
- Store tokens in `pass` (`secrets/omnical/<identity>/<client>`).
- Frontend passwords (argon2) are only for the web UI; DAV never sees them.

### 5.4 Collections

- RustiCal creates default calendar + addressbook per principal on first use;
  create explicitly via frontend (or MKCOL) if you want named ones (e.g.
  `personal`, `work-hawksnest`, `tasks`).
- Task lists are ordinary CalDAV calendars that accept VTODO (Tasks.org/Thunderbird
  treat them as task lists) — verify `calendar-collection` supports both VEVENT and
  VTODO component-set during the Phase 14 tests.
- Birthdays: RustiCal auto-exposes
  `/caldav/principal/<id>/_birthdays_<addressbook_id>` — bonus, no config needed.

---

## Phase 6 — Sync Hub for Any Provider (vdirsyncer)

**Deliverable: the router is the convergence hub for all identities regardless of
which provider hosts them.**

> **STATUS (2026-09-04): DONE — 6.1, 6.2, and 6.3 steps 1–2 executed and verified;
> 6.3 step 3 (canonicality flip) deliberately deferred per its own gate (needs the
> verification matrix, i.e. Phase 7 client rows, to pass first).**
> - **Safety net:** hot SQLite `.backup` pulled before any change
>   (`~/backups/omnical/db-pre-phase6-20260904.sqlite3`, 148 KB) and again after
>   migration (`db-post-phase6-20260904.sqlite3`, 3.5 MB).
> - **Pre-flight scratch validation** (throwaway pair in `/tmp/opencode`, 3 real
>   Google events → temporary `_vdirtest` calendar on the server, then deleted):
>   MKCOL/PUT/etag/GET/idempotent-re-sync all pass. Content round-trips
>   semantically identical — RustiCal adds RFC-5545-correct CRLF and re-folds long
>   lines; vdirsyncer normalizes both, so its second sync was 0 actions.
> - **6.1 DONE — 9 new pairs + 8 new storages** appended to
>   `~/.config/vdirsyncer/config` (pre-edit backup: `config.pre-phase6-20260904.bak`).
>   Existing Google pairs untouched — the new pairs **share the same filesystem
>   storages** (that IS the hub: Google ↔ vdir ↔ RustiCal). Initial sync:
>   **721 items uploaded a→b, 0 downloaded, 0 errors**. Deviations from the plan's
>   example, forced by vdirsyncer 0.20 reality:
>   - `collections` uses the 0.20 three-item entry form `["config-name", "a-name",
>     "b-name"]` instead of `"from a"` (names genuinely differ per side); `null` is
>     allowed in the a/b slots and is used for the three flat, local-only khard
>     addressbooks (no collection subdir — the dir itself is the collection).
>   - `password.fetch = ["command", "pass", "show", "secrets/omnical/<id>/vdirsyncer"]`
>     is 0.20's spelling of the plan's "password-command". Verified headless
>     (gpg-agent reachable with HOME only; `pass` resolvable from both
>     `/system/index/bin` and `/usr/bin`), so the existing cron can use it.
>   - URL is `https://0115d8cf.duckdns.org:8443/` (Phase 0.3 amendment; NAT
>     hairpin verified from the dev machine).
> - **Data mapping (all 7 identities covered):**
>   - Calendars: `google_calendar_local`/`nfcarlton@gmail.com` (195 events) →
>     nfcarlton@gmail.com `personal`; `google_calendar_local_hawksnest`/
>     `nicholas@hawksnestsoftware.com` (0 events) → hawksnest `personal`, and its
>     second collection `cln2…@virtual` — Google's read-only **"Holidays in United
>     States"** calendar (317 events) → **new `holidays` collection** created via
>     MKCOL on the router (displayname "US Holidays") so the holiday vdir is
>     hub-covered too and khal can keep it after the 6.3 flip.
>   - Contacts: `burningserenity-gmail`/`nfcarlton-gmail`/`nfcalaway-gmail`/
>     `hawksnest` (Google-paired, collection `default`, 104/104/0/0 vcards) → each
>     identity's `personal` addressbook; flat local-only `carltonaudio` (0),
>     `novo-ordo` (1), `novo-ordo-zero` (0) → `personal` of
>     nicholas@carltonaudio.com / nfcalaway@novo-ordo.com / zero@novo-ordo.com.
> - **6.2 DONE** — discover + metasync + sync all clean for the new pairs; second
>   sync = 0 actions; full all-pairs `vdirsyncer sync` (the exact cron command) =
>   0 actions across 17 collections. **Scheduler already existed** (`*/15 * * * *
>   vdirsyncer sync` in the user crontab — exactly the plan's migration cadence),
>   so no new timer was added. `metasync` is a by-design no-op here (a pair's
>   `metadata` keys default to empty — no displayname files were written into the
>   vdirs).
> - **6.3:** step 1 DONE (the 721 uploads; server-side counts verified equal to
>   local: 195/317/0 + 104/104/1/0/0/0). Step 2 (strays) — both investigated, both
>   no-ops: `~/.calendars/local/` is **empty** (0 `.ics`), and
>   `~/.contacts/contacts.csv` (90 rows) is a **stale Google export** — every
>   phone (90/90) and email (6/6) already exists in the synced vCard addressbooks,
>   so importing would only create duplicates → skipped. Step 3 NOT done (matrix
>   gate). **Reminder for the 6.3 flip:** khal's `google_hawksnest` calendar points
>   at a *stale* dir (`~/.calendars/google/nicholas@hawksnestsoftware.com/`, 1
>   stale `.ics`) instead of the actively-synced `~/.calendars/hawksnest/…` — fix
>   when repointing khal.
> - **Two-way proof (Google-independent):** server-side PUT of a test vCard into
>   carltonaudio's `personal` → downloaded into the local khard dir; server-side
>   DELETE → deletion propagated locally (needed `--force-delete` — vdirsyncer's
>   empty-storage guard, expected for a 1-item collection); both sides back to
>   pristine 0 items.
> - **Router impact:** DB 151 KB → 3.5 MB; SQLite WAL spiked to 7 MB during the
>   bulk push → overlay briefly **2.6 M free (below the ≥5 M floor)**;
>   `PRAGMA wal_checkpoint(TRUNCATE)` folded it back (online-safe, service live) →
>   **9.3 M free (91 %)**. **Note for Phase 8:** include a WAL checkpoint in the
>   nightly backup cron; keep the 8.4 disk watch.
> - **Findings for later phases:**
>   - `burningserenity-gmail` and `nfcarlton-gmail` khard addressbooks **both
>     sync from the nfcarlton@gmail.com Google account** (100/104 cards
>     byte-identical) — that 104-contact set therefore now exists twice in
>     RustiCal (once per identity), mirroring the user's existing khard topology.
>     Kept deliberately (identity-based mapping per §3.4); revisit at the 6.3
>     flip if de-duplication is wanted.
>   - RustiCal hrefs = the item UID when UID-safe; UIDs containing `@` (all
>     `…@google.com` ones) get generated UUID hrefs — harmless (vdirsyncer tracks
>     hrefs via listing, not construction).
>   - Benign noise: vdirsyncer's discovery PROPFIND on `/` logs 405 "client
>     error" in rustical's `logread` (frontend root is GET/HEAD-only — known from
>     Phase 1.5); discovery proceeds via `/.well-known/*` and works.
>   - Verification-matrix **row 10 now passes** (clean discover+sync incl. ETags,
>     no items lost; both directions exercised). Rows 11+ remain for the matrix
>     run / Phase 7.

### 6.1 Extend `~/.config/vdirsyncer/config` — **DONE (2026-09-04; see STATUS for the 0.20 syntax deviations)**

Keep the existing Google↔vdir pairs untouched; add RustiCal pairs per identity.
Example for one identity (repeat pattern per identity):

```ini
[pair rustical_calendar_nfcarlton]
a = "nfcarlton_calendar_local"          # existing filesystem storage (~/.calendars/google/nfcarlton@gmail.com)
b = "rustical_calendar_nfcarlton_remote"
collections = ["from a"]                 # or pin collection hrefs after discover
conflict_resolution = "b wins"           # during migration; revisit after verification

[storage rustical_calendar_nfcarlton_remote]
type = "caldav"
url = "https://0115d8cf.duckdns.org/"
username = "nfcarlton@gmail.com"
password = "<app token from pass>"

[pair rustical_contacts_nfcarlton]
a = "nfcarlton_contacts_local"
b = "rustical_contacts_nfcarlton_remote"
collections = ["from a"]

[storage rustical_contacts_nfcarlton_remote]
type = "carddav"
url = "https://0115d8cf.duckdns.org/"
username = "nfcarlton@gmail.com"
password = "<app token from pass>"
```

(App tokens can be referenced via vdirsyncer's password-command with `pass` to keep
them out of the config file.)

### 6.2 Run & schedule — **DONE (2026-09-04; existing `*/15 * * * * vdirsyncer sync` cron reused — see STATUS)**

```sh
vdirsyncer discover && vdirsyncer metasync && vdirsyncer sync
```

Add to cron/timer if no vdirsyncer scheduler exists today (this plan does not assume
one; check `crontab -l` / systemd timers first). Suggest every 15 min during
migration, then hourly.

### 6.3 Migration & khal/khard repointing — **steps 1–2 DONE 2026-09-04 (both no-ops — see STATUS); step 3 gated on the verification matrix as written**

1. Initial sync: vdirs (Google truth) → RustiCal collections.
2. Import strays: `~/.calendars/local/*.ics` and `~/.contacts/contacts.csv`
   (convert to vCard, e.g. with `khard` tooling) via RustiCal's frontend import.
3. After the verification matrix passes, flip conflict resolution and/or repoint
   khal/khard to the RustiCal-side vdir trees so the router becomes canonical:
   ```ini
   [[omnical_nfcarlton]]
   path = ~/.calendars/omnical/nfcarlton@gmail.com/
   type = calendar
   ```

---

## Phase 7 — End-User Clients & Cross-Domain Invitations

**Deliverable: every approved client class working against the router.**

| Client | Setup | Notes |
|---|---|---|
| **DAVx5** | Login via **Nextcloud flow**: URL `https://0115d8cf.duckdns.org`, use frontend login → generates app token automatically; collections auto-discovered | WebDAV Push gives near-instant sync (enable dav_push transport per docs) |
| **Tasks.org** | Add CalDAV account via DAVx5; task lists appear as calendars with VTODO support | The "tasks" requirement of this project |
| **Apple Calendar** | Use `/caldav-compat` paths (Apple mishandles multi-home `calendar-home-set`), or install the **configuration profile** generated by RustiCal's frontend (token section) | Requires real TLS (satisfied by LE) |
| **Apple Contacts** | CardDAV account, server `0115d8cf.duckdns.org`, user id + app token, path `/carddav` | |
| **Thunderbird** | New Account → Calendar → On the Network → root URL `https://0115d8cf.duckdns.org` + app token; same for CardDAV | Group calendars discovered properly |
| **khal / khard** | via vdirsyncer hub (Phase 6) | CLI stays exactly as today, now backed by the router |

**Cross-domain invitations (iMIP):** RustiCal does not implement RFC 6638
server-side scheduling. Invitations to attendees on **any** domain are sent
client-side: Apple Calendar and Thunderbird compose an `.ics` iTIP message and email
it via whichever SMTP identity the user picks — attendee replies flow back the same
way and update the event. This works with every email domain by construction; it is a
documented product limitation that the *server itself* doesn't also maintain an
inbox/outbox/scheduling queue (see §17 for the future custom extension).

---

## Phase 8 — Backups & Maintenance

**Deliverable: nightly verified backups, health monitoring, sysupgrade runbook.**

### 8.1 Nightly pull-based backup (dev machine cron; reuses `router` SSH alias)

```sh
# 02:30 local, daily
ssh router 'sqlite3 /usr/local/share/rustical/db.sqlite3 ".backup /tmp/omnical-bu.db" \
            && tar -C /tmp -czf - omnical-bu.db /etc/rustical' \
  > ~/backups/omnical/$(date +\%F).tar.gz && rm -f ~/backups/omnical/*.tar.gz.(older than 30d)
```

- SQLite `.backup` is online/safe (no service stop).
- Retention: 30 days local; optionally rsync a weekly copy to a VM (`wg-vm1`/`wg-vm3`).
- **Restore drill:** test-restore the DB into a scratch RustiCal instance on the dev
  machine at least once (this is part of the verification matrix).

### 8.2 Monitoring

- procd respawn covers crashes; add `rustical health` to the existing
  `screech-watchdog`-style pattern if desired (every 5 min cron on router).
- `logread -e rustical | tail` for triage; keep `tracing` at warn/info.

### 8.3 Sysupgrade runbook (router firmware updates)

1. Backups are current (check nightly artifact exists).
2. `/etc/sysupgrade.conf` already preserves `/etc/rustical` + data dir (Phase 2.6).
3. After sysupgrade: re-run `~/router-dav/deploy.sh` (re-pushes binaries + init
   scripts), verify `rustical health` + external reachability.

### 8.4 Storage watch

`df -h /` weekly expectation: overlay stays ≥ 5 MB free (binary + DB + certs are the
only consumers). Add to backup cron: `ssh router df -h / | tail -1` in the log.

---

## 14. Verification Matrix

Run in order; each row is a gate. Do not migrate khal/khard canonicality (Phase 6.3)
before row 10 passes.

| # | Test | Command / Method | Expected |
|---|---|---|---|
| 1 | Service health | `ssh router /usr/sbin/rustical health` | healthy |
| 2 | Bound correctly | `netstat -tlnp` on router | rustical on 127.0.0.1:4000; dav-tls on 0.0.0.0:443; uhttpd on 8443 |
| 3 | TLS chain (external) | `openssl s_client` + `curl -sI https://0115d8cf.duckdns.org/.well-known/caldav` from LTE/VM | LE issuer, 200/301 |
| 4 | DAV headers | `curl -X OPTIONS -u user:token https://…/caldav/` | `DAV: 1, calendar-access` (and `addressbook` on /carddav/) |
| 5 | PROPFIND | `curl -X PROPFIND -H 'Depth: 1' -u … https://…/caldav/` | 207 multistatus, principal home-set lists collections |
| 6 | calendar-query REPORT | curl REPORT with time-range filter on a test calendar | 207, only matching VEVENTs |
| 7 | VTODO | PUT a VTODO; REPORT with `calendar-query` comp-filter VTODO | 207; Tasks.org sees the task |
| 8 | addressbook-query REPORT | curl REPORT on `/carddav/…` | 207, vCards match filter |
| 9 | Well-known | `curl -sI https://…/.well-known/caldav` and `…/carddav` | 30x to correct roots (client autodiscovery path) |
| 10 | vdirsyncer | `vdirsyncer discover && sync` (new pairs) | clean two-way sync incl. ETags; no items lost (diff before/after) — **✓ verified 2026-09-04 (Phase 6: 721 items a→b, 0 errors; server-side PUT/DELETE round-trip b→a; idempotent re-sync; server counts == local counts)** |
| 11 | khal / khard | create/edit event & contact via CLI in the omnical vdirs | appears on server (verify via curl REPORT) and on other clients |
| 12 | DAVx5 + Tasks.org | Android account; create/edit event, contact, task | syncs both directions; WebDAV Push = near-instant when enabled |
| 13 | Thunderbird | calendar + cardbook/tasks accounts at root URL | discovers all own collections + group calendars |
| 14 | Apple Calendar/Contacts | caldav-compat path or config profile | account works; create/edit round-trips; contacts sync |
| 15 | Sharing | group collection visible to member identities (all 4 domains) | cross-domain share works via membership |
| 16 | iMIP invitation | Thunderbird invite to an external address on a different domain; attendee accepts | reply updates organizer's event |
| 17 | Reboot persistence | `reboot` router; re-check services + data | everything returns; DB intact (proves not-in-/var) |
| 18 | Restore drill | restore nightly backup tar into scratch instance on dev machine | DB opens, data present |
| 19 | Firewall hygiene | nmap 4000 from LAN/WAN; port-scan WAN IP | 4000 closed; only 22/443(+53) exposed |

---

## 15. Rollback / Recovery

Every step is reversible; nothing destructive is done to the router.

- **Services:** `/etc/init.d/dav-tls disable && stop`; `/etc/init.d/rustical disable && stop`.
- **Files:** `rm /usr/sbin/{rustical,dav-tls} /etc/init.d/{rustical,dav-tls}`; `rm -rf /etc/rustical` (keep a copy of the DB first).
- **uhttpd:** restore `listen_https` to `:443` entries; `uci commit uhttpd; restart`.
- **Firewall:** delete `Allow-Dav-TLS` rule; `uci commit firewall; restart`.
- **Upstream:** remove port-forward on TP-Link.
- **Data:** the SQLite DB is a single file — copy it off with the Phase 8 procedure at any time; restoring = stop service, replace file, start.
- **Google side is never modified by this plan** — original vdirs/Google pairs stay intact until you explicitly migrate canonicality (Phase 6.3), so the pre-Omnical state is always recoverable.

---

## 16. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| RustiCal binary > 35 MB stripped (42 MB overlay budget) | Medium | Blocks internal-flash decision | strip → size-opt profile → LTO/panic=abort → upx --lzma → escalate USB-stick decision to user |
| RustiCal is young software; protocol bugs with picky clients (Apple) | Medium | Client-specific breakage | Pin 0.16.1; full verification matrix incl. Apple before cutover; upstream has tested-client list; issues tracker active |
| No RFC 6638 scheduling | Certain (accepted) | No server-side invitations | Client-side iMIP covers invitations; future extension (§17) |
| RFC 6578 sync-token partial → slower large-collection syncs | Low | Performance only | Collections here are personal-scale (thousands of items max); acceptable |
| Frontend dist not committed upstream → extra build step | Low | Build friction | bun is installed; check at clone (Phase 1.2) |
| Dynamic IP / duckdns updater gap | Low | Brief outages on IP change | Verify updater in Phase 0; add router cron if needed (2.7) |
| Upstream port-forward impossible on TP-Link | Low | No public exposure | VM wireguard relay fallback documented (§4.2) |
| Flash wear from SQLite on f2fs | Low | Long-term hardware | Personal write volumes are tiny; SQLite WAL; nightly `.backup` keeps data safe regardless |
| Zig/rustc version drift breaks cross-build | Low | Build breakage | Pin toolchain in rustup/notes; router-nym proved zig 0.16 + musl target works |
| f2fs overlay corruption / sysupgrade data loss | Low | Data loss | sysupgrade.conf entries + nightly off-router backups + restore drill (rows 17–18) |

---

## 17. Future / Optional Enhancements

1. **RFC 6764 autodiscovery SRV/TXT records** at DNS providers for domains you control
   (`hawksnestsoftware.com`, `carltonaudio.com`, `novo-ordo.com`):
   `_caldavs._tcp.<domain> SRV 0 1 443 0115d8cf.duckdns.org`,
   `_carddavs._tcp.<domain> SRV …`, `TXT "path=/caldav"` — enables
   "just type your email address" account discovery in Apple/DAVx5/Thunderbird for
   those domains. (gmail.com etc. obviously can't be annotated — direct URL login
   still works for any address.)
2. **Server-side scheduling (RFC 6638) / iMIP gateway**: a small custom Rust service
   (same deployment pattern) that watches an IMAP inbox for `text/calendar` replies
   and injects them / sends invitations via SMTP — would make invitations fully
   server-driven while remaining domain-agnostic. Substantial; only if client-side
   iMIP proves insufficient.
3. **WebDAV Push transports** (WebSocket/WebPush) tuning for instant DAVx5 sync
   (RustiCal ships support; configure in `dav_push` after Phase 7).
4. **IPv6**: publish AAAA on duckdns once a stable GUA exists on WAN; same firewall
   rule already covers it (`family='any'`).
5. **Ujail/seccomp hardening** of both services via procd jail params, once stable.
6. **Dedicated VM off-site replica** (via existing wg-vm tunnels) running the same
   binaries — a warm standby with hourly DB copy.

---

## Appendix A — RustiCal Reference Notes

- Version pinned: **0.16.1** · repo: `https://github.com/lennart-k/rustical` · docs:
  `https://lennart-k.github.io/rustical/` · license: AGPL-3.0-or-later
- Subcommands: `serve` · `principals` (users/groups) · `membership` · `gen-config`
  (prints full default config — use it to confirm exact TOML keys before editing) ·
  `health`
- URL map: `/.well-known/caldav|carddav` → `/caldav`, `/caldav-compat`,
  `/carddav`; principals at `/caldav/principal/<user_id>`; group impersonation user
  `user$group`; birthdays auto-collection
  `_birthdays_<addressbook_id>`
- Auth model: frontend login = argon2 password; DAV/CalDAV/CardDAV = HTTP Basic with
  **user id + app token** (pbkdf2-verified random tokens, per-client, generated in web
  UI; DAVx5 can self-provision via Nextcloud login flow)
- User id constraints: no `:` or `$` characters — **full email addresses of any domain
  are valid**
- Storage: single SQLite DB (sqlx, bundled); needs writable `/tmp`; CA certs at
  `/etc/ssl/certs` (present on router)
- Upstream config env-var mapping example (Docker):
  `RUSTICAL_DATA_STORE__SQLITE__DB_URL=/var/lib/rustical/db.sqlite3` — **do not use
  /var on this router (tmpfs)**
- Not implemented: RFC 6638 scheduling (client-side iMIP instead); RFC 6578
  sync-token partial
- Features present: WebDAV Push (web-push), Nextcloud login flow, Apple configuration
  profiles, group-based sharing, deleted-calendar recovery, frontend import/export,
  OIDC (optional; not used in this plan)

## Appendix B — Key Commands & File Paths

**Router paths**
- `/usr/sbin/rustical`, `/usr/sbin/dav-tls` — binaries (re-deploy after sysupgrade)
- `/etc/rustical/config.toml` — server config (0600)
- `/etc/rustical/tls/{fullchain.pem,key.pem}` — LE cert/key (0600)
- `/usr/local/share/rustical/db.sqlite3` — THE database (persistent overlay)
- `/etc/init.d/rustical`, `/etc/init.d/dav-tls` — procd services
- `/etc/sysupgrade.conf` — includes `/etc/rustical`, `/usr/local/share/rustical`
- `/etc/crontabs/root` — existing cron home (backups run from dev machine instead)

**Dev machine paths**
- `~/router-dav/` — this project (scripts, dav-tls source, deploy.sh, out/)
- `~/router-nym/build/bin/zig-musl-cc` (+ ar/ranlib/nm symlinks) — reused wrappers
- `~/.acme.sh/` — acme.sh home, duckdns DNS-01 + install-cert deploy hook
- `~/.config/vdirsyncer/config` — extended with RustiCal pairs (Phase 6)
- `~/backups/omnical/` — nightly backup artifacts
- `pass` entries: `secrets/duckdns/0115d8cf.duckdns.org/token`, `secrets/omnical/<identity>/<client>`

**One-liners**
```sh
ssh router                                        # shell on router
ssh router '/usr/sbin/rustical health'            # health check
ssh router 'logread | grep -i rustical | tail'    # logs
ssh router 'sqlite3 /usr/local/share/rustical/db.sqlite3 ".backup /tmp/bu.db"'  # hot backup
curl -sI https://0115d8cf.duckdns.org/.well-known/caldav                        # liveness
vdirsyncer discover && vdirsyncer sync            # hub sync
```

## Appendix C — Relevant RFCs

| RFC | What it defines | Status in this deployment |
|---|---|---|
| 4918 | WebDAV core (PROPFIND, MKCOL, locks) | Implemented by RustiCal |
| 3253 | Versioning / REPORT method | Implemented (REPORT used heavily by clients) |
| 4791 | CalDAV | Implemented (VEVENT + VTODO → tasks) |
| 5545 | iCalendar (VEVENT/VTODO/VJOURNAL, iTIP) | Object format; VTODO = the tasks requirement |
| 6352 | CardDAV | Implemented |
| 6578 | Collection synchronization (sync-token) | Partial in RustiCal; clients fall back to full sync |
| 6638 | CalDAV scheduling (server-side invitations) | **Not implemented** → client-side iMIP (RFC 6047 iTIP-in-IMAP; email invitations from any domain) |
| 6764 | DAV service discovery (SRV/TXT + /.well-known) | `/.well-known` served by RustiCal; SRV records optional (§17.1) |
| 8291/8030 | WebPush | Used by RustiCal's WebDAV Push (optional enable) |

---

*End of plan. Execute phases in order; gates are called out at Phases 1, 2, 3, and the
verification matrix. Escalate to the user if any size-gate, port-forward, or Apple
client verification fails.*
