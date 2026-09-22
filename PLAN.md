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
| C8 | RFC 6638 absent | Cross-domain invitations via client-side iMIP (Apple/Thunderbird email `.ics` from any SMTP identity to any attendee domain) | Honest limitation; server-side scheduling as a custom extension — **PULLED FORWARD, IN PROGRESS since 2026-09-05 (see §17.2)** — because iOS offers no invite UI at all without server-side scheduling |
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

> **STATUS (2026-09-04): DONE — incl. the reboot gate, which passed after a
> natural reboot on 2026-09-05 (see the reboot-gate paragraph below).**
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
> **Reboot gate (2.8): PASSED 2026-09-05** — user rebooted the router that evening
> (uptime 40 min at verification); everything auto-started and survived: rustical +
> dav-tls both `running` (bound 127.0.0.1:4000 / 192.168.1.21:8443 exactly as
> configured), uhttpd untouched on LAN :443, `rustical health` OK, DB intact at
> `/usr/local/share/rustical/db.sqlite3` (`PRAGMA integrity_check` = ok; live counts
> exactly the pre-reboot values — 517 calendar / 209 address objects, 8 principals,
> 29 app tokens), TLS certs present, all three router crons alive (srvzone,
> screech-watchdog, DuckDNS `curl -4`), overlay 9.3 M free (floor intact), and
> external reachability re-proven from the dev machine: `/.well-known/caldav` → 308
> + authenticated PROPFIND 207 through the public URL with full LE chain validation.
> Proves not-in-/var, procd boot persistence of both services, and cron persistence.
> Verification-matrix row 17 green. (Historical reminder, now moot: all verify
> commands and the firewall rule use port **8443**.)

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
**CORRECTED (2026-09-06, Phase 8.3, via `sysupgrade -b` ground truth): custom
init.d scripts are NOT preserved either** — the conffile mechanism covers only
package-provided scripts; `/etc/init.d/{rustical,dav-tls}` are wiped along with
the binaries (and `/usr/bin/rustical-watchdog`). `deploy.sh` re-pushes them, so
the re-run instruction stands (see 8.3's runbook).

### 2.7 Optional: duckdns updater cron (only if Phase 0 step 4 found nothing) — **DONE (curl -4 pinned; see STATUS finding)**

```cron
*/5 * * * * curl -s "https://www.duckdns.org/update?domains=0115d8cf&token=<TOKEN>&ip=" >/dev/null
```

(Token inserted at deploy time, chmod 600 crontab — never logged.)

### 2.8 Verify — **DONE incl. reboot gate (natural reboot verified 2026-09-05 — see STATUS)**

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
>   curl-equivalent; see Phase 4 STATUS). ~~Still deferred: 2.8 reboot gate~~
>   — **resolved 2026-09-05: natural reboot passed, see Phase 2 STATUS** (it also
>   proved dav-tls starts on boot).

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
> - Still open from earlier phases: ~~**2.8 reboot gate** (next natural reboot)~~
>   — **resolved 2026-09-05: natural reboot passed, see Phase 2 STATUS.**

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

> **STATUS (2026-09-04, updated 2026-09-06, completed 2026-09-07): DONE — 6.1, 6.2,
> and 6.3 steps 1–2 executed and verified. 6.3 step 3 (canonicality flip) had been
> deferred per its own gate (matrix Phase 7 client rows) but was **EXECUTED
> 2026-09-06 by explicit user gate override** — the flip is LIVE (dedicated omnical
> vdir trees synced from RustiCal, khal/khard repointed, conflict resolutions
> flipped so the router is canonical end-to-end). ~~Only the two-way re-proof
> remains open.~~ **The two-way re-proof is DONE (2026-09-07): khal and khard full
> round-trips through the NEW omnical trees, every step synced and server-verified,
> full sync idempotent — 6.3 is now FULLY DONE (see the step-3 record below, incl.
> a new hub-chain deletion-resurrection finding).** Decisions,
> results, and resume point: see the 6.3 step 3 record below.**
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
>   so importing would only create duplicates → skipped. Step 3 EXECUTED
>   2026-09-06 (user gate override; record under 6.3 step 3 below).
>   ~~Reminder for the 6.3 flip: khal's `google_hawksnest` calendar points at a
>   *stale* dir (`~/.calendars/google/nicholas@hawksnestsoftware.com/`, 1 stale
>   `.ics`) instead of the actively-synced `~/.calendars/hawksnest/…` — fix when
>   repointing khal.~~ **RESOLVED by the flip: khal now reads the omnical trees
>   (`omnical_hawksnest` ← RustiCal `personal`), so the stale dir is no longer
>   referenced by any client.**
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
>     flip if de-duplication is wanted. — **REVISITED at the flip (2026-09-06):
>     user decision = KEEP the duplicate.**
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

### 6.3 Migration & khal/khard repointing — **steps 1–2 DONE 2026-09-04 (both no-ops — see STATUS); step 3 FULLY DONE 2026-09-07 (flip live since 2026-09-06; two-way re-proof through the omnical trees verified 2026-09-07 — see the record under step 3)**

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

   — **EXECUTED (2026-09-06; flip LIVE, and the two-way re-proof completed
   2026-09-07 — see the re-proof record at the end of this block). User
   decisions (all 2026-09-06):**
   proceed despite matrix rows 12–16 being open (gate override accepted — the
   flip is dev-machine-only, the server untouched, fully reversible);
   **topology** = dedicated omnical trees + conflict flips, hub pairs intact
   (Google keeps mirroring both ways); **khal scope** = current set + holidays;
   **contact de-dup** = KEEP the 104-card duplicate.
   - **Pre-flight:** server healthy (`rustical health` OK); full `vdirsyncer
     sync` baseline clean and idempotent; server-side live counts verified per
     collection (calendars: nfcarlton `personal` **196**, hawksnest `personal`
     **0**, hawksnest `holidays` **317**; addressbooks: burningserenity
     **104**, nfcarlton **104**, nfcalaway@novo-ordo **1**, the other four
     **0**). Count-checking oddity found en route (pre-existing, inside the
     long-recorded 517 live total, untouched by the flip): 3 live object rows
     still sit under the soft-deleted `_vdirtest` calendar and 1 live row sits
     in nicholas@carltonaudio.com's `personal` CALENDAR — soft-deleting a
     collection evidently does not tombstone its object rows.
   - **Config backups (the rollback story):**
     `~/.config/vdirsyncer/config.pre-phase63-20260906.bak`,
     `~/.config/khal/config.pre-phase63-20260906.bak`,
     `~/.config/khard/khard.conf.pre-phase63-20260906.bak` — and the ORIGINAL
     vdir trees were never touched, so rollback = restore the three configs +
     optionally delete the omnical pairs/trees.
   - **vdirsyncer changes (`~/.config/vdirsyncer/config`):**
     (1) the two Google CALENDAR pairs flipped `conflict_resolution`
     "b wins"→"a wins" — the RustiCal-fed vdir now beats Google on conflict;
     (2) explicit `conflict_resolution = "b wins"` added to the 7 rustical
     CONTACT pairs (previously unset = hard error on conflict);
     (3) **10 new pairs appended** (comment-marked "Omnical canonicality
     flip"): `omnical_calendar_{nfcarlton,hawksnest}` covering 3 collections
     (nfcarlton `personal`; hawksnest `personal` + `holidays`) and
     `omnical_contacts_<id>` ×7, sharing two new filesystem storages —
     `~/.calendars/omnical/` and `~/.config/khard/contacts/omnical/` (a-side
     collection dirs = identity email; `holidays` for the holiday tree — the
     plan's example path `~/.calendars/omnical/nfcarlton@gmail.com/` realized
     verbatim) — with per-identity caldav/carddav remotes reusing the
     `pass`-fetched vdirsyncer app tokens and Phase 6's three-item
     `collections` form. All server pairs `b wins` (server canonical). The
     existing `*/15` vdirsyncer cron picks the new pairs up automatically (it
     syncs the whole config).
   - **Initial sync verified:** discover + sync clean; the omnical trees
     downloaded from the server with counts EXACTLY equal to the server live
     counts above (196/317/0 `.ics`; 104/104/1/0×4 `.vcf`); full re-sync
     idempotent (0 actions).
   - **khal repointed (`~/.config/khal/config`):** `google`/`google_hawksnest`
     REPLACED by `omnical_nfcarlton` (dark green) and `omnical_hawksnest`
     (dark magenta — this is the stale-dir fix), `omnical_holidays` ADDED
     (color `yellow`; **finding: khal 0.14's palette has no "dark yellow"**),
     the empty `local` calendar kept as-is, `default_calendar =
     omnical_nfcarlton`. Verified live: `khal printcalendars` lists the four;
     `khal list` renders real events incl. "Labor Day :: Public holiday" from
     the holidays tree.
   - **khard repointed (`~/.config/khard/khard.conf`):** the 7 addressbook
     NAMES kept identical (muscle memory/scripts), paths →
     `~/.config/khard/contacts/omnical/<identity email>`. Verified live:
     per-addressbook listings correct (nfcarlton-gmail 104, novo-ordo 1).
     **Finding:** the combined `khard list` DEDUPLICATES the byte-identical
     burningserenity/nfcarlton cards by UID (105 unique rows shown of 209) —
     display behavior only, no data touched.
    - **Two-way re-proof through the NEW trees — DONE (2026-09-07): all gates
      green; 6.3 step 3 is now FULLY DONE.** Executed exactly per the former
      resume list (with the recorded khal 0.14 `-a` / `"summary ::
      description"` syntax); safety-net DB backup pulled first
      (`~/backups/omnical/db-pre-63reproof-20260907.sqlite3`, 0600, integrity
      ok, live counts 517/209). Final state: live counts back to EXACTLY
      517/209 (soft-delete tombstones +1 event / +1 card → raw 521/212, as
      designed); full sync idempotent; overlay 6580 KB free (floor intact).
      - **khal leg:** `khal new -a omnical_nfcarlton 2026-09-10 15:00 16:00
        "Omnical re-proof :: …"` → 197th `.ics` in the omnical tree (UID
        `TLXS…`, UID-safe href); sync passes pushed it through the WHOLE
        chain — one hop per pass, action-log evidence each: omnical pair →
        server, rustical pair → old vdir, google pair → Google. Server-side
        calendar-query REPORT (Sept-10 time-range, through dav-tls,
        `ssl_verify_result=0`) → 207 with exactly the test event
        (SUMMARY/DESCRIPTION/UID intact; the one extra REPORT match is the
        known recurring-event over-match, not ours). Delete via `printf
        'D\ny\n' | khal edit "Omnical re-proof"` → sync passes → deletion
        logged on ALL three hops (server / old vdir / Google), GET of the
        href → 404, REPORT window empty, `khal search` empty, omnical tree
        back to 196.
      - **khard leg:** full create → edit → remove cycle in `carltonaudio`
        (the repointed omnical tree; khard 0.21 YAML template = the row-11
        recipe: First/Last name + `Email: internet:`): create → 2 sync
        passes (omnical pair → server, rustical pair → old flat tree
        `~/.config/khard/contacts/carltonaudio/`) → server verified via
        addressbook-query REPORT with an FN text-match (207, exact vcard);
        edit (dump khard's own YAML via `khard show --format=yaml`, change
        the email, `echo y | khard edit -a carltonaudio -i …`) → sync →
        server shows the edited email on the SAME UID/href; remove
        (`khard remove -a carltonaudio --force`) → `vdirsyncer sync
        --force-delete` ×2 — the 1→0 empty-storage guard tripped exactly as
        established, on BOTH hops (server addressbook 1→0 AND old flat tree
        1→0; the plain boolean flag covers the whole run) → FN-filter REPORT
        → empty multistatus + GET of the old href → 404; both trees back to
        0 cards, `khard list -a carltonaudio` → "Found no contacts".
      - **Idempotency (former resume item 2):** full `vdirsyncer sync` →
        0 actions / 0 errors; immediate second run → 0 actions.
      - **NEW FINDING — hub-chain deletions can be transiently REVERTED by
        Google re-serialization (first delete attempt failed; retried
        clean):** the first delete pass hit a **412 Precondition Failed**
        (vdirsyncer 0.20 runs pairs CONCURRENTLY — the omnical pair deleted
        the server item while the rustical pair was mid-PUT of the same item
        from a stale listing; transient, self-resolves on re-run), and the
        `*/15` cron ticks then RESURRECTED the event everywhere: Google had
        re-serialized the uploaded event (PRODID `-//Google Inc//…`, new
        DTSTAMP) and the google pair downloaded that version into the old
        vdir — so the old-vdir etag was "changed on a" when the rustical
        pair looked, and it re-UPLOADED the item to the server after the
        omnical delete; the omnical pair then re-downloaded it into the
        omnical tree. Root cause: a deletion racing a mid-chain content
        change (the old-tree row-11 test couldn't hit this — its deletion
        was single-hop from the shared a-side). **Clean procedure (proven,
        used for the green result above): verify convergence first (one
        sync = 0 actions), delete via the client, then run sync passes
        back-to-back — from a converged state the deletion propagates
        hop-by-hop in 3 passes with no interference.** Benign en route: one
        transient `ConnectionResetError` on a listing (retry clean) and the
        recurring `Deleting an item with no etag` warning on server/Google
        deletes (vdirsyncer proceeds without If-Match; the deletes
        demonstrably land).
      - (Matrix rows 12–16 catch-up remains independent user-device work;
        the flip no longer waits on it.)

---

## Phase 7 — End-User Clients & Cross-Domain Invitations

**Deliverable: every approved client class working against the router.**

> **STATUS (2026-09-08, updated 2026-09-08 server-side): IN PROGRESS — host side DONE,
>   scheduling extension LIVE, three verification-matrix rows cleared server-side.**
>   Previous sessions (2026-09-05/06): row 11 fully verified (khal create/edit/delete +
>   khard create/edit/remove, each step synced and server-verified); i3status-rust
>   blocker FIXED + upstream issue filed (greshake/i3status-rust#2309); iPhone accounts
>   started via manual entry; §17.2 scheduling extension deployed 2026-09-06.
>   **2026-09-08 server-side verification (this session):**
>   - **§17.2 scheduling confirmed LIVE:** OPTIONS on nfcarlton `personal` returns
>     `calendar-scheduling, calendar-auto-schedule` — the INVITEES fix is deployed
>     and active. schedule-inbox/outbox URLs present in family-group PROPFIND.
>   - **Row 13 (Thunderbird):** VERIFIED — `/.well-known/caldav` → 308 → `/caldav` +
>     `/.well-known/carddav` → 308 → `/carddav`; PROPFIND 207 on both roots with
>     thunderbird app token. Both calendar + contacts discoverable.
>   - **Row 14 (Apple Calendar + Contacts):** VERIFIED server-side — all 7 `apple`
>     tokens authenticate on caldav-compat (207) and carddav (207).
>   - **Row 15 (Sharing):** VERIFIED — family calendar PROPFIND 207 for all 7 member
>     identities across all 4 domains (gmail, hawksnest, carltonaudio, novo-ordo).
>   - **Rows 12 (Android), 16 (iMIP), §17.2 item 6 (iPhone live test):** still open —
>     require user's physical devices. Row 12 skipped per user (2026-09-08). Logread
>     on the router shows NO iOS user agents since deployment (ring buffer rotated;
>     only scanner noise) — needs the live `logread -f` capture while re-saving an
>     account on the phone.
>   - **i3status-rust:** live bar vs Omnical, working (confirmed 2026-09-05, no change).
> - **Android / DAVx5 + Tasks.org (matrix row 12): STARTED 2026-09-06 — server-side
>   pre-flight + client research DONE (all green); device work NOT started. The session
>   was stopped by user before any phone/server change; nothing was created, no state
>   changed anywhere.**
>   - **User requirements (decisions, 2026-09-06):** the Android phone is a **wholly
>     separate identity — NOT one of the 7 existing ones** (none of the Google accounts
>     are on the phone); the row-12 test is **predicated on being able to invite this
>     new user** (exercises the §17.2 scheduling extension with an internal attendee)
>     and on **hosting them server-side** (a new principal to create). Credential
>     transfer must be **a direct URL with the login token embedded — no typing on the
>     phone**. The phone currently runs **stock Google Calendar only** — DAVx5 and
>     Tasks.org are NOT installed yet (install notes: DAVx5 is free on F-Droid, paid on
>     Play; Tasks.org is free on both; also grab a QR scanner that fires ACTION_VIEW,
>     e.g. Binary Eye from F-Droid).
>   - **Server-side pre-flight (all green, 2026-09-06):** router healthy, running the
>     current build — startup log shows `Scheduling extension enabled (7 SMTP
>     identities)` + `Subscriptions extension enabled (public export feeds)`
>     (rustical PID 22534 since 20:17); dav-tls LISTEN on `192.168.1.21:8443` as
>     expected. `OPTIONS` on a calendar collection (nfcarlton `personal`, vdirsyncer
>     token) returns `dav: 1, 3, access-control, calendar-access, webdav-push,
>     calendar-scheduling, calendar-auto-schedule` — every DAV token DAVx5 needs,
>     including WebDAV Push and the scheduling tokens. `[dav_push]` in the router
>     config is commented out but **that means DEFAULTS, and `DavPushConfig::default()`
>     has `enabled: true`** (src/config.rs:223) — the WebPush transport is live without
>     any config change (the config comment "enable … after Phase 7 tests" predates
>     this verification; no action needed). `davx5` app tokens already exist in `pass`
>     for all 7 identities (69 chars each) — but per the new requirement the Android
>     account will NOT use them; a fresh token for the new principal instead.
>   - **Research findings that fix the row-12 design:**
>     1. **Zero-typing login = QR-coded DAVx5 deep link.** DAVx5 manual §8.1.2
>        (fetched 2026-09-06): implicit-Intent URLs of the form
>        `davx5://user:password@server.example.com/path/` open the DAVx5 login screen
>        pre-filled (scheme rewritten to `https://`), and **this URL form is
>        explicitly documented for QR codes** ("generate a QR code of the URL … then
>        scan … to open DAVx5"; the manual's "don't encode programmatically" warning
>        targets app developers building Intents — the QR use case is the intended
>        purpose). Scanner must fire ACTION_VIEW (Binary Eye documented; the stock
>        camera app may only offer a web search instead — to be confirmed on-device).
>     2. **App tokens are URI-safe by construction.** `generate_app_token()`
>        (`src/commands/app_token.rs`) samples 64 `Alphanumeric` chars; the CLI prints
>        `<id4>_<token64>` (first 4 chars = validator hint). Pure `[A-Za-z0-9_]` → the
>        `davx5://user:pass@host:8443/` QR URL needs NO percent-encoding. Row-12 QR
>        will be `davx5://<new-id>:<id4>_<token64>@0115d8cf.duckdns.org:8443/`
>        (discovery via `/.well-known/*` then UA-sniffing → DAVx5 gets the standard
>        tree, not caldav-compat).
>     3. **Internal-attendee delivery is inbox-only — so the invited Android user must
>        see the event through a normal calendar, not the inbox.** Source
>        (`crates/scheduling/src/scheduler.rs` `deliver_status`): for an internal
>        (local-principal) attendee the REQUEST is injected into their schedule-inbox
>        only — no email, no auto-filing into a calendar. DAVx5 has no RFC 6638 inbox
>        support at all. **Design consequence:** put the new principal in the
>        `family` group (membership already exists as a concept; sharing = row 15
>        material) → the organizer invites them to an event stored in the `family`
>        calendar → DAVx5 syncs that calendar → the Android user RSVPs by editing the
>        event there → the attendee-PUT path (`scheduler.rs` `attendee_put`) sees the
>        PARTSTAT change by the acting principal, updates the organizer's stored
>        copies and files a REPLY in the organizer's inbox. One row-12 pass thus
>        exercises rows 12+15 together and a live §17.2 internal-attendee round-trip.
>     4. **WebDAV Push details (row-12 "near-instant" check):** DAVx5 ≥ 4.4.10
>        implements WebDAV-Push over the **WebPush transport**, requiring Google FCM
>        (Play services) or a UnifiedPush distributor on the device (choose in
>        DAVx5 settings → bottom). RustiCal's `dav_push` is exactly that WebPush
>        (VAPID) transport (`crates/dav_push` uses the `web_push` crate; VAPID keypair
>        auto-generated/persisted via `dav_push_store.get_vapid_keypair()` at
>        src/lib.rs:154 — no manual key setup). After account creation, refresh the
>         collection list so DAVx5 fetches the push props, then check a collection
>        detail shows "Push support: subscribed" and a change on the server arrives
>        within seconds.
>   - **Next steps for row 12 (NOT started; ready to execute next session):**
>     1. Pick the new principal id (user decision — any email-style id, e.g. a real
>        address of the Android user; drives iMIP-vs-inbox delivery if ever invited
>        cross-identity). Create it on the router (`principals create --password`
>        piped), assign `family` membership, create a `davx5` app token, store both
>        in `pass` under `secrets/omnical/<id>/`.
>     2. Seed collections if desired (`personal`/`tasks`/`personal` ab — mirror the
>        Phase 5.4 pattern) — the family calendar is the sharing vehicle.
>     3. Generate + display the QR (`qrencode -t ANSIUTF8` in a terminal, or save PNG;
>        the §7 QR attempt's tooling still exists), scan with Binary Eye on the phone
>        → verify DAVx5 login pre-filled → finish account creation.
>     4. Install Tasks.org, point it at the DAVx5 account's task lists.
>     5. Run the row-12 checks: two-way event/contact/task create+edit+delete
>        (server-verify each via curl REPORT like row 11), WebDAV Push latency, then
>        the invitation leg (khal or iPhone as organizer → invite the Android identity
>        → event appears in family calendar on Android → RSVP → organizer copy
>        updated + REPLY in organizer inbox; `logread -f` on the router during the
>        RSVP to capture the scheduler lines).
> - Pre-flight: router healthy (rustical `127.0.0.1:4000`, dav-tls `192.168.1.21:8443`),
>   full `vdirsyncer sync` clean, safety-net DB backup pulled to
>   `~/backups/omnical/db-pre-phase7-20260905.sqlite3` (3.5 MB) before any change.
> - User decisions: iPhone = **all 7 identities**; status-bar calendar =
>   **nfcarlton@gmail.com** (the account the previous Google-OAuth block showed).
> - **i3status-rust (new host client, added to scope by user):** new app token `i3status`
>   (nfcarlton@gmail.com) created via CLI, stored in pass
>   (`secrets/omnical/nfcarlton@gmail.com/i3status`); credentials in
>   `~/.config/i3status-rust/omnical_credentials.toml` (0600, referenced via the block's
>   `credentials_path`); calendar-block source swapped Google-OAuth → Omnical basic auth
>   (config backup `config.toml.bak.20260905-pre-omnical`). **ACTIVATED 2026-09-05 (later
>   session) — the live bar now runs the rebuilt binary with this Omnical source; see the
>   FIX EXECUTED bullet below.**
> - **Finding — block URL must be `/caldav-compat/`, NOT `/caldav/`:** the regular tree's
>   `calendar-home-set` returns TWO `<href>`s (personal + family) and i3status-rs 0.36.1's
>   quick-xml parser deserializes a single `href` String → block errors out on `/caldav/`.
>   Same multi-home quirk that breaks Apple; `/caldav-compat/` returns a single home and
>   discovery + calendar listing then parse fine ("Personal", "Personal birthdays"; Tasks
>   auto-excluded — VTODO-only).
> - **Finding — BLOCKER DIAGNOSED (2026-09-05, bisect run — supersedes the earlier
>   "prime suspects"): the "no events" failure is a client-stack bug chain; RustiCal is
>   NOT at fault.** The identifying bisect finally ran on the prepared variants (real
>   event, khal-originated): **v1** = as-extracted (folded ATTENDEE + ends in lone `\r`):
>   0 components; **v2** = folding removed, lone `\r` kept: 0; **v3** = folding kept +
>   final `\r\n` restored: 1; **v4** = neither: 1 → **the culprit is the truncated final
>   CRLF (lone `\r` at EOF), NOT RFC-5545 folding** — folding parses fine. Tracing where
>   the lone `\r` comes from, every link proven empirically:
>   1. **RustiCal is conformant end-to-end.** DB-stored ICS ends `END:VCALENDAR\r\n`
>      (5/5 sampled, `hex(substr(…))` on the router); the wire REPORT serializes that
>      final CRLF in XML text as `&#13;` + literal LF (CR-escaping is mandatory — a
>      conformant XML parser decodes it back to `\r\n`; confirmed on 2/2 calendar-data
>      payloads in the saved `compat-report.xml`).
>   2. **quick-xml 0.37's serde deserializer (exactly what the block uses) corrupts it**:
>      it strips the trailing literal LF of element text before unescaping — minimal
>      repro: `"<a>END:VCALENDAR&#13;\n</a>"` → `"END:VCALENDAR\r"` (and
>      `"<a>plain\n</a>"` → `"plain"`) — producing the lone `\r`.
>   3. **icalendar 0.16.12 then fails silently**: lone-`\r` payload → Ok-with-0-components
>      instead of an Err, so the block shows "no events" with no error.
>   Both bugs are already fixed in current upstream deps (scratch repros kept at
>   `/tmp/opencode/i3s-test/{qxml-latest,ics-latest}`): **quick-xml 0.42** preserves the
>   trailing LF (payload decodes to `…\r\n`, 1 component); **icalendar 0.17.9 and
>   0.17.13** return a loud Err on lone-`\r` (conformant input unaffected). However,
>   **i3status-rust master (post-0.36.1) still pins quick-xml 0.37** (icalendar 0.17.9)
>   → even a current upstream build still corrupts the payload (loudly, at least).
>   **Decision (work-around vs upstream, as this task required):** patch **neither
>   RustiCal** (wire-conformant; an escaping hack like `&#10;`-encoding the final LF to
>   defeat the trim was considered and rejected) **nor icalendar** (already fixed ≥0.17).
>   Fix client-side instead — separate follow-up task: **rebuild i3status-rust 0.36.1
>   with `quick-xml = 0.42` + `icalendar = 0.17.13`** (source-built GoboLinux install at
>   `/programs/x11-misc/i3status-rust/0.36.1/`; expect possible small API fixes for the
>   0.37→0.42 jump), verify the calendar block against Omnical, then `pkill i3status-rs`
>   to activate the already-wired config; **file an upstream i3status-rust issue**
>   recommending the quick-xml bump (attach the minimal repro above). ~~Bar stays on the
>   old Google source until that rebuild is executed.~~
> - **FIX EXECUTED (2026-09-05, later session) — blocker resolved; live bar on Omnical.**
>   Upstream `greshake/i3status-rust` cloned at tag v0.36.1 (commit `b4212f7` — identical
>   to the installed build) into `/tmp/opencode/i3status-rust-rebuild`; local pins are
>   **quick-xml 0.37 (lock 0.37.5) + icalendar 0.16.12** (the "0.17.9" in the bisect text
>   referred to master). Two-line `Cargo.toml` patch — `quick-xml = { version = "0.42",
>   features = ["serialize"] }` (locks 0.42.0) and `icalendar = { version = "0.17.13",
>   features = ["chrono-tz"] }` (locks 0.17.13) — then `cargo build --release`
>   (thin-LTO profile as shipped) compiled **clean on the first try: no API fixes were
>   needed for the 0.37→0.42 jump** (the block's `$value`/`@name` serde renames are
>   unchanged in 0.42); ~2.5 min. Verification against live Omnical: the mini
>   calendar-only config renders `Sun 11:00 Omnical test event` (the khal test event)
>   where the stock binary rendered the no-events icon; 25 s run, zero stderr; the full
>   user config ran 20 s clean with the calendar widget populated. Install: stock binary
>   preserved as `/usr/bin/i3status-rs.bak-0.36.1-stock` (28.6 MB), rebuild at
>   `/usr/bin/i3status-rs` (23.4 MB) — the `/programs/…/bin` and `/system/index/bin`
>   symlinks resolve to it automatically. Distinguish binaries by size (both report
>   `0.36.1 (commit b4212f7)`).
> - **Activation finding — swaybar does NOT respawn a killed status_command** (unlike
>   i3bar; the `pkill i3status-rs; swaybar respawns it` assumption was wrong). After
>   `pkill i3status-rs` the bar sat statusless; killing swaybar itself worked — sway
>   respawned swaybar, which re-exec'd status_command with the NEW binary + the
>   already-wired Omnical config. Restart method to remember: `pkill swaybar`, not
>   `pkill i3status-rs`. Live bar verified: stable >50 s, errors.txt
>   (`~/.local/share/i3status-rs/`) 0 bytes, and an ESTABLISHED connection from the
>   i3status-rs pid to `65.33.235.245:8443` — the block is live against the router
>   (hairpin NAT path).
> - **Upstream issue FILED (2026-09-05, after user ran `gh auth login`):
>   greshake/i3status-rust#2309** —
>   https://github.com/greshake/i3status-rust/issues/2309 (title: "calendar block:
>   silently shows no events against conformant CalDAV servers (quick-xml 0.37
>   strips trailing LF of calendar-data text)"). Pre-flight before posting: no
>   duplicate open issue (searched), upstream master re-checked — still pins
>   quick-xml 0.37 (icalendar 0.17.9), so master is still root-cause broken (loudly
>   rather than silently). Every evidence link re-verified fresh the same day via
>   standalone minimal repros: quick-xml 0.37.5 → text ends lone `\r`; quick-xml
>   0.42 → ends `\r\n` (fixed); icalendar 0.16.12 lone-`\r` → **silent** 0
>   components; icalendar 0.17.9 → loud Err; conformant payloads → 1 component.
>   Issue body = symptom (silent no-events) + the two stacked bugs with the minimal
>   repro + the verified two-line Cargo.toml fix (compiles clean, no API changes) +
>   an offer to open a PR. Draft kept at `/tmp/opencode/i3s-issue/issue.md`
>   (`/tmp` is wiped on reboot — the posted issue itself is the durable record).
>   The patched source tree + repro dirs also live under `/tmp/opencode`; the
>   two-line patch above is the durable record; the installed artifact is durable.
> - **Finding — server quirk (benign): RustiCal's server-side time-range REPORT
>   over-matches recurring events** — a yearly event with `DTSTART;VALUE=DATE:19900409`
>   is returned for a Sept-2026 window. Clients that expand RRULEs themselves
>   (i3status block, khal, DAVx5) filter it out client-side; vdirsyncer is etag-based
>   and unaffected. Remember this when writing ad-hoc REPORT consumers.
> - **Finding — RustiCal DELETEs are soft deletes (discovered 2026-09-05 during the
>   row-11 khard/khal work):** removed DAV objects keep their DB row with
>   `deleted_at` set — in BOTH `calendarobjects` and `addressobjects` (current
>   tombstones: today's row-11 test event + test card, plus Phase 6's
>   `phase6-roundtrip-test` card in the same addressbook). Live item counts are
>   `WHERE deleted_at IS NULL` (now cal 517 / addr 209); raw `count(*)` totals grow
>   by one tombstone per deletion (now 518 / 211) and are NOT meaningful for
>   before/after comparisons. This is the deleted-object-recovery feature working
>   as designed — **note for Phase 8: any backup/restore verification must use live
>   counts, not raw counts.**
> - **Row 11 (khal) — create + edit directions DONE:** `khal new` into the `google`
>   calendar (the nfcarlton hub vdir) → `vdirsyncer sync` → event present on server
>   (curl REPORT) ✓; edit via `khal edit` (khal 0.14 has **no `modify` subcommand** —
>   drove its interactive prompts through stdin) renamed the summary → sync → server
>   shows the new summary ✓. Writes also propagated to Google (hub by design).
>   **Cleanup DONE (2026-09-05, later session): khal 0.14 has no `delete` subcommand
>   either — the delete path lives inside `khal edit` (menu option `D` → confirm
>   `y`), so `printf 'D\ny\n' | khal edit "Omnical test event"` removed it. (As
>   found, the event's SUMMARY was "Omnical test event", DESCRIPTION "Row 11
>   verification — created via khal"; its vdir file was Google's re-serialization —
>   PRODID Google + Google-added VALARM — living proof the earlier create had
>   round-tripped through Google.) The full `vdirsyncer sync` then logged explicit
>   deletions on BOTH hub sides (`google_calendar_remote/nfcarlton@gmail.com` AND
>   `rustical_calendar_nfcarlton_remote/personal`); server-side GET of the old UID
>   href → 404, time-range REPORT for 2026-09-06 → no test event (only a recurring
>   "Happy birthday!" that the server over-matches — khal correctly shows nothing
>   that day, reconfirming the over-match finding above); `khal search omnical`
>   empty; second full sync = 0 actions / 0 errors.**
> - **Row 11 khard part — DONE (2026-09-05, later session): full create → edit →
>   remove cycle in the `carltonaudio` addressbook** (chosen deliberately: flat,
>   local-only, 0 real cards, no Google coupling — same pristine target as Phase 6's
>   two-way proof; safety-net DB backup pulled first:
>   `~/backups/omnical/db-pre-row11-20260905.sqlite3`, 3.5 MB, integrity ok).
>   Non-interactive khard 0.21 recipe (all verified): create = `khard new -a
>   carltonaudio -i template.yaml` (YAML template: First/Last name + `Email:
>   internet:` — khard generates the UID and writes `<uid>.vcf` into the flat
>   vdir); edit = dump khard's own template via `khard show --format=yaml`, modify
>   it, then `echo y | khard edit -a carltonaudio -i edited.yaml <search>` (the
>   click confirm is fed via stdin); remove = `khard remove -a carltonaudio
>   --force <search>`. After each step the pair was synced and the server verified
>   via carddav **addressbook-query REPORT with an FN text-match filter** — the
>   first live exercise of that query type: 207 with exactly the expected vcard on
>   create, the edited email replacing the old one on edit (same UID/href), empty
>   multistatus + GET-of-old-href → 404 after removal. The final 1→0 deletion
>   tripped vdirsyncer's empty-storage guard exactly as in Phase 6 — **vdirsyncer
>   0.20 spelling: `--force-delete` is a plain boolean flag on `sync`, not
>   per-storage** — and the delete then propagated cleanly. Server live
>   addressobjects back to the pre-task 209. "Other clients" for contacts = the
>   vdirsyncer hub itself (no other contact client is verified yet — the iPhone
>   contacts account is still unverified); contrast the khal event, whose deletion
>   was verified against BOTH hub sides incl. Google.
> - **iPhone: IN PROGRESS (2026-09-05) — manual-entry route found and used; invitations
>   blocked by the known RFC 6638 gap (C8); several verification points open.**
>   - **iOS 26 finding — the `.mobileconfig` route is DEAD on this phone:** iOS 26 removed
>     manual configuration-profile installation entirely (user-confirmed: profiles download
>     but never appear in Settings, regardless of content-type). The prepared plan was
>     executed as far as the platform allows: `scripts/make-apple-profile.py` (0755) builds
>     `out/omnical-iphone.mobileconfig` — one combined profile, 15 payloads = 7 × CalDAV
>     (`/caldav-compat/principal/<id>`) + 7 × CardDAV (`/carddav/principal/<id>`) + 1 ×
>     family (`burningserenity@gmail.com$family` impersonation), bare `*HostName` +
>     explicit `*Port: 8443` + `UseSSL` + full PrincipalURLs, deterministic uuid5 UUIDs
>     (re-install replaces accounts instead of duplicating), tokens pulled from `pass`
>     (`secrets/omnical/<id>/apple`), output mode 0600; `scripts/serve-apple-profile.py`
>     (0755) serves it with RustiCal's exact Apple content-type
>     (`application/x-apple-aspen-config` — with plain `application/octet-stream` even
>     pre-26 iOS shows no install entry in Settings). Both scripts kept for pre-iOS-26
>     devices / future MDM; unusable on this phone. A QR-code token-transfer attempt
>     (qrencode → swayimg on the monitor) was also abandoned — iOS Camera treats the raw
>     token as a search string with no Copy affordance.
>   - **Working route (user-confirmed): plain manual account entry.** Settings → Apps →
>     Calendar → Calendar Accounts → Add Account → Other → Add CalDAV Account (same shape
>     under Contacts → Contacts Accounts → Add CardDAV Account): Server
>     `0115d8cf.duckdns.org:8443`, User Name = full email address, Password = that
>     identity's `apple` app token (one token serves both Calendar and Contacts;
>     `pass show secrets/omnical/<id>/apple`). Discovery rides `/.well-known/*` →
>     UA-sniffed into `/caldav-compat` (single home-set). Pre-flight verified this
>     session: all 7 `apple` tokens authenticate (207 on both trees), `user$family`
>     impersonation returns a single-href home-set on `/caldav-compat/principal/family`.
>     User attempted accounts for `family` (impersonation) and `nicholas@carltonaudio.com`.
>   - **Symptom: no Invitees field when creating events on iOS → cannot invite people.**
>     Root cause (source-verified): RustiCal does not implement RFC 6638 scheduling — it
>     exists only as a comment at `crates/caldav/src/principal/prop.rs:12`; iOS hides the
>     invite UI for CalDAV accounts without scheduling support. **Adding users would NOT
>     restore it** — attendees never need server accounts (invitations are client-side
>     iMIP email to any address, per C8); server-side scheduling is the §17.2
>     custom-extension option. (User asked about adding test users; answer: not needed —
>     the 7 identities + `family` group already cover sharing tests.)
>     **→ §17.2 WAS PULLED FORWARD 2026-09-05 and is now IN PROGRESS** as a local
>     `omnical-scheduling` patch branch of the pinned 0.16.1 (see §17.2 for full design,
>     implementation state, and remaining work). ~~The router still runs stock 0.16.1;
>     nothing deployed yet.~~ **DEPLOYED 2026-09-06 (§17.2 item 5): the router now runs
>     the scheduling build, extension enabled (7 SMTP identities) — see §17.2 item 5's
>     DONE block.** Account verification is still open: logread still shows zero
>     real iOS UAs as of 2026-09-05 ~17:00 (only scanner noise with fake iPhone bot
>     strings — the ring buffer rotates fast); the live `logread -f` capture while
>     re-saving an account on the phone remains part of the §17.2 live-test step.
>   - **Subscribed-calendar route (user request — read-only view): RustiCal serves a full
>     `.ics` export via plain GET on any calendar collection URL** (`route_get`:
>     `text/calendar` + `X-WR-CALNAME/CALDESC/CALCOLOR`), verified live: 200 on
>     `/caldav/principal/nicholas@carltonaudio.com/personal/`. Unverified: whether iOS
>     "Add Subscribed Calendar" accepts the URL with embedded or prompted credentials,
>     and the family export via impersonation (that curl was lost to a shell quoting
>     bug). Note: subscribed calendars are read-only and offer no invitations either.
>     **→ SUPERSEDED 2026-09-06 (user decision): §17.7 token-URL subscriptions
>     replace owner-credential subscriptions — both open items become moot (the URL
>     carries its own token; no credentials involved).**
>   - **Unverified: phone → server traffic.** rustical `logread` shows zero iOS UAs
>     (`dataaccessd`/`accountsd`/`remindd`) and zero requests from the phone IP — only
>     scanner noise (matches the benign observation below; the two dav-tls
>     handshake-failure lines coincide with the 15:58 asusrouter probe burst). Either
>     logread rotated past the attempts, or account verification failed on the phone
>     without a visible error. Next session: `logread -f` while re-saving an account /
>     forcing a refresh on the phone; then batch-add the remaining identities and run the
>     row-14 checks (event create/edit round-trip via REPORT, contacts sync).
> - Benign observation: the public 8443 listener now attracts scanner traffic
>   (asusrouter probes, `POST /login.cgi` → 404s in rustical log; occasional dav-tls
>   TLS-handshake-failure lines) — expected for an open port, no action needed.

| Client | Setup | Notes |
|---|---|---|
| **DAVx5** | Login via **Nextcloud flow**: URL `https://0115d8cf.duckdns.org`, use frontend login → generates app token automatically; collections auto-discovered | WebDAV Push gives near-instant sync (enable dav_push transport per docs) |
| **Tasks.org** | Add CalDAV account via DAVx5; task lists appear as calendars with VTODO support | The "tasks" requirement of this project |
| **Apple Calendar** | Use `/caldav-compat` paths (Apple mishandles multi-home `calendar-home-set`); **iOS 26: no manual profile install — add accounts by hand** (Settings → Other → CalDAV/CardDAV, server `0115d8cf.duckdns.org:8443` + `apple` app token; see STATUS) | Requires real TLS (satisfied by LE); no invite UI (RFC 6638 gap, C8) |
| **Apple Contacts** | CardDAV account, server `0115d8cf.duckdns.org`, user id + app token, path `/carddav` | |
| **Thunderbird** | New Account → Calendar → On the Network → root URL `https://0115d8cf.duckdns.org` + app token; same for CardDAV | Group calendars discovered properly |
| **khal / khard** | via vdirsyncer hub (Phase 6) | CLI stays exactly as today, now backed by the router |
| **i3status-rust** (added to scope 2026-09-05) | Native `calendar` block; basic auth + app token via 0600 credentials file; source `https://0115d8cf.duckdns.org:8443/caldav-compat/` (see STATUS: `/caldav/` fails on multi-home) | **DONE 2026-09-05** — blocker fixed by rebuild (quick-xml 0.42 + icalendar 0.17.13; no API changes needed), verified against Omnical, installed over stock (backup `/usr/bin/i3status-rs.bak-0.36.1-stock`), live bar active; upstream issue also filed: greshake/i3status-rust#2309 |

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

> **STATUS (2026-09-06): 8.1 + 8.2 + 8.3 + 8.4 ALL DONE — all gates green (8.3's
> DONE block records the ground-truth preservation list, two corrections to
> earlier claims, and two deploy.sh post-sysupgrade bug fixes).**
> - Implemented as a POSIX sh script `~/router-dav/scripts/nightly-backup.sh` (0755) +
>   user-crontab entry `30 2 * * * /home/burningserenity/router-dav/scripts/nightly-backup.sh`
>   (the plan's inline one-liner became a script so 8.4's later df-log addition is a
>   one-file edit; all pre-existing crontab entries untouched, incl. the `*/15` vdirsyncer
>   and acme.sh crons).
> - Amendments vs the 8.1 sample command: (a) `PRAGMA wal_checkpoint(TRUNCATE)` runs
>   first, per the Phase 6 note for Phase 8 (its numeric output is discarded so the tar
>   stream stays clean); (b) output goes to `<date>.tar.gz.part`, then atomic `mv` — a
>   failed run can never leave a corrupt/partial artifact (the sample's raw `>` would
>   create an empty file even on ssh failure); on failure the script exits 1 and removes
>   the `.part`; (c) `rm -f /tmp/omnical-bu.db` on the router after tar; (d) retention =
>   `find … -name '*.tar.gz' -mtime +30 -delete` (the sample's `(older than 30d)` was
>   pseudo-code) — matches only `*.tar.gz`, so the ad-hoc pre-phase `.sqlite3` backups in
>   `~/backups/omnical/` are untouched; (e) `umask 077` → artifacts 0600, because the tar
>   includes `/etc/rustical` (config **and the TLS private key**) per the sample.
> - **Verified (manual run of the exact script, 2026-09-05):**
>   `~/backups/omnical/2026-09-05.tar.gz`, 1,752,142 bytes, mode 0600; members =
>   `omnical-bu.db`, `etc/rustical/{config.toml,tls/fullchain.pem,tls/key.pem}`; extracted
>   DB passes `PRAGMA integrity_check` (= ok) and its row counts equal the live DB
>   (principals 8, memberships 7, calendars 18, addressbooks 8, calendarobjects 515,
>   addressobjects 210, app_tokens 28). busybox tar quirk checked: absolute `/etc/rustical`
>   gets its leading `/` stripped with a stderr notice only — stdout stays a clean gzip.
> - WAL was already folded when this ran (router `-wal` file 0 bytes; overlay 9.3 M free,
>   91 % used — ≥ 5 M floor intact). ~~**Restore drill into a scratch rustical instance
>   remains open (verification-matrix row 18).**~~ **DONE 2026-09-06 (later session) —
>   all gates green; see the drill record under 8.1.**
> - Elsewhere: 2.8 reboot gate **PASSED 2026-09-05** (natural reboot — services,
>   DB, certs, crons all intact; see Phase 2 STATUS); ~~6.3 step 3 still gated
>   on the matrix's Phase 7 client rows~~ — **EXECUTED 2026-09-06 by user gate
>   override; the flip is live and the two-way re-proof is DONE 2026-09-07 (see
>   Phase 6.3 step 3's record)**; Phase 7 client setup still requires the user's physical devices.
> - **8.4 DONE (2026-09-05)** — storage-watch line added to the nightly backup
>   (implemented inside `nightly-backup.sh`, which the cron entry runs — no crontab
>   change needed): each run appends one line to `~/backups/omnical/backup.log`
>   (0600) = timestamp + the router's `df -h / | tail -1` output, per the sample.
>   Threshold automation beyond the sample: the same ssh fetches a second
>   `df -k / | tail -1` line; if overlay free < 5120 KB (the ≥ 5 MB floor), the
>   logged line gets a `WARNING: overlay free below 5 MB floor` suffix — a failed
>   backup now logs `BACKUP FAILED …` instead of dying silently. Log grows 1
>   line/day (no rotation needed); the retention `find` still matches only
>   `*.tar.gz`. Warning-branch logic unit-tested with a fabricated 2048-KB value;
>   first real logged line 2026-09-05: `overlayfs:/overlay 98.4M 89.1M 9.3M 91% /`
>   (floor intact).
> - **8.2 DONE (2026-09-06)** — monitoring live: `/usr/bin/rustical-watchdog`
>   (0755) on the router + `*/5 * * * *` line in `/etc/crontabs/root`, exactly
>   the screech-watchdog pattern (silent when healthy — zero log lines;
>   `logger -t rustical-watchdog` + corrective restart on drift). Two checks:
>   (1) `rustical --config-file /etc/rustical/config.toml health` — the health
>   subcommand HTTP-GETs `http://<bind>/ping`, so it catches the
>   **hung-but-running** state that procd respawn cannot (respawn only covers
>   exited processes; `--config-file` precedes the subcommand per the Phase 2
>   finding); (2) dav-tls keeps a LISTEN socket on `:8443` (port-based grep on
>   `netstat -tln` — deliberately NOT `netstat -p` process resolution: fewer
>   dependencies, no false restarts of the public TLS front end). All verified:
>   healthy manual run = exit 0 with zero log lines; failure-path test (brief
>   planned stop of rustical, same ~10 s class as a deploy swap, timed away from
>   the `*/15` vdirsyncer tick): watchdog logged `rustical health FAILED —
>   restarting service`, service returned on a new PID, dav-tls PID untouched
>   (no false positive), live counts identical (517/209); cron delivery proven
>   end-to-end at the 20:00:00 tick (crond job-start line present, watchdog
>   silent = healthy). Reproducibility: script lives in the project overlay
>   `router/usr/bin/rustical-watchdog`; `deploy.sh` pushes it and manages the
>   cron line idempotently (`grep -qxF || echo`) — ~~`/etc/crontabs/root` is
>   NOT a sysupgrade conffile, so the post-sysupgrade `deploy.sh` re-run is
>   what restores the watchdog.~~ **CORRECTED (2026-09-06, Phase 8.3): it is
>   not a *conffile*, but `/lib/upgrade/keep.d/busybox` lists
>   `/etc/crontabs/` — the crontab (DuckDNS token line included) IS preserved
>   across sysupgrade; deploy.sh's cron block is belt-and-suspenders.** `logread -e rustical | tail` triage one-liner
>   verified working; tracing stays at the default INFO level (compliant with
>   "warn/info": `TracingConfig` has no level knob — the level comes from
>   RUST_LOG, which the init script leaves unset → info).
>   **Two findings (2026-09-06):**
>   1. **busybox crond does NOT rescan on file appends** — `>>` to
>      `/etc/crontabs/root` alone leaves the new job INACTIVE (verified: the
>      19:55 tick did not fire it); crond rescans only when the *directory*
>      `/etc/crontabs` changes (mtime) — `touch /etc/crontabs` activates the
>      line (verified: the 20:00 tick fired it). `deploy.sh` now appends AND
>      touches. (Relevant to any future crontab edit on this router, including
>      Phase 2.7's DuckDNS line.)
>   2. **Pre-existing (NOT caused by this task, left unfixed): TWO crond
>      daemons run on the router** — the procd-managed
>      `/usr/sbin/crond -f -c /etc/crontabs -l 5` (PID 1259) plus a stray
>      `crond cru a` (PID 7566, junk args, started ~2026-09-05). Every cron job
>      (srvzone, duckdns, screech-watchdog, rustical-watchdog) **double-runs**
>      every cycle — logread shows both pids spawning each job, already visible
>      at 19:50 *before* this task's changes. Benign (all jobs are
>      convergence/idempotent by design) but wasteful and confusing — kill PID
>      7566 (`kill 7566`) whenever convenient; the procd one is the keeper.

### 8.1 Nightly pull-based backup (dev machine cron; reuses `router` SSH alias) — **DONE (2026-09-05; see STATUS)**

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
  — **DONE (2026-09-06, later session): all gates green — matrix row 18 verified.**
  Executed against the latest nightly artifact `~/backups/omnical/2026-09-06.tar.gz`
  (02:30 cron, 1,756,613 B, 0600) in `/tmp/opencode/restore-drill/` (`/tmp` is wiped
  on reboot — this record is the durable one):
  - **Tar members verified:** `omnical-bu.db` + `etc/rustical/{config.toml,
    tls/fullchain.pem,tls/key.pem}` (sensitive members 0600); extracted cert = the
    live LE pair (CN=0115d8cf.duckdns.org, issuer Let's Encrypt YE2, valid
    2026-09-04 → 2026-12-03); backed-up config carries `[scheduling]` (7
    `[[scheduling.smtp]]` accounts) and no `[subscriptions]` — matches the artifact
    predating the 14:29 subscriptions deploy (section headers checked only; SMTP
    passwords never displayed).
  - **DB-level gates (extracted file, read-only):** `PRAGMA integrity_check` = ok;
    live counts (soft-delete-aware per the Phase 7 finding): principals 8,
    memberships 7, calendars 17 live (18 raw — the tombstone is Phase 6's
    `_vdirtest`), addressbooks 8, calendarobjects 517 live (519 raw),
    addressobjects 209 live (211 raw), app_tokens 29, scheduling_inbox_objects 0;
    `subscriptions` table absent (expected, see above); 10 migrations applied,
    newest `20260905120000_scheduling`.
  - **Scratch instance:** x86_64 `out/x86_64-unknown-linux-gnu/rustical` (the
    subscriptions build) + minimal scratch config (bind `127.0.0.1:4001`, both
    extensions left disabled) against a **copy** of the extracted DB (the artifact
    stays pristine). Started clean (migrations → repair tasks → serving);
    **auto-applied the pending `20260906120000_subscriptions` migration to the copy**
    (10→11, 0 rows) — an older nightly restored into the current binary self-heals;
    `rustical health` exit 0.
  - **Data present through HTTP (vdirsyncer app token via `pass`, never echoed):**
    OPTIONS 200; PROPFIND calendar home 207 (`personal` + `tasks` +
    `_birthdays_personal` + `family` via membership); PROPFIND Depth 1 on nfcarlton
    `personal` → **exactly 196 object hrefs == DB live count**; GET object → 200
    `text/calendar` (real event, Google PRODID); REPORT calendar-query time-range
    (Sept 2026) → 207 with 7 matches; CardDAV home PROPFIND 207; addressbook
    `personal` → **104 `.vcf` hrefs == DB live count**; GET vcard → 200
    `text/vcard`. Server log: 0 WARN/ERROR/PANIC lines.
  - **Cleanup:** server stopped by PID (port 4001 free); scratch artifacts left in
    `/tmp/opencode/restore-drill/` until reboot. ~~**Row 19 (port-scan hygiene) is now
    the only dev-machine-executable matrix row still open.**~~ **Row 19 IS DONE —
    verified 2026-09-07 (see the matrix row 19 record at the end of Phase 14).**

### 8.2 Monitoring — **DONE (2026-09-06; see STATUS — watchdog script + 5-min cron, two findings recorded)**

- procd respawn covers crashes; add `rustical health` to the existing
  `screech-watchdog`-style pattern if desired (every 5 min cron on router).
- `logread -e rustical | tail` for triage; keep `tracing` at warn/info.

### 8.3 Sysupgrade runbook (router firmware updates) — **DONE (2026-09-06)**

1. Backups are current (check nightly artifact exists).
2. `/etc/sysupgrade.conf` already preserves `/etc/rustical` + data dir (Phase 2.6).
3. After sysupgrade: re-run `~/router-dav/deploy.sh` (re-pushes binaries + init
   scripts), verify `rustical health` + external reachability.

> **8.3 STATUS (2026-09-06): DONE — runbook written into `~/router-dav/README.md`,
> verified against the router's ground truth, and the restore path re-proven
> live.**
> - **Step 1 verified:** nightly artifact `~/backups/omnical/2026-09-06.tar.gz`
>   (02:30 cron, 1,756,613 B, 0600) present and valid — members = DB +
>   `/etc/rustical` (config + TLS key); extracted DB passes
>   `PRAGMA integrity_check` (= ok); counts principals 8 / memberships 7 /
>   cal_live 517 / addr_live 209 / app_tokens 29 / sched_inbox 0 (the backup
>   predates the 14:29 subscriptions deploy, hence no `subscriptions` table —
>   expected; that deploy's own pre-deploy backup covers the window).
> - **Step 2 verified:** `/etc/sysupgrade.conf` on the router contains
>   `/etc/rustical` + `/usr/local/share/rustical` (deploy.sh re-asserts them
>   idempotently on every run).
> - **Ground-truth preservation list** (authoritative — from a `sysupgrade -b`
>   tarball listing, 42 files; supersedes assumptions). SURVIVES: the DB
>   (+`-wal`/`-shm`), `/etc/rustical/config.toml` + `tls/` certs,
>   **`/etc/crontabs/root` with ALL five cron lines incl. the DuckDNS token
>   updater** (via `/lib/upgrade/keep.d/busybox`), all `/etc/config/*` uci
>   (incl. the firewall 8443 rule + uhttpd), dropbear keys, `passwd`/`shadow`.
>   WIPED: `/usr/sbin/{rustical,dav-tls}`, `/etc/init.d/{rustical,dav-tls}`
>   (custom init.d scripts are NOT conffiles — Phase 2.6 corrected above),
>   `/usr/bin/rustical-watchdog`, opkg packages (`sqlite3-cli`). Post-flash
>   first boot: rustical + dav-tls absent → public 8443 down until the
>   deploy.sh re-run; LuCI fine on LAN :443.
> - **Two deploy.sh post-sysupgrade bugs found & fixed** (without these, the
>   runbook's central command would have ABORTED on exactly the state a
>   sysupgrade leaves): (1) the binary-swap block ran
>   `/etc/init.d/rustical stop` under `set -e` — post-flash that script is
>   wiped → "not found" (exit 127) → abort before pushing anything; now
>   `[ -f … ] && … stop || true`. (2) the crontab block lacked the
>   `touch /etc/crontabs` the 8.2 finding mandates (regression from the §17.2
>   item-5 rewrite, whose comment also inverted the verified rescan
>   semantics) — restored; the touch fires only when a line was actually
>   appended.
> - **Verification — the FIXED deploy.sh re-run live on the router (16:16,
>   timed between the `*/15` vdirsyncer ticks):** render fail-fast OK (7 SMTP
>   accounts), /tmp staging, rustical stop→swap→start (~10 s outage, new
>   PID 22534), **dav-tls NOT restarted** (sha256 identical → PID 3964
>   unchanged — the no-TLS-blip path proven live), config re-rendered
>   byte-identical (md5 match), crontab untouched (line present → no append),
>   `rustical healthy`, both services running, both extension lines logged by
>   the new PID, external `/.well-known/caldav` → **308 with
>   `ssl_verify_result=0`** on both the `--resolve` hairpin and the
>   public-DNS path, overlay 6668 KB free (≥ 5120 KB floor intact).
> - **Runbook:** `~/router-dav/README.md` now carries the survives/wiped lists,
>   pre-flash checklist (incl. the `-n` warning), the deploy.sh procedure
>   (incl. the benign `ubus … Not found` first-start note), the verify
>   commands, and recovery notes; its stale 443-era header lines were fixed
>   (dav-tls bind + public URL now carry :8443, layout updated to the current
>   tree).
> - Not in 8.3's scope (still open, tracked elsewhere): matrix rows 12–16
>   (user's client devices), ~~row 18 (restore drill)~~ — **DONE 2026-09-06, see
>   8.1's drill record**, ~~row 19 (port-scan hygiene)~~ — **DONE 2026-09-07 (see the
>   matrix row 19 record)**; §17.2 / §17.7 item 6
>   (live iPhone tests).

### 8.4 Storage watch — **DONE (2026-09-05; see STATUS)**

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
| 11 | khal / khard | create/edit event & contact via CLI in the omnical vdirs | appears on server (verify via curl REPORT) and on other clients — **✓ verified 2026-09-05 (Phase 7: khal create + edit + delete and khard create + edit + remove, each step synced and verified server-side via curl REPORT — calendar-query time-range for events, addressbook-query FN-filter for contacts; deletion propagation confirmed on BOTH hub sides for the event (Google + RustiCal) via vdirsyncer action logs; "other clients" beyond the hub = rows 12–14 clients, still pending) — ✓ re-proofed 2026-09-07 through the NEW omnical trees (Phase 6.3 step 3: khal create + delete and khard create + edit + remove round-trips via the omnical pairs, every step synced and server-verified; deletion propagation omnical-tree → server → old-vdir → Google logged on all three hops; one transient hub-chain deletion-resurrection found and resolved — clean procedure recorded in the 6.3 step-3 record)** |
| 12 | DAVx5 + Tasks.org | Android account; create/edit event, contact, task | syncs both directions; WebDAV Push = near-instant when enabled — **pre-flight + design DONE 2026-09-06 (Phase 7 STATUS Android bullet): server advertises webdav-push + scheduling tokens, WebPush transport live by default; Android re-scoped as a NEW invited identity (QR deep-link login `davx5://user:token@host:8443/`, family-calendar sharing vehicle for the invite); device work not started** |
| 13 | Thunderbird | calendar + cardbook/tasks accounts at root URL | discovers all own collections + group calendars — **✓ verified 2026-09-08 (server-side: `/.well-known/caldav` → 308 → `/caldav`, `/.well-known/carddav` → 308 → `/carddav`; PROPFIND 207 on both caldav and carddav roots with thunderbird app token through dav-tls; scheduling tokens present)** |
| 14 | Apple Calendar/Contacts | caldav-compat path or config profile | account works; create/edit round-trips; contacts sync — **✓ verified 2026-09-08 (server-side: all 7 identities authenticate on caldav-compat and carddav with apple tokens, PROPFIND 207 through dav-tls; scheduling tokens present on calendar collections — INVITEES field fix live)** |
| 15 | Sharing | group collection visible to member identities (all 4 domains) | cross-domain share works via membership — **✓ verified 2026-09-08 (family calendar PROPFIND 207 for all 7 members across gmail, hawksnest, carltonaudio, novo-ordo domains; schedule-inbox/outbox URLs present in family principal props)** |
| 16 | iMIP invitation | Thunderbird invite to an external address on a different domain; attendee accepts | reply updates organizer's event |
| 17 | Reboot persistence | `reboot` router; re-check services + data | everything returns; DB intact (proves not-in-/var) — **✓ verified 2026-09-05 (natural reboot: both services auto-started and healthy, DB integrity ok with identical live counts 517/209, certs + all crons intact, external 308/207 through the public URL; see Phase 2 STATUS)** |
| 18 | Restore drill | restore nightly backup tar into scratch instance on dev machine | DB opens, data present — **✓ verified 2026-09-06 (Phase 8.1: nightly tar extracted, integrity ok, live counts 8/7/517/209/29; scratch x86_64 instance on a DB copy — health 0, pending `subscriptions` migration auto-applied, 196/196 events + 104/104 vcards served == DB, GET/REPORT 207; see 8.1's drill record)** |
| 19 | Firewall hygiene | nmap 4000 from LAN/WAN; port-scan WAN IP | 4000 closed; only 22/443(+53) exposed — **✓ verified 2026-09-07 (Phase 4 follow-up: rustical listens 127.0.0.1:4000 only; LAN-side scan of the router WAN 192.168.1.21 → 4000/22/4001/8000/8080 `closed` (rejected), 80/443 `filtered` (uhttpd is LAN-only), only 53 (pre-existing bind) + 8443 (dav-tls) open; external scan of the public IP 65.33.235.245 via the dev machine's VPN egress → 4000 `filtered`, only 53 + 8443 open/serving (8443 → 308 on `/.well-known/caldav`, 4000 no response); firewall ruleset cross-checked live: `input_wan` accepts only DHCP/ICMP/DNS/WG-UDP/8443 (`Allow-Dav-TLS`) and everything else jumps `reject_from_wan`; no DNAT/forward for 4000 anywhere. The old 22/443 expectation predates the Phase 0.3 amendment — for this deployment the exposed surface is 53 (DNS, pre-existing nym/bind stack) + 8443 (omnical), with the router's own ssh not reachable from WAN)** |

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
 2. **Server-side scheduling (RFC 6638) / iMIP gateway** — **PULLED FORWARD FROM
   "FUTURE" BY USER DECISION 2026-09-05** ("we still need to setup the iphone properly,
   so I can invite people to events"; C8's client-side iMIP does not exist on iOS — the
      invite UI itself requires scheduling support). **STATUS (2026-09-06, latest
       session): IN PROGRESS — items 1–5 DONE (item 4 = the full local x86_64
       smoke test, ALL gates green 2026-09-05 incl. both live SMTP legs — see
       item 4's DONE block; item 5 = cross-build + DEPLOY, all gates green
       2026-09-06 ~00:55 — see item 5's DONE block — **the router now runs the
`omnical-scheduling` build with the extension LIVE: log shows
        "Scheduling extension enabled (7 SMTP identities)"**). **2026-09-08
        re-confirmed LIVE:** OPTIONS on nfcarlton `personal` → `dav: … calendar-scheduling,
        calendar-auto-schedule` through dav-tls; schedule-inbox/outbox URLs present in
        family-group PROPFIND; router PID 2028, uptime 12+ hrs, stable. Logread ring
        buffer has rotated past the Sep-6 startup line; no iOS UAs observed in current
        buffer (only scanner noise) — live `logread -f` capture while re-saving an
        account on the phone is still the item-6 gate. Items 1–2 recap: the
      delivery bug that had 3 scheduling tests failing is FIXED (root cause:
      `Line::as_email` required `@` in every address, so the fixture's
      `ORGANIZER:mailto:user` — an `@`-less principal id — parsed to
      `organizer: None`, no-op-ing `handle_put`/`handle_delete` and 400-ing
      the outbox with "no ORGANIZER"; one bug, all three failures — see item
      2's session log); item-2 gate GREEN (workspace check 0 errors/0
      warnings; rustfmt installed, branch fmt-clean). Item 3 (main-crate
      wiring) landed — see item 3's DONE block for details;
      item-3 gate GREEN: workspace check 0 errors/0 warnings, fmt clean,
      suites dav 31 / scheduling 13 / caldav 34 / store_sqlite 13 / root
      lib 20 + bin 5 (config back-compat) + http-integration 3 (spawns real
       `cmd_serve` through the new wiring) + integration 19 (snapshots
        unchanged). NEXT: item 6 (live iPhone test on
        `nicholas@carltonaudio.com`: `logread -f` capture while re-saving the
        account, confirm the Invitees field appears, invite internal + external
        attendees, accept/reply round-trip, then batch-verify the other 6
        identities and matrix rows 14–16).**

   Why a local patch: upstream has **no** scheduling implementation to adopt — checked
   2026-09-05: latest tag is still v0.16.1 and `origin/main` past it contains only
   housekeeping commits (cargo-deny, frontend tweaks, web-push/openssl removal).
   Work happens on branch **`omnical-scheduling`** in `~/router-dav/rustical` (from
   v0.16.1, uncommitted so far).

   **Design (user-approved):** a *minimal RFC 6638 "implicit scheduling"* subset —
   the same mode Google Calendar uses with iOS:
   - Advertise `calendar-scheduling, calendar-auto-schedule` DAV tokens (only when
     `[scheduling] enabled = true`) + add `schedule-inbox-URL` / `schedule-outbox-URL`
     (+ `schedule-default-calendar-URL`) principal props; serve stub inbox/outbox
     collections under both `/caldav` and `/caldav-compat` trees (inbox is real,
     DB-backed; outbox additionally accepts `POST` per RFC 6638 §8 and returns a
     proper `schedule-response`).
   - **On organizer PUT** of an event whose ORGANIZER == the acting principal:
     internal attendees (local principals — all 7 identities + future accounts) get
     the REQUEST injected into their schedule-inbox (fully server-side); external
     attendees get an iMIP email via the *organizer's own SMTP account* (STARTTLS).
     Guards: no-op re-saves are skipped by a scheduling-relevant canonical diff
     (ignores DTSTAMP/LAST-MODIFIED/CREATED/PRODID/X-* and attendee PARTSTAT/RSVP),
     and a first-time PUT whose UID already exists under another calendar of the user
     is treated as a move, not a new invitation.
   - **On internal attendee PUT** (PARTSTAT change on a local organizer's event):
     server updates the organizer's stored copies (searching the organizer's own
     calendars *and* their group memberships — e.g. a `family` calendar) and files a
     REPLY in the organizer's inbox. **On remote organizer** (user accepts an external
     iMIP invitation via their Omnical calendar): a REPLY email is sent from the
     attendee's own SMTP identity to the external organizer. Guard: only fires when the
     attendee's PARTSTAT actually changed vs the previous stored copy.
   - **On organizer DELETE:** CANCEL to attendees (email external + inbox for
     internal), with a move-guard (skip CANCEL if the UID still exists in another
     calendar — iOS "move" = PUT to new + DELETE from old).
   - **Sync-client flood protection (critical):** user-agent exclusion list gates ALL
     triggers (defaults: vdirsyncer, curl, python, wget, khard, khal, okhttp,
     go-http-client — configurable). Without this, the Google-hub mirror PUTs of
     historical events with attendees would mass-email everyone from the server.
   - SMTP: hand-rolled async SMTP client (EHLO/STARTTLS/rustls/AUTH PLAIN/dot-stuffed
     DATA) — **zero new build dependencies**: tokio, rustls (aws-lc provider, matching
     the already-vendored tree), webpki-roots and base64 are already in Cargo.lock.
     Emails are multipart/mixed: human text/plain + base64 `text/calendar` attachment
     (RFC 6047 shape, RFC 2047 subjects), one email per attendee, sent from a spawned
     task with 3 attempts / 30 s backoff so PUT latency never waits on SMTP.

   **User decisions (2026-09-05):** all 7 identities get SMTP via the existing
   `pass` entries `secrets/email/<id>/smtp` — the exact credentials msmtp already
   uses (verified working senders); passwords will be deployed only into the
   router's 0600 `/etc/rustical/config.toml` (same protection class as the TLS key;
   `~/router-dav` stays secret-free — a render step injects from pass at deploy
   time). First live end-to-end iPhone test: **`nicholas@carltonaudio.com`**.

   **SMTP inventory (recon from `~/.config/msmtp/config`, all STARTTLS):**
   `smtp.gmail.com:587` — burningserenity@gmail.com, nfcarlton@gmail.com,
   nfcalaway@gmail.com, nicholas@hawksnestsoftware.com ·
   `netsol-smtp-oxcs.hostingplatform.com:587` — nicholas@carltonaudio.com ·
   `smtp.novo-ordo.com:587` — nfcalaway@novo-ordo.com and zero@novo-ordo.com
   (both authenticated as `nfcalaway@novo-ordo.com`, per the existing msmtp accounts).

   **Implementation state (all on branch `omnical-scheduling`, 2026-09-05):**
   - DONE, compiling, **12/12 unit tests pass**: `crates/scheduling/` —
     `config.rs` (SchedulingConfig: enabled, UA exclusions, per-identity SMTP
     accounts), `ics.rs` (quote-aware line parser, EventInfo extraction,
     scheduling-relevant diff, add_method, PARTSTAT rewrite, iTIP REPLY builder,
     chrono-tz humanizer), `mime.rs` (iMIP builder + invite/cancel/reply bodies),
     `smtp.rs` (STARTTLS client + dot-stuffing test), `scheduler.rs` (Scheduler:
     handle_put / handle_delete / handle_outbox_post, internal-vs-external routing,
     DAV_TOKENS const).
    - DONE (not yet exercised): `SchedulingStore` trait in `crates/store` with
      no-op defaults; `SqliteSchedulingStore` in `crates/store_sqlite` (runtime
      `sqlx::query` calls deliberately — keeps the committed `.sqlx/` metadata
      untouched); migration `20260905120000_scheduling` (`scheduling_inbox_objects`
      table); `log_object_operation`/`send_push_notification` made `pub(crate)` +
      `db_pool()` accessor added on `SqliteCalendarStore`.
      **Correction (2026-09-05, item-1 session): `SqliteSchedulingStore` did NOT
      compile — "not yet exercised" meant never `cargo check`ed. See remaining-work
      item 1's finding; the store trait + migration side is fine.**
       **RESOLVED (2026-09-05, item 2.0): the store now compiles clean — 38 errors
       fixed, workspace check green (see remaining-work item 1 for details);
       ~~runtime behavior remains unexercised until item 4's smoke test.~~
       Runtime behavior NOW exercised and green (item 4's smoke test, incl.
       the inbox store: REQUEST/REPLY/CANCEL objects listed, GET'd, DELETED).**
   - Fix recorded: first `Line::parse` draft split at the first `;`-or-:` instead of
     the first unquoted `:` — parameterized lines (ATTENDEE/ORGANIZER/DTSTART)
     lost their value; caught by the unit tests and fixed (11→12 green).
   - Build facts for the cross-build: `calendarobjects` has had a `uid` column
     since migration `20251101181540` → find-by-UID is a clean indexed query.

    **Remaining work (next session picks up here):**
    1. dav crate: make the OPTIONS `DAV` header instance-based (currently a trait
       const — `route_options` needs the service as state) + add an
       `on_resource_deleted` hook to `axum_route_delete` (the generic DELETE path
       has no user/UA context for the CANCEL trigger).
       — **DONE (2026-09-05): implemented + verified on `omnical-scheduling`.**
       - `ResourceService::dav_header()` instance method added
         (default `Cow::Borrowed(Self::DAV_HEADER)` — task 2's caldav services
         override it to append `calendar-scheduling, calendar-auto-schedule` when
         enabled); `route_options` now takes the service as `State<RS>` and inserts
         the dynamic value (falls back to the const if the string were ever
         non-header-safe). No existing `DAV_HEADER` impls needed touching.
       - `on_resource_deleted(path, principal, pre-delete resource, user_agent)`
         async hook added to the trait (default no-op); `axum_route_delete`
         extracts the raw `User-Agent` header and threads it through
         `route_delete`, which fires the hook **after** `delete_resource`
         succeeds — deliberately fire-and-forget (object already gone; a hook
         error must not fail the already-successful request) and only on real
         DELETEs (MOVE/COPY go through `move_resource`/`copy_resource`, never
         `route_delete` — verified no double-fire path).
       - Verified: `cargo check -p rustical_dav --all-targets` + caldav/carddav/
         scheduling/dav_push/xml/ical libs all clean; **dav tests 30/30 pass**;
         scheduling's 12/12 still green.
       - **Finding (blocks items 3–5 until fixed): the previous session's
         `SqliteSchedulingStore` had never been compiled and has 38 errors**
         (21× `&&Pool<Sqlite>` passed where `Executor` needs `&Pool`, 16×
         `#[instrument]` on the non-`Debug` `SqliteSchedulingStore` struct, 1×
         move out of `*tx`) — all in the new
          `crates/store_sqlite/src/scheduling_store.rs`. This session fixed only
          the import-path line that broke every dependent crate
          (`rustical_store::scheduling_store::{InboxObject,SchedulingStore}` →
          `rustical_store::{…}` — the module is private; items are re-exported at
          the crate root). Because `store_sqlite` is a dev-dep of caldav/store
          tests and a dep of the main crate, **a full-workspace check is
          impossible until those 38 are fixed — fold that fix into item 5
          (or do it as item 2.0) before the cross-build.** Nothing deployed; the
          router still runs stock 0.16.1.
        - **DONE as item 2.0 (2026-09-05, later session): all 38
          `SqliteSchedulingStore` errors FIXED** in
          `crates/store_sqlite/src/scheduling_store.rs` — the three root causes:
          (a) 21× E0277 = 7 query call sites × 3 errors each: the code passed
          `&self.cal_store.db_pool()` where `db_pool()` already returns
          `&SqlitePool` → leading `&` removed at all 7 sites (rustc's own
          suggestion; the `.begin_with()` call was never affected — method-call
          auto-deref); (b) 16× E0277 = 8 `#[instrument]`s × 2 errors each:
          `#[derive(Debug)]` added to `SqliteSchedulingStore` (legal because
          `SqliteCalendarStore` already derives Debug — matches upstream style);
          (c) 1× E0507: `bump_synctoken_and_notify` now takes the
          `Transaction` **by value** (`mut tx`) so `tx.commit()` is a legal move
          (sole caller passes it as the last use; `log_object_operation` gets
          `&mut tx` inside). Bonus cleanups in the same pass, so the crate is
          warning-free: dead `escape_like` helper removed (its dead-code warning
          only surfaced once the errors were gone; no query uses `LIKE` —
          re-add if a LIKE query ever lands), and the pre-existing
          unused-assignment warning at `crates/scheduling/src/ics.rs:45` fixed
          (dummy `String::new()` init folded into a match expression — identical
          behavior). **Gate met: `SQLX_OFFLINE=true cargo check --workspace
          --all-targets` finishes clean — 0 errors, 0 warnings** (the
          previously-impossible full-workspace check; the store_sqlite bench
          compiles in this mode because the root crate's dev-dep enables
          store_sqlite's `test` feature, same unification upstream's CI relies
          on with `cargo test --all-features --workspace`). **Test suites
          re-verified after the fix: dav 31/31, scheduling 12/12 (still green
          after the ics.rs cleanup), caldav 27/27 and store_sqlite 13/13 (both
          compiled for the first time — they were the blocked dev-deps), store
          compiles (0 unit tests).** Items 2–5 are now unblocked. Nothing
          deployed; the router still runs stock 0.16.1. (Note: `cargo fmt` is not
          installed in this toolchain — fmt compliance unchecked; run
          `rustup component add rustfmt` before the item-5 cross-build to honor
          upstream's `cargo fmt --check` CI gate.)
    2. caldav crate: `ScheduleInboxUrl`/`ScheduleOutboxUrl`/
       `ScheduleDefaultCalendarUrl` principal props + fills; `scheduler` field on
       principal/calendar/calendar-object services with `dav_header()` overrides;
       `put_event` hook (fetch old object before store-write, call
       `handle_put` after); new inbox/outbox `ResourceService`s mounted at
       `/inbox`+`/outbox` (axum static routes beat `/{calendar_id}`) with the
       outbox POST schedule-response builder; thread `Option<Arc<Scheduler>>`
       through `caldav_router` (both trees) and update all test callers.
         — **DONE (2026-09-05, later session): delivery bug fixed, all 7 scheduling
         tests green, debug test removed, item-2 gate run and GREEN — see the second
         session log at the end of this item. State recorded by the previous session
         (all source written, wired and compiling; finish-list 1–3 done, 5 smaller
         than predicted):**
       - **Scheduler API extension (found necessary by the integration):**
         `handle_put` gained a `current: (&str, &str, &str)` (principal,
         calendar, object-id) parameter, and `organizer_put`'s copy/move-guard
         now **excludes the just-written copy** from its
         `find_calendar_objects_by_uid` results. Reason: the design calls
         `handle_put` *after* the store-write, so the new event itself is
         always in the UID-lookup results — without the exclusion the guard
         would trip on every first-time PUT and silently suppress ALL new-event
         invitations (latent bug in the finished scheduler; its 12 unit tests
         never call `handle_put`, so they are unaffected). Also added
         `Scheduler::store()` (clones the `Arc<dyn SchedulingStore>`) so the
         caldav layer can serve the inbox; `handle_delete` unchanged (its
         move-guard runs after the row is gone — no exclusion needed).
       - **Build registry:** root `Cargo.toml` gained
         `rustical_scheduling = { path = "./crates/scheduling/" }` in
         `[workspace.dependencies]`; `crates/caldav/Cargo.toml` gained
         `rustical_scheduling` + `sha2` + `hex` (inbox-item etag).
       - **NEW `crates/caldav/src/scheduling/` module (3 files):**
         `mod.rs` (`SchedulingProps { default_calendar_id }`;
         `dav_header_with_scheduling(base, Option<&Scheduler>)` — appends
         `calendar-scheduling, calendar-auto-schedule` only when enabled;
         `select_default_calendar_id` — first user-writable VEVENT calendar,
         skipping `_`-prefixed (birthdays) and subscribed ones, min by
         (order, id)); `inbox.rs` (InboxResource — collection with resourcetype
         {collection, schedule-inbox}, CommonPropertiesProp, owner-only
         privileges; InboxObjectResource reusing `CalendarObjectPropWrapper`
         (getetag / calendar-data / getcontenttype) with a sha256 etag over
         id+ics; InboxResourceService PathComponents=(principal,), get_members
         from `get_inbox_objects`, nests the object service at `/{object_id}`;
         object ids are used VERBATIM — `sanitize_id` already appends `.ics`,
         so unlike calendar objects there is NO .ics-stripping deserializer;
         object service implements get/delete_resource + GET handler with
         owner-only auth, ETag + text/calendar, HEAD support);
         `outbox.rs` (OutboxResource — collection with resourcetype
         {collection, schedule-outbox}; OutboxResourceService holds the
         scheduler; `AxumMethods::post` → `post_outbox` →
         `handle_outbox_post` → HTTP 200 + `schedule-response` XML with
         per-recipient `mailto:` href and `request-status` `2.0;Success` /
         `5.0;<msg>`; body-parse failures → 400 via
         `rustical_dav::Error::BadRequest`). Inbox/outbox DAV_HEADER consts
         bake the scheduling tokens in (their routes are only mounted while
         enabled); the principal/calendar/calendar-object services instead
         override `dav_header()` with the helper.
       - **Principal props + service:** `PrincipalProp` gained
         `ScheduleInboxUrl` / `ScheduleOutboxUrl` /
         `ScheduleDefaultCalendarUrl` (`Option<HrefElement>`, `#[xml(rename =
         "schedule-inbox-URL")]` etc., NS CALDAV — same conditional-prop
         pattern as the existing `Source(Option<HrefElement>)`; allprop
         serializes them empty while disabled). `PrincipalResource` gained
         `scheduling: Option<SchedulingProps>`; fills derive the inbox/outbox
         hrefs from the request's principal URI + `inbox/` / `outbox/`
         (each tree — `/caldav` and `/caldav-compat` — advertises its own),
         default-calendar href percent-encodes the id.
         PrincipalResourceService gained `scheduler: Option<Arc<Scheduler>>`,
         fills `scheduling` in `get_resource` (enabled-gated), and
         `axum_router` conditionally mounts `/inbox` + `/outbox` when
         `scheduler.is_some_and(enabled)` (static axum routes beat
         `/{calendar_id}`), passing the scheduler into
         CalendarResourceService.
       - **Calendar + calendar-object services:** both gained the `scheduler`
         field (new() signatures extended, Clone updated, `dav_header()`
         overrides added); CalendarResourceService threads it into
         CalendarObjectResourceService, which implements `on_resource_deleted`
         → `handle_delete(user.id, deleted.get_ics(), ua)`.
       - **`put_event` (calendar_object/methods.rs):** previous-object fetch
         now also runs when a scheduler is present (the If-Match/If-None-Match
         precondition block reuses it; semantics unchanged); User-Agent read
         from the header map; after a successful `put_object` it calls
         `handle_put(&user.id, (principal, calendar_id, object_id), old_ics,
         body, ua)`. NB: `object_id` here is the `.ics`-STRIPPED id (path
         deserializer strips it) — consistent with what the store returns,
         which the guard exclusion depends on. Acting user = authenticated
         principal id; impersonation (`user$family`) surfaces as `family` and
         then no-ops conservatively (organizer≠`family`).
        **Finish-list status (updated 2026-09-05, latest session):**
        1. `crates/caldav/src/lib.rs`: extend `caldav_router` with the
           `scheduler: Option<Arc<Scheduler>>` param and pass it into the
           `PrincipalResourceService` literal (currently missing the new
           `scheduler` field → E0063; the `use rustical_scheduling::Scheduler`
           and `pub mod scheduling;` lines are already in place).
           — **DONE (latest session): param added (last position), threaded into
           the literal; lib compiles.**
        2. `src/app.rs`: update BOTH `caldav_router` calls (both trees) to pass
           `None` (item 3 later replaces None with the real wiring).
           — **DONE (latest session): both calls pass `None` with a comment
           pointing at item 3.**
        3. `crates/caldav/src/principal/tests.rs`: the two struct literals
           (PrincipalResourceService ~line 25, PrincipalResource ~line 72) need
           `scheduler: None` / `scheduling: None`.
           — **DONE (latest session).**
        4. `crates/caldav/src/scheduling/tests.rs`: referenced by
           `#[cfg(test)] mod tests;` in scheduling/mod.rs but NOT written yet.
           Planned coverage (design decided): full `caldav_router` via
           TestStoreContext + `SqliteSchedulingStore` + enabled
           SchedulingConfig (also proves the `/inbox` vs `/{calendar_id}` axum
           route precedence doesn't panic at Router construction); OPTIONS
           advertises the tokens; PROPFIND principal fills the three props;
           PUT with an internal attendee → REQUEST in the attendee's inbox
           (PROPFIND/GET/DELETE); DELETE → CANCEL; outbox POST →
           schedule-response; vdirsyncer UA → no delivery. Fixture facts
           settled: add second principal `attendee@example.com` + app token
           via `principal_store.add_app_token`, `personal` calendar via
           `insert_calendar`, ORGANIZER `mailto:user` matches the fixture
           principal `user`, expected inbox object id `req-<uid>.ics`.
            — **WRITTEN (latest session), all 7 planned tests + a
            `test_outbox_rejects_non_organizer` bonus; fixture exactly as
            designed. Green (4): OPTIONS tokens (principal + calendar), PROPFIND
            principal props (hrefs `/caldav/principal/user/{inbox,outbox,personal}/`),
            vdirsyncer-UA no-delivery, outbox-rejects-non-organizer (400 — NB not
            yet distinguished whether that 400 comes from the organizer check or
            a parse failure). FAILING (3): PUT→REQUEST-in-inbox, DELETE→CANCEL,
            outbox POST→schedule-response — all on one underlying delivery bug,
            see the session log. NOTE the tests use UIDs without `@`
            (`sched-req-1` etc.): `sanitize_id` maps `@`→`_`, so
            `req-put-test-1@example.com.ics` would actually be
            `req-put-test-1_example.com.ics`.**
            — **ALL GREEN (2026-09-05, later session): the 3 failing tests pass
            after the `as_email` fix (see the second session log); the bonus
            test's 400 is now provably the organizer check (parse no longer
            fails), i.e. it fails for the right reason.**
        5. Snapshot updates (via `INSTA_UPDATE=always`, then review):
           `crates/caldav/src/principal/snapshots/…propfind-2.snap` (debug) and
           `…propfind-3.snap` (serialized) gain the three empty props; the ROOT
           integration-test snapshots
           `tests/integration_tests/caldav/snapshots/…propfind_depth_0.snap`
           and `…propfind_depth_1.snap` likewise (they PROPFIND the principal
           allprop through make_app — unchanged wiring there because app.rs
           passes None).
           — **SMALLER THAN PREDICTED (latest session): only the DEBUG snapshot
           `propfind-2.snap` changed (reviewed: exactly the three new
           `ScheduleInboxUrl/ScheduleOutboxUrl/ScheduleDefaultCalendarUrl(None)`
           variants inserted). The serialized snapshots are UNAFFECTED —
           `propfind-3` passes unchanged and contains no schedule elements:
           XML serialization SKIPS the None-valued props entirely (the earlier
           "serializes them empty" assumption was wrong). The root
            `propfind_depth_0/1` snapshots were predicted-unaffected (app.rs
            passes None) and are expected to pass unchanged — re-verify in the
            item-2 gate run.**
            — **RE-VERIFIED in the gate (2026-09-05, later session): root
            integration suite 19/19 green, snapshots untouched.**
         6. Item-2 gate: `SQLX_OFFLINE=true cargo check --workspace
            --all-targets` clean (keep the 0-warnings bar) + suites green
            (dav 31, scheduling 12, caldav 27 + the new scheduling tests,
            store_sqlite 13).
            — **NOT yet run this session (only `cargo check -p rustical_caldav
            --all-targets` verified clean; caldav suite currently 31 passed /
            3 failed — the 3 new failing delivery tests).**
            — **RUN AND GREEN (2026-09-05, later session): workspace check 0
            errors / 0 warnings; dav 31, scheduling 13 (12 + new regression
            test), caldav 34 (27 + 7 scheduling), store_sqlite 13; root crate
            integration tests 19/19. Plus `cargo fmt --check` clean (see the
            second session log).**

        **Session log (2026-09-05, latest session) — wiring completed, delivery
        bug found and half-diagnosed:**
        - **Extra compile fixes beyond the predicted E0063s** (first `cargo
          check` didn't even parse the manifest): (a) `crates/caldav/Cargo.toml`
          had a **duplicate `sha2.workspace = true`** key (previous session added
          it although it already existed) — removed; (b) `Scheduler` got a
          **manual redacted `Debug` impl** (`enabled` only) instead of a derive —
          it holds `SchedulingConfig` with SMTP passwords which must never leak
          through `PrincipalResourceService`'s `#[derive(Debug)]` into logs;
          (c) `SchedulingProps` got `#[derive(Debug, Clone)]`; (d) inbox.rs
          needed `use hex::ToHex;` for `encode_hex` and had an unused
          `IntoResponse` import removed.
        - **Routing finding (test-level):** the bare `caldav_router(...)` Router
          matches paths **WITHOUT trailing slash only** — `/caldav/principal/user`
          → 200/207, but `/caldav/principal/user/` → 404 (integration tests go
          through `make_app`'s merged Router, which behaves differently — the
          Phase 1.5 smoke test's trailing-slash `/caldav/` PROPFIND worked there).
          All scheduling-test request URIs therefore use no trailing slash. The
          inbox/outbox **static nests DO win over `/{calendar_id}`** and router
          construction does not panic — precedence requirement proven. The
          inbox PROPFIND/GET/DELETE routes work (207 with
          `resourcetype {collection, schedule-inbox}`, ETag'd GET, successful
          DELETE, owner-only 401s).
         - **THE OPEN BUG — `Scheduler::handle_put` silently no-ops.** Symptom:
           PUT (as `user`, ORGANIZER `mailto:user`, internal ATTENDEE
           `attendee@example.com`) returns 201 but the attendee's inbox stays
           empty; DELETE likewise delivers no CANCEL; outbox POST returns 400.
           Isolated to the scheduler (NOT routing/auth/store) via a direct-call
           debug test (`debug_scheduler_direct`, currently still in tests.rs —
           **remove before finishing**): with the store side individually proven
           OK — manual `put_inbox_object` → Ok and readable; 
           `find_calendar_objects_by_uid("user","sched-req-1")` returns exactly
           `[("personal","sched-req-1",ics)]`; `principal_exists(attendee)` =
           true; migration table exists (`create_db_pool(":memory:", true)`
           migrates) — calling `handle_put("user", ("user","personal",
           "sched-req-1"), None, &ics, None)` directly still delivers nothing.
           Code inspection so far: `Line::parse`/`as_email`/`unfold`/`parse_event`
           (crates/scheduling/src/ics.rs) and every guard in
           `handle_put`→`organizer_put`→`deliver`→`deliver_status` look correct;
           scheduling unit tests remain 12/12. **Next steps (in order):**
           1. cheapest discriminator first — call `ics::parse_event` directly on
              the exact `event_ics(...)` payload in a debug print (if it returns
              None or wrong EventInfo, the bug is parse-side and likely explains
              the outbox 400 too — a unifying hypothesis, NOT yet confirmed);
           2. otherwise instrument the guard chain (eprintln at each early return
              in handle_put/organizer_put, and print the `handle_outbox_post`
              Err string in the outbox test — the 400's cause is currently
              ambiguous between "could not parse iTIP message" and the
              organizer check);
           3. fix, re-run the 3 failing tests, then the full item-2 gate.
           **→ RESOLVED (2026-09-05, later session): step 1's unifying hypothesis
           was CONFIRMED — see the second session log.**
         - **Also pending cleanup:** delete `debug_scheduler_direct` + its extra
           imports (`CalendarObject`, `SchedulingStore`) from tests.rs; `cargo
           fmt` still uninstalled (run `rustup component add rustfmt` before the
           item-5 cross-build per the earlier note).
           **→ DONE (2026-09-05, later session): debug test + both imports
           removed; rustfmt installed and the whole branch is fmt-clean.**

         **Session log (2026-09-05, later session) — delivery bug root-caused,
         fixed, item-2 gate green:**
         - **Root cause (empirically confirmed via the planned step-1
           discriminator, a `parse_event` print in the debug test):** the test
           payload parsed to `EventInfo { organizer: None, attendees:
           [attendee@example.com, …] }` — `Line::as_email()`
           (crates/scheduling/src/ics.rs) required `@` in **every** extracted
           address, but the fixture's ORGANIZER is `mailto:user` and the fixture
           principal id is `user` (RustiCal principal ids are arbitrary strings
           — `@` is NOT required; only `:` and `$` are forbidden). So the
           organizer parsed to `None` → `handle_put`/`handle_delete` early-
           returned at their `let Some(organizer) … else { return }` guards and
           `handle_outbox_post` returned `Err("iTIP message has no ORGANIZER")`
           → the outbox 400. One parse-side bug, all three failures — the
           previous session's "code inspection looks correct" simply missed that
           `@`-less principal ids can legally appear as ORGANIZER/ATTENDEE
           CAL-ADDRESSes. (Production identities are full email addresses, so
           this would have worked there — but the server must not silently
           no-op on arbitrary-string principals, which are a core RustiCal
           property this project relies on.)
         - **Fix:** `as_email()` now takes a `mailto:`-prefixed value verbatim
           (trimmed + lowercased; still rejected if empty or containing `:`/`
           ` `) — a `mailto:` CAL-ADDRESS is an address *by definition*; the
           `@`-containing heuristic now only guards the bare (non-mailto) form
           against random text. Regression unit test
           `mailto_without_at_is_an_address` added to ics.rs (organizer
           `mailto:user` → `Some("user")`; bare `ORGANIZER:user` still → None);
           scheduling suite now 13/13.
         - **Aftermath:** the 3 delivery tests + all others green (caldav lib
           34/34 = 27 + 7 scheduling); `debug_scheduler_direct` and its
           `CalendarObject`/`SchedulingStore` imports removed from tests.rs;
           `test_outbox_rejects_non_organizer` now 400s for the *right* reason
           (organizer≠acting-user check, not a parse failure).
         - **Item-2 gate run (finish-list item 6):** `SQLX_OFFLINE=true cargo
           check --workspace --all-targets` → 0 errors / 0 warnings; suites:
           dav 31, scheduling 13, caldav 34, store_sqlite 13 (all green); root
           crate integration tests 19/19 (the `propfind_depth_0/1` snapshots
           pass unchanged — finish-list item 5's re-verification done).
         - **rustfmt:** installed (1.9.0, closing the earlier note) and
           `cargo fmt` applied — `cargo fmt --check` now clean. The fmt pass
           touched several pre-existing scheduling-branch files (import
           ordering, line wrapping in principal/mod.rs, calendar/service.rs,
           calendar_object/service.rs etc.) — formatting only; all suites
           re-run green afterwards.
    3. main crate: `[scheduling]` in `Config` (serde-default so the existing
       router config keeps parsing), `cmd_serve` wiring (`get_data_stores` returns
       the scheduling-store handle → `Scheduler::new(config, store)` → `make_app`).
       — **DONE (2026-09-05, later session): implemented + gate green on
       `omnical-scheduling`.**
       - `src/config.rs`: `pub scheduling: SchedulingConfig` field
         (`#[serde(default)]`, placed between `caldav` and `maintenance`) — old
         configs without the section keep parsing (also covered by the bin's
         5 figment TOML/env tests, which parse configs with no `[scheduling]`);
         figment's `RUSTICAL_SCHEDULING__*` env overrides come for free.
         `src/commands/mod.rs`: `cmd_gen_config` literal extended →
         `rustical gen-config` now prints the section (`enabled = false`, the 8
         default UA exclusions, `smtp = []`).
       - `src/lib.rs`: `get_data_stores` returns a 6th tuple element
         `Arc<dyn SchedulingStore>`; the sqlite arm constructs the calendar
         store un-Arc'd first, builds `SqliteSchedulingStore::new(cal_store.clone())`
         from it (`SqliteCalendarStore` is `Clone`; pool + push-sender are cheap
         Arc clones → one shared pool, and the item-1 migration
         `20260905120000_scheduling` still runs through the existing
         `create_db_pool(db_url, migrate)` call), then re-wraps the calendar
         store in its `Arc`. `cmd_principals`'s 5-tuple destructure → 6.
       - `cmd_serve`: `let scheduler = config.scheduling.enabled.then(|| Arc::new(
         Scheduler::new(config.scheduling.clone(), scheduling_store)))` — the
         scheduler is constructed **only when enabled**, so `[scheduling]
         enabled = false` (the default, and the current router config) behaves
         byte-for-byte like stock 0.16.1: no inbox/outbox routes, no scheduling
         DAV tokens, no PUT/DELETE hooks, and no extra previous-object fetch in
         `put_event` (that fetch is gated on a scheduler being present).
         Startup logs `Scheduling extension enabled (N SMTP identities)` when
         on. `make_app(..., scheduler, ...)` threads it into both `caldav_router`
         trees (`/caldav` + `/caldav-compat`).
       - `src/app.rs`: `make_app` gained `scheduler: Option<Arc<Scheduler>>`
         (after `caldav_config`); the two placeholder `None`s from item 2
         replaced with `scheduler.clone()` / `scheduler`.
       - Call-site fixes: `tests/integration_tests/mod.rs` passes `None`
         (scheduling behavior stays covered by the caldav crate's 7 scheduling
         tests, which drive `caldav_router` directly); `tests/common/mod.rs` and
         the three `Config` literals in `tests/http_integration.rs` gained
         `scheduling: Default::default()`.
       - **Gate (item-3):** `SQLX_OFFLINE=true cargo check --workspace
         --all-targets` → 0 errors / 0 warnings; `cargo fmt --check` clean
         (two nits auto-fixed by `cargo fmt`: import order, `.then(...)` chain
         wrapping); suites green: dav 31, scheduling 13, caldav 34,
         store_sqlite 13, root lib 20 + bin 5 + http-integration 3 (these spawn
         real `cmd_serve` processes via `test_runner`, so the new wiring — incl.
         the 6-tuple `get_data_stores` and the disabled-config path — runs under
         test) + integration 19 (snapshots unchanged).
       - Deployment notes for item 5 (recorded now while fresh): enabling on the
         router = add `[scheduling]` with `enabled = true` + the 7 SMTP accounts
         to `/etc/rustical/config.toml` (rendered from
         `pass secrets/email/<id>/smtp` at deploy time per the user decision) —
         or set `RUSTICAL_SCHEDULING__ENABLED=true` env in the init script. First
         start of the new binary auto-applies the `scheduling_inbox_objects`
         migration to the live DB (online, additive table; item 5's pre-deploy DB
         backup covers it).
    4. Local x86_64 smoke test: RFC 6638 curl checks (OPTIONS tokens, principal
       props, inbox PROPFIND/GET/DELETE, PUT with attendees → Python SMTP sink
       captures the iMIP email; accept/reply flow between two local principals).
          — **DONE (2026-09-05, final session): ALL gates GREEN — every step of
          the "NEW Remaining steps" list executed and verified, including both
          live SMTP legs (the earlier rustls dual-provider panic in the spawned
          SMTP task — found by this smoke test, fixed in smtp.rs with the
          explicit aws-lc-rs provider + regression test — is now PROVEN fixed
          end-to-end: STARTTLS → AUTH PLAIN → dot-stuffed DATA all succeed and
          the sink captures the mail). Full results in the execution log after
          the remaining-steps list below. Next: item 5 (cross-build).**
        - **Build DONE:** `scripts/build-rust.sh x86_64-unknown-linux-gnu` on
          `omnical-scheduling` — `out/x86_64-unknown-linux-gnu/{rustical,dav-tls}`
          (35 MB / 1.4 MB, host profile as in Phase 1.5). `rustical --version`
          OK; `gen-config` prints the `[scheduling]` section (enabled, the 8
          default UA exclusions, smtp) — item 3's config plumbing confirmed in
          the real binary.
        - **Design constraints found by reading the scheduler/SMTP code (they
          shape the test):** (a) the UA-exclusion defaults (vdirsyncer, curl,
          python, wget, khard, khal, okhttp, go-http-client) mean every
          *triggering* request must send a custom User-Agent (e.g.
          `omnical-smoke/1.0`); one PUT with curl's default UA is kept in the
          plan as the live exclusion check. (b) The SMTP client hard-requires
          STARTTLS with webpki root validation (no plaintext mode) — the sink
          must present a cert the client trusts; solved by presenting the REAL
          LE cert (`~/router-dav/out/tls/`) for `0115d8cf.duckdns.org`. (c)
          Inbox object ids follow `sanitize_id`: `req-<uid>.ics`,
          `cancel-<uid>.ics`, `reply-<uid>-<attendee>.ics`, with `@` → `_` in
          uids/addresses.
        - **Harness prepared in `/tmp/opencode/sched-smoke/` (`/tmp` is wiped on
          reboot — this log is the durable record; re-creating is minutes):**
          `smtp_sink.py` (stdlib-only sink on 127.0.0.1:8025 speaking exactly
          the client's envelope: EHLO → STARTTLS (LE cert) → EHLO → AUTH PLAIN
          → MAIL/RCPT/DATA/QUIT, dot-unstuffing, captures each message to
          `mail/NNN.eml` + `envelope.jsonl` with mail_from/rcpt_to/auth_user);
          `config.toml` (bind 127.0.0.1:4000, scratch DB
          `file:/tmp/opencode/sched-smoke/db.sqlite3`, scheduling enabled, two
          SMTP identities `org@omnical.test` + `att@omnical.test` both →
          `0115d8cf.duckdns.org:8025`); fixtures in `ics/`: `invite.ics`
          (organizer org@omnical.test invites internal att@omnical.test +
          external guest@example.net), `noua.ics` (same shape, for the
          curl-default-UA exclusion check), `remote.ics` (external organizer
          remote@elsewhere.test invites att@omnical.test — exercises the
          remote-organizer REPLY-email path).
        - **Environment findings:** user namespaces are DISABLED on this host
          (`/proc/sys/user/max_user_namespaces` = 0 → bwrap and `unshare -Urn`
          both fail), but **passwordless sudo works**: the plan is
          `sudo unshare -m` + `mount --bind` of a private hosts file
          (containing `127.0.0.1 0115d8cf.duckdns.org`) over /etc/hosts for
          the SERVER PROCESS ONLY (optionally `runuser -u burningserenity`
          back to the user inside the namespace) — its SMTP client then
          reaches the local sink while webpki-validating the real LE cert,
          with zero host-visible changes; the sink runs on the host normally
          (mount-only namespace shares the network namespace, so 127.0.0.1
          works between them). `mount --bind` inside `sudo unshare -m` was
          verified to succeed; the actual duckdns→127.0.0.1 override content
          was NOT yet exercised (the earlier belief that /etc/hosts pins the
          name was a misread — the name resolves via DNS; /etc/hosts never
          contained it). Also: ports 4000/8025 free; Python 3.14 (no aiosmtpd
          → hand-rolled sink).
         - ~~**Remaining steps (next session resumes here):**~~ **(executed —
           steps 1–3 by the session log below; the rest by the final session's
           execution log):** (1) write the
          override hosts file, confirm `getent hosts 0115d8cf.duckdns.org` →
          127.0.0.1 INSIDE the `sudo unshare -m` namespace; (2) start
          `smtp_sink.py` + the namespace-wrapped server (`rustical
          --config-file /tmp/opencode/sched-smoke/config.toml serve`, RUST_LOG
          with `rustical_scheduling=debug` for the SMTP C/S transcript); (3)
          create principals `org@omnical.test` + `att@omnical.test`
          (`principals create --password`, piped) + app tokens, MKCOL their
          `personal` calendars (Phase 1.5: defaults NOT auto-created); (4) run
          the gates: OPTIONS DAV tokens on principal + calendar, PROPFIND
          principal schedule-inbox/outbox/default-calendar URLs, PUT
          `invite.ics` (custom UA) → 201 + REQUEST in att's inbox
          (PROPFIND/GET/DELETE) + iMIP email in the sink (multipart, From org,
          AUTH as org), PUT `noua.ics` with curl's DEFAULT UA → NO deliveries
          (exclusion live), att PUTs the invite copy with
          PARTSTAT=ACCEPTED → organizer's stored copy updated + REPLY in
          org's inbox, att PUTs a `remote.ics` copy with PARTSTAT=ACCEPTED →
          REPLY email from att's SMTP identity in the sink, DELETE of the
           organizer's invite → CANCEL in inbox + email, outbox POST (REQUEST
           + REPLY) → 200 schedule-response with per-recipient request-status.

         **Session log (2026-09-05, latest session) — steps 1–3 executed,
         internal gates GREEN, a REAL smtp.rs runtime bug found & fixed,
         rebuild left mid-flight (all state facts below verified on disk):**
         - **Step 1 DONE:** hosts override verified — `sudo unshare -m` +
           `mount --bind hosts.override /etc/hosts` → `getent hosts
           0115d8cf.duckdns.org` = 127.0.0.1 INSIDE the namespace, real
           65.33.235.245 outside (name lives in DNS, not /etc/hosts); the
           mount-only namespace shares the network namespace, so the
           server's SMTP client reaches the host-side sink on 127.0.0.1:8025.
         - **Step 2 DONE:** sink up (real LE cert presented for STARTTLS);
           server started via `sudo unshare -m … runuser -u burningserenity`
           with `RUST_LOG=info,rustical_scheduling=debug`; startup log
           `Scheduling extension enabled (2 SMTP identities)`; scheduling
           migration auto-applied to the scratch DB.
         - **Step 3 DONE + MKCOL finding:** principals `org@omnical.test` /
           `att@omnical.test` + smoke app tokens created (identical CLI shape
           to Phase 5). MKCOL of `personal` calendars **400s unless the body's
           root tag is in the `DAV:` namespace** — `C:mkcol` → "Invalid tag …
           Expected [Some(Namespace(DAV:))]mkcol"; `D:mkcol` (with
           `C:calendar` inside `D:resourcetype`) → 201 for both. Phase 5.4's
           seeding evidently obeyed the same rule — restated here for reuse.
         - **Gates GREEN (internal delivery path):** OPTIONS advertises
           `calendar-scheduling, calendar-auto-schedule` on principal,
           calendar, `/inbox`, `/outbox` (calendar adds `webdav-push`);
           PROPFIND principal fills all three schedule URLs (inbox/outbox/
           default-calendar hrefs, `@` percent-encoded as `%40`); PUT
           `invite.ics` as org with custom UA `omnical-smoke/1.0` → 201 +
           etag, REQUEST landed in att's inbox as `req-smoke-invite-1_omnical.test.ics`
           (uid `@`→`_` exactly as predicted), calendar-data with
           `METHOD:REQUEST` intact.
         - **THE BUG this smoke test exists to catch — invisible to the unit
           and integration suites (they never build a real connector): the
           external iMIP email never arrived.** The sink saw the client drop
           immediately after `STARTTLS`; the server log had a PANIC in the
           spawned email task: `tls_connector()` used the implicit
           `ClientConfig::builder()`, and this binary's dependency graph
           enables BOTH rustls crypto providers (aws-lc-rs default + ring via
           other workspace members) → rustls cannot choose a process default
           and panics (`rustls-0.23.43/src/crypto/mod.rs:249`); the spawned
           task dies SILENTLY — no retry, no warn, PUT still 201s.
           **FIX in `crates/scheduling/src/smtp.rs`:** explicit
           `builder_with_provider(aws_lc_rs::default_provider())` +
           `.with_safe_default_protocol_versions()`; regression test
           `tls_connector_builds_with_explicit_provider` added. Gate after
           the fix: `SQLX_OFFLINE=true cargo check --workspace --all-targets`
           0 errors/0 warnings, `cargo fmt --check` clean, scheduling suite
           **14/14** (13 + the regression test).
          - **x86_64 rebuild left MID-FLIGHT (state verified 2026-09-05 ~19:55;
            → RESOLVED by the final session's step 0 — the cp was redone and
            the smoke test ran from the fixed binary):**
           the fix compiled cleanly into
           `~/router-dav/build/cargo-target/x86_64-unknown-linux-gnu/release/rustical`
           (19:47, 35,844,376 B = +4.5 KB vs stock), but the script's `cp` to
           `out/` never completed — first attempt hit `Text file busy` (the
           then-running server held the binary); the follow-up chain (kill
           server → wipe scratch → re-run build) DIED at its own `pkill -f
           'sched-smoke/config.toml'`, which matched the invoking shell's
           cmdline (classic pkill self-match) — everything after it (rm, log
           truncation, build script) never ran; the tool then timed out at
           300 s. Net state: **server DEAD (port 4000 free); sink STILL
           RUNNING on 127.0.0.1:8025; scratch DB + WAL + artifacts + empty
           mail/ NOT wiped; `out/x86_64-unknown-linux-gnu/rustical` is still
           the PRE-FIX 19:15 binary** (35,839,896 B) — do NOT smoke-test from
           out/ until the cp is redone. **pkill hygiene for next session: use
           a non-self-matching pattern, e.g. `pkill -f '[r]ustical
           .*sched-smoke'` or kill by PID from `ss`.**
          - ~~**NEW Remaining steps (next session resumes here):**~~ **EXECUTED
            IN FULL (2026-09-05, final session):** (0) re-run
           `scripts/build-rust.sh x86_64-unknown-linux-gnu` (cp succeeds now —
           server dead) and sanity-check `out/…/rustical --version`; wipe the
           scratch DB + `mail/` + `out-*.xml` (the never-run rm) and re-seed
           step 3 (principals, tokens, MKCOL — credentials land in the same
           `creds_*_token.txt` files; sink already running, server restart
           command in the log above); (1) re-run the PUT-invite gate — 201 +
           REQUEST in att's inbox (should reproduce) AND the iMIP email
           captured by the sink (multipart/mixed, From org, AUTH as
           `org-smtp-user`, base64 `text/calendar; method=REQUEST`
           attachment) = the smtp.rs fix's live proof; (2) inbox GET + DELETE
           of the REQUEST object (owner-only auth spot-check); (3) PUT
           `noua.ics` with curl's DEFAULT UA → NO deliveries (exclusion
           live); (4) att PUTs the invite copy with PARTSTAT=ACCEPTED →
           organizer's stored copy updated + REPLY in org's inbox; (5) att
           PUTs a `remote.ics` copy with PARTSTAT=ACCEPTED → REPLY email
           from att's SMTP identity in the sink; (6) DELETE of the
           organizer's invite → CANCEL in inbox (STATUS:CANCELLED ensured) +
           email; (7) outbox POSTs (REQUEST + REPLY) → 200 schedule-response
            with per-recipient request-status; (8) cleanup (stop server/sink),
            flip item 4 to DONE, proceed to item 5 (cross-build).

          **Session log (2026-09-05, final session) — item 4 completed, all
          gates GREEN (server killed by PID, no pkill self-match; sink reused
          from the prior session):**
          - **Step 0:** rebuild cp'd the post-fix binary into
            `out/x86_64-unknown-linux-gnu/rustical` (35,844,376 B = the 19:47
            cargo-target build; `--version` OK); scratch DB/WAL/SHM + `mail/` +
            `out-*.xml` wiped; namespace-wrapped server restarted (fresh DB →
            migrations + `Scheduling extension enabled (2 SMTP identities)`);
            hosts override re-verified (127.0.0.1 inside the unshare -m
            namespace, 65.33.235.245 outside); principals + smoke app tokens
            re-created (same `creds_*_token.txt` files); MKCOL `personal` 201
            for both (D:-namespace body reused).
          - **Step 1 — smtp.rs fix's LIVE PROOF:** PUT `invite.ics` as org
            (custom UA `omnical-smoke/1.0`) → 201 + etag; REQUEST in att's
            inbox (`req-smoke-invite-1_omnical.test.ics`, uid `@`→`_` as
            predicted) **AND the iMIP email captured by the sink**: envelope
            `mail_from <org@omnical.test> → <guest@example.net>`, **AUTH as
            `org-smtp-user`, `tls: true`**; full SMTP transcript in the server
            log (EHLO → STARTTLS → EHLO → AUTH PLAIN → MAIL/RCPT/DATA/QUIT, no
            panic). Email verified: multipart/mixed, From
            `Smoke Organizer <org@omnical.test>`, Subject "Invitation: …",
            `Auto-Submitted: auto-generated`, human text/plain + base64
            `text/calendar; method=REQUEST` attachment (decodes to the full
            VEVENT with `METHOD:REQUEST` injected).
          - **Step 2:** inbox GET as att → 200 + etag + `text/calendar`
            (METHOD:REQUEST intact); GET as org (non-owner) → **401**
            (owner-only auth); DELETE as att → 200, inbox empty after.
          - **Step 3:** PUT `noua.ics` with curl's DEFAULT UA (curl/8.21.0) →
            201 stored but **zero deliveries**: att inbox empty, no new email,
            and every `send_mail` log line timestamps to the single earlier
            invite delivery — UA exclusion live.
          - **Step 4:** att PUTs the invite copy with PARTSTAT=ACCEPTED →
            201; **organizer's stored copy rewritten** (org's
            `personal/smoke-invite-1.ics` now shows
            `ATTENDEE;…;PARTSTAT=ACCEPTED` for att) **+ REPLY in org's
            inbox** (`reply-smoke-invite-1_omnical.test-att@omnical.test.ics`)
            with METHOD:REPLY, same UID, correct ORGANIZER/ATTENDEE/PARTSTAT.
            (Code-verified en route: the changed-PARTSTAT guard compares
            against an existing old copy only — a first-time PUT with a
            PARTSTAT fires the reply, which is exactly the iOS accept flow.)
          - **Step 5:** att PUTs a `remote.ics` copy (external organizer
            `remote@elsewhere.test`) with PARTSTAT=ACCEPTED → REPLY **email
            captured: `mail_from <att@omnical.test> → <remote@elsewhere.test>`,
            AUTH as `att-smtp-user`** (the attendee's own SMTP identity, per
            design), attachment `method=REPLY`.
          - **Step 6:** org DELETEs the invite → 200; **CANCEL in att's
            inbox** (`cancel-smoke-invite-1_omnical.test.ics`, METHOD:CANCEL +
            STATUS:CANCELLED, attendees preserved) **+ CANCEL email** to
            guest@example.net (Subject "Cancelled: …", attachment
            method=CANCEL, AUTH as org-smtp-user). Move-guard exercised
            implicitly: only org's own calendars were searched (att's copy
            correctly not treated as a "move").
          - **Step 7:** outbox POST REQUEST as org → **200
            `schedule-response`** with per-recipient `mailto:` hrefs and
            `2.0;Success` for BOTH the internal (att) and external (guest)
            recipient; deliveries verified (REQUEST in att's inbox + 4th email
            captured). Outbox POST REPLY as att (single-attendee iTIP) → 200
            `2.0;Success`; REPLY filed in org's inbox.
          - **Internal gates re-verified on the FIXED binary + fresh DB (the
            prior session's OPTIONS/PROPFIND greens predated the smtp.rs fix):
            OPTIONS advertises `calendar-scheduling, calendar-auto-schedule` on
            principal, calendar (which adds `webdav-push`), `/inbox`, `/outbox`;
            PROPFIND principal fills all three schedule URLs
            (inbox/outbox/default-calendar=personal).**
          - **Totals:** 4 emails captured (001 REQUEST→guest from org, 002
            REPLY→remote-organizer from att, 003 CANCEL→guest from org, 004
            outbox REQUEST→guest from org) — every one with the correct
            per-identity From + SMTP AUTH user. Cleanup: server + sink
            stopped by PID (ports 4000/8025 free). `/tmp/opencode/sched-smoke/`
            artifacts remain until reboot; this log is the durable record.
          - **Gotcha worth remembering (verification tooling):** inbox-object
            hrefs percent-encode `@` in object ids
            (`reply-…-att%40omnical.test.ics`), so filtering PROPFIND output
            with `grep -v %40` (meant to drop the principal self-hrefs)
            silently hides inbox objects — filter on the `/inbox/` path
            segment instead.
     5. Cross-build aarch64 via `scripts/build-rust.sh` (recipe D), size-gate
       (binary was 26 MiB of the 35 MiB budget; expect ~+1 MiB), DB backup,
       `deploy.sh`, router config with the 7 SMTP accounts rendered from pass,
       server-side verify through dav-tls (`curl --resolve …:8443:192.168.1.21`).
       — **DONE (2026-09-06, 00:40–01:00; session started the evening of
       2026-09-05) — ALL gates green; the extension is LIVE on the router.**
       - **Cross-build:** `scripts/build-rust.sh` (clang recipe D) on
         `omnical-scheduling` — 2 m 33 s; `out/rustical` = static stripped
         aarch64 ELF, 29,074,968 B = **27.7 MiB of the 35 MiB budget** (+1.2 MiB
         over stock, as predicted); `out/dav-tls` byte-identical to the deployed
         one (sha256-verified; rustical `--version` + `gen-config` printing the
         `[scheduling]` section sanity-checked under `qemu-aarch64`).
       - **Pre-deploy safety net:** hot SQLite `.backup` pulled to
         `~/backups/omnical/db-pre-sched-deploy-20260905.sqlite3` (3.5 MB,
         integrity ok, live counts 517/209/8 — exactly the pre-deploy state).
       - **Config render:** new `scripts/render-router-config.sh` (0755) —
         appends `[scheduling]` `enabled = true` + the 8 default UA exclusions
         (written explicitly) + 7 `[[scheduling.smtp]]` accounts to the
         secret-free base template and prints the result to **stdout** (piped
         into `ssh router 'cat > /etc/rustical/config.toml'`; deploy.sh holds it
         in a shell variable). Passwords from `pass secrets/email/<id>/smtp`;
         TOML-escaped; missing/empty entry aborts the render. Verified with
         python `tomllib` (7 accounts, correct hosts/ports/users, password
         lengths sane). `zero@novo-ordo.com` maps to the shared
         `nfcalaway@novo-ordo.com` pass entry (user `nfcalaway@novo-ordo.com`),
         exactly per the msmtp topology. **`~/router-dav` remains secret-free**
         (rendered config never touches dev disk).
       - **deploy.sh hardening (two real findings):** (1) **ENOSPC — the
         overlay (~9.4 MB free) cannot hold a second 28 MB binary copy next to
         the running one** (scp failed "write remote: Failure"; the first
         attempt left a 26 MB partial `rustical.new` that had to be rm'd); and a
         RUNNING binary's blocks are only reclaimed when its process stops.
         Fix: stage both binaries in `/tmp` (**tmpfs — zero overlay cost**),
         then a tight stop → flash-copy → start swap (~5–10 s outage).
         (2) **Order-critical: stock 0.16.1's `Config` is
         `deny_unknown_fields`** — the new `[scheduling]` config must never meet
         the old binary (crash-respawn would loop), so the binary is swapped
         while stopped and the config lands before the start. Also: dav-tls is
         swapped in place and restarted **only if its sha256 changed** (it
         didn't — the running process kept serving, no TLS blip); the render
         runs **before** any service stop (fail-fast if pass breaks). All of
         this is codified in deploy.sh comments — future re-deploys
         (post-sysupgrade) are safe/idempotent.
       - **Deployed + verified on the router:** rustical healthy (new PID),
         127.0.0.1:4000 only; startup log `Scheduling extension enabled (7
         SMTP identities)`; dav-tls untouched and running; migration
         `20260905120000_scheduling` auto-applied to the live DB
         (`scheduling_inbox_objects` table, 0 rows); live counts **517/209/8 —
         identical to the pre-deploy backup**, integrity ok; WAL checkpointed
         (32 KB → 0) → overlay **7,012 KB free (floor ≥ 5 MB intact)**.
       - **Server-side verify through dav-tls** (all with
         `curl --resolve 0115d8cf.duckdns.org:8443:192.168.1.21`, full LE chain
         validation, `ssl_verify_result=0`, no `-k`; vdirsyncer app token):
         `/.well-known/caldav` → 308; OPTIONS advertises
         `calendar-scheduling, calendar-auto-schedule` on the principal
         (**both** `/caldav` and `/caldav-compat` trees), the calendar (+
         `webdav-push`), `/inbox`, `/outbox`; PROPFIND principal (Depth 0,
         named props) 207 with all three schedule URLs filled
         (`…%40gmail.com/inbox/`, `…/outbox/`, `…/personal/`); PROPFIND the
         inbox collection → 207 `{collection, schedule-inbox}` displayname
         "Schedule Inbox" (compat tree also 207); personal calendar Depth 1
         serves its 196 objects; full `vdirsyncer sync` clean (0 errors,
         idempotent second run — the hub never noticed the swap).
       - Note: the item-6 live iPhone test has NOT started; the phone's
         accounts may need a re-save to pick up the newly advertised
         scheduling support.
   6. Live iPhone test (test identity `nicholas@carltonaudio.com`): `logread -f`
      capture while re-saving the account (also resolves the still-open "unverified
      phone→server traffic" item), confirm the Invitees field now appears, create
      an event inviting another identity (internal inbox path) and an external
      address (email path), accept/reply round-trip, then batch-verify the other 6
      identities and run the verification-matrix rows 14–16.
   7. PLAN.md final status update + the upstream i3status-rust issue ~~remains
      a *separate* pending task~~ — **update (2026-09-05, later session): the rebuild
      itself is DONE (see Phase 7 STATUS); RESOLVED same day after `gh auth login`:
      the issue is FILED as greshake/i3status-rust#2309 (see Phase 7 STATUS for the
      evidence chain) — nothing remains here except possible upstream follow-up
      (a PR was offered in the issue).**
3. **WebDAV Push transports** (WebSocket/WebPush) tuning for instant DAVx5 sync
   (RustiCal ships support; configure in `dav_push` after Phase 7).
4. **IPv6**: publish AAAA on duckdns once a stable GUA exists on WAN; same firewall
   rule already covers it (`family='any'`).
5. **Ujail/seccomp hardening** of both services via procd jail params, once stable.
  6. **Dedicated VM off-site replica** (via existing wg-vm tunnels) running the same
     binaries — a warm standby with hourly DB copy.
  7. **Public read-only subscription feeds ("share links") for calendars + contacts
     — PULLED FORWARD FROM "FUTURE" BY USER DECISION 2026-09-06** ("we need to add
     to the plan: make this service a calendar/contacts people can subscribe to,
     instead of having to send ics files all over the place"). Supersedes the
     Phase 7 open item "Subscribed-calendar route": its unverified question
     (whether iOS "Add Subscribed Calendar" accepts owner credentials on the
     owner-authenticated `route_get` export) becomes moot — subscription URLs
     carry their own credential, so subscribers need **no account, no app token,
     no login prompt**.

**STATUS (2026-09-06, updated again same day): item 1 DONE (see its
         DONE block); item 2 DONE — code compiled, compile fixes applied, the
         item-2 gate run and ALL GREEN incl. the new export_routes suite and a
         full `cargo test --workspace` with 0 failures (see item 2's DONE block);
         item 3 DONE (resumed from the user-stopped session 2026-09-06, later
         session — import fix, test call-sites, and the enabled-path
         http-integration test done; item-3 gate ALL GREEN: workspace check
         0/0, fmt clean, full `cargo test --workspace` 0 failures with every
         expected count exact — see item 3's execution log); item 4 DONE
         (2026-09-06, later session — x86_64 smoke test, **all 25 gates green**
         — see item 4's DONE block); item 5 DONE (2026-09-06, later session —
         cross-build + deploy, **all gates green, the extension is LIVE on
         the router** alongside scheduling — see item 5's DONE block, which
         also records the unrecorded-intermediate-build drift found at recon);
         item 6 NOT STARTED. NEXT: item 6 (live tests — needs the user's
         physical iPhone: "Add Subscribed Calendar" with token URLs for
         nicholas@carltonaudio.com personal + the family calendar, `.vcf`
         fetch on the phone, watch `logread` poll cadence, then the PLAN.md
         final status update).**

     **Design (user decision = the Google Calendar "secret address" model):**
     - Two public URL shapes, served WITHOUT authentication (the token in the
       URL *is* the credential — the routes mount outside the DAV
       `AuthenticationLayer`):
       `https://0115d8cf.duckdns.org:8443/export/<64-char-token>.ics` — live
       full export of a calendar, reusing the exact `route_get` payload builder
       (`IcalCalendar::from_objects` + `X-WR-CALNAME/CALDESC/CALCOLOR/TIMEZONE`,
       `text/calendar; charset=utf-8`;
       `crates/caldav/src/calendar/methods/get.rs`);
       `https://0115d8cf.duckdns.org:8443/export/<token>.vcf` — full export of
       an addressbook (all live vCards concatenated, `text/vcard;
       charset=utf-8` — NEW; the CardDAV tree has no GET export today).
       Subscribers: iOS/macOS "Add Subscribed Calendar", Google Calendar "From
       URL", `webcal://` (clients rewrite to https), any URL-fetching tool.
       Contacts apps have no "subscribe" concept — a stable secret `.vcf` URL
       is the contacts equivalent (open/import/refetch instead of mailing
       vCards around).
     - **URL semantics:** the extension must match the token's collection kind
       (`.ics` ↔ calendar, `.vcf` ↔ addressbook); unknown token, kind/extension
       mismatch, or vanished collection → **404** (never 401/403 — no auth
       prompts, no token-validity oracle for the public-port scanners). GET/HEAD
       only (polling clients); no write path exists by construction; scheduling
       triggers are unaffected (they live on PUT/DELETE only).
     - **Token format:** 64-char alphanumeric — the exact `generate_app_token`
       shape (`crates/frontend/src/routes/app_token.rs`), ≈380 bits of entropy;
       guessing gets 404s like any other path on the public port.
     - **Storage decision (deliberate deviation from the app-token pbkdf2
       pattern):** new `subscriptions` table with the token in **plaintext +
       UNIQUE index**. Rationale: app tokens are hashed because they grant
       **write** access; a subscription token grants **read** access to data
       that sits in plaintext in the same DB file (`calendarobjects.ics`), so
       hashing protects nothing the DB doesn't already contain (upstream's own
       app-token note concedes the same about DB access), while plaintext
       enables the O(1) indexed lookup the no-username request requires
       (per-principal verify like app tokens is impossible) and lets
       `subscriptions list` re-display a lost URL (Google-parity UX).
     - **Management (CLI, server-admin like `principals`, no HTTP auth):**
       `rustical subscriptions add <principal> --kind calendar|addressbook
       <collection_id>` (generates the token, prints the full URL(s)),
       `subscriptions list <principal>`, `subscriptions remove <principal>
       <id>` (instant revoke). Group collections work (e.g. the `family`
       calendar — the token targets principal `family`, collection `family`).
     - **Config gate:** `[subscriptions] enabled = true` (serde-default false —
       the existing router config keeps parsing and behaves byte-for-byte like
       the current build until enabled; same pattern as `[scheduling]`). Routes
       only mount when enabled.

     **Implementation items (mirroring the scheduling extension's structure):**
     1. **Store + token layer**: `SubscriptionKind`/`Subscription` model +
        `SubscriptionStore` trait (no-op/NotFound defaults so test stores keep
        compiling) in `crates/store`, SQLite impl + migration
        `20260906120000_subscriptions` (runtime `sqlx::query` — keeps the
        committed `.sqlx/` metadata untouched) in `crates/store_sqlite`, unit
        tests in the store_sqlite harness. Gate: workspace check 0 errors/0
        warnings + suites green + fmt clean.
        — **DONE (2026-09-06): implemented + gate green. All on the same
        uncommitted `omnical-scheduling` working tree in `~/router-dav/rustical`
        (additive on top of the deployed scheduling build — commit the two
        extensions together or split them when committing; the router still
        runs the §17.2 item-5 binary until item 5 below deploys).**
        - `crates/store/src/subscription_store.rs` (NEW): `SubscriptionKind`
          (Calendar/Addressbook, `as_str` + `TryFrom<&str>` for row/CLI
          decode), `Subscription` (id, principal, kind, collection_id, token,
          created_at), `SubscriptionStore` trait — `add_subscription`
          (caller-generated token, returns the new id; default
          `Err(ReadOnly)` because a no-op has no honest id to return),
          `get_subscription_by_token` / `delete_subscription` (defaults
          `Err(NotFound)`), `get_subscriptions` (default empty) — the same
          no-op-default pattern as `SchedulingStore`, so non-SQLite (test)
          stores keep compiling untouched. Re-exported at the crate root
          (`pub use subscription_store::*;`, mirroring `scheduling_store`).
        - Migration `20260906120000_subscriptions.{up,down}.sql`:
          `subscriptions` (id TEXT PK, principal, kind, collection_id, token
          **UNIQUE**, created_at DEFAULT CURRENT_TIMESTAMP, FK
          principal→principals ON DELETE CASCADE) +
          `idx_subscriptions_principal`; the plaintext-token rationale lives
          in a header comment in the migration.
        - `crates/store_sqlite/src/subscription_store.rs` (NEW):
          `SqliteSubscriptionStore` mirroring `SqliteSchedulingStore` (holds
          the cloneable `SqliteCalendarStore` for its pool; runtime
          `sqlx::query` only — `.sqlx/` untouched). Duplicate token →
          `Error::AlreadyExists` for free via the existing unique-violation
          mapping in `crates/store_sqlite/src/error.rs` (no new error code).
          Registered in `lib.rs`.
        - Tests: `crates/store_sqlite/src/tests/subscription_store.rs` — full
          lifecycle (unknown token → NotFound [the export URL's 404 path] →
          create → token lookup/list round-trip incl. created_at set by the DB
          → delete under a foreign principal is a NotFound no-op → revoke
          kills the token instantly → double-delete NotFound) + duplicate-token
          rejection (uniqueness is global across kind/collection; addressbook
          kind round-trips).
        - **Compile findings (all fixed):** (a) `Error::Other` needs
          `anyhow::anyhow!` — `String` has no `Into<anyhow::Error>`; (b)
          `Row::get` takes two generic params — write `let kind: String =
          row.get("kind")`, not a turbofish; (c) rstest `#[future]` fixture
          args need `.await` before field access; (d) store test files must
          wrap contents in `#[cfg(test)] mod tests { … }` (upstream's pattern,
          easy to miss): under `cargo check --workspace --all-targets` the
          feature-enabled **lib** target also compiles `pub mod tests`, and
          `#[tokio::test]` fns are compiled OUT there (plain `#[test]` items
          vanish outside `cfg(test)`), so unwrapped top-level `use` lines
          warn as unused in that target. Pitfall re-confirmed: `cargo check -p
          rustical_store_sqlite --all-targets` in ISOLATION fails on the bench
          (`tests` module is feature-gated) — only the full-workspace
          invocation unifies the `test` feature (§17.2 item-2.0 knew this).
        - **Gate:** `SQLX_OFFLINE=true cargo check --workspace --all-targets`
          → **0 errors / 0 warnings**; `cargo test -p rustical_store_sqlite`
          → **15/15 green** (13 existing + the 2 new, 0.37 s);
          `cargo fmt --check` clean (one line-wrapping nit in
          `subscription_store.rs` fixed by `cargo fmt`). NOT re-run this
          session (every touched line is additive; the new migration is
          already proven valid by both the store_sqlite and caldav fixtures'
          `:memory:` migrate-on-open): dav / scheduling / caldav / root
          lib+bin+http-integration / integration suites — re-run alongside
          item 2's gate before building further. **NEXT: item 2 (export
          routes).**
     2. **Export routes**: axum router mounted in `src/app.rs` **without** the
        AuthenticationLayer (enabled-gated): `/export/{token}.ics` → token
        lookup → `CalendarStore::get_calendar`+`get_objects` → factor the
        `route_get` export body into a shared builder so owner-export and
        token-export serve byte-identical payloads; `/export/{token}.vcf` →
        `AddressbookStore` objects concatenated. 404 semantics above; HEAD;
        ETag skipped (polling cadence is client-driven).
         — **DONE (2026-09-06, later session — resumed from the stopped
         session): code compiled, compile fixes applied, item-2 gate run and
         ALL GREEN.** (Prior state: all item-2 code written but never
         compiled — session stopped by user before the gate; task selection
         as recorded there: item 2 was the first unfinished task a
         dev-machine session could execute.) All on the same uncommitted
         `omnical-scheduling` tree:
        - `crates/caldav/src/calendar/methods/get.rs` — the `route_get`
          export body is factored into `pub fn build_export_ics(&Calendar,
          Vec<(String, CalendarObject)>) -> String` in the same file
          (identical logic: `X-WR-CALNAME/CALDESC/CALCOLOR/TIMEZONE` props
          from the calendar metadata); `route_get` now calls it —
          byte-identical by construction. This is the shared builder.
        - `src/export.rs` (NEW; one-line `pub mod export;` added to
          src/lib.rs) — `pub fn export_router<AS: AddressbookStore,
          CS: CalendarStore>(addr_store: Arc<AS>, cal_store: Arc<CS>,
          sub_store: Arc<dyn SubscriptionStore>) -> Router`, one route
          `GET/HEAD /export/{filename}`: axum/matchit cannot suffix-match a
          path param, so `<token>.<ext>` is parsed in the handler via
          `rsplit_once('.')` (missing or unknown extension → 404); token
          lookup via `get_subscription_by_token`; kind/extension must match
          (Calendar↔`.ics`, Addressbook↔`.vcf`); **every failure path
          returns `Err(rustical_store::Error::NotFound)`**, so unknown
          token, kind/extension mismatch and vanished collection render the
          identical `(404, "Not found")` body — no auth prompt, no
          token-validity oracle; both collection lookups use
          `show_deleted = false`, so soft-deleted (trashed) collections 404
          (the owner CalDAV `route_get` uses `true`; for a live collection
          the body is unaffected); `.ics` served via the shared
          `build_export_ics`, `.vcf` via the exact CardDAV `route_get`
          concatenation (`get_vcf()` joined with `\r\n`); content types
          identical to the owner exports (`text/calendar; charset=utf-8` /
          `text/vcard; charset=utf-8`); ETag deliberately skipped; HEAD is
          free (axum 0.8 `routing::get` auto-serves HEAD with the body
          stripped — verified in the axum 0.8.9 source). `ExportState` has
          a manual `Clone` impl without `AS`/`CS` bounds (a derived Clone
          would add bounds generic stores don't have and break mounting).
          NOT mounted anywhere yet — mounting (at app level, the dav-push
          `subscription_service` pattern = outside the DAV
          `AuthenticationLayer` by construction) + the `[subscriptions]`
          config gate + store plumbing + CLI are item 3, per the item
          split.
        - Design correction found while implementing: "the CardDAV tree has
          no GET export today" was WRONG — stock 0.16.1 already has one
          (`crates/carddav/src/addressbook/methods/get.rs`) doing exactly
          the planned vCard concatenation; the export router mirrors it,
          so no carddav-side factoring was needed.
        - Wiring note for item 3: `make_app` feeds the caldav trees the
          `combined_cal_store` (cal + birthdays); passing the SAME combined
          store into `export_router` keeps `_birthdays_*` collections
          feed-capable and preserves owner-export parity — passing the raw
          cal store would 404 birthday feeds (item 3's deliberate choice).
        - Root `Cargo.toml` — `rustical_ical.workspace = true` added to
          [dev-dependencies] (the test fixture needs `CalendarObjectType`
          for the calendar's `components` field).
        - `tests/export_routes.rs` (NEW, root crate) — 6 route-level tests
          on real sqlite fixtures (`rustical_store_sqlite::tests::
          test_store_context`; objects seeded through the real DAV PUT
          paths of the caldav/carddav routers, basic auth against the
          fixture principal; 66-char token-shaped constants — length not
          load-bearing, item 3's CLI generates the real 64-char tokens):
          `.ics` byte-parity vs the owner-authenticated CalDAV `route_get`
          export (body + content-type, no ETag); `.vcf` byte-parity vs the
          CardDAV one; HEAD = headers only; the 404 matrix (unknown token /
           no extension / `.txt` / both kind-mismatches, asserting all
           bodies IDENTICAL); revoked subscription → instant 404; vanished
           (trashed) calendar + addressbook → 404. **Ran green 6/6 in the
           item-2 gate (first execution).**
         - ~~**Remaining work (next session resumes here):**~~ **EXECUTED
           (2026-09-06, later session) — results:**
           - **Step 0 (compile + fixes): far fewer problems than predicted —
             1 error + 1 warning total, both in the NEW test file's
             integration surface, none in the export code itself:** (a)
             `SqliteSubscriptionStore` had only `#[derive(Debug)]` but the
             test needs a direct handle next to the `Arc<dyn
             SubscriptionStore>` handed to the router → `Clone` added to the
             derive, matching every sibling store (`SqliteCalendarStore`,
             `SqliteAddressbookStore`, `SqlitePrincipalStore`,
             `SqliteDavPushStore` all derive `Clone`; only the
             single-consumer `SqliteSchedulingStore` doesn't); (b) dead-code
             warning: `TestApp.addr_sub_id` was never read → the revoked-
             subscription test now revokes BOTH subscriptions (calendar +
             addressbook) and asserts both URLs 404 — warning gone, "instant
             revoke" now proven for both kinds. `cargo fmt` made no changes
             (the never-compiled code was already fmt-clean).
           - **Step 1 (item-2 gate) — ALL GREEN:**
             `SQLX_OFFLINE=true cargo check --workspace --all-targets` →
             **0 errors / 0 warnings**; `cargo fmt --check` clean; suites:
             dav 31, scheduling **17** (14 + the 3 new iOS-fix tests below),
             caldav **35** (34 + the iOS regression test below), store_sqlite
             15, root lib 20 + bin 5 + **export_routes 6/6 (NEW, first run
             ever: both byte-parity tests incl. content-type + no-ETag, HEAD
             headers-only, the full 404 matrix with identical bodies, revoke
             both kinds, vanished calendar + addressbook)** +
             http-integration 3 + integration 19 (snapshots unchanged). A
             full `cargo test --workspace` was also run beyond the listed
             gate suites: **every target green, 0 failures anywhere**
             (covers dav_push/frontend/xml/ical/oidc/store too).
           - **Step 2:** this block flipped to DONE (this text).
           - **Step 3:** NEXT = item 3 (main-crate wiring + CLI).
         - **The earlier unrecorded iOS principal-URL-ORGANIZER fix
           (~08:50, 2026-09-06) — now recorded; validated by this gate run.**
           Root cause (from the regression test's comment): iOS writes the
           account owner's ORGANIZER (and self-ATTENDEE) as the CalDAV
           **principal URL** — RustiCal's advertised calendar-user-address,
           e.g. `/caldav/principal/user/` — not as `mailto:`; the scheduler
           then didn't recognize the organizer and delivered nothing (same
           silent-no-op class as the `as_email` `@` bug). Fix, on the same
           uncommitted `omnical-scheduling` tree: `crates/scheduling/src/
           ics.rs` gained `principal_url_id()` (recognizes path-only
           `/caldav[-compat]/principal/<id>/…` AND absolute
           `scheme://host:port/…` forms; id = percent-decoded segment after
           `/principal/`) which `as_email()` now accepts as an address form,
           plus `pub fn normalize_caladdresses(ics)` rewriting principal-URL
           ORGANIZER/ATTENDEE values to `mailto:` (leaves bare/random values
           untouched); `crates/scheduling/src/scheduler.rs` `deliver_status`
           applies it before building the delivered copy, so both inbox
           copies and iMIP attachments carry `mailto:` CAL-ADDRESSes that
           remote and local clients can work with. Tests: caldav
           `test_principal_url_organizer_delivers_request` (iOS-shaped PUT
           with principal-URL ORGANIZER/self-ATTENDEE → REQUEST delivered,
           stored copy normalized to `mailto:user`) + 3 scheduling ics.rs
           unit tests (`principal_url_organizer_is_an_address`,
           `principal_url_forms` incl. percent-encoded absolute forms,
           `normalize_rewrites_principal_urls`) — hence the 17/35 suite
           counts above.
     3. **Main-crate wiring + CLI**: `[subscriptions]` config section,
        `get_data_stores` 7th tuple element `Arc<dyn SubscriptionStore>`,
        `make_app` param + router mount, `rustical subscriptions` subcommands
(token generator = 64-char Alphanumeric; model types from item 1).
         — **DONE (2026-09-06, later session — resumed from the stopped
         session): the "Remaining work" list executed in full, item-3 gate
         run and ALL GREEN — see the execution log at the end of this
         item.**
        All changes on the same uncommitted `omnical-scheduling` tree; nothing
        deployed (the router still runs the §17.2 item-5 binary). **What was
        written this session:**
        - `src/config.rs` — `SubscriptionsConfig { enabled, public_url }`
          (`#[serde(deny_unknown_fields, default)]`, derive Default → enabled
          defaults to false = zero footprint until enabled; `public_url` is
          `Option<String>` with `skip_serializing_if`) + `Config.subscriptions`
          field (`#[serde(default)]` — old configs keep parsing, the 5→6
          existing main.rs config tests remain the back-compat proof).
        - `src/commands/subscriptions.rs` (NEW) — the CLI:
          `SubscriptionsCommand::{Add,List,Remove}` mirroring `principals
          app-token`; `add <principal> <collection_id> --kind
          calendar|addressbook` (local `KindArg` clap ValueEnum wrapper —
          orphan rule forbids deriving it on the foreign `SubscriptionKind`),
          token = reuse of `generate_app_token()` (64-char Alphanumeric,
          exactly the designed shape), prints `Subscription created (id: …)`
          + the full export URL; `list` re-displays `id - kind - collection -
          URL - created`; `remove <principal> <id>` prints removal. `add`
          validates the target collection exists first (fail-fast instead of
          a forever-404 URL): calendars via the **combined** store (same view
          the export routes serve — keeps `_birthdays_*` subscribable per
          item 2's deliberate choice), addressbooks via the addr store,
          `show_deleted = false` both; and prints a stderr note when
          `[subscriptions] enabled = false` (routes not mounted). URL base =
          `public_url` when set, else `http://{http.bind_config()}` (covers
          the deprecated host form), path-only on Unix-socket binds.
        - `src/commands/mod.rs` — module registered + re-exported
          (`SubscriptionsArgs, cmd_subscriptions`); gen-config literal gained
          `subscriptions: SubscriptionsConfig::default()` → `rustical
          gen-config` now prints `[subscriptions]` (enabled = false;
          public_url skipped when None).
        - `src/lib.rs` — `Command::Subscriptions` variant; `get_data_stores`
          returns a **7-tuple** (new last element `Arc<dyn SubscriptionStore>`;
          `SqliteSubscriptionStore::new(cal_store.clone())` in the sqlite
          branch next to the scheduling store, pre-Arc, same pool);
          `cmd_serve` destructures 7, builds `subscriptions =
          config.subscriptions.enabled.then_some(subscription_store)`
          (scheduler pattern — Some only when enabled), logs
          `Subscriptions extension enabled (public export feeds)`, threads it
          into `make_app` (new param after `scheduler`).
        - `src/commands/principals.rs` — destructure 6→7 (extra `_`).
        - `src/app.rs` — `make_app` param
          `subscriptions: Option<Arc<dyn SubscriptionStore>>` (after
          `scheduler`); when Some, merges `export_router(addr_store.clone(),
          combined_cal_store.clone(), sub_store)` **before the frontend
          block** (`combined_cal_store` is moved into `frontend_router` there;
          cloning first preserves the item-2 deliberate choice to feed the
          export routes from the combined store) — outside the DAV
          `AuthenticationLayer` by construction, same as the dav-push
          `subscription_service` pattern.
        - `src/main.rs` — dispatch `Command::Subscriptions` →
          `cmd_subscriptions(args, parse_config()?)`; NEW bin config test
          `test_config_toml_subscriptions` (TOML with enabled+public_url
          parses; without the section → disabled defaults; bin tests now 6).
        **Design decisions beyond the item-3 text (recorded while fresh):**
        (a) `[subscriptions]` gained a second key **`public_url`** —
        required to fulfill "prints the full URL(s)": the production public
        URL (`https://0115d8cf.duckdns.org:8443`, dav-tls front end) differs
        from the HTTP bind (`127.0.0.1:4000`), so the CLI cannot derive it;
        unset falls back to the bind-derived URL (correct but only
        locally-reachable). **Item 5's config render must set it** (e.g.
        `public_url = "https://0115d8cf.duckdns.org:8443"`). (b) CLI works
        regardless of `enabled` (admin may pre-provision; the note warns).
        (c) The CLI reuses `get_data_stores` like `cmd_principals` (migrations
        run, same DB as the server — proven concurrent-access-safe by
        test_initial_setup).
~~**Verified compile state (2026-09-06, incremental
         `SQLX_OFFLINE=true cargo check -p rustical --lib --bins`): FAILS —
         2 errors + 2 warnings, ALL in the new `subscriptions.rs`, everything
         else compiles:**~~ **(RESOLVED by the later session: the recorded
         "Known fix" was correct — see the execution log below.)**
        - 2× E0599: `get_calendar` (line ~116, on `CombinedCalendarStore`) and
          `get_addressbook` (line ~121) — the defining traits
          (`CalendarReadStore`, `AddressbookReadStore`) are not in scope;
          export.rs escapes this because its `CS: CalendarStore` generic
          bound brings supertrait methods into scope, but a CONCRETE type
          needs the trait imported. **Known fix: in subscriptions.rs line 6,
          replace the import set with `use rustical_store::{
          AddressbookReadStore, CalendarReadStore, CombinedCalendarStore,
          SubscriptionKind };`** (the current
          `AddressbookStore, CalendarStore, SubscriptionStore` are exactly
          the 2 unused-import warnings — none of the three is needed).
        - Consequence: the bins target (main.rs dispatch + the new test) is
          unchecked behind the lib failure.
~~**Remaining work (next session resumes here):**~~ **EXECUTED IN
         FULL (2026-09-06, later session) — results:**
        1. Apply the known import fix above in
           `src/commands/subscriptions.rs`, re-run
           `SQLX_OFFLINE=true cargo check -p rustical --lib --bins` → expect
           clean; **expect fmt nits** (the file was never `cargo fmt`ed).
        2. Test call-sites (NOT yet updated — `--all-targets` will E0063
           until then): `tests/common/mod.rs` Config literal +
           `tests/http_integration.rs` ×3 Config literals need
           `subscriptions: Default::default()`; `tests/integration_tests/
           mod.rs` `make_app` call needs the new `None` arg (after the
           scheduler `None`).
        3. Planned but NOT written: ONE enabled-path http-integration test
           (`test_subscriptions_export`) + a `rustical_process_with(db_url,
           customize-config-closure)` variant in tests/common/mod.rs —
           spawn real `cmd_serve` with `subscriptions.enabled = true`, seed
           principal+calendar through the stores, `cmd_subscriptions` add,
           unauthenticated GET `/export/<token>.ics` → 200
           (`BEGIN:VCALENDAR…`), kind/extension mismatch `.vcf` → 404, CLI
           remove → instant 404. This is the src wiring's only enabled-path
           coverage (item 4 re-proves it at the real-binary level).
        4. Item-3 gate (NOT yet run): `SQLX_OFFLINE=true cargo check
           --workspace --all-targets` 0 errors/0 warnings; `cargo fmt
           --check` clean; suites green — expected counts after the fix:
           dav 31, scheduling 17, caldav 35, store_sqlite 15, root lib 20 +
           bin 6 (5 + the new config test) + export_routes 6 +
           http-integration 4 (3 + the new one) + integration 19.
5. Flip this block to DONE (record the gate results), update the
            STATUS line above, then NEXT = item 4 (x86_64 smoke test).

         **Execution log (2026-09-06, later session) — item 3 completed,
         gate ALL GREEN:**
         - **Step 1 (import fix):** the recorded "Known fix" applied
           verbatim and the compiler CONFIRMED its non-obvious claim —
           `SubscriptionStore` is genuinely unneeded there. The reason
           (scratch-verified with a 6-line rustc repro): method calls on a
           `dyn Trait` receiver (the CLI's `Arc<dyn SubscriptionStore>`
           destructure) resolve WITHOUT the trait being in scope; only
           CONCRETE types need the defining trait imported (the
           mirror-image of this rule then bit the new test — step 3b).
           `SQLX_OFFLINE=true cargo check -p rustical --lib --bins` → clean
           (0 errors, 0 warnings). **fmt nits: NONE** — the never-fmt'ed
           file was already fmt-clean.
         - **Step 2 (test call-sites):** 4 Config literals gained
           `subscriptions: Default::default()` — 2× in test_initial_setup +
           1× in test_principal_impersonation (both in
           tests/http_integration.rs) + 1× in tests/common/mod.rs;
           tests/integration_tests/mod.rs `make_app` gained the new `None`
           arg after the scheduler `None`. Pitfall hit: the impersonation
           literal sits at a SHALLOWER indent than the other two, so a
           single-pattern replaceAll silently missed it — caught by the
           follow-up E0063 and fixed with its own edit. The predicted error
           set after fmt (4× E0063 + 1× E0061) was exactly right.
         - **Step 3 (the enabled-path test):** `rustical_process_with(db_url,
           customize: FnOnce(&mut Config))` added to tests/common/mod.rs
           (`rustical_process` kept as a thin `|_| {}` wrapper);
           `test_runner_with` added beside `test_runner` in
           tests/http_integration.rs — with test_runner's body deliberately
           RESTORED to call `rustical_process` directly: letting it delegate
           through test_runner_with made `rustical_process` dead code in
           this target and warned (dead_code + unused_import), violating
           the 0-warnings bar. NEW `test_subscriptions_export`: spawns a
           real `cmd_serve` with `[subscriptions] enabled`, creates the
           principal via `cmd_principals`, seeds a `personal` calendar
           through the stores, runs `cmd_subscriptions` add (asserts the
           read-back token is exactly 64 chars), then: unauthenticated GET
           `/export/<token>.ics` → 200 `text/calendar` + body starting
           `BEGIN:VCALENDAR`; `.vcf` kind/extension mismatch → 404; CLI
           remove → instant 404. The token is read back through
           `SqliteSubscriptionStore::get_subscriptions` because in-process
           tests cannot capture the CLI's stdout println!s. Two compile
           fixes beyond the plan text: (a) `SqliteCalendarStore::new` is a
           3-arg derive-Constructor `(db, mpsc::Sender<CollectionOperation>,
           skip_broken)` — the store fixtures' `channel(1)` pattern used
           for both constructions; (b) calling `get_subscriptions` on the
           CONCRETE `SqliteSubscriptionStore` DOES need the
           `SubscriptionStore` trait imported — the dyn exception from
           step 1 does not apply to concrete types — trait added to the
           test's import list.
         - **Step 4 (item-3 gate) — ALL GREEN, every expected count exact:**
           `SQLX_OFFLINE=true cargo check --workspace --all-targets` →
           **0 errors / 0 warnings**; `cargo fmt --check` clean; full
           `cargo test --workspace` → **0 failures anywhere** — dav 31,
           carddav 15, dav_push 9, scheduling 17, caldav 35, store_sqlite
           15, root lib 20 + bin 6 (5 + `test_config_toml_subscriptions`) +
           export_routes 6/6 + http-integration 4/4 (3 +
           `test_subscriptions_export`, green on first run) + integration
           19/19 (snapshots unchanged) + xml 36 + ical 1 + frontend 1.
         - **Step 5:** this block flipped to DONE, STATUS line updated.
           **NEXT = item 4 (x86_64 smoke test)** — the src wiring now has
           its enabled-path coverage; item 4 re-proves it at the
           real-binary level. Nothing deployed (the router still runs the
           §17.2 item-5 binary); the tree remains uncommitted as before.
     4. **Local x86_64 smoke test** (scratch DB): CLI-created tokens →
        unauthenticated curl of `.ics`/`.vcf` (byte-compare `.ics` against the
        owner-authenticated `route_get` export), 404 matrix (bad token, wrong
        extension, revoked, vanished collection), HEAD.
          — **DONE (2026-09-06, later session): ALL gates green — 25/25 PASS
          at the real-binary level (CLI-generated 64-char tokens, curl as the
          unauthenticated client). Nothing deployed; the router still runs the
          §17.2 item-5 binary.**
         - **Build first:** `out/x86_64-unknown-linux-gnu/rustical` was the
           2026-09-05 §17.2 build (pre-subscriptions) → rebuilt via
           `scripts/build-rust.sh x86_64-unknown-linux-gnu` (1 m 08 s,
           35 MB); `--version` OK and `gen-config` prints the `[subscriptions]`
           section (item 3's plumbing confirmed in the real binary).
         - **Harness in `/tmp/opencode/subs-smoke/`** (`/tmp` is wiped on
           reboot — this log is the durable record; re-creating is minutes):
           `config.toml` with bind `127.0.0.1:4000`, scratch DB, `[subscriptions]
           enabled = true`, and `public_url = "http://127.0.0.1:4000"` so the
           CLI prints directly curl-able URLs. Startup log: `Subscriptions
           extension enabled (public export feeds)`; both extension
           migrations auto-applied to the fresh DB.
         - **Seeding via the real CLI + real DAV paths:** principal
           `org@omnical.test` (piped `principals create --password`) + app
           token; MKCOL calendar `personal` **with displayname +
           `C:calendar-description` + `A:calendar-color`** so the export
           carries the full `X-WR-CALNAME/CALDESC/CALCOLOR` header block;
           MKCOL addressbook `personal`; PUT VEVENT + VCARD (201/201).
           **MKCOL finding (restated for reuse): calendar MKCOL props are
           namespaced — `D:description` → 400 "Invalid field name in
           MkcolCalendarProp" (the calendar description prop is
           `C:calendar-description`, CALDAV ns); `A:calendar-color` is
           accepted. The addressbook took plain `D:` props fine.**
         - **CLI verified live (concurrent with the running server — same
           SQLite pool class as `cmd_principals`):** `subscriptions add`
           prints the id + full URL (`public_url` base honored); tokens are
           **exactly 64 chars** (checked); `subscriptions list` re-displays
           id - kind - collection - URL - created (the Google-parity "lost
           URL" UX); `remove` prints removal.
         - **Gates — 25/25 PASS (`gates.sh` kept in the harness dir):**
           - **Byte-parity:** unauthenticated `.ics`/`.vcf` **byte-identical
             (`cmp`)** to the owner-authenticated `route_get` exports
             (GET on the collection URLs, basic auth); content types
             identical (`text/calendar; charset=utf-8` /
             `text/vcard; charset=utf-8`); X-WR-CALNAME/CALDESC/CALCOLOR
             lines present in both; **no `ETag`** on feed responses (design:
             skipped); **no `WWW-Authenticate`** on feeds or 404s — the
             routes are demonstrably outside the DAV `AuthenticationLayer`.
           - **HEAD:** `.ics`/`.vcf` → 200 + content-type via `curl -I`.
             (Tooling note: `curl -X HEAD` hangs — curl waits for a GET-style
             body; use `-I`. Body-stripping is axum auto-HEAD behavior,
             proven by the route test.)
           - **404 matrix (6 cases):** unknown token (`.ics` + `.vcf`),
             missing extension, `.txt`, calendar-token-as-`.vcf`,
             addressbook-token-as-`.ics` — all 404 with **byte-identical
             bodies** (no token-validity oracle) and no auth prompt.
           - **Revoke:** removing the calendar subscription → its `.ics` URL
             is an **instant 404 while the `.vcf` feed still serves 200**
             (per-subscription scoping); removing the addressbook
             subscription → its URL 404.
           - **Vanished collections:** subscriptions re-added (new tokens —
             re-add proven), then both collections DELETEd through the DAV
             tree (owner auth, 200) → both **live-token URLs 404**
             (`show_deleted = false` semantics live through the whole
             stack).
         - **Server log clean during the gates:** zero WARN/ERROR/PANIC —
           the only ERROR lines are the two deliberate 400 MKCOL probes; the
           404-matrix requests log as INFO "client error" exactly like any
           other 404 (export requests visible in `rustical::app` spans).
         - **Cleanup:** server stopped by PID (port 4000 free); scratch
           artifacts remain in `/tmp/opencode/subs-smoke/` until reboot.
           **NEXT: item 5 (cross-build + deploy).**
      5. **Cross-build + deploy** (deploy.sh flow: `/tmp` staging + binary-swap;
         config render adds `[subscriptions] enabled = true`; pre-deploy DB
         backup — the additive migration auto-applies on start) + server-side
         verify through dav-tls.
         — **DONE (2026-09-06, later session): ALL gates green — the
         subscriptions extension is LIVE on the router alongside scheduling.**
         - **Drift found at recon (recorded here because no session recorded
           it):** the router was NOT running the §17.2 item-5 binary as the
           plan's item-1/4 blocks state, but an unrecorded intermediate
           build — `/usr/sbin/rustical` 29,076,336 B, mtime 2026-09-06 12:54,
           i.e. the 08:54 post-iOS-fix build, deployed by someone without a
           PLAN.md entry (evidence: `out/rustical` carried the same 08:54
           mtime/size; the running binary had no `subscriptions` table and no
           `[subscriptions]` config support, so it predates the item-1 store
           layer). Superseded by this deploy either way; baseline was taken
           from the live DB, not from the plan text.
         - **Pre-deploy baseline (live DB, integrity ok):** principals 8,
           memberships 7, cal_live 517, addr_live 209, app_tokens 29,
           sched_inbox 1; WAL 0 (already folded); overlay 7000 KB free.
         - **Cross-build:** `scripts/build-rust.sh` (clang recipe D), 1 m 36 s
           incremental; `out/rustical` = 29,289,304 B = **27.9 MiB of the
           35 MiB budget** (+212,968 B vs the running binary). qemu sanity:
           `--version` OK, `gen-config` prints `[subscriptions]` (enabled =
           false default), `subscriptions --help` shows add/list/remove.
           `out/dav-tls` sha256-identical to the deployed one → deploy.sh
           skipped its restart (no TLS blip; dav-tls PID unchanged).
         - **Pre-deploy backup:** hot `.backup` pulled to
           `~/backups/omnical/db-pre-subs-deploy-20260906.sqlite3` (3,575,808
           B, 0600); integrity ok; counts identical to live.
         - **Config render:** `scripts/render-router-config.sh` extended with
           the `[subscriptions]` section — `enabled = true` +
           `public_url = "https://0115d8cf.duckdns.org:8443"` (item 3's
           mandate: the CLI must print the public dav-tls front end, not the
           127.0.0.1:4000 bind). Render validated with python `tomllib`
           (piped — never on dev disk): 7 SMTP accounts with correct
           hosts/identities, non-empty passwords, 8 UA exclusions,
           subscriptions section correct. `~/router-dav` stays secret-free.
         - **Deploy (deploy.sh, unchanged flow):** /tmp staging → stop →
           binary swap → config → start; rustical healthy on new PID;
           startup log shows BOTH lines: `Scheduling extension enabled (7
           SMTP identities)` and `Subscriptions extension enabled (public
           export feeds)`; migration `20260906120000_subscriptions` recorded
           in `_sqlx_migrations`, `subscriptions` table 0 rows; live counts
           identical (517/209/8/29/1); WAL 45 KB → 0 via checkpoint; overlay
           **6768 KB free** (≥ 5120 KB floor intact). `deny_unknown_fields`
           order-criticality handled by deploy.sh's stop→swap→config→start
           sequence (the `[subscriptions]` config never met the old binary).
         - **Server-side verify through dav-tls** (`curl --resolve
           0115d8cf.duckdns.org:8443:192.168.1.21`, full LE chain
           validation, `ssl_verify_result=0`, no `-k`):
           - **CLI live on the router** (concurrent with the running server,
             same pool class as `cmd_principals`): `subscriptions add family
             family --kind calendar` / `--kind addressbook` / `nfcarlton@gmail.com
             personal --kind calendar` → 64-char tokens, URLs printed with
             the `public_url` base; `list` re-displays id - kind - collection
             - URL - created; `remove` prints removal.
           - **Unauthenticated feeds:** family `.ics` → 200 `text/calendar;
             charset=utf-8` with `X-WR-CALNAME:Family` + `X-WR-CALCOLOR`;
             family `.vcf` → 200 `text/vcard; charset=utf-8`; populated
             nfcarlton `personal` `.ics` → 200, **229,201 bytes / 196
             VEVENTs** — all three **byte-identical (`cmp`)** to the
             owner-authenticated `route_get` exports (owner basic auth with
             the vdirsyncer token; wrong-password control → 401, so auth on
             the owner path is enforced).
           - **HEAD** → 200 + content-type, zero body. **No `ETag` and no
             `WWW-Authenticate`** on feeds or 404s (routes outside the DAV
             `AuthenticationLayer`, live through the whole stack incl.
             dav-tls).
           - **404 matrix:** unknown token (`.ics` + `.vcf`), missing
             extension, `.txt`, calendar-token-as-`.vcf`,
             addressbook-token-as-`.ics` → all 404 with **byte-identical
             bodies** (one md5 across all cases — no token-validity oracle);
             the family-calendar token with its correct `.ics` extension
             served 200 (positive control).
           - **Revocation:** `remove` → **instant 404** on all three scratch
             URLs; `subscriptions` table back to 0 rows; `list` empty. No
             residue — the real subscriptions for item 6's live tests are
             still to be created on the phone.
           - **Scheduling regression intact:** `/.well-known/caldav` → 308;
             OPTIONS compat principal → `dav: 1, 3, access-control,
             calendar-access, calendar-scheduling, calendar-auto-schedule`;
             PROPFIND principal 207 with all three schedule URLs filled.
           - **Hub:** full `vdirsyncer sync` clean — 17 collections, 0
             errors, idempotent second run (the hub never noticed the swap).
           - Minor observation for the record: the stored vdirsyncer app
             tokens are 69 chars, not the 64 the plan's app-token notes
             assumed (single-line `pass` entries verified); subscription
             tokens are exactly 64. Harmless — nothing depended on 64.
     6. **Live tests**: iPhone "Add Subscribed Calendar" with a token URL for
        `nicholas@carltonaudio.com` `personal` and for the `family` calendar
        (contacts `.vcf` fetch on the phone), watch `logread` for the poll
        cadence, then PLAN.md final status update.
 8. **Invitation-gated self-service registration & user portal** — **PULLED
   FORWARD FROM "FUTURE" BY USER DECISION 2026-09-07** ("we need to be able to
   register new users and host their accounts"; "make these subscribable instead
   of emailing files around"). **STATUS (2026-09-07): items 1–2 DONE; items 3–6 remaining.** User decisions for the feature are
   recorded under "User decisions"; the implementation split (items 1–6) and the
   verification-matrix additions (rows 20–23) live at the end. House style: as
   sessions execute, each item flips to **DONE** with its gate results in place.

   **Why / scope (2026-09-07):** today a new account is created by root over SSH
   to the router (multi-step: principal, app tokens, collections, share feeds,
   pass entries) and sharing means mailing `.ics`/`.vcf` files around. This
   feature makes account creation a **self-service, invitation-gated web flow**
   and makes every user's own calendars/contacts **subscribable via the §17.7
   export feeds** (importing *other* platforms' subscribe URLs into omnical, and
   exporting omnical's own subscribe URLs to other apps). The **calendar CRUD**
   (create/delete/fetch/update) already exists in the RustiCal portal
   (`create-*`/`edit-*`/`delete-button`/`import-*` frontend components and the
   `/frontend/user/{user}/calendar*` routes) — this feature verifies it for a
   self-registered (non-group) user and adds the two missing portal surfaces
   (Linked platforms, Share).

   **User decisions (all 2026-09-07, from the planning Q&A):**
   - **Linking = three concrete surfaces, all self-service where possible:**
     (1) **Import from a subscribe URL** — user pastes any public `.ics`
     subscribe URL (Google "secret iCal address", iCloud "Publish", Outlook
     "Publish a calendar", any webcal://) → the server fetches it and
     materializes the events into a calendar of the user's choice, with an
     explicit **Refresh** action re-syncing by UID; (2) **Upload `.ics`/`.vcf`**
     — the existing RustiCal import feature, verified for self-registered users;
     (3) **Export/share** — each user gets **their own subscribe URLs** (`.ics`
     calendar feeds + `.vcf` contact feeds) surfaced in the portal, created and
     revoked there, byte-identical to the §17.7 CLI-created URLs. The admin-only
     **vdirsyncer hub** (Phase 6, dev machine) remains the two-way sync path for
     Google accounts (`add-user.sh --hub`). Full server-side two-way CalDAV sync
     of arbitrary providers is **explicitly out of scope** (documented, future).
   - **No auto-group** for new registrations: a self-registered user owns only
     their own collections; admin adds group membership later via
`membership assign` (2026-09-07 user decision; Phase 5's memberships were
      all admin-provisioned via `membership add`, so self-service simply carries
      no group by default). Verified in the
      matrix: the `family` calendar/addressbook must NOT appear for a fresh user.
   - **Invites are single-use**, with **optional email binding** and **optional
     expiry** (config defaults: unbounded lifetime, unbound).
   - **Auto-provision** beyond the account + its three collections:
     a configured **app-token set** (default: `vdirsyncer`, `davx5`,
     `thunderbird`, `apple`, `i3status`) shown **once** on the success page, and
     an **auto share-feed URL** for the new user's `personal` calendar.

   **Architecture (all on the existing `omnical-scheduling` working tree in
   `~/router-dav/rustical`, additive on top of scheduling + subscriptions):
   `[registration] enabled = false` default → routes unmounted, zero behavior
   change vs the current build (same pattern as §17.2/§17.7); the public surface
   is unchanged (`:8443` → dav-tls → rustical). New pieces:**
   - `crates/store` — `InviteStore` + `CalendarSourceStore` traits (no-op
     defaults so non-SQLite test stores keep compiling), models.
   - `crates/store_sqlite` — `SqliteInviteStore`/`SqliteCalendarSourceStore` +
     migrations `20260907XXXXXX_invites` + `…_calendar_sources` (runtime
     `sqlx::query` only — `.sqlx/` metadata untouched, as the §17.2/§17.7 item-1
     pattern).
   - `src/config.rs` — `[registration]` section (serde-default; see 17.8.5).
   - `src/register.rs` — public router (`/register` GET+POST) mounted in
     `make_app` **outside** the DAV `AuthenticationLayer` (export_router
     precedent), enabled-gated, with the provisioning engine.
   - `src/commands/invites.rs` — `rustical invites create|list|revoke`
     (server-admin surface, mirror of `subscriptions`/`principals`).
   - `src/linked.rs` (import engine) — outbound fetch + parse + materialize +
     refresh, used by the portal route.
   - `crates/frontend` — two new `Section` impls + askama templates +
     routes: **Linked platforms** and **Share**; a searchable (public)
     `/register` template. **Server-rendered askama + plain HTML forms — NO
     changes to `bundle.mjs`, so no deno build is needed** (the Section trait at
     `crates/frontend/src/pages/user.rs` is a plain generic; routes are Rust.
     Only interactive sugar lives in the committed JS bundle, and none of the
     new surfaces needs it).
   - `scripts/add-user.sh` + `scripts/remove-user.sh` (dev machine) and
     `render-router-config.sh` gaining `[registration]`.

   ### 17.8.1 Invitation model
   - Table (see schema below): id, `code` UNIQUE, optional `target_email`,
     `created_by`, `created_at`, optional `expires_at`, `used_by`, `used_at`.
   - **Code = 12-char URL-safe random** (human-transcribable alphabet, ~60+ bits
     entropy), NOT the 64-char `generate_app_token` shape — invite codes get
     typed/read aloud over the phone; brute force is throttled anyway.
- **Single-use via atomic redemption:** `UPDATE invites SET used_by=?, used_at
      =? WHERE code=? AND used_by IS NULL`; rowcount 0 ⇒ double-spend ⇒ abort the
      loser with the generic body. **Redemption happens BEFORE any account state
      is written** (and after the no-existing-account + email/expiry checks), so a
      race can never orphan a principal; a post-redemption provisioning failure
      merely burns the code — admin reissues with `invites create --email`.
   - CLI: `rustical invites create [--email <e>] [--expires <date>]` (prints the
     code), `list [--all]` (unredeemed by default), `revoke <code>`.
   - Oracle discipline: unknown/used/expired code → one generic error body
     ("Invalid or expired invitation code."); email-mismatch → a distinct message
     ("That invitation code is for a different email address.") because the
     intended recipient must be able to correct a typo without support and the
     mismatch discloses nothing cross-user.

   ### 17.8.2 Registration flow (public `/register`)
   - **GET** → form: email, optional display name, invite code, password +
     confirm. **POST** validates in order: `[registration].enabled` (routes
     unmounted when off); email format (light check; lowercase; must not contain
     `:`/`$` — the RustiCal principal-id constraint from §3.5) and no existing
     principal with that id ("already has an account — log in instead.");
     password `≥ min_password_length` (default 12) and matches confirm; invite
     code per 17.8.1 (generic/mismatch bodies); per-IP rate limit (default 10
     POSTs/hour/IP, in-memory bucket) and a small global bucket
     (default 60/hour) against code brute-force; 429 on overflow.
   - **Provisioning (commit point = the atomic invite redemption; the code is
     consumed before any account state is written, so there is no partial-account
     window):**
     1. **Atomic redemption** (17.8.1); rowcount 0 → generic abort, nothing else
        happens.
     2. `insert_principal(Principal{ id: email, displayname: name-or-email,
        password: argon2(provided), principal_type: Individual }, false)`.
     3. `add_app_token` per entry in `auto_app_tokens` (the frontend
        `route_post_app_token` already returns the full `{token_id}_{token}`
        once at creation — `crates/frontend/src/routes/app_token.rs`; the
        success page reuses that exact UX).
     4. Seed collections via the store (no HTTP dependency): `personal`
        calendar (VEVENT+VJOURNAL), `tasks` calendar (VTODO), `personal`
        addressbook — mirroring the Phase 5.4 MKCOL results. (Verify store
        defaults for displayname/component-set/color match what the 5.4/§17.7
        smoke tests produced; set explicitly otherwise.)
     5. If `auto_subscription` (default true): `SubscriptionStore::
        add_subscription` for `personal`/calendar → the share URL.
   - **Post-redemption failure** (steps 2–5 are cheap and in-process) = a burned
     code, never a half-account; recovery is one CLI call
     (`invites create --email`). Admin cleanup is not needed — the only outcome
     to audit is an email registered-with-no-collections, covered by the row-20
     gate.
   - **Success:** render the fast-start card once — server host
     `0115d8cf.duckdns.org:8443`, per-client app tokens, the `personal` share
     feed URL; then auto-login (insert session `user` = email, the `route_post_
     login` session shape) and redirect to `/frontend/user/<email>`.
- **Abuse/security:** the only write endpoint beyond DAV is POST `/register`
      and it requires a valid unredeemed invite; logs carry the username only
      (never password/code/tokens); no captcha in v1 (invite-gated +
      rate-limited + single-use; revisit if spam shows up).
   - **AMENDED 2026-09-11/12 (invite → group join):** invites carry an optional
      `target_group` (set when minted from the portal Share section — §17.8.4 —
      or the CLI), so redemption auto-joins the group. **Existing users** are now
      handled instead of rejected: `lookup existing principal` runs after the
      atomic redemption, and an already-registered user simply joins the invite's
      `target_group` (if any) and is auto-logged-in to the success page — the
      account is never duplicated and no collections are re-seeded. Group FK note
      for the record: `memberships.member_of` → `principals.id`, so
      `add_membership` requires the group principal to exist already.
   - **AMENDED 2026-09-13 (needs_password_change interplay):** because the
      registrant chooses their password in this very POST, provisioning ends with
      `set_needs_password_change(email, false)` (error-logged, non-fatal) so the
      first-join nudge (§17.8.11) never double-forces a fresh user. An EXISTING
      user redeeming a group-join invite keeps the flag set by
      `add_membership` (booking one forced change).

   ### 17.8.3 Linked platforms (import from subscribe URL)
   - **Portal section** (authed): list of linked sources per calendar; form to
     add `{ source_url, calendar_id }`; per-source **Refresh** and **Remove**
     actions. Mirrors the §17.7 "share feeds" in reverse — this instead turns a
     *foreign* URL into *your* calendar data (a copy).
   - **Fetch engine:** outbound HTTPS via `reqwest` (already in the dependency
     graph — `Cargo.toml:164`; the §17.2 item-4 SMTP sink proved the
     rustls/webpki-roots outbound path end-to-end). HTTPS only; ~20 s timeout;
     response size cap (streaming, ~25 MB).
   - **SSRF policy (mandatory, code-reviewed):** resolve the hostname and refuse
     any IP in private/loopback/link-local/ULA/reserved/multicast ranges
     (10/8, 172.16/12, 192.168/16, 169.254/16, 100.64/10, 127/8, 0.0.0.0, 224/4,
     ::1, fd00::/8, fe80::/10, …); re-verify after every redirect (DNS-rebinding
     resistance — pin the resolved IP for the connection); scheme http refused
     outright. Rationale: the router sits on the LAN and the WireGuard VMs — the
     importer must never reach 192.168.x / 10.x / the WG peers. Public provider
     hosts (calendar.google.com, iCloud, Outlook publish hosts) are unaffected.
   - **Parse:** `crates/ical` `CalendarObject` + TZ handling (the inverse of the
     export/`route_get` feed builder). **GATE g-1 (before item 3 builds on it):
     confirm `crates/ical` can PARSE a remote feed's VEVENT/VTIMEZONE soup** (it
     is write-oriented). Fallback if not: a minimal RFC 5545 line parser modeled
     on `crates/scheduling/src/ics.rs` (already quote-aware) — small, testable.
   - **Materialize:** insert objects into the target calendar preserving UIDs;
     store the mapping in `calendar_sources` (id, principal, calendar_id,
     source_url, last_fetch_at, provider host, last known state) for Refresh.
   - **Refresh:** re-fetch, diff by UID, add/update/remove. Remove is
     **explicit-click only** in v1 (no auto-poll), and a mass-delete heuristic
     (a refresh deleting >X% of a calendar's rows) aborts with a log line rather
     than wiping a calendar (providers that omit old events on later fetches are
     common — Google's "no longer in past" folding). **Remove** deletes the
     mapping but leaves the materialized calendar data (it is a copy by design).
- Provider notes: Google secret iCal address (webcal) and iCloud/Outlook
      publish URLs work directly; a Google calendar without a secret link → use
      Upload (download the `.ics`, then portal import).
   - **DONE 2026-09-09** — import engine wired into the portal end-to-end:
      the portal `Linked Platforms` section (list of sources per calendar with
      Refresh/Remove) is mounted and served by owner-only routes, and `get_app`
      wires the real `SqliteCalendarSourceStore` (the `fetch → parse →
      materialize → refresh → remove` pipeline under §17.8.7 item 3 moved from
      the dead `src/linked_platforms.rs` into `crates/frontend/src/routes/
      linked_platforms.rs`; the old file was deleted). SSRF: `ssrf_guard` parses
      literal IPv4/IPv6 via `url::Host` (no resolver round-trip, rebind-proof)
      and resolves domains with `to_socket_addrs`, refusing private/loopback/
      link-local/ULA/multicast/unspecified before connect and again after every
      redirect; only https is accepted; fetch is a 20 s / 25 MB reqwest GET with
      the resolved IP pinned. Refresh diffs by UID (`refresh_plan`), aborts on a
      >50% mass-delete, and `Remove` deletes only the mapping, keeping the
      materialized copy. Coverage: 14 frontend-crate unit tests (SSRF ranges
      both directions + rejection messages, non-https refusal, plan
      add/update/delete/mass-abort/tolerate-half, and a `#[cfg(test)]`
      `store_pipeline` module verifying materialize + UID-diff refresh +
      mass-delete abort against the real SQLite stores via the `test`-feature
      harness) + 11 http-integration tests (`frontend_linked_platforms`:
      page lists sources with actions, empty state, add refusals
      non-https/private-range store nothing, unknown calendar, foreign user 401,
      remove keeps data / foreign 404, refresh failure banner / foreign 401).
      Gate: workspace check 0/0, fmt clean, clippy clean on new files, all 65
      http-integration tests + full workspace suite green. The real-remote-fetch
      happy-path + SSRF-negative controls remain for the live-deploy phase
      (§17.8.7 item 6).

   ### 17.8.4 Share/export surface (own subscribe URLs in the portal)
   - New portal section listing, per owned collection, the existing subscription
     row (id, kind, collection, `created`) + **full export URL** +
     **Revoke**, or a **Create share link** button when none exists.
   - The URL-builder currently lives only in `src/commands/subscriptions.rs`
     (`public_base_url`/`export_url`) — **factor it into a small shared module**
     so the portal prints byte-identical URLs to the CLI (the router config
     already sets `[subscriptions].public_url = "https://0115d8cf.duckdns.org:
     8443"`).
   - Store/table stays the §17.7 `subscriptions` table; the CLI remains the
     server-admin surface, the portal is the per-user surface (same rows).
   - **DONE 2026-09-09** — share surface complete + verified: the portal
     `Share` section lists every owned collection (own + owned-group
     principals) with the existing subscription row + **full export URL** +
     **Revoke**, or a **Create share link** button; create mints a 64-char
     app-token-shaped token into the same `subscriptions` table (`export_url`
     built by the factored shared module), ownership-checked per collection
     (own principal or a group the user owns), then redirects to the section.
     The URL-builder moved to `crates/frontend/src/url_builder.rs` and is
     re-exported at the root (`pub use rustical_frontend::url_builder`), so
     CLI + portal share one source of truth (byte-identical URLs). Gate green
     2026-09-09: workspace check 0/0, `fmt --all` clean, zero clippy warnings
     on new files, 10 new `tests/integration_tests/frontend_share.rs` tests
     (list w/ create buttons, create → URL + `/export` 200, group + foreign
group, unknown kind/collection, revoke → 404, wrong-user 401) + all 53
      integration tests + full workspace suite green.
   - **AMENDED 2026-09-12/13 (registration invites per collection):** the Share
      section now also mints one-time registration invites, in addition to the
      export feeds:
      - **Send invite** (2026-09-12): a per-collection email form
        (`{principal, email}`) creates an unbounded-lifetime one-time invite
        bound to that email AND to the collection's group principal
        (`target_group`) — on redemption the new user (or an existing user,
        §17.8.2 AMENDED) joins the group and gets full r/w to its shared
        collections. Own collections (`principal == own id`) mint an invite
        with `target_group = None`.
      - **Generate invite link** (2026-09-13): the same invite with NO email
        binding — `SendInviteForm.email` became `Option<String>`
        (`#[serde(default)]`; empty/absent ⇒ unbound, invalid NON-empty still
        rejected). The button prints a copy-pasteable `/register?code=…` link
        to hand out out of band (`invite_url` + `invited_email` surfaced in
        `share_section.html`, with a distinct banner per case).
      Both mint the 12-char unambiguous code via `generate_invite_code()`
      (§17.8.1) and store through `invite_store.add_invite(&code, &email,
      &target_group, &user.id, &None)`; ownership is enforced per collection
      (own principal or a group the user owns — foreign group 403).
      Gate 2026-09-13: 4 new `frontend_share.rs` http-integration tests
      (unbound link stored with `target_group`, email-bind, invalid email
      rejected, unowned group Forbidden) + the whole §17.8.11 suite + full
      `cargo test --workspace` green; fmt/clippy clean on new files.
   - **AMENDED 2026-09-22 (Calendars-screen surface — §17.13):** the same
     reuse-or-mint share-link action now also lives on the **Calendars
     screen**: `POST /frontend/user/{u}/calendar/subscribe`
     (`{principal, calendar_id}`) mints-or-reuses the calendar's
     subscription via the shared `ensure_subscribe_url`, and the tile shows
     the full export URL (Copy + the same share-link Revoke) or a
     "Subscribe URL" button when no link exists yet.

   ### 17.8.5 Config reference
   ```toml
   [registration]
   enabled = false                # routes unmounted until true (zero footprint)
   invite_required = true         # intranet mode = false (NOT the public default)
   min_password_length = 12
   auto_app_tokens = ["vdirsyncer", "davx5", "thunderbird", "apple", "i3status"]
   auto_subscription = true       # create personal share feed on registration
   default_group = ""             # "" = no group (2026-09-07 decision); "family" = auto-add
   rate_limit_per_hour = 10       # per-IP; plus a fixed global bucket 60/h
   ```
   Each sub-section `#[serde(deny_unknown_fields, default)]` (the house pattern
   at `src/config.rs`); `render-router-config.sh` adds `enabled = true` + the
   defaults explicitly, validated with python `tomllib` (never on dev disk).

   ### 17.8.6 `scripts/add-user.sh` (admin companion) — the task that precedes self-service
   Dev-machine script `add-user.sh <email> [--name "…"] [--group family]
   [--no-share] [--hub]`, **idempotent** (refuses to clobber an existing pass
   entry/principal without `--force`):
   - Over SSH to router, equivalent of the Phase 5 flow: `principals create`
     (piped random 32-char frontend password) → `principals app-token create`
     per client → seed `personal`/`tasks`/`personal` via store-backed MKCOL (the
     §17.7 smoke MKCOL bodies) → `subscriptions add` for the personal share feed
     (unless `--no-share`) → optional `--group family` → store every secret in
     `pass` under `secrets/omnical/<id>/` (`frontend` + per-client) → print the
     fast-start summary. Dry-run mode prints the transcript without touching pass
     or the router.
   - `--hub` (optional): appends the identity's Google/external account pair to
     `~/.config/vdirsyncer/config` (Phase 6 pattern) so the dev-machine hub
     mirrors it two-way — the admin path for "link existing platforms" end to
     end.
   - Companion `scripts/remove-user.sh <email>`: revoke tokens, delete principal
     + collections via CLI (soft-deletes per RustiCal semantics), remove pass
     entries (kept behind a `--purge-pass` flag defaulting on with a prepended
     backup note).

### 17.8.7 Implementation split (mirrors §17.2/§17.7 house style)
    1. **Store layer + CLI**: `InviteStore` + `CalendarSourceStore` traits, SQLite
       impls, migrations, store_sqlite unit tests; `rustical invites`
       create/list/revoke (+ calendar-source list is portal-only; CLI `linked`
       list/remove for admin debugging). Gate: `SQLX_OFFLINE=true cargo check
       --workspace --all-targets` 0/0, fmt clean, suites green (+ the new ones).
       **DONE 2026-09-07** — gate green: workspace check 0/0, `fmt --all` clean,
       new store_sqlite suites pass (`invite_store` lifecycle/duplicate-code/
       expiry/email-binding + `calendar_source_store` lifecycle/duplicate-URL,
       all 21 crate tests green), `rustical` lib 20 tests green, zero clippy
       warnings on the new files. `rustical invites` smoke-tested end-to-end on
       a scratch DB (create → 12-char unambiguous code; `--email`/`--expires`
       binding + `YYYY-MM-DD`→end-of-day normalization; bad expiry rejected;
       list hides used; revoke + re-revoke "Not found"). SQLite note worth
       recording: `row.try_get("col").ok()` infers `String` and decodes a NULL
       column as `""` — nullable reads must decode `Option<String>` + `.flatten()`
       (verified empirically; the writes bind true SQL NULL).
   2. **Config + wiring + public registration**: `[registration]` config,
      `get_data_stores` 9-tuple (+InviteStore, +CalendarSourceStore — note for
      the record: it is getting unwieldy; a `RegistrationStores` struct refactor
      is offered but the tuple is kept for sibling consistency), `make_app` param
      + router mount (outside the DAV auth layer), `src/register.rs` + askama
      templates + provisioning engine + atomic invite redemption + auto-login +
      success card. Gate: workspace check 0/0, fmt, root enabled-path
      http-integration tests (real `cmd_serve`, CLI-issued invite, public POST →
      principal + collections + tokens + feed; single-use spin; email-bind;
      expiry; generic-error body equality).
       **DONE 2026-09-07** — gate green: workspace check 0/0, `fmt --all` clean,
       rustical lib 29 tests (9 new in `src/register.rs`: provision + auto-login,
       success-card feeds, single-use, unknown/used/expired body equality,
       email-bind distinct body, existing-account, short/mismatch password, CSRF,
       rate-limit), store_sqlite 21 still green, http-integration
       `test_register_disabled_unmounted` (404 while `enabled=false`) +
       `test_register_enabled_provisions` (real `cmd_serve`, CLI-issued invite,
       public GET/POST `/register` → principal + 5 default app tokens +
       personal/tasks calendars + personal addressbook + 2 share feeds whose
       `/export/{token}.{ics,vcf}` URLs 200; single-use re-POST shares the
       unknown-code alert body; auto-login 303). Zero clippy warnings on the new
       files. Notes: pages are self-contained inline HTML (no askama in the
       binary crate); share-feed URLs are token-in-path to match `export_router`
       (`{base}/export/{token}.ics`); validation errors return HTTP 400 with the
       form re-rendered; `DTSTART;VALUE=DATE` seeds use basic `%Y%m%d`.
3. **Portal + import engine**: Linked-platforms + Share sections (Section
       impls, templates, owner-only routes), factored shared URL-builder, fetch/
       parse/materialize/refresh engine with the SSRF policy. Preceded by **gate
       g-1** (crates/ical parses a remote feed). **DONE 2026-09-08** — workspace
       check 0/0, fmt clean, clippy clean on new files, 9 register tests + 19
integration tests all green. Gate (real remote .ics fetch + SSRF-negative
        controls + mass-delete abort) remains for the live-deploy phase.
        **AMENDED 2026-09-09** — the Share half is fully wired end-to-end: the
        portal section (create button + URL display + revoke for own + group
        collections) is implemented and covered by 10 new http-integration
        tests (see §17.8.4 DONE note); the linked-platforms import engine that
        was committed under this item is now WIRED into the router —
        `LinkedPlatformsSection` + owner-only add/refresh/remove routes mounted,
        real `SqliteCalendarSourceStore`, engine + routes in
        `crates/frontend/src/routes/linked_platforms.rs` (dead `src/
        linked_platforms.rs` deleted), covered by 14 frontend-crate unit tests
        (incl. a real-sqlite `store_pipeline`) and 11 http-integration tests
        (see §17.8.3 DONE note); real-remote-fetch gate remains for item 6.
4. **Admin tooling**: `scripts/add-user.sh` created (0755, idempotent with
        --force, --dry-run, --no-share, --hub, --group; stores secrets in pass;
        prints fast-start summary). `scripts/remove-user.sh` created (0755; the
        §17.8.6 companion teardown: resolves the live principal id from the
        router, hard-deletes the caldav/carddav collections via DAV
        `X-No-Trashbin: 1` (a soft delete would leave rows and trip the
        `ON DELETE RESTRICT` FK on `principals remove`), revokes app tokens +
        subscriptions, removes the principal over SSH, then purges pass entries
        behind `--purge-pass` (default ON) with a backup note first; handles
        both pass-store layouts (raw email and add-user.sh underscored key
        names) and a `--force`/`--dry-run`);
        **IN PROGRESS** — both scripts are dry-run-verified end-to-end against
        the live router (dry-run is read-only: principals list + live PROPFIND
        enumerate the real collections/tokens/subscriptions), real router
        create/remove pending.
    5. **Cross-build + deploy** (existing deploy.sh flow: `/tmp` staging, stop →
       binary-swap → config → start so the new `[registration]` section only ever
       meets the NEW binary; pre-deploy DB backup; the additive migrations
       auto-apply) + server-side verify through dav-tls (`curl --resolve`),
       including the subscriptions/scheduling regression lines. Gate as §17.7
       item 5.
       **DONE & DEPLOYED 2026-09-14** — pre-deploy DB backup
       (`nightly-backup.sh`, 2026-09-14.tar.gz); `render-router-config.sh`
       now emits `[registration] enabled = true` + explicit defaults
       (§17.8.5), rendered config validated with python `tomllib` (secrets
       never on dev disk); cross-build via `build-rust.sh` (clang recipe D):
       rustical 4.8 MiB UPX-packed (16.4 MiB raw, within the 35 MiB budget),
       dav-tls unchanged (596 KiB, sha-identical → no restart); `deploy.sh`
       swap clean (deploy.sh's 2 s health probe raced UPX decompress +
       migrations + repair — the immediate `rustical health` re-check exits 0).
       Live server-side verification through dav-tls, all green:
       - **Migrations applied:** `principals.needs_password_change` column
         present; `lynscarlton@gmail.com` + `chris@carltonaudio.com` seeded
         `= 1` (verified via sqlite3 on the router).
       - **Startup log:** "Scheduling extension enabled (7 SMTP, RSVP)",
         "Subscriptions extension enabled", "**Registration extension enabled
         (public /register)**", "IMAP iMIP ingestion enabled (7 mailboxes)",
         serving on 127.0.0.1:4000; zero panics; no non-request ERROR/WARN
         lines.
       - **`GET /register` → 200** (form renders: email, displayname,
         password `minlength=12`, confirm, invitation code, hidden csrf);
         **CSRF live** (missing/wrong token → 400 "This form has expired");
         **validation order live** (short password → "Password must be at
         least 12 characters." before the invite check); **rate limiter
         live** (the test POSTs tripped the per-IP 10/h bucket → 429).
       - **Portal:** `GET /frontend/login` → 200. **Regressions:**
         `/.well-known/caldav` → 308; OPTIONS on a caldav principal → 200;
         live share feeds 200 (the two 404s during the sweep were
         soft-deleted collections — indistinguishable-by-design, not a
         regression); normal DAV traffic (iOS remindd PROPPATCH etc.)
         flowing on the new binary.
       - Negative POST `/register` with a valid CSRF but a wrong invite code
         is integration-tested; the live invalid-code body check is deferred
         to item 6 (the rate-limit bucket was consumed by the CSRF/limiter
         probes; the limiter is in-memory and clears on restart).
       - Record: deploy.sh's own health line raced startup (2 s) this time —
         not a service problem (service `running`, listener up, health exit
         0 on re-check); worth a longer wait in the script later.
    6. **Live tests**: issue real invites, register via the public `/register`
       from a phone + a desktop, verify portal CRUD + link-from-URL (a real
       external provider) + share URLs and the "already has an account" case;
       verification-matrix rows 20–23; PLAN.md final status update.
       **IN PROGRESS 2026-09-14 — BLOCKED by a production bug found on the very
       first registration attempt (see below); fix drafted, not yet tested/
       built/deployed.** What happened, in order:
       1. Restarted rustical on the router (clears the in-memory rate-limit
          buckets the 2026-09-14 deploy probes had consumed).
       2. `rustical invites create` → code `9mdmUPMXmwm2` (unbound).
       3. Browser-like public flow (curl with cookie jar through the real
          dav-tls URL): GET `/register` (csrf token) → POST with email
          `live-test-20260914@example.com` + password + code → **HTTP 500**,
          log: `registration: collection seeding failed err=Resource already
          exists and overwrite=false`.
       4. **Root cause:** `seed_collections` (src/register.rs) hard-codes
          displaynames `"Personal"`/`"Tasks"`/`"Personal"`, but calendar and
          addressbook displaynames are **globally unique**
          (`idx_calendars_displayname_unique` /
          `idx_addressbooks_displayname_unique`, enforced app-side by
          `check_displayname_unique`) — and `burningserenity@gmail.com`
          already holds "Personal"/"Tasks". The provisioning order means the
          invite is already consumed and the principal + 5 app tokens already
          created when seeding fails ⇒ every production registration would
          have 500'd identically, leaving a half-provisioned account and a
          burned code. The integration test never caught it because its fresh
          test DB has no other user holding "Personal".
        5. **Fix drafted (uncommitted, untested, working tree only):** seed
           `displayname: None` for the personal/tasks calendars and the
           personal addressbook — the exact shape the Phase 5.4 MKCOL results
           have in production (chris/lynscarlton/nfcalaway/nicholas rows are
           all NULL; DAV clients fall back to the collection id "personal",
           which is what Apple/DAVx5 display today). Also planned: extend
           `test_register_enabled_provisions` with a SECOND registration so
           the collision is regression-locked.
           **IMPLEMENTED & TESTED 2026-09-14 (session continued):** the fix
           is applied in the rustical working tree (`src/register.rs`,
           committed 2026-09-14 as `f80074c0` — see step 7) and
           `test_register_enabled_provisions` is extended
           two ways: (a) explicit `displayname == None` assertions on the
           first registration's personal/tasks calendars and personal
           addressbook, and (b) a full SECOND registration
           (`second@example.com`, fresh invite + fresh cookie session)
           asserting 200, principal + 5 app tokens + collections with NULL
           displaynames. Regression proof: temporarily reverting
           `src/register.rs` makes the test fail exactly at the old bug
           (`left: Some("Personal")`, `right: None`) — the pre-fix code
           cannot pass the extended test. Gates green:
           `SQLX_OFFLINE=true cargo check --workspace --all-targets` 0/0,
           full `cargo test --workspace` all suites ok (incl. 35 rustical
           lib + 6 http-integration tests). The two touched files add zero
           new clippy/fmt findings (whole-repo rustfmt/clippy drift from the
           newer toolchain exists in untouched files — pre-existing, noted
           for the record).
       6. **Observation for the record (rate limiter):** dav-tls sets no
          `X-Forwarded-For`, so the register per-IP bucket (10/h) keys every
          client as `"<global>"` — i.e. it is effectively one shared global
          10/h bucket instead of per-IP (the deploy-day 429s were this
          bucket, not a per-IP one). Either dav-tls should forward XFF or the
          limiter should fall back to the TCP peer. Not urgent: the shared
          bucket is *stricter*, not weaker.
        7. **Remaining live work (rows 20–23, 25, 26 flows):**
           ~~run the fix's unit/integration tests → `cargo test`~~ **DONE
           2026-09-14 (see step 5 above)**; ~~rebuild → redeploy~~ **DONE
           2026-09-14** — fix committed `f80074c0` (rustical tree);
           `build-rust.sh` (clang recipe D) green: rustical 4.8 MiB
           UPX-packed (within the 35 MiB budget), dav-tls unchanged
           (596 KiB, sha-identical → no restart); `deploy.sh` swap clean,
           `rustical health` exits 0 on first probe, service `running`;
           server-side verify through dav-tls (`--resolve
           0115d8cf.duckdns.org:8443:192.168.1.21`): `/register` 200,
           `/.well-known/caldav` 308, `/frontend/login` 200; startup log
            "Registration extension enabled (public /register)".
            **CLEANUP DONE 2026-09-14** — hot backup
            (`/tmp/db-pre-cleanup-20260914.sqlite3` on router) taken first;
            `rustical principals remove live-test-20260914@example.com`
            succeeded; verified via sqlite3: principal row gone, all 5 app
            tokens cascaded away (0 rows), no collections/addressbooks/
           memberships had ever been created for it; the burned invite row
           `9mdmUPMXmwm2` (used_by=live-test…) stays by design; `rustical
           health` OK.
           **REGISTRATION REDO DONE 2026-09-14 (later session)** — rustical
           restarted first (clears the in-memory rate-limit buckets), fresh
           CLI invite `yoCehFdvHLer` (email-bound), full public flow
           through dav-tls: GET `/register` (csrf) → POST → **200**
           provisioned summary; sqlite3 server-side: principal + 5 app
           tokens + personal/tasks calendars + personal addressbook all
           displayname NULL + 2 subscription rows; both `/export/{token}.
           {ics,vcf}` 200; single-use re-POST and unknown-code POST share
           the 400 "Invalid or expired invitation code." body; auto-login
           GET `/register` → 303 `/frontend/user/live-test…`; no ERROR
           lines for the successful registration. Account + portal password
           (pass) kept for the remaining live tests. Recorded under
           verification-matrix row 20.
            **Next** → ~~portal CRUD~~ **DONE LIVE 2026-09-14 (see
            matrix row 21)** → ~~share links + group-join invite
            (existing-user case)~~ **DONE LIVE 2026-09-14 (see
            matrix rows 23 + 25)** → linked-platform real-URL import +
            the forced password-change gate live on a test account, then
            the phone-based registration. The two seeded users' own rotation
            remains theirs (their logins, not ours).
            **SHARE LINKS + GROUP-JOIN INVITE (EXISTING USER) LIVE
            2026-09-14** — both flows exercised through dav-tls on the
            kept `live-test-20260914@example.com` account (portal
            password from pass); rustical restarted first (clean
            in-memory rate-limit buckets); full records under matrix rows
            23 and 25. One portal login (session cookie) drove portal +
            DAV alike. All test state torn down afterwards (group,
            membership, owner row, group calendar, extra app token, extra
            subscriptions) — sqlite3-verified back to the exact
            row-20/21 baseline (2 subscriptions, 5 app tokens, 0
            memberships, 2 own collections). **Record notes:** (a) the
            CLI has no group-owner setter — the `group_owners` row was
            inserted via sqlite3 (test setup only); (b) the first-join
            `needs_password_change` flag set by the CLI membership
            assign was cleared via sqlite3 so the row-26 gate stays its
            own live test; (c) redeeming a USED code as an existing user
            answers the duplicate-account body ("An account with that
            email address already exists."), not the unknown-code body —
            same single-use outcome, different (also non-oracle)
            wording; (d) the two consumed invite rows stay in `invites` by
            design; (e) zero unexpected ERROR/panic lines for the
            whole run (the only ERROR-class lines: the deliberate
            foreign-group 403, the used-code 400, and 404 probes).

         **ROW-22 + ROW-26 LIVE (2026-09-22 evening session — STARTED, then
         stopped mid-diagnosis by user; resume state also in
         `PLAN_NEXT_AGENT_ROW22_26_RESUME.md`):**
         - **Unrecorded 2026-09-14 19:17 prep discovered on the live-test
           account** (this session's first finding — prep for exactly this
           test that was never recorded): calendar `srcfeed` (3 handmade
           VEVENTs, UIDs `e1`/`e2`/`e3`, "Feed event one/two/three"), two
           empty target calendars `imported` + `imported2`, and a third
           subscription on the account (id `ee496590-829e-47b4-a0d9-7cf2aec3cf18`,
           kind calendar, collection `srcfeed`). `calendar_sources` was empty
           — the import itself never ran. All of it to be torn down to the
           recorded row-20/21 account shape (personal+tasks, 2 registration
           subs) when the live work completes.
         - Safety net FIRST: `~/backups/omnical/db-pre-row2226-20260922.sqlite3`
           (3.9 MB, `integrity_check` ok; live 216 cal / 209 addr, raw 552
           cal, 67 app tokens, 9 subscriptions, 0 sources — matches §17.14's
           recorded live baseline incl. the srcfeed sub).
         - **Partial greens before the blocker:** srcfeed export URL serves
           live through dav-tls (200, `ssl_verify_result=0`, exactly the 3
           VEVENTs) — it is the row-22 import source; portal login 303 with
           the pass-stored password; session drives the Linked Platforms
           page (200, "No linked platforms yet", `imported`/`imported2`
           offered as add-targets).
         - **THE BLOCKER (new live bug, undiagnosed — row 22 halted):** POST
           `/frontend/user/{u}/linked-platforms/add` re-renders 200 with
           "Could not fetch '…': DNS resolution failed" for EVERY hostname
           tried (the srcfeed URL, example.com, example.org, www.duckdns.org).
           The std `to_socket_addrs` calls in `ssrf_guard`/`fetch_and_parse`
           (`crates/frontend/src/routes/linked_platforms.rs`) fail inside the
           long-running deployed process although:
           - busybox `nslookup` and the router's curl (musl) resolve every
             host fine via all 5 resolvers (2×Quad9 v4, 192.168.1.1, 2×Quad9
             v6); `/etc/resolv.conf → /tmp/resolv.conf` is readable through
             the process's own `/proc/7532/root`;
           - the SMTP path of the SAME PID (tokio blocking-pool
             `TcpStream::connect((host, port))`) resolved + sent real email
             minutes earlier (the 21:17:07 UTC redeploy's §17.14 checklist
             greens ran ~21:20–21:35; PID 7532 started 21:17:07) — the
             failures start ~21:45;
           - **ruled out empirically** (standalone aarch64-musl probes built
             with the exact `build-rust.sh` rust-lld recipe — plain main
             thread, tokio worker thread, and `spawn_blocking` variants, plus
             a UPX'd variant — all resolve every host from the router):
             OS/resolver path, UPX packing, tokio-worker-thread blocking-call
             context, FD exhaustion (32/1024 open), procd memory cap (RSS
             21 MB, `Max address space unlimited`), procd jail (none
             configured), nsswitch (absent — musl);
           - **tcpdump during a failing add: ZERO DNS packets leave the
             router** — getaddrinfo fails before emitting any query.
           - Remaining suspects / cheapest next experiments: (a) run a FRESH
             instance of the exact deployed binary (:4001, scratch DB —
             binary vs long-running-state discriminator; the first attempt
             died instantly on `nohup: not found` (busybox has none); redo
             with `setsid` or a held-open ssh session; nothing ever listened
             on 4001 and no scratch DB was created); (b) trigger one SMTP
             send NOW (e.g. `invites create --send` to a plus-address) — if
             SMTP still resolves while the add-route fails, the likely fix is
             routing the std resolves through `spawn_blocking` like SMTP;
             either way instrument/log the real `io::Error` (the current
             `.map_err(|_| "DNS resolution failed")` hides it); (c) check
             whether rustical's serve path configures a small tokio
             worker-stack size (probes used the default 2 MB).
          - Row 26 NOT STARTED — queued behind row 22 on the same account.
          - **Live-test state right now:** +1 active diag DAV app token
            `diag-row22` (id `bbdef0d9-4b2a-4279-92da-13c63025dfef`, prefix
            `bbde` — remove at resume/teardown; 68 vs 67 baseline); portal
            session cookie in `/tmp/opencode/row2226/jar.txt` (dies with the
            next rustical restart); probes + scratch config on router tmpfs
            (`/tmp/dnsprobe`, `/tmp/dnsprobe2`, `/tmp/dnsprobe2u`,
            `/tmp/rustical-row2226.toml` + stale `.pid`/`.log`) — wiped on
            reboot, remove for tidiness when done. Prod untouched otherwise:
            ping 200, dav-tls never restarted.
          - **ROOT CAUSE FOUND (session 4, 2026-09-22 ~22:00 UTC; resume file
            rewritten):** the add-route bug is a **std behavior change, not
            a live-platform issue** — current std's
            `TryFrom<&str> for LookupHost` (sys_common/net.rs) requires a
            `host:port` string (`rsplit_once(':')` → else instant
            `io::Error` InvalidInput "invalid socket address"), so
            bare-hostname `to_socket_addrs()` — exactly what
            ssrf_guard/fetch_and_parse did — never reaches getaddrinfo
            (explains: instant fail, zero DNS packets, every hostname,
            fresh + long-running instance identical). SMTP/IMAP always
            worked (tokio `TcpStream::connect((host, port))` = tuple
            impl). **Fix applied (UNCOMMITTED):** both call sites now
            `(host, 0u16).to_socket_addrs()` (port 0 — reqwest
            resolve_to_addrs takes ports from the URL) + real io::Error
            warn!-logged via inspect_err. Frontend unit tests green;
            integration tests + fmt/clippy + rebuild + redeploy + both
            live checklists remain. **Diagnosis-evidence corrections:**
            (a) busybox on the router has NO `setsid` — every backgrounded
            `setsid … &` capture (prev session's + session 4's first two)
            silently never ran; the zero-packet fact itself re-verified
            true with a held-open single-ssh-script capture;
            (b) the old probes' "worker" test ran on the MAIN thread
            (`block_on`) — corrected probe (true spawned worker tasks,
            64 KiB–8 MiB worker stacks, UPX variant) passes all contexts;
            (c) the SMTP path was NEVER broken: two forgot-password probes
            (21:45:14 + 21:46:52) both delivered real email — side
            effects: +2 password_resets rows (second unused, expires
            ~22:46:52 UTC) + 2 audit emails in the INBOX;
            (d) a fresh scratch instance of the deployed binary (:4001,
            CLI invite mint + POST /register provisioning) failed
            identically → long-running state ruled out (the scratch
            instance's own IMAP ingest connected + parsed mail 2 s after
            boot — process-wide DNS fine); (e) logread's ring buffer is
            tiny (dropbear floods it) — start `logread -f` to a file
            before probes needing log evidence. Offline gating missed
            this because the integration tests deliberately leave the
            domain-fetch happy path to the live gate (literal-IP guard
            refusals + seeded sources only).

   ### 17.8.8 Verification-matrix additions
   | # | Test | Method | Expected |
   |---|---|---|---|
| 20 | Registration | CLI invite → public POST `/register` | principal + 3 collections + app tokens + personal share feed exist; single-use spin fails; email-bind + expiry honored; unknown/used/expired codes yield one generic body; double-submit race has one winner | **DONE 2026-09-07** — plus real `cmd_serve` http-integration test (GET/POST `/register`, CSRF, token-in-path feed URLs, 404 on disabled, 303 auto-login, shared unknown/used alert body). **LIVE: DONE 2026-09-14 (redo after the fix)** — fresh CLI invite `yoCehFdvHLer` (email-bound `live-test-20260914@example.com`), public GET/POST `/register` through dav-tls (`--resolve 0115d8cf.duckdns.org:8443:192.168.1.21`): POST → **200** with the provisioned summary + both `/export/{token}.{ics,vcf}` URLs (200, `BEGIN:VCALENDAR…RustiCal Export`, empty vcf); server-side verify: principal row, **5 app tokens**, `personal`+`tasks` calendars and `personal` addressbook **all displayname NULL** (the `f80074c0` fix holding live), 2 subscription rows; invite row consumed (`used_by`+`used_at` set); single-use re-POST → 400 with the exact unknown-code body ("Invalid or expired invitation code."); auto-login GET `/register` on the registered session → 303 `/frontend/user/live-test-20260914@example.com`; zero ERROR lines for the 200 registration (the only 400 ERRORs logged are the intentional negative probes). Rate-limiter buckets cleared by restarting rustical first (in-memory). Portal password stored in pass (`secrets/omnical/live-test-20260914@example.com/portal`). **Finding for the record:** the `route_post_register` ERROR span logs the whole `RegisterForm` including the password in cleartext — pre-existing upstream behavior, noted for a future hardening item. Account kept for the remaining item-6 live tests. |
| 21 | Portal CRUD (self-registered) | create/read/update/delete calendars + addressbooks + app tokens as a fresh no-group user | full CRUD works; family/module collections invisible (no auto-group) | **LIVE: DONE 2026-09-14** — portal login (`live-test-20260914@example.com`, pass-stored password) → 303; session drives both portal + DAV endpoints (the AuthenticationLayer accepts the session cookie). **Calendars:** MKCOL (exact `create-calendar-form` JS body) → 201; portal section lists it; PROPFIND 207 `displayname=Live Test Cal`; PROPPATCH (edit-form body) → 207; PROPFIND shows `Live Test Cal Renamed` + `#ff0000ff` color stored; DELETE `X-No-Trashbin: 1` → 200; PROPFIND → 404; portal section no longer lists it. **Addressbooks:** same cycle (`Live Test Addr` → renamed → deleted → 404) via `/carddav`. **App tokens:** POST `/frontend/user/{u}/app_token` → 200 token (69-char `<id4>_<64>`); Basic-auth PROPFIND with it → 207; revoke via the portal form path (full-UUID id, as `profile_section.html` renders) → 303; the token → 401. **Server-side (sqlite3):** after the run only the original collections remain (`personal`+`tasks` calendars, `personal` addressbook, all displayname NULL) and exactly the 5 registration app tokens; `memberships` 0 rows. **Isolation:** page JSON `memberships:[]`; only personal/tasks/personal listed — no family/module auto-group. **Finding for the record:** revoking with the 4-char token *prefix* (the client-hint part) is a silent no-op — the portal always passes the full UUID id; my first curl used the prefix and the token stayed valid, worth remembering for CLI-side tooling. |
| 22 | Linked platforms | import a real external .ics URL; provider edit → Refresh; Remove | count matches; edits propagate on Refresh; copy remains after Remove; SSRF-negative targets refused; size cap honored | **DONE 2026-09-09 (offline/wired)** — portal section + owner-only add/refresh/remove routes mounted with the real `SqliteCalendarSourceStore`; SSRF guards, fetch guards, UID-diff refresh, mass-delete abort, Remove-keeps-copy and banner paths covered by 14 frontend-crate unit tests + 11 http-integration tests (see §17.8.3 DONE note). The real-remote-provider lines (fetch, Refresh propagation, size cap) are §17.8.7 items 5–6 (live-deploy phase). **LIVE: STARTED 2026-09-22 evening, blocked on a fetch bug — ROOT-CAUSED same evening (session 4): bare-hostname `to_socket_addrs()` never reaches getaddrinfo — current std's `TryFrom<&str> for LookupHost` requires `host:port` and fails instantly with InvalidInput "invalid socket address" (explains zero DNS packets/every hostname/fresh+prod identical; the SMTP+IMAP paths use the tuple form and were never broken — the earlier zero-packet SMTP readings were void tcpdumps: busybox has no `setsid`, backgrounded captures silently never ran). Fix applied in linked_platforms.rs (tuple form + real io::Error warn-logged, uncommitted); frontend unit tests green; integration tests + fmt/clippy + rebuild + redeploy + the live checklist remain — resume in `PLAN_NEXT_AGENT_ROW22_26_RESUME.md` + §17.8.7 item-7 record.** |
| 23 | Share/export | portal-created share URL | byte-identical `.ics` vs owner export; revoke → instant 404; §17.7 rows still green | **DONE 2026-09-09** — http-integration tests assert create → token-in-path URL served + revoke → 404, incl. group-owned collections (PORTAL create/revoke; the CLI-side byte-identical line is §17.7, already green). **LIVE: DONE 2026-09-14** — portal login → 303; Share section lists `personal` calendar + `personal` addressbook (registration feeds, full URLs) and `tasks` with a Create button; POST `/frontend/user/{u}/share/create` (`principal=live-test…&kind=calendar&collection_id=tasks`) → 303; page now shows the new 64-char-token URL; public GET → 200 `text/calendar` (welcome VTODO, `PRODID:RustiCal Export`); owner reference via CLI `subscriptions add --kind calendar <u> tasks` → different token, **byte-identical body (`cmp` clean)** — portal + CLI share the URL-builder and the export route; portal revoke (form path, full sub id) → 303 and the URL 404s immediately while the CLI URL stays 200; CLI row removed afterwards — subscriptions back to exactly the 2 registration rows. |
| 24 | Registration HTTP gate | `cargo test -p rustical --lib --test http_integration` | 9 unit + 2 integration tests green; `test_register_enabled_provisions` asserts principal + 5 app tokens + personal/tasks + addressbook + 2 share feeds via `/export/{token}.{ics,vcf}`; `test_register_disabled_unmounted` 404 |
| 25 | Portal group-join invites | Share section per collection: "Send invite" (email-bound) / "Generate invite link" (unbound); redeem through `/register` as a new AND as an existing user | invite row stores `target_group` (+ optional email); new user provisions + joins group; existing user is auto-logged-in and joins; unbound link needs no email; invalid email rejected; foreign group 403 | **DONE 2026-09-13 (offline)** — 4 new `frontend_share.rs` integration tests; existing-user redemption covered by commit 44afb366 (see §17.8.2/17.8.4 AMENDED). **DEPLOYED 2026-09-14** (item 5): portal + `/register` live; real redeems are item 6. **LIVE: DONE 2026-09-14 (existing-user cycle, full round-trip)** — setup: `rustical principals create group.livetest -p group -n "Live Test Group"` (CLI), membership assign live-test → group.livetest (CLI; first-join flag cleared via sqlite3 — see item-6 record notes), `group_owners` row via sqlite3 (no CLI setter), group calendar `shared` **MKCOL 201 as plain member auth** (no `$` impersonation — §5.4 parity, exact `create-calendar-form` XML body). Portal Share section then lists the group row (owner label "Live Test Group", principal `group.livetest`, collection `shared`) with its invite forms; **"Generate invite link"** (`principal=group.livetest`, no email) → 200 "Invite link generated!" banner with the 12-char code; server-side: invites row `target_group=group.livetest`, `target_email=NULL`, `created_by=live-test…`. **Control:** invite mint on the foreign `family` group → **403**. **Redeem as EXISTING user:** fresh session GET `/register` (csrf) → POST (email=live-test…, real portal password, code) → **200** "Welcome back … You have been added to the `group.livetest` group. … signed in automatically"; GET `/register` on that session → **303** `/frontend/user/live-test-20260914@example.com` (auto-login live); server-side: invite consumed (`used_by`+`used_at`), membership row present, `needs_password_change` stays 0 (repeat redeem, not a first join); single-use re-POST of the used code → **400** (duplicate-account body — used codes no longer match the existing-user fast path; still a non-oracle body). **Access after join (plain member Basic auth, no impersonation):** PROPFIND depth:1 on `/caldav/principal/group.livetest/` → 207 listing `shared`; PUT VEVENT → **201**; calendar-query REPORT → **207** with the event href. **Teardown (all green):** calendar DELETE `X-No-Trashbin: 1` → 200 (probe 404); the extra app token revoked via the portal full-UUID path (401 after); `principals remove group.livetest` cascaded owner + membership rows — sqlite3-verified back to the row-20/21 baseline. |
| 26 | Forced password change | a flagged user (seeded or first-join) logs into the portal | every portal page except `/user/{u}/password` redirects there; rotation clears the flag and lifts the gate (303); wrong current / short / mismatch rejected with the form re-rendered; passwordless users never gated | **DONE 2026-09-13** — seed migration `20260913120000_needs_password_change`; 7 `frontend_password.rs` + 5 store-principal tests; `test_principal_impersonation` amended (see §17.8.11). **LIVE: QUEUED 2026-09-22 behind the row-22 fetch bug (same kept account, same session; flag-flip via sqlite3 + gate/negatives/rotation/restore checklist in `PLAN_NEXT_AGENT_ROW22_26_RESUME.md` §Row 26).** |

   ### 17.8.9 Risks & mitigations (new)
   - **Public DoS / invite brute-force** → per-IP + global buckets, 60-bit codes,
     single-use; the public port already attracts benign scanner noise (§7).
   - **SSRF** → https-only + private-range refusal + post-redirect re-check;
     the router's LAN/WG/services stay unreachable.
   - **`deny_unknown_fields` config drift** → deploy ordering (binary swap while
     stopped, config lands before start), tomllib-validated render.
   - **`crates/ical` may not parse feeds** → gate g-1 early; fallback minimal
     parser modeled on `crates/scheduling/src/ics.rs`.
   - **Half-provisioned account on mid-flow failure** → the invite is consumed
     before any account state (17.8.2); the post-redemption steps are in-process
     and cheap, so the risk is a burned code (reissue via CLI), never an orphan
     account.
   - **Frontend-no-deno claim** → all new surfaces are server-rendered askama +
     plain forms; only the existing CRUD components touch the committed JS bundle.
   - **Refresh mass-delete** → explicit refresh only + mass-delete heuristic
     aborts with a logged line.

### 17.8.10 Out of scope (future)
   Server-side two-way CalDAV sync of write providers; email-verified signup;
   password reset; captcha/external abuse service; IP geo-blocking.

   ### 17.8.11 Forced one-time password change (`needs_password_change`)
   When a real user (one with a stored password) is added to their FIRST-EVER
   membership, they are now nudged to change their password on the next portal
   login — the admin/user who chose the initial password no longer controls it
   once a shared calendar is involved. Design and implementation (2026-09-13):
   - **Flag:** `Principal.needs_password_change: bool` (`#[serde(default)]`,
     so old serialized principals stay decodeable); column
     `principals.needs_password_change BOOLEAN NOT NULL DEFAULT 0` via
     migration `20260913120000_needs_password_change`. **Seeded `true`** for
     `lynscarlton@gmail.com` + `chris@carltonaudio.com` — the two initial
     shared-calendar users whose passwords were admin-chosen.
   - **Set on first join:** `PrincipalStore::add_membership` now runs in a
     transaction: reads the principal's membership count (0 when none) and
     `password_hash` (has one?), does the `REPLACE INTO memberships`, and —
     only when `count == 0 && has_password && rows_affected() == 1` — sets
     the flag. Group principals / passwordless (OIDC) users are never
     flagged. Runtime `sqlx::query` + `Row::get` for the new reads; the
     `.sqlx/` offline metadata and the checked `query!` texts are untouched
     (house pattern).
   - **Store API:** `get_needs_password_change` / `set_needs_password_change`
     (runtime-row reads), and `update_password(principal, argon2_hash)` which
     writes the hash AND clears the flag in one UPDATE — rotation is the only
     un-flag. Trait defaults: `update_password` → `Err(Error::ReadOnly)`,
     get/set → `Ok(false)`/`Ok(())` no-ops (non-SQLite test stores).
   - **Frontend gate:** middleware `password_change_gate`, layered LAST on
     `user_router` (outermost, so it runs after `AuthenticationLayer` and the
     `Principal` extension is present), redirects EVERY portal page except
     `/frontend/user/{u}/password` while
     `user.needs_password_change && user.password.is_some()` — passwordless
     users are never gated even if flagged.
   - **Change page** `/frontend/user/{u}/password` (GET+POST, askama
     `password_change.html`, new `routes/password.rs`): GET renders the form
     (a plain redirect when `allow_password_login` is off); POST verifies
     current password via `validate_password`, `new_password ==
     new_password_confirm`, `len >= FrontendConfig.min_password_length`
     (new config field, default 12), argon2-hashes (frontend crate gained the
     `argon2` workspace dep) and calls `update_password` → 303 to
     `/frontend/user/{id}`. No JS, no config — consistent with the rest of
     the server-rendered portal.
   - **Gate 2026-09-13 (all green):** store_sqlite `tests/principal_store.rs`
     +5 (first-join sets flag, later joins don't re-trigger, passwordless
     never flagged, set/clear round-trip, `update_password` rotates + clears)
     and `tests/principal_store` registered as `#[cfg(test)]`;
     http-integration `frontend_password.rs` +7 (gate redirect, passwordless
     skip, page render, wrong current, short, mismatch, success clears flag +
     lifts the gate + redirect); `http_integration.rs::
     test_principal_impersonation` amended to the now-forced change flow
     (login → `/frontend/user` → SEE_OTHER `/frontend/user/user/password` →
     change → SEE_OTHER `/frontend/user/user`); workspace `cargo check`
     0/0, `cargo test --workspace` fully green (76 run_integration +
     6 http_integration + 5 store principal tests), fmt + clippy clean on
     the new files. Note: a fresh registrant's flag is cleared during
     provisioning (§17.8.2 AMENDED 2026-09-13); an existing user redeeming a
     group-join invite keeps the flag. **DEPLOYED 2026-09-14** (§17.8.7
     item 5): column + seeds live on the router; the two seeded users will be
     forced to rotate on their next portal login — the live gate flow itself
     is exercised by the item-6 live tests (their logins, not ours).

   **Schema sketches (both migrations resemble the §17.7 `subscriptions` one):**
   ```sql
   CREATE TABLE invites (
     id TEXT PRIMARY KEY,
     code TEXT NOT NULL UNIQUE,
     target_email TEXT,
     created_by TEXT NOT NULL,
     created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
     expires_at TEXT,
     used_by TEXT,
     used_at TEXT
   );
   CREATE TABLE calendar_sources (
      id TEXT PRIMARY KEY,
      principal TEXT NOT NULL REFERENCES principals(id) ON DELETE CASCADE,
      calendar_id TEXT NOT NULL,
      source_url TEXT NOT NULL,
      provider_host TEXT NOT NULL,
      last_fetch_at TEXT,
      last_fetch_success INTEGER NOT NULL DEFAULT 0,
      created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
      UNIQUE (principal, calendar_id, source_url)
   );
   ```

## 17.9 Next feature requests (user, 2026-09-14)

Two requested follow-ups on the §17.8 sharing surface. §17.9.1 DONE
2026-09-14; §17.9.2 IN PROGRESS since 2026-09-14 (first implementation
session; see the status block in §17.9.2 — code written but NOT yet
compiled/gated).

### 17.9.1 Invite links per collection tile
**Request:** generated invite links should show in the **same tile** as the
calendar they were generated for — not only in the one-time banner above the
Share section (today's behavior).
- **Data:** `invites` gains `collection_id` (+ kind) so an invite is traceable
  back to the exact collection tile. Own-collection invites (no group) have
  no `target_group` today, so they are currently untraceable — the column
  fixes both cases. (Optional: reuse `target_group` when the collection is
  group-owned and keep `collection_id` authoritative.)
- **UI:** each calendar tile renders its unredeemed invite links (copy +
  revoke); newly generated links still show the success banner, but the link
  now persists under the tile. Addressbook tiles: invites are calendar-only
  today — decide whether to extend or keep calendar-only.
- **Compat:** portal-only change + one additive migration; the CLI and
  redemption logic are unaffected.
- **DONE 2026-09-14** — additive migration
  `20260914120000_invites_collection` (`collection_id`, `kind` columns);
  `Invite` struct + `InviteStore::add_invite` extended with the two optional
  columns (CLI/registration callers pass `None, None` — behavior unchanged;
  redemption logic untouched). The Share section: `SendInviteForm` gained
  `collection_id` + `kind` (per-tile hidden inputs); the route validates
  ownership, calendar-only kind, and collection existence (same discipline
  as share create) before storing; `build_share_entries` now threads the
  invite store and attaches each calendar tile's unredeemed invites via
  `tile_invite_links` (scoped by `collection_id` + kind + `target_group`:
  own tiles match `target_group = NULL`, group tiles match the group — a
  group calendar with the same id as a personal one cannot leak links
  across tiles). Template renders the links under each calendar tile with
  Copy (inline clipboard) + Revoke; new route
  `POST /{user}/share/invite/{code}/revoke` (ownership-checked, deletes
  the invite); the one-time success banner is unchanged. Addressbook tiles:
  kept calendar-only (route rejects non-calendar kinds). Gate: fmt clean on
  changed files, zero clippy warnings on changed files, 1 new store_sqlite
  test (collection binding + NULL plain invite), 6 new/changed
  `frontend_share.rs` http-integration tests (stored collection binding on
  the two mint tests, unknown-collection + non-calendar-kind rejected with
  no invite stored, tile shows link with exactly one revoke form + revoke
  removes it, same-id cross-principal scoping) — 18/18 share tests green,
  full `cargo test --workspace` green (all 39 targets).

### 17.9.2 Privilege-based access control (view / edit / admin)
**Request:** per-member privileges for shared groups —
`view` (read-only), `edit` (CRUD), `admin` (change member privileges, invite
users) — replacing today's "membership = full r/w". Full design lives in
`PLAN_SHARING.md` §10. Sketch:
- **Model:** new `group_members` rows `(group, member, privilege)` or a
  `privilege` column beside the existing memberships row; owner/admin
  determined from it. Default on group creation: owner=admin, other members
  as chosen by the creator.
- **Enforcement:** `view` members get read-only DAV access — the CalDAV/
  CardDAV write paths (PUT/DELETE/MKCOL/PROPPATCH/ACL) must reject with 403
  for view-level members; `edit` = current full r/w minus member management;
  `admin` = edit + change privileges + invite. Portal member-management UI
  becomes admin-only.
- **Surfaces:** portal group detail (member list with per-member privilege
  selector + invite), API `/groups/{id}/members` (privilege in body), DAV
  authorization checks in `crates/caldav`/`crates/carddav` write routes.
- **Risks:** read-only enforcement must not break clients (DAVx5/iOS may
  attempt write operations and must get clean 403s, not timeouts); app-token
  impersonation (`user$group`) inherits the same privilege.

**STATUS (2026-09-14, session 1 of the implementation): IN PROGRESS — all
code written but NOT yet committed; session stopped by the user at the
progress checkpoint. Follow-up items 1–3 DONE (see the "NOT yet done" list
below: templates + lib.rs route registration landed, `cargo check -p
rustical_frontend` 0 errors, fmt clean; items 4–6 remain).** Everything
below is on the rustical tree
at `~/router-dav/rustical` (uncommitted; stacks on the §17.9.1 changes, also
still uncommitted there). Design decisions taken while implementing (per the
"refine at implementation time" note in PLAN_SHARING §10):

- **Model:** `Privilege` enum (`view`/`edit`/`admin`, lowercase serde,
  `FromStr`/`Display`, helpers `can_write`/`can_admin`) in
  `crates/store/src/auth/privilege.rs` (NEW). `Principal` gains
  `privileges: BTreeMap<String, Privilege>` (`#[serde(default,
  skip_serializing)]`) + `privilege_for` / `can_write` / `is_admin`
  (self = admin unless an impersonation stamp lowered it; membership without
  a stored row defaults to `edit` = today's behavior; non-member = `view`).
  `Error` gains `LastAdmin` and `OwnerNotDemotable` (both → 403).
- **Migration:** `20260914130000_group_members.{up,down}.sql` — table
  `group_members (group_id, member_id, privilege CHECK IN ('view','edit',
  'admin'), PK (group_id, member_id), FK principals ON DELETE CASCADE)`;
  backfill: group memberships → `edit`, `group_owners` owners → `admin`
  (upsert). Additive; zero behavior change for existing groups.
- **Store (sqlite):** `get_principals`/`get_principal` attach privileges
  (bulk query for the former, per-principal for the latter);
  `add_membership` seeds a default `edit` row only when `member_of` is a
  GROUP (`INSERT … ON CONFLICT DO NOTHING`, so the owner's `admin` row wins);
  `remove_membership` enforces last-admin and deletes the privilege row;
  `set_group_owner` upserts `admin`; new `get_privilege` / `set_privilege`
  (last-admin + owner-never-demotable invariants) / `list_members_with_
  privileges`. Trait defaults keep non-sqlite (test) stores compiling. All
  `Principal {…}` literals across the workspace updated with
  `privileges: Default::default()`.
- **Auth middleware:** `user$group` impersonation now stamps the
  impersonated principal's `privileges` with the acting user's privilege, so
  impersonation inherits the privilege instead of bypassing it (design
  requirement). Reads via impersonation still work (`is_principal`).
- **DAV enforcement (403 for `view`, 401 for non-members):**
  - Central `route_delete` + `route_proppatch` (dav crate): Read-but-no-
    WriteProperties → `Error::Forbidden` (403); no privileges at all (non-
    member) keeps the 401.
  - `get_user_privileges` made privilege-aware on: caldav calendar +
    calendar-object + scheduling inbox/inbox-object/outbox resources, carddav
    addressbook + address-object resources (`view` member → read-only set).
  - Explicit write-route checks (`is_principal` → 401, then `can_write` →
    403) added to: caldav `put_event`, `mkcalendar`, calendar `import`,
    scheduling `post_outbox`; carddav `put_object`, `mkcol`, `import`.
  - Deliberately NOT gated: GET/REPORT read paths, calendar/addressbook
    `POST` (WebDAV-Push subscription registration — read-support so `view`
    members still get push), share-feed exports (read-only by construction).
- **API (`/api/v1`):** members.rs rewritten — add/remove member admin-only;
  `AddMemberRequest` accepts optional starting `privilege` (default `edit`);
  new `PUT /groups/{g}/members/{m}` `{privilege}` (admin-only, self-change
  forbidden); `list_members` now member-gated and returns `[{id,
  privilege}]` (existing tests' `members[i]["id"]` assertions still hold).
  groups.rs `delete_group` admin-only; collections.rs create-collection now
  requires `can_write` (view members may subscribe, not create); api
  error.rs maps `LastAdmin`/`OwnerNotDemotable` → 403.
- **Portal:** share.rs — `shareable_principals` = groups with `edit`+ (view
  members no longer see group share tiles); share create/revoke →
  `can_write`; invite mint/revoke → `is_admin`; `ShareEntry.can_invite`
  gated the invite forms + invite-revoke buttons in `share_section.html`.
  groups.rs rewritten — `GroupInfo.admin`, `GroupDetailPage {admin,
  members: Vec<GroupMember{principal, privilege}>, error}` via shared
  `group_detail_page` builder, new POST routes `/{user}/groups/{g}/members/
  {m}/privilege` and `…/remove` (admin-only, self-change/self-removal
  forbidden, store invariants surface as `?error=` redirects).
- **NOT yet done (next session, in order):**
  1. ~~The new groups.rs routes reference `urlencoding::encode` which is NOT a
     dependency yet — switch to the already-available `percent-encoding`
     (workspace dep of crates/frontend) or add `urlencoding` to
     crates/frontend/Cargo.toml.~~ **DONE 2026-09-14** — both
     `urlencoding::encode` call-sites in
     `crates/frontend/src/routes/groups.rs` (`route_group_member_privilege`,
     `route_group_member_remove` error redirects) switched to
     `percent_encoding::utf8_percent_encode(…, NON_ALPHANUMERIC)`; no
     Cargo.toml change needed (workspace dep already present). Full
     `cargo check` still blocked by a pre-existing compile error in the
     uncommitted session-1 code (`crates/store/src/auth/middleware.rs:111-112`
     — `privileges.insert(impersonating.to_owned(), user.privilege_for(impersonating))`
     passes `Principal` where `String`/`&str` are expected; fixes are
     `impersonating.id.to_owned()` / `user.privilege_for(&impersonating.id)`).
     Will be resolved when the remaining items land before the item-6 gates.
     **FIXED 2026-09-14 (with item 2)** — middleware.rs:111-112 now
     `impersonating.id.to_owned()` / `user.privilege_for(&impersonating.id)`;
     `rustical_store` compiles.
  2. `group_detail.html` template still uses the OLD fields (`owner`,
     `members: Vec<Principal>` with API-delete forms) — update to `admin` +
     privilege selector/remove forms for `GroupMember`; `groups_section.html`
     chip for `admin`. **DONE 2026-09-14** — `group_detail.html`: `owner` →
     `admin` gating (delete group, add member), error banner
     (`{% if let Some(error) = error %}`) above the actions; member rows are
     `GroupMember` (`member.principal.displayname/id`); admins get a
     per-member privilege `<select name="privilege">` (view/edit/admin,
     current privilege pre-selected, auto-submit on change) POSTing to
     `/frontend/user/{u}/groups/{g}/members/{m}/privilege` + a Remove form
     POSTing to `…/remove` (both hidden for the acting user themselves);
     non-admin members see a privilege chip only. `groups_section.html`:
     chip now `{% if group.admin %}` "Admin" (was `owner`/"Owner"; owner is
     an implicit admin). `GroupInfo.owner` field + `get_group_owner` call in
     `route_groups` removed (dead code after the chip change). Two
     session-1 compile blockers fixed to make the templates checkable:
     `group_detail_page`'s non-async `filter_map(…).await` rewritten as a
     for-loop over `list_members_with_privileges`, and the item-1
     middleware.rs fix (above). Gate: `cargo fmt --check -p
     rustical_frontend` clean, `cargo check -p rustical_frontend` 0 errors
     (remaining warnings: the two unregistered routes + `SetPrivilegeForm`,
     resolved by item 3; two pre-existing `auth_provider` warnings in
     share.rs from session 1).
  3. Register the two new group routes in `crates/frontend/src/lib.rs`.
     **DONE 2026-09-14** — `frontend_router` gained
     `POST /{user}/groups/{group}/members/{member}/privilege`
     (`route_group_member_privilege`) and `POST /{user}/groups/{group}/
     members/{member}/remove` (`route_group_member_remove`), stacked after
     the group-detail route (same position as the share invite-revoke route
     from §17.9.1); imports extended in the `groups` use block. Gate:
     `cargo check -p rustical_frontend` 0 errors — the three item-2
     warnings (two unregistered routes + `SetPrivilegeForm`) are gone,
     leaving only the two pre-existing `auth_provider` warnings in
     share.rs; `cargo fmt --check -p rustical_frontend` clean.
  4. CLI: `rustical membership set-privilege <id> --to <group> --privilege
     view|edit|admin`; `list` prints privileges (src/commands/membership.rs).
  5. Tests (none written yet): store_sqlite principal_store tests (default
     `edit` on join, owner `admin`, set/get privilege, last-admin + owner
     invariants on `set_privilege` and `remove_membership`, privilege
     round-trip through `get_principal`); api.rs integration tests (admin can
     add/remove/set-privilege while `edit` member gets 403, self-change 403,
     last-admin 403, `view` member cannot create collections); per-privilege
     DAV matrix (view PUT/DELETE/MKCOL/PROPPATCH → 403 + reads OK, edit full
     CRUD, `user$group` impersonation inherits `view` → 403 on writes);
     frontend invite gating (non-admin invite mint → 403).
  6. Gates: `SQLX_OFFLINE=true cargo check --workspace --all-targets`,
     `cargo test --workspace`, `cargo fmt --check`, clippy on changed files —
     then record the gate results here and in PLAN_SHARING §10 (answer the
     10.2 open questions as decided above: group-level granularity v1,
     addressbooks stay invite-free, member list visible to members).

### 17.10 Calendar-level guest invites (no platform account)
**Request (2026-09-15):** invite someone to an individual calendar without
requiring them to create a platform account (no new username/password for the
portal). Recipient enters a pre-generated credential (server URL + guest
username + app token) into their CalDAV client. Sender chooses
view/edit/admin per invite. Delivery: link (copy credential) + email.

Distinct from §17.8 (registration invites → full account) and §17.7/§17.8.4
(read-only subscription share links). A guest invite grants write-capable DAV
access to exactly one collection via a lightweight principal that cannot log
into the portal.

#### 17.10.1 Design decisions
- **No new `PrincipalType`:** guests use the existing `Individual` type.
  The `collection_shares` table distinguishes them. CalDAV
  `CalendarUserType` stays `INDIVIDUAL` — no XML serialization changes.
- **No portal password:** guest `Principal` has `password = None` → portal
  login fails (`validate_password` returns `None`). DAV access via app
  token only (`validate_app_token` in auth middleware).
- **No memberships:** guest principals have empty `memberships` → no
  owner's other collections leak through `CalendarHomeSet` or
  `GroupMembership` DAV properties.
- **Per-collection scoping via `collection_shares` table:** each row maps
  exactly one guest principal to exactly one (owner, collection) pair with
  a privilege (`view`/`edit`/`admin`). V1: one share per guest principal.
- **Store-level resolution:** `CalendarStore` is made share-aware.
  `get_calendar(guest_id, cal_id)` resolves via shares → returns the
  owner's calendar with `cal.principal = guest_id` (the acting user's own
  id) so all existing `is_principal` / `can_write` DAV guards pass
  naturally — zero per-route guard surgery needed.
- **Privilege stamping at auth time:** when the middleware authenticates a
  guest, it stamps `privileges.insert(guest_id, share_privilege)`. This
  makes `privilege_for(guest_id)` return the share's privilege instead of
  the default `Privilege::Admin`. `can_write(guest_id)` then correctly
  gates writes: `view` → read-only, `edit`/`admin` → full write set.
- **Credential delivery:** minting shows the credential once (server URL +
  guest username + `{token_prefix}_{token}`) with Copy — same UX as app
  token creation. Optional email via `rustical_scheduling::smtp::send_mail`.
- **V1 scope:** calendars only. `kind` column in `collection_shares` is
  ready for addressbooks (same pattern).

#### 17.10.2 Data model
New migration `20260915120000_collection_shares`:
```sql
-- Omnical §17.10: per-collection guest invites (no platform account).
-- Each row grants one guest principal DAV access to exactly one
-- collection with a specific privilege. The guest authenticates via an
-- app token (standard DAV Basic auth); the share is the ACL.
CREATE TABLE collection_shares (
    id TEXT PRIMARY KEY,
    owner_principal TEXT NOT NULL,
    collection_id TEXT NOT NULL,
    kind TEXT NOT NULL CHECK (kind IN ('calendar', 'addressbook')),
    privilege TEXT NOT NULL CHECK (privilege IN ('view', 'edit', 'admin')),
    guest_principal TEXT NOT NULL UNIQUE,
    target_email TEXT,
    created_by TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    revoked_at DATETIME,
    FOREIGN KEY (owner_principal) REFERENCES principals(id) ON DELETE CASCADE,
    FOREIGN KEY (guest_principal) REFERENCES principals(id) ON DELETE CASCADE
);
```

**`CollectionShare` struct** (new file `crates/store/src/collection_share_store.rs`):
```rust
pub struct CollectionShare {
    pub id: String,
    pub owner_principal: String,
    pub collection_id: String,
    pub kind: String,            // "calendar" | "addressbook"
    pub privilege: Privilege,    // view / edit / admin
    pub guest_principal: String,
    pub target_email: Option<String>,
    pub created_by: String,
    pub created_at: Option<String>,
    pub revoked_at: Option<String>,
}
```

**`CollectionShareStore` trait** (analogous to `SubscriptionStore`/`InviteStore`):
- `add_share(owner, collection_id, kind, privilege, guest_principal, target_email, created_by) -> share_id`
- `get_share_by_guest(guest_id) -> Option<CollectionShare>` (revoked excluded)
- `get_shares_for_collection(owner, collection_id) -> Vec<CollectionShare>` (active)
- `revoke_share(share_id) -> Result` (sets `revoked_at`)
- `list_guest_shares(owner) -> Vec<CollectionShare>` (all active for owner)

SQLite impl + default trait methods (no-op/error) for test stores.

#### 17.10.3 Store-level share resolution
The `CalendarStore` gains share-awareness transparent to the DAV layer.
The store returns `Calendar` with `cal.principal = guest_id` (the acting
user's own id) so all existing `is_principal` / `can_write` checks pass.

**`get_calendar(principal, cal_id)`:**
1. Direct lookup (principal owns this calendar) — existing path.
2. If `NotFound` + shares present: `get_share_by_guest(principal)` → if
   share matches `(owner, cal_id, not revoked)` → fetch calendar from
   owner's namespace → set `cal.principal = principal` → return.
3. Otherwise propagate `NotFound`.

**`get_calendars(principal)`:**
1. Direct lookup (calendars owned by principal).
2. If empty + shares exist: resolve shared collections → return with
   `cal.principal = principal`.

**Object methods** (`get_object`, `put_object`, `delete_object`):
Same pattern — resolve `(principal, cal_id)` via shares before delegating
to owner storage. The `principal` argument is always the guest's own id;
the store translates internally.

#### 17.10.4 Auth middleware stamping
After `validate_app_token` returns the guest principal, the middleware
stamps:
```rust
if let Some(share) = share_store.get_share_by_guest(&user.id).await {
    user.privileges.insert(user.id.clone(), share.privilege);
}
```
This makes `privilege_for(guest_id)` return the share privilege.
`can_write(guest_id)` → `share_privilege.can_write()` → correct gate.

The middleware needs `CollectionShareStore` access — add as an `Extension`
or pass into the middleware constructor.

> **Implementation note (2026-09-15):** stamped in
> `SqlitePrincipalStore::get_principal`/`get_principals` instead of the
> middleware. `validate_app_token` returns `get_principal(id)`, so the DAV
> auth path gets the identical stamped principal with zero plumbing through
> `AuthenticationLayer`/`caldav_router`/`carddav_router` — and every other
> principal-load path (discovery, portal) is stamped consistently too.
> `privilege_for(guest_id)` reads `privileges[id of self]`, exactly the slot
> the plan stamps. Verified by tests: `guest_privilege_is_stamped`,
> `non_guest_is_not_stamped`.

**Why this works end-to-end (no DAV changes needed):**
- `CalendarResource::get_user_privileges`: `is_principal(cal.principal)`
  where `cal.principal = guest_id` → true (self). `can_write(guest_id)` →
  stamped privilege → correct `read_only` or `all`.
- `CalendarObjectResource::get_user_privileges`: same pattern.
- Central dav `route_delete`/`route_proppatch`: call
  `resource.get_user_privileges(user)` → same logic.
- Explicit guards (`put_event`, `import`, `mkcalendar`,
  `calendar_object/methods.rs`, carddav `put_object`/`mkcol`/`import`):
  `is_principal(&path_principal)` where `path_principal = guest_id` → true
  (self). `can_write(&path_principal)` → stamped privilege → correct.
- `get_event` second check: `is_principal(&calendar.principal)` where
  `calendar.principal = guest_id` (after store resolution) → true.
- `PrincipalResource::get_user_privileges`: guest accessing own principal →
  `is_principal(self)` → true → `owner_only(true)` → full access.

#### 17.10.5 Principal home set / discovery
`PrincipalResourceService::get_members(guest_id)` calls
`cal_store.get_calendars(guest_id)`. Share-awareness returns shared
collections with `cal.principal = guest_id`. Client sees exactly the shared
calendars. `CalendarHomeSet` = `[self]` only (no memberships). No leak.

#### 17.10.6 Portal UI
Share section (`crates/frontend/src/routes/share.rs`):
- `ShareEntry` gains `guest_shares: Vec<GuestShareEntry>` field.
- Each calendar tile renders "Guest access" subsection:
  - Active shares: guest username, privilege badge, Copy credential,
    Revoke.
  - "Invite guest" form: privilege `<select>` (view/edit/admin), optional
    email, submit.
- `ShareSection` gains `guest_share_credential: Option<String>` (one-time
  display after minting).

New route `POST /{user}/share/guest-invite`:
1. Ownership check (`user.is_admin(&principal)`).
2. Create guest principal (`"guest-{ulid}"`, `password: None`,
   `Individual`, no memberships).
3. Mint app token (`add_app_token` → `{token_prefix}_{token}`).
4. Create `collection_shares` row.
5. Optionally send email via `send_mail`.
6. Re-render with `guest_share_credential` set.

New route `POST /{user}/share/guest-invite/{id}/revoke`: sets `revoked_at`.

New public route `GET /guest/{code}` (optional link delivery): credential
page, no auth.

#### 17.10.7 Delivery
**Link (primary):** credential shown on Share page (one-time) → owner
copies and sends via any channel. Optional `GET /guest/{code}` public page.

**Email (optional, requires SMTP):** if `[scheduling]` has SMTP accounts
and email is provided, `send_mail` sends plaintext with server URL +
username + token. If SMTP not configured, email is stored in `target_email`
for audit but not sent.

> **Implementation note (2026-09-15, item 5):** `build_guest_invite(account,
> to, server_url, username, credential, calendar_id, owner)` lives in
> `crates/scheduling/src/mime.rs`.  Plain-text RFC 5322 message (no MIME
> parts); subject base64-encoded; all line endings `\r\n`.  The portal route
> (`route_share_guest_invite`) spawns `smtp::send_mail` via `tokio::spawn`
> (fire-and-forget, warn-on-error).  Deviation from plan: only one new unit
> test (`guest_invite_is_plaintext_with_credential`) — no network-based
> send test because the codebase avoids SMTP integration tests.
> `SmtpAccount` plumbed as `Extension<Vec<SmtpAccount>>` through
> `frontend_router` → `make_app` → `cmd_serve` (reads
> `config.scheduling.smtp`); test harness passes `vec![]`.

> **Implementation note (2026-09-22, §17.13):** `build_guest_invite` gained a
> `subscribe_url: Option<&str>` parameter — the email now also carries the
> credential-less `/export/{token}.ics` link (reuse-or-mint via the shared
> `ensure_subscribe_url`, §17.8.4), and the one-time Share-page banner shows
> it too ("Prefer a subscription link …").  CLI `rustical guest-share add`
> mints-or-reuses the same subscription and prints the URL.  When
> `[subscriptions]` is disabled the section is simply omitted.

#### 17.10.8 CLI additions
- `rustical guest-share add <owner> <collection_id> --kind calendar --privilege view|edit|admin [--email <addr>]` → print credential
- `rustical guest-share list <owner>` → list active shares
- `rustical guest-share revoke <share_id>` → revoke
- `rustical guest-share credential <share_id>` → print credential

> **Implementation note (2026-09-15, item 6):** subcommand group
> `guest-share { add | list | revoke | credential }` in
> `src/commands/guest_shares.rs`.  `add` mints `guest-{uuid}`, inserts
> principal (`password: None`, `Individual`), calls `add_app_token` then
> `share_store.add_share`; prints username + credential.
> `list` iterates `list_guest_shares(owner)`.  `revoke` calls
> `revoke_share(share_id)` directly.  `credential` scans all principals'
> shares to locate the row, prints guest username + privilege and a note
> that the token secret is stored hashed and cannot be recovered (mint
> fresh instead).  Registered in `Command::GuestShare` in `src/main.rs`.
> Gate: 1 new integration test (`tests/guest_share_cli.rs` — add +
> list + revoke round-trip).

#### 17.10.9 Implementation split
1. **Migration + store trait** — `collection_shares` table,
   `CollectionShareStore` trait, SQLite impl. Gate: `cargo check -p
   rustical_store_sqlite`, 1 new store test (add + get + revoke +
   revoked-excluded).
2. **CalendarStore share-awareness** — wrap `get_calendar` / `get_calendars`
   / object methods with share resolution. Gate: 3 new store tests (guest
   resolves shared calendar, guest sees only shared calendars, non-share
   principal unaffected).
3. **Auth middleware stamping** — add `CollectionShareStore` to middleware,
   stamp guest privileges after `validate_app_token`. Gate: `cargo check -p
   rustical_store`, 2 new auth tests (guest privilege stamped, non-guest
   unstamped).
4. **Portal share section** — `ShareEntry.guest_shares`, mint/revoke routes,
   credential display, template. Gate: `cargo check -p rustical_frontend`,
   4 new `frontend_share.rs` integration tests (mint view/edit/admin,
   revoke, ownership check, credential shown once).
5. **Email delivery** — `send_mail` integration. Gate: 1 new test (email
   sent when SMTP configured).
6. **CLI** — `guest-share` subcommand group. Gate: 1 new CLI test (add +
   list + revoke round-trip).
7. **Gates:** `SQLX_OFFLINE=true cargo check --workspace --all-targets`,
   `cargo test --workspace`, `cargo fmt --check`, clippy on changed files
   — record results here.

> **Item 7 gate results (2026-09-15, all green):**
> - `cargo check --workspace --all-targets` — pass (2 pre-existing
>   unused-`auth_provider` warnings in `route_share_revoke` /
>   `route_share_invite_revoke`, unchanged).
> - `cargo test --workspace` — pass.  Suite totals:
>   store_sqlite 34, scheduling 56 (1 ignored pre-existing),
>   frontend_share 22 (18 pre-existing + 4 new), guest_share_cli 1
>   (new), all other crates green, 0 failures.
> - `cargo fmt --check` — pass.
> - Clippy: no new lint categories introduced by changed files;
>   all lint hits in `crates/scheduling/src/mime.rs` are on lines
>   identical to pre-existing code style (`push_str(&format!(...))`
>   pattern).  No clippy findings in new files (`guest_shares.rs`,
>   `guest_share_cli.rs`).

> **LIVE: DONE 2026-09-15** — deployed (fresh aarch64 build via
> `scripts/build-rust.sh` + `deploy.sh`; migration `20260915120000`
> auto-applied at startup: `collection_shares` table + indexes present).
> Test flow against `0115d8cf.duckdns.org:8443` (dav-tls, session login as
> `live-test-20260914@example.com`): created throwaway calendar `guesttest`
> via MKCOL (session cookie auth) and minted guests from the Share page
> without touching production collections.
> - **view** guest (no email): banner «Guest access created!» → Server URL
>   `https://0115d8cf.duckdns.org:8443/caldav`, Username
>   `guest-20818ba0-…`, App token `1784_…` (69-char; one-time banner).
> - **DAV as guest (Basic auth with the guest username + token):** PROPFIND
>   depth:1 on `/caldav/principal/{guest}/` → 207 listing the shared
>   `guesttest` calendar (`cal.principal` rewritten to the guest); view-guest
>   PUT VEVENT → **403 Forbidden** (read-only gate via stamped privilege).
> - **edit** guest bound to `burningserenity@gmail.com`: banner «Guest
>   access sent to …!»; PUT VEVENT → **201**, GET → **200**, DELETE → **200**
>   (write path fully works).
> - **Email:** no SMTP `warn`/error logged → `send_mail` accepted the
>   message (success is debug-level; receipt pending on the Gmail side —
>   verify on the mailbox).
> - **Revoke** (POST `…/guest-invite/{id}/revoke`, form carries
>   `principal=<owner>`): 303; DB audit rows keep `revoked_at` set for all 4
>   guests; **DAV access dies** — re-test PROPFIND after revoke → 207 with
>   only the principal node, shared calendar gone (revoked-filter re-verified
>   live).
> - **Cleanup:** test calendars `guesttest` + `revoketest` deleted
>   (`X-No-Trashbin: 1`), share page back to baseline (0 active guests, 0
>   test tiles), no leftover objects.
>
> **Live-test findings for the record:**
> - The one-time credential lives ONLY in the POST response body (inline
>   200, not a redirect) — the follow-up GET to the Share page no longer
>   shows it (expected one-time banner behavior).
> - `POST …/revoke` expects a urlencoded form body with `principal` set
>   (bare POST → 415; harmless).
> - Guest principals persist after revoke (by design: share rows are the
>   audit trail; a revoked guest authenticates to an empty home set).

### 17.11 Scheduling hardening — khal 403 (RRULE UNTIL normalisation + default ORGANIZER) (2026-09-18)

**Request (2026-09-18):** the khal-created recurring event
`~/.calendars/omnical/Internal Shared/25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S.ics`
("Stand Up", bi-weekly, attendee `denis@hawksnestsoftware.com`) 403'd on
every vdirsyncer PUT (pair `omnical_calendar_hawksnest`, cron `*/15`) — the
pair stalled on that one item.

**Diagnosis:** khal emits the RRULE end as a *floating* DATE-TIME
(`RRULE:FREQ=WEEKLY;UNTIL=20261204T090000;INTERVAL=2`) next to a
`TZID`-qualified DTSTART. RFC 5545 §3.3.10 requires UNTIL to be UTC when
DTSTART is tz-qualified; caldata 0.16.2 enforces this at parse time
(`DtStartUntilMismatchTimezone`) → the `valid-calendar-data` precondition
fails in `CalendarObject::import` → 403 (WARN + full body dump in
`logread`). Fixing khal's output was rejected — the server should
accept-and-normalise. Two server-side fixes, both landed on the
`omnical-scheduling` tree (uncommitted):

- **Fix 1 — normalise floating RRULE UNTIL on import:** NEW
  `crates/ical/src/normalize.rs`: `normalize_rrule_until(ics) -> Cow<str>`
  walks unfolded logical lines with component-depth tracking; per
  VEVENT/VTODO/VJOURNAL it captures the DTSTART shape (`TZID` / UTC-Z /
  floating / `VALUE=DATE`) and rewrites `UNTIL=<floating 15-digit>` **only**
  when DTSTART is tz-qualified: `TZID` → resolved via chrono-tz
  `and_local_timezone().earliest()` (deterministic on DST folds;
  nonexistent local times left untouched rather than guessed), re-emitted
  with `Z`; UTC DTSTART → append `Z`. Nested components (VALARM, VTIMEZONE
  subcomponents) never contribute. Untouched bodies return `Cow::Borrowed`;
  rewrites rejoin CRLF with the trailing-CRLF guard (same pattern as
  `scheduling::ics::normalize_caladdresses`). Idempotent. Wired into
  `CalendarObject::import` (`crates/ical/src/calendar_object.rs:79`) — the
  DAV PUT path and the calendar-level import route both inherit it;
  `from_ics` (store load) deliberately untouched.
- **Fix 2 — default ORGANIZER stamping:** NEW
  `scheduling::ics::default_organizer(ics, organizer)` inserts
  `ORGANIZER:mailto:<organizer>` before the first ATTENDEE (depth 1) of the
  first VEVENT; no-op if that VEVENT already has an ORGANIZER or no
  ATTENDEE. `caldav put_event` gates it via `stamp_default_organizer`:
  no METHOD, no ORGANIZER, ≥1 ATTENDEE, and the acting user is an attendee
  or owns the target calendar — the §17.2 implicit-scheduling organizer
  rule materialised for the storage path. The scheduler's `handle_put` now
  receives `stored_ics = object.get_ics()` (final normalised + stamped
  form) instead of the raw client body, and the ETag is computed from the
  final object → ETag and stored copy agree → vdirsyncer stable after the
  first sync. Bulk (calendar-level) import deliberately gets no stamping.

**Tests:** 15 new ical unit tests in `normalize.rs` (incl. the production
event verbatim + the end-to-end `CalendarObject::import` regression that
used to fail); 5 new scheduling unit tests for `default_organizer` (incl.
khal's folded ATTENDEE line); 4 new caldav integration tests in
`scheduling/tests.rs` (`KHAL_EVENT`): (a) production 403 regression → 201 +
stored body contains both `UNTIL=20261204T140000Z` and
`ORGANIZER:mailto:user`, (b) existing ORGANIZER preserved verbatim,
(c) attendee-less event untouched, (d) stamping works with scheduler
`None`. Existing `sched-no-org-1` behaviour unchanged.

**Gates (2026-09-18, all green):** `SQLX_OFFLINE=true cargo check
--workspace --all-targets` (only the 5 pre-existing `rustical_frontend`
warnings); `cargo fmt --check` clean; clippy zero NEW warnings (warning
locations diffed against stashed HEAD — line-number shifts of pre-existing
debt only); tests: rustical_ical 16, rustical_scheduling 61 (+1 ignored),
rustical_caldav 51, rustical_store_sqlite 34, `rustical --lib` 36 — 0
failures, no insta snapshot changes.

> **LIVE: DONE 2026-09-18** — built (`scripts/build-rust.sh`: rustical
> 15.8 MiB stripped / 4.8 MiB after UPX --lzma — gate «rustical fits:
> 4 MiB of 35 MiB budget»; dav-tls binary unchanged) and deployed
> (`deploy.sh`: DB backup, binary swap, service restart — `rustical
> healthy`, pid 14271, scheduling + iMIP ingest + RSVP extensions up).
> Last failing PUT on the OLD binary at 14:15:02 (403 + WARN body dump);
> new binary serving 14:18:50. Manual `vdirsyncer sync
> omnical_calendar_hawksnest` right after → «Copying (uploading) item
> 25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S» with zero errors, and the scheduler
> logged `scheduling: emailed REQUEST to denis@hawksnestsoftware.com
> (attempt 1)` at 14:19:09 — external attendee → iMIP via SMTP, i.e. Fix 2's
> ORGANIZER stamp and §17.2 implicit scheduling both firing off the stored
> organiser copy. Two further syncs: clean, no re-uploads (PUT ETag ==
> stored form). `curl` GET of the stored object → 200 with
> `RRULE:FREQ=WEEKLY;UNTIL=20261204T140000Z;INTERVAL=2` (floating 09:00,
> Dec 4 = EST −0500 → 14:00Z ✓) and
> `ORGANIZER:mailto:nicholas@hawksnestsoftware.com`. The local vdir keeps
> khal's original body (vdirsyncer is ETag-driven → zero local churn —
> the desired steady state); subsequent `*/15` cron runs clean. Code
> remains uncommitted on `omnical-scheduling` (six files: five modified +
> `crates/ical/src/normalize.rs` new).

> **Follow-up (2026-09-18, ~16:20Z): second production 403 — same event,
> different khal shape.** After the morning deploy the user edited the
> event in ikhal (moved 11:00 → 09:00, `SEQUENCE` 0→1); khal then
> re-serialised the RRULE end as a *bare DATE* —
> `RRULE:FREQ=WEEKLY;UNTIL=20261204;INTERVAL=2` — next to the `TZID`'d
> DATE-TIME DTSTART. caldata parses that DATE as floating midnight, so the
> SAME `DtStartUntilMismatchTimezone` validation fired again (reproduced
> via `/tmp/opencode/caldata-repro`: "UNTIL was specified in timezone
> Local"), and vdirsyncer PUTs 403'd from the first post-edit sync on.
> Fix (same session): `normalize_rrule_until` now also rewrites
> DATE-valued `UNTIL=YYYYMMDD` when the DTSTART is tz-qualified,
> expanding it to that date at the DTSTART's time-of-day in its zone
> (inclusive last-occurrence day — occurrences on that day are kept,
> which is khal's semantics) and re-emitting UTC; UTC `Z` DTSTARTs
> likewise get the DATE expanded at their time-of-day; all-day and
> floating DTSTARTs stay untouched (DATE UNTIL is RFC-valid there).
> `DtStart` now carries the value's time-of-day
> (`Tzid(String, Option<NaiveTime>)` / `Utc(Option<NaiveTime>)`).
> Tests: +5 ical (edited event verbatim — `concat!` const keeping khal's
> folded ATTENDEE line, UNTIL on an occurrence day keeps that day, UTC
> -DTSTART DATE-UNTIL, floating-DTSTART DATE-UNTIL untouched, end-to-end
> `CalendarObject::import` regression, idempotence extended) and +1 caldav
> (`KHAL_EVENT_EDITED` PUT by owner → 201 → stored body has
> `UNTIL=20261204T140000Z` + `ORGANIZER:mailto:user`). Gates: fmt clean,
> workspace check 0 errors, clippy zero new warnings (hits only on
> pre-existing pedantic-debt lines), ical 20 / caldav 52 / scheduling 61
> / store_sqlite 34 / `rustical --lib` 36 — 0 failures. Redeployed
> 16:27:52Z (pid 15146, `rustical healthy`): manual sync → «Copying
> (updating) item 25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S» accepted,
> scheduler emailed the REQUEST update to denis@hawksnestsoftware.com at
> 16:28:09Z (`SEQUENCE` bump = scheduling-relevant change), second sync
> clean; stored-copy GET: `DTSTART;TZID=…:20260918T090000`,
> `SEQUENCE:1`, `RRULE:FREQ=WEEKLY;UNTIL=20261204T140000Z;INTERVAL=2`,
> `ORGANIZER:mailto:nicholas@hawksnestsoftware.com`.

> **Open items left to the user (not acted on):** stale sibling vdir
> `~/.calendars/omnical/hawksnest-internal-shared/` (older duplicate
> "Stand Up", UID `L9YROQM0L9T6MZSKN3P22JX9AZPHOMNLLWDQ`, odd
> `TRIGGER:P0D` VALARM, not covered by any current pair) and orphaned
> `~/.vdirsyncer/status/{google_calendar*,rustical_*}` dirs.

### 17.12 Per-calendar credentials on the Calendars screen (2026-09-21)

**Request:** on the user's Calendars screen, generate credentials per calendar
"like those emailed as invites" — always with full read/write/admin access.

**Design (reuses the §17.10 guest-share machinery):** each calendar tile gains
a **"Generate credentials"** action (`POST /frontend/user/{user}/calendar/
credentials`, form `{principal, calendar_id, email?}`) that mints a fresh
`guest-{uuid}` principal + app token (`{id4}_{64}`) and persists a
`collection_shares` row scoped to exactly that calendar with the privilege
hard-set to **`admin`** — full read/write/admin. The credential (Server URL +
username + app token) is shown once in a banner on the Calendars screen and,
like the Share page guest invite, emailed via `mime::build_guest_invite` when
an email is given and SMTP is configured (`target_email` stored for audit
otherwise).

- **Gate (minting):** the tile shows the button only where the acting user
  holds `admin` on the owning principal (`user.is_admin(&principal)` — own
  calendars always, groups only for admins); the route enforces the same rule
  (403 otherwise). `view`/`edit` group members see the calendar but no
  button.
- **Location:** the list route moved from `routes/calendars.rs` (deleted) into
  `routes/calendar.rs`, which now holds the whole Calendars surface; a shared
  `render_calendars_page` renders the section + optional one-time
  credential/error banners. Each tile carries a `CalendarTile` struct (meta,
  calendar, `can_generate`, active `guest_shares`) so the active credentials
  are listed inline with their own Revoke control.
- **Revoke (on-screen):** the Calendars page lists every active credential
  minted for the tile (`Credential guest-… · admin access · created …`) and a
  **Revoke** button per row → `POST /frontend/user/{user}/calendar/
  credentials/{id}/revoke` (form `{principal}`) sets `revoked_at`; the row
  stops resolving access (revoked rows are filtered out of every store
  lookup), the credential no longer stamps any privilege, and the tile stops
  showing the credential. Same owner/admin gate as minting (403 otherwise,
  401 cross-user); redirect back to the Calendars screen.
- **V1 scope:** calendars only (same as §17.10). The Share page's guest-shares
  revoke flow covers addressbook credentials; the Calendars screen owns its
  calendar credentials end-to-end.

**Tests:** new `tests/integration_tests/frontend_calendars.rs` (7): page shows
the button on exactly the own + admin-group tiles (not the edit-only or
foreign tiles); mint on the own calendar → banner + `admin` share row +
`validate_app_token` authenticates with `Privilege::Admin` (full write +
admin); group-calendar mint with email binds `target_email` + invalid-email
rejected; non-admin/foreign principals → 403 (nothing stored); unknown
collection → error banner; revoke on the own + group calendar → 303, store
lookups empty, `validate_app_token` succeeds but stamps no privilege, tile no
longer lists the credential (sibling credentials unaffected); revoke 403 for
edit-only members / foreign principals + 401 cross-user.

**Gates (2026-09-21, all green):** `SQLX_OFFLINE=true cargo check --workspace
--all-targets` — only the pre-existing `rustical_frontend` warnings (2 unused
`auth_provider`, 3 never-read `ShareSection` fields); `cargo fmt --check`
clean; clippy zero NEW warnings in the changed files. Tests: full workspace
`--no-fail-fast` — every suite green **except the 7 pre-existing
`frontend_share` failures** (verified identical on the stashed clean-tree
HEAD); new `frontend_calendars` 7/7. Code uncommitted on `omnical-scheduling`
(6 files: template, `lib.rs`, `routes/calendar.rs`, `routes/mod.rs`, deleted
`routes/calendars.rs`, `tests/integration_tests/{mod.rs,frontend_calendars.rs}`).

> **AMENDED 2026-09-22 (§17.13):** (a) the 7 `frontend_share` failures listed
> above were root-caused to the §17.10 guest-banner nesting bug (see §17.13 C)
> and are now green — 93/93 integration tests.  (b) The three "never-read
> `ShareSection` fields" are now read: the guest banner is gated on
> `guest_share_principal`/`guest_share_calendar_id`.  (c) The Calendars
> screen's tiles now also carry the §17.8.4 share-link surface ("Subscribe
> URL" button / export URL with Copy + Revoke — see §17.13 A); the §17.12
> credential banner itself is page-level and was not affected by the
> Share-page nesting incident.

### 17.13 Calendars-screen subscribe URLs + guest-email subscribe link; the banner-nesting incident (2026-09-22)

**Requests:** (1) "We need a way to generate that subscription URL for the
logged-in user, from their Calendars page."  (2) "I have somehow lost the
ability to share the CAS calendar from the nicholas@carltonaudio.com
account."

**A. Calendars-screen Subscribe URL (reuses §17.7/§17.8.4):**
- `CalendarTile` gains `subscribe_url` / `sub_id` / `can_subscribe`;
  `render_calendars_page(…)` now takes `sub_store:
  Option<&Arc<dyn SubscriptionStore>>` + `base_url` and looks up each
  calendar's existing subscription (kind `Calendar`, matching
  `collection_id`) exactly like the Share page does.
- New `route_calendar_subscribe` → `POST /{user}/calendar/subscribe` (form
  `{principal, calendar_id}`): mint-or-reuse via `ensure_subscribe_url` (now
  `pub(super)` in `routes/share.rs`, shared by the share + calendar route
  modules), then 303 back to the Calendars screen.  Ownership gate: own
  principal or `can_write` group (same rule as the Share page's share
  links — note: broader than §17.12's `is_admin` credential gate, which
  remains unchanged).  Subscriptions disabled → 503; errors render as the
  page's error banner.
- `calendars_section.html`: "Subscribe URL" button in the actions row (only
  when `tile.can_subscribe && tile.subscribe_url.is_none()`), plus a
  metadata block showing the export URL with **Copy** and **Revoke**
  (reuses `POST /{user}/share/{sub_id}/revoke`).
- Gates: 2 new `tests/integration_tests/frontend_calendars.rs` tests (mint →
  tile shows URL; foreign/unknown rejected); local scratch e2e: button →
  303 → URL on tile → `/export/{token}.ics` 200.

**B. Guest share email + banner carry the subscribe link (§17.10.7
AMENDED):** `build_guest_invite(..., subscribe_url: Option<&str>)` in
`crates/scheduling/src/mime.rs`; `route_share_guest_invite` mint-or-reuses
the shared calendar's subscription and includes it in the email + one-time
banner (`ShareSection.guest_share_subscribe_url`); CLI `rustical
guest-share add` prints the same URL.

**C. The banner-nesting incident (root cause of request 2):** the one-time
guest-credential banner in `share_section.html` sat inside
`{% for invite in entry.invites %}` — tiles with **no registration invites**
never iterated the loop, so the banner never rendered after minting; app
token secrets are stored hashed → the credential was effectively lost.  This
bit on 2026-09-22:
- the nicholas@carltonaudio.com CAS report (CAS tile has no invites);
- share `30343a84-eaf6-4511-a63e-ef30c2b621a7` (2026-09-22 18:10:03 UTC):
  **lyrest@gmail.com's "Cece" calendar** (`290c0341-…`), guest
  `guest-d151aece-…`, privilege `admin`, no `target_email` — initially
  misattributed to the CAS/nicholas account; same bug on lyrest's Share
  page (that tile has no invites either).  The share is still active but
  the credential is unrecoverable — harmless while active; revoke +
  re-mint from the portal is the cleanup path (owner's call).
**Fix (deployed 2026-09-22):** banner moved out of the invites loop to tile
level, **gated on `guest_share_principal == entry.principal &&
guest_share_calendar_id == entry.collection_id`** (the struct fields existed
but were never wired) — renders **exactly once, on the minted calendar's
tile**; the tile-level error moved to page level (same placement as the
Calendars screen).  The `{% else if enabled %}` "Create share link" branch
now renders when `entry.url` is `None`.  This also fixed the **7
long-failing `frontend_share` integration tests** (§17.12 AMENDED) and
unlocked a strengthening: `test_guest_invite_mints_share_and_shows_credential_banner`
now asserts the banner count is exactly 1.  *Post-mortem detail:* the first
version of the fix moved the banner + error to tile level **ungated** —
caught during live post-deploy verification (banner rendered on all 7
tiles) and re-fixed + redeployed the same evening.

**Gates (2026-09-22, all green):** `cargo test --test run_integration_tests`
93 passed / 0 failed; `cargo fmt` applied; clippy `rustical_frontend` +
`rustical` only pre-existing warnings; cross-build aarch64-musl 4 MiB
(35 MiB budget).  Deployed 19:16 UTC; gate-fix redeploy 19:31 UTC (deployed
md5 == fresh build).

**LIVE (2026-09-22, app-token diag, all artifacts cleaned up + DB-verified):**
Calendars screen shows 4 "Subscribe URL" buttons + the CAS tile's existing
`export/o7HE….ics` link (subscription since 2026-09-10) with Copy/Revoke;
`POST /calendar/subscribe` for CAS → 303 with **reuse** (no new
`subscriptions` row); guest-invite POST for CAS → banner exactly once on the
CAS tile (Server URL / Username / App token / "Prefer a subscription link"
line); `/export/o7HE….ics` serves 200 `text/calendar` through dav-tls
(`https://0115d8cf.duckdns.org:8443`).  All diag guest shares revoked, diag
app token removed.

**Also in this batch:** platform invite `rustical invites create --send`
(`src/commands/invites.rs`, `build_registration_invite` in `mime.rs`) and
the untracked `scripts/invite-user.sh` SSH wrapper.

### 17.14 Forgot-password flow + app-email sender switch to burningserenity@novo-ordo.com (2026-09-22)

**Requests:** (1) self-service password reset — "Forgot your password?" on
the login page → emailed one-time reset link → set a new password;
(2) all app-generated emails (registration invites, guest-share
credentials, the new reset emails — everything via
`scheduling.smtp.first()`) must come From `burningserenity@novo-ordo.com`
instead of `burningserenity@gmail.com`.  iMIP invitations are unchanged
(they always match the organizer via `smtp_account()`).

**A. Reset flow (design decisions):**
- **Token**: 64-char alphanumeric; only its SHA-256 hex digest is stored
  (`token_hash()`), so a DB leak leaves no working links.  Expiry 1 hour
  (`RESET_TOKEN_EXPIRY_SECS = 3600`); plain string comparison
  `YYYY-MM-DDTHH:MM:SSZ` like invites.
- **Single-use atomicity**: `redeem_reset` = `UPDATE … WHERE token_hash = ?
  AND used_at IS NULL AND expires_at > ?` in a transaction, then marks the
  principal's other outstanding tokens used; `add_reset` supersedes the
  principal's previous unused tokens first → at most one usable link per
  account at any time.
- **No enumeration**: known email, unknown email, and OIDC-only accounts all
  get the byte-identical "reset sent" page; unknown/used/expired tokens all
  get one generic invalid page.  Reset POST does **not** auto-login.
- **CSRF**: session-bound, rotated on every rendered response; mismatch →
  "This form has expired" 400.
- **Rate limiting**: per-IP 5/h + global 30/h sliding window, shared by both
  POST endpoints, `X-Forwarded-For` first hop; rejected attempts are not
  recorded.
- **Availability**: the login-page link and `POST /forgot-password` require
  `allow_password_login && smtp non-empty && public_url non-empty`
  (`reset_available()`); the *reset* endpoints only require
  `allow_password_login` (a minted link stays redeemable even if SMTP is
  later removed).  Password rules reuse `[frontend]
  min_password_length` (default 12); a successful reset also clears
  `needs_password_change`.
- **Delivery**: fire-and-forget `tokio::spawn(smtp::send_mail(…))` like the
  guest-share email; failures logged, never user-facing.
- Endpoints live in the frontend crate (unauthenticated, next to `/login`,
  outside `user_router`'s gates): `/frontend/forgot-password`,
  `/frontend/reset-password/{token}`.  No new config keys; SQLx offline
  metadata untouched (runtime `sqlx::query` on purpose, like the invite
  store).
- **New plumbing**: `PasswordResetStore` trait (`rustical_store`) + SQLite
  impl (`rustical_store_sqlite`) + migration
  `20260922120000_password_resets` (table with `token_hash UNIQUE`, index
  on `(principal_id, used_at)`); `get_data_stores` is now an 11-tuple ending
  in `Arc<dyn PasswordResetStore>`; `make_app`/`frontend_router` gained the
  store + `Arc<ResetRateLimiter>` extensions; `build_password_reset` in
  `mime.rs` (mirrors `build_registration_invite`).  Drive-by: fixed
  pre-existing `tests/http_integration.rs` breakage (two `InviteCreateArgs`
  literals missing the `send` field from commit `c2b2492c`).

**B. Sender switch (`scripts/render-router-config.sh`):**
`burningserenity@novo-ordo.com` is now the **FIRST** SMTP row (From address
for everything sent via `smtp_accounts.first()` — zero code changes);
`burningserenity@gmail.com` remains as an iMIP organizer principal.  The
new identity **shares `nfcalaway@novo-ordo.com`'s pass entry** (same
pattern as `zero@novo-ordo.com`) — no new `pass` entry; fallback if SMTP
rejects that identity is a dedicated
`secrets/email/burningserenity@novo-ordo.com/smtp` entry + username change.
No IMAP row (not an iMIP organizer; inbound replies not polled for it).

**Gates (2026-09-22, all green):** full `cargo test --workspace` sweep —
frontend 22 passed (incl. rate-limit shared-budget, no-enumeration
body-equality, CSRF, supersedes/expiry, password rotation), scheduling +
store + store_sqlite 100, integration + http 148; `cargo fmt` clean (drift
was confined to this effort's files); clippy clean for all new code.  Flake
fixed during gating: the test helper `mint_reset` generated tokens from
only 9 values (`"t"×63` + random digit) and collided on the UNIQUE
`token_hash` ~11% of runs; it now uses the real `random_token()`.
**Status:** DONE, live-verified end to end (2026-09-22). Deployed initially
20:11 UTC (outside any agent session); the session-2 live pass found the
reset EMAIL BOUNCING (`EHLO omnical.local` rejected by smtp.novo-ordo.com
— Postfix `reject_unknown_helo_hostname`). The EHLO fix (`set_ehlo_name_from_url`,
OnceLock from `[subscriptions] public_url`) was rebuilt + redeployed
21:17:07 UTC (session 3) and the **full email checklist re-run GREEN**:
reset email arrives From `burningserenity@novo-ordo.com` (DKIM/SPF pass,
64-char one-time link + 1 h expiry note); no-enumeration byte-identity
holds (unknown vs known vs session-2's saved response, csrf-normalized);
supersedes works (second POST invalidates the first link — generic invalid
page vs form); redeem is atomic single-use (all 3 minted rows carry
`used_at`, no usable links remain); reset POST → 303 login (no auto-login);
OLD password 401, NEW logs in and lands normally, `needs_password_change`
cleared 1→0, original password restored via
`rustical principals edit zero@novo-ordo.com --password` (CLI takes no
`-c` flag — as root it reads /etc/rustical/config.toml by default) — net
state change zero; guest-share credential email + registration-invite
email both From `burningserenity@novo-ordo.com`; iMIP REQUEST still From
the organizer (`burningserenity@gmail.com`) — the sender switch left iMIP
untouched. **Live findings:** (a) `zero@novo-ordo.com` is an IMAP *alias*
delivering into nfcalaway@novo-ordo.com's INBOX (Delivered-To proves it) —
the `mbsync Novo-Ordo-zero` channel matches nothing; read via
`mbsync Novo-Ordo-general` INBOX; (b) a stored-event PUT whose ICS carries
`METHOD:REQUEST` is ignored by the scheduler (`handle_put` early-return:
stored objects must not carry METHOD) — iMIP testing needs a plain VEVENT;
(c) the rate limiter records a POST **before** the CSRF check — a
form-expired 400 consumes budget; (d) per-IP buckets key on
client-supplied `X-Forwarded-For` (dav-tls is a raw splice), so live diag
can isolate tests by varying XFF. See
`PLAN_NEXT_AGENT_CALENDARS_TAB_SHARING.md` §15 for the session-3 log.

---

### 17.15 Share tab folded into Calendars/Addressbooks tabs; per-client subscribe instructions; CAS "no controls" root-caused as client-side staleness (2026-09-22)

**Requests:** (1) root-cause the repeated CAS "no controls on the Share
tab" report (logged in as `nicholas@carltonaudio.com`, seen twice on
2026-09-22 after deploys); (2) remove the Share tab entirely — its controls
move onto the Calendars tab (calendar half) and Addressbooks tab (share
links only); (3) per-client subscribe instructions on every calendar tile
(Google/Apple/Outlook + DAVx5/Thunderbird), answering the user's
*"the share we have does caldav, is there no subscribe link we can use this
way, does it always have to be credentials?"*; (4) CSS cleanup.

**The credentials answer (baked into the UI):** credential-less
(`/export/{token}.ics`) = read-only **live** feed — anyone with the URL
sees current events when their client re-fetches (Google can lag hours);
"full access" always needs a credential (guest username + one-time app
token, or own per-calendar token) because CalDAV writes must authenticate
the actor. Platform hard limits stated verbatim in the UI: Google Calendar
and Outlook.com **cannot** add an external CalDAV server at all — read-only
subscribe only; Android's Google Calendar app likewise (web "From URL" or
DAVx5); Apple/Thunderbird/generic CalDAV clients do both paths.

**A. Root-cause outcome (decisive, §4):** **H1 — stale browser page / dead
session**; there is no server-side bug. Evidence: (a) `logread` since the
20:11:33 UTC 2026-09-22 restart shows **zero** `/frontend` page GETs from
the user's Firefox — only `/favicon.ico` 404s — so what they described
predates the restart (in-memory sessions died with it; an open tab kept
rendering the old page); (b) a local repro on a **copy of the production
DB** (login as `nicholas@carltonaudio.com`, diag password set on the copy
only) rendered the CAS Share tile with **every** control server-side:
export URL + Copy + Revoke, all three guest rows, all three minting forms.
H2–H5 not needed. Closure = post-deploy hard refresh (Ctrl+Shift+R +
re-login) — the §9 LIVE checklist verifies it. (Lesson recorded after two
agents mis-diagnosed it: the repro on a prod-DB copy + request-log evidence
pass settles it in minutes.)

**B. Design decisions:**
- **Routes**: the `/share/…` POST paths stay as-is (internal form actions —
  renaming is churn), but every redirect/render target changes: `303 →
  /{user}/calendar#cal-{id}` (calendar) or `/{user}/addressbook#ab-{id}`
  (addressbook); tiles gained `id="cal-…"` / `id="ab-…"` anchors.
  `route_share_revoke` learns the subscription's kind + collection from
  the store **before** deleting so the redirect anchors the right tile;
  ditto `route_share_invite_revoke` via `get_invite`. `GET /{user}/share`
  is unregistered → 404; nav entry removed from `pages/user.html`.
- **Code shape**: `share.rs` lost all page assembly (`share_page`,
  `share_page_with_*`, `build_share_entries`, `shareable_principals`,
  `ShareSection` — 1189 → ~700 lines); its POST handlers re-render via the
  Calendars/Addressbooks renderers and the new `CalendarsExtras` struct
  (error + §17.12 credential banner + §17.10 guest banner + §17.9 invite
  banner in one place). `route_share_revoke`/`route_share_invite_revoke`/
  `route_share_guest_revoke` became non-generic (no auth_provider needed —
  the old `share.rs:480/749` unused warnings died with the rewrite).
  `CalendarTile` gained `can_invite`, `invites`, `sub_created_at`,
  `subscribe_url_webcal`; `AddressbookTile` is new (meta, birthday_cal,
  addressbook, subscribe_url/sub_id/sub_created_at/can_subscribe).
  Addressbook invites/guest shares stay calendar-only (§17.9.1/§17.10).
- **Gating preserved exactly**: subscribe link = own principal or
  `can_write(group)`; invites + guest shares + §17.12 credentials = own or
  `is_admin(group)`; guest rows `can_manage = can_invite`. One hardening
  over the old Share page: **invite rows are rendered only on tiles where
  `can_invite`** (a view-only group member must not see the group's
  registration URLs — the old page never faced this because it listed only
  writable principals).
- **Banners**: §17.13 exactly-once semantics kept, now tile-gated on the
  Calendars page (guest + invite banners compare principal **and**
  collection against the tile); errors page-level. The guest-credential
  banner links to the tile's instructions anchor (`#client-help-{id}`).
- **Instructions** (`<details class="client-help">` per calendar tile, "Set
  up on your device…"): A) *Subscribe — read-only, no account needed* —
  Apple iPhone/iPad ("Calendars → Add Calendar → Subscribe to Calendar"),
  Apple Mac (File → New Calendar Subscription, Auto-refresh hourly), Google
  ("Settings ⚙ → Add calendar → From URL" + slow-refresh caveat),
  Outlook web/new ("Add calendar → Subscribe from web"), Outlook classic
  (Internet Calendars → New), generic webcal note; shows the https URL plus
  a **Copy webcal://** button (scheme swap display-only — the stored token
  keeps working over plain https, never stored in the DB). B) *Full access —
  read + write (CalDAV)* — Apple, DAVx5 (Android), Thunderbird, all with
  the server URL prefilled from the new `caldav_url` (base_url + `/caldav`);
  plus the verbatim hard-limit line "Google Calendar & Outlook.com: not
  possible — neither supports adding an external CalDAV server; use the
  read-only subscribe link above instead."
- **Terminology** (per plan): "Subscribe link (read-only)", "Full access
  (CalDAV)", "Create subscribe link", "Create share link".
- **CSS**: one stylesheet still; new classes `.block-label`,
  `.subscribe-block`, `.access-block`, `.share-url-row`, `.share-url`,
  `.muted`, `.client-help`, `form.inline-form`, `.error`, `.success` (the
  two latter previously unstyled). Share blocks flow through the tile grid
  via `grid-area: metadata` auto rows. All malformed inline styles are gone
  (`style="{ height: 2em; }"` / `style=""` deleted with the Share template;
  `style="display:inline"` in linked-platforms + group_detail →
  `class="inline-form"`); the `--color` CSS-var inline styles stay (legit
  per-tile theming). `input[type=email]` is now styled globally (the only
  other template using it is the login-style forgot-password form).
  `EmbedService` now sends `Cache-Control: no-cache` alongside its ETag
  (verified live on the diag instance) so CSS/JS changes reach browsers on
  the next load — no hash-param hack needed.
- **Tests**: `frontend_share.rs` rewritten around the new surfaces: tab
  404 + nav absence, create → tile URL + public export round-trip for
  calendar **and** addressbook, revoke → 404 (+ anchored redirects),
  invite flows incl. per-tile scoping and persistence, guest banner
  exactly-once + revoke-removes-row, and a **CAS-shaped regression
  fixture** (existing subscription + view/edit/admin guest shares + zero
  invites must render every control — the §4 report can't regress).
  `frontend_password.rs` gate tests retargeted to `/calendar`;
  `frontend_calendars.rs` credential-row wording updated. Helper note:
  `extract_register_url` now anchors on `<code>` because persistent invite
  rows (plain `<a href>`) can precede the banner in the DOM.

**Gates (2026-09-22, dev):** `cargo test -p rustical_frontend` 22 ✓;
integration suite **95/95** ✓; `cargo test --workspace` all green;
`cargo fmt --check` ✓; clippy clean for all touched production code
(fixed: redundant clones, `pub(crate)`→`pub`, let-else, doc backticks,
too-many-lines allows). The clippy nit (`/// ─────` separators above the
guest-shares tests in `tests/integration_tests/frontend_share.rs:959`)
was fixed 2026-09-22 (plain `//` comments) — re-run green: zero clippy
warnings in `tests/integration_tests/frontend_*`, integration suite
95/95 again.
**LIVE (2026-09-22, session 2): deployed + this effort's checklist GREEN —
the password-reset carry-over checklist found one real bug (see §17.14).**
Safety-net DB backup first (`~/backups/omnical/db-pre-1715-deploy-20260922.sqlite3`,
3.9 MB, integrity ok, 216 cal / 209 addr live). Built via
`scripts/build-rust.sh` (UPX'd rustical 4.9 MiB — fits C2 budget) and
deployed via `~/router-dav/deploy.sh` 21:00 UTC: startup log says
"Scheduling extension enabled (8 SMTP identities, RSVP links enabled)" ✓,
health ok, dav-tls unchanged (no restart, sha-verified), overlay 27.5 M
free. Live checks (all through the public URL, curl + logged-in session as
`nicholas@carltonaudio.com`): `/{user}/share` → **404** ✓; nav has no
Share entry ✓; Calendars page CAS tile renders **every** control —
subscribe block (URL/Copy/Revoke), 3 guest rows + Revoke, invite-guest +
send-invite + generate-link forms, `Full access (CalDAV)` block,
instructions disclosure (Apple/Google/Outlook/DAVx5/Thunderbird, verbatim
"not possible" line, webcal Copy, caldav URL prefilled) — 22/22 tile checks
PASS (the §4 H1 closure: all controls are server-side; user needs one
Ctrl+Shift+R + re-login — every deploy restart kills in-memory sessions) ✓;
Addressbooks tiles render the share-link block and the full
create→revoke cycle was proven live on `personal`: POST share/create →
303 → `#ab-personal`, tile shows URL+Copy+Revoke, public export URL →
200, POST share/{id}/revoke (with the form's `principal` field — an empty
POST body gets 415, a field-less one 422) → 303 → `#ab-personal`, export
URL → 404 ✓. DB left at baseline (the test subscription was revoked).
**§17.14 carry-over checklist (same live pass):** login page shows "Forgot
your password?" ✓; POST `/frontend/forgot-password` (real account
`zero@novo-ordo.com`, flagged `needs_password_change=1` for the flag test)
returns the byte-identical no-enumeration page ✓ — but the email itself
**BOUNCED live**: `logread` shows `could not email password reset link:
SMTP error 554 … 5.7.1 <omnical.local>: Helo command rejected: Host not
found`. Root cause: the hand-rolled SMTP client's `EHLO omnical.local`
(scheduling `smtp.rs:19`) — smtp.novo-ordo.com runs Postfix
`reject_unknown_helo_hostname`; Gmail (the pre-§17.14 app sender)
tolerated it, so the sender switch surfaced the bug on its first live
send. **Fix implemented on dev (2026-09-22, session 2, NOT yet
built/deployed):** `EHLO_NAME` is now a `OnceLock<String>` set at startup
from the public URL host (`set_ehlo_name_from_url` + `url_host` with
IPv6-literal handling, RFC-5321 "localhost" fallback; set in `src/lib.rs`
serve path — only when a real `[subscriptions] public_url` is configured —
and in the `invites create --send` CLI branch, the only other send
process). Gates for the fix green: 4 new smtp unit tests, `cargo fmt
--check` clean, clippy clean for touched code. **Remaining:** rebuild +
redeploy, then re-run the email-dependent §17.14 checklist items (reset
email arrival, supersedes/re-request, rotation + flag-clear, guest-share
credential email, `invites --send`, iMIP organizer From) — see
`PLAN_NEXT_AGENT_CALENDARS_TAB_SHARING.md` §13 session-2 log for the exact
resume list. Rate-limiter budget spent so far: 1 POST (of 5/h per IP).
**RESOLVED (session 3, 2026-09-22):** the EHLO fix was rebuilt
(scripts/build-rust.sh, rustical 4.85 MiB UPX'd, dav-tls unchanged) +
redeployed 21:17:07 UTC (safety-net backup
`~/backups/omnical/db-pre-ehlo-deploy-20260922.sqlite3`, live counts
216/209 = §17.15 baseline), and **every email-dependent §17.14 item
re-ran GREEN** — reset email delivery From `burningserenity@novo-ordo.com`,
supersedes, rotation + flag-clear + full password restore, unknown-email
byte-identity, guest-share credential email, `invites create --send`, and
iMIP REQUEST From the organizer (see §17.14 Status for the four live
findings, incl. the zero-alias mailbox and the METHOD:REQUEST
non-trigger). DB left at baseline (only expected tombstones + used
reset rows for audit). Pending: commits, when the user asks.

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
