# PLAN_NEXT_AGENT_PASSWORD_RESET — finish & ship the forgot-password flow + novo-ordo sender switch

Date: 2026-09-22
Repos:
- **Code**: `/home/burningserenity/router-dav/rustical` — branch `omnical-scheduling`, **all changes UNCOMMITTED** (do not commit unless the user asks).
- **Deploy scripts**: `/home/burningserenity/router-dav/scripts/render-router-config.sh` (also uncommitted).
- This planning repo (`~/Documents/ByteWheel/omnical`) is docs-only; no code here.

## 1. User request (what this effort delivers)

1. Self-service password reset: a "Forgot your password?" link on the login
   page → email to the user with a one-time reset link → set a new password.
2. App-generated emails (registration invites, guest-share credentials, and
   the new reset emails — everything sent via `scheduling.smtp.first()`)
   must come **From `burningserenity@novo-ordo.com`** instead of
   `burningserenity@gmail.com`. iMIP calendar invitations are **unchanged**
   (they always match the organizer's identity via `smtp_account()`).

User decision 2026-09-22: `burningserenity@novo-ordo.com` **shares
`nfcalaway@novo-ordo.com`'s SMTP credentials** (same pattern as
`zero@novo-ordo.com`) — no new `pass` entry needed.

## 2. State: everything is implemented and green except ONE test

`cargo check --workspace` passes. `cargo clippy` is clean for all new code
(only pre-existing warnings elsewhere, e.g. `share.rs:480,749` unused
`auth_provider`). Test status at handoff:

| Suite | Result |
|---|---|
| `cargo test -p rustical_scheduling -p rustical_store -p rustical_store_sqlite` | **100 passed** |
| `cargo test -p rustical` (unit + `tests/integration_tests` + `tests/http_integration`) | **148 passed** |
| `cargo test -p rustical_frontend` | **21 passed, 1 FAILED** |

The one failure is a **bug in the test itself, not the handler**:
`routes::password_reset::tests::rate_limit_is_shared_by_both_endpoints`
panicked at `crates/frontend/src/routes/password_reset.rs:982` —
`reset 2: left: 429, right: 400`. The per-IP budget is 5/h and the loop does
3×(forgot+reset) = **6** POSTs, so the 6th POST (3rd reset) is correctly
rejected with 429. The limiter and handlers behave exactly as designed.

## 3. THE FIX (step 1 of your work)

In `crates/frontend/src/routes/password_reset.rs`, replace the body of
`rate_limit_is_shared_by_both_endpoints` (currently ~line 966) so exactly 5
attempts are *allowed* before the 6th is rejected, and assert both endpoints
hit the shared budget:

```rust
#[tokio::test]
async fn rate_limit_is_shared_by_both_endpoints() {
    let rig = TestRig::new().await;
    // Burn the per-IP budget (5/h) across both endpoints: 5 attempts are
    // allowed, the 6th is rejected no matter which endpoint it hits
    // (rejected attempts are not recorded).
    for i in 0..3 {
        let resp = rig
            .post("/frontend/forgot-password", &[("email", "a@b.cd")])
            .await;
        assert_eq!(resp.status(), StatusCode::BAD_REQUEST, "forgot {i}");
    }
    for i in 0..2 {
        let resp = rig
            .post("/frontend/reset-password/whatever", &[("password", "x")])
            .await;
        assert_eq!(resp.status(), StatusCode::BAD_REQUEST, "reset {i}");
    }
    let resp = rig
        .post("/frontend/forgot-password", &[("email", "a@b.cd")])
        .await;
    assert_eq!(resp.status(), StatusCode::TOO_MANY_REQUESTS);
    let resp = rig
        .post("/frontend/reset-password/whatever", &[("password", "x")])
        .await;
    assert_eq!(resp.status(), StatusCode::TOO_MANY_REQUESTS);
}
```

Then:

```
cargo test -p rustical_frontend        # expect 22 passed, 0 failed
cargo test --workspace                 # full sweep, expect all green
```

Also run `cargo fmt` and review the diff (limit to the files touched by this
effort if the repo has pre-existing drift — it was not run yet).

## 4. What was built (file inventory)

### New files
- `crates/store/src/password_reset_store.rs` — `PasswordReset` struct +
  `PasswordResetStore` trait (`add_reset` / `get_reset` / `redeem_reset`,
  default impls error like `InviteStore`).
- `crates/store_sqlite/src/password_reset_store.rs` — SQLite impl
  (`SqlitePasswordResetStore`, wraps `SqliteCalendarStore` for the pool like
  the invite store; runtime `sqlx::query` deliberately, no `.sqlx` churn).
- `crates/store_sqlite/migrations/20260922120000_password_resets.{up,down}.sql`
  — table `password_resets(id, principal_id, token_hash UNIQUE, created_at,
  expires_at, used_at)` + index on `(principal_id, used_at)`.
- `crates/frontend/src/routes/password_reset.rs` — handlers
  (`route_get/post_forgot_password`, `route_get/post_reset_password`),
  `ResetRateLimiter` (per-IP 5/h + global 30/h sliding window, shared by both
  POSTs, `X-Forwarded-For` first hop), helpers, and a full `mod tests` rig
  (in-memory SQLite + `frontend_router` + session layer).
- Templates: `crates/frontend/public/templates/pages/forgot_password.html`,
  `reset_password.html`, `password_reset_invalid.html` (askama, extend
  `layouts/default.html`, same `login_window` style as the login page).

### Modified files
- `crates/store/src/lib.rs`, `crates/store_sqlite/src/lib.rs` — module +
  re-export.
- `crates/scheduling/src/mime.rs` — `build_password_reset(account, to,
  reset_url, expires_at)` (RFC 5322 plaintext, mirrors
  `build_registration_invite`; From = account identity; "If you did not
  request this email…") + 2 tests (`password_reset_carries_reset_url`,
  `password_reset_has_no_dot_lines`).
- `crates/frontend/Cargo.toml` — added `sha2.workspace = true` and
  `sqlx.workspace = true` (dev-dependencies).
- `crates/frontend/src/routes/mod.rs` — `pub mod password_reset;`
- `crates/frontend/src/routes/login.rs` — `LoginPage` gained
  `forgot_password_available`; `route_get_login` now also extracts
  `Extension<Vec<SmtpAccount>>` + `Extension<String>` (public_url) and calls
  `reset_available()`.
- `crates/frontend/public/templates/pages/login.html` — "Forgot your
  password?" link, gated on `forgot_password_available`.
- `crates/frontend/src/lib.rs` — routes mounted next to `/login`
  (`/forgot-password`, `/reset-password/{token}`), `frontend_router` gained
  `password_reset_store: Arc<dyn PasswordResetStore>` param, new Extensions:
  store + `Arc<ResetRateLimiter>` (created inside `frontend_router`).
- `src/app.rs` — `make_app` gained `password_reset_store` param, passed to
  `frontend_router`.
- `src/lib.rs` — `get_data_stores` now returns an **11-tuple** ending in
  `Arc<dyn PasswordResetStore>`; `cmd_serve` destructures + passes it;
  `SqlitePasswordResetStore` constructed alongside the invite store.
- `src/commands/{invites,subscriptions,guest_shares,principals}.rs` —
  positional tuple patterns got a trailing `_` for the 11th element.
- `tests/integration_tests/mod.rs` — `make_app` call gains
  `password_reset_store` (built from `SqlitePasswordResetStore::new(
  cal_store.clone())`).
- `tests/http_integration.rs` — **pre-existing breakage fixed as drive-by**:
  two `InviteCreateArgs` literals were missing the `send` field added by the
  previous commit (`c2b2492c`); now `send: false`. Without this the suite
  could not compile.
- `scripts/render-router-config.sh` — see §5.

## 5. Design decisions (do not re-derive; keep if touching the code)

- **Token**: 64-char alphanumeric (like `register.rs::random_token`); only
  its SHA-256 hex digest is stored (`token_hash()`), so a DB leak leaves no
  working links. Expiry **1 hour** (`RESET_TOKEN_EXPIRY_SECS = 3600`), plain
  string comparison `YYYY-MM-DDTHH:MM:SSZ` like invites.
- **Single-use atomicity**: `redeem_reset` = `UPDATE … WHERE token_hash = ?
  AND used_at IS NULL AND expires_at > ?` inside a transaction, then marks
  the principal's *other* outstanding tokens used. `add_reset` supersedes the
  principal's previous unused tokens first. So: at most one usable link per
  account at any time.
- **No enumeration**: known email, unknown email, and OIDC-only accounts
  (password `None` → no email) all get the byte-identical
  `RESET_SENT_MSG` page; unknown/used/expired tokens all get the identical
  `INVALID_RESET_MSG` page. Frontend tests assert body equality.
- **CSRF**: session-bound `csrf` key, rotated on every rendered response;
  mismatch → 400 "This form has expired. Please load it again."
- **Password rules**: reuses `[frontend] min_password_length` (default 12);
  `update_password` also clears `needs_password_change` (existing SQLite
  behavior, `principal_store.rs:379`), so a reset satisfies the forced-change
  nudge.
- **Reset POST does NOT auto-login** — redirects to `/frontend/login`.
- **Availability**: the login page only shows the link, and `POST
  /forgot-password` only works, when `allow_password_login && smtp non-empty
  && public_url non-empty` (`reset_available()`). The *reset* endpoints only
  require `allow_password_login` (a minted link stays redeemable even if
  SMTP is later removed).
- **Email delivery**: fire-and-forget `tokio::spawn(smtp::send_mail(…))`
  exactly like the guest-share email in `share.rs:1016-1043` — failures are
  logged, never user-facing. Sender = `scheduling.smtp.first()`.
- **Endpoints live in the frontend crate** (unauthenticated, next to
  `/login`, outside `user_router`'s gates); mounted only when the frontend is
  enabled. No new config keys.
- **SQLx offline metadata untouched** (new queries use runtime API on
  purpose — see invite store header comment).

## 6. Sender switch details (`render-router-config.sh`)

- `ACCOUNTS` table: `burningserenity@novo-ordo.com|smtp.novo-ordo.com|587|
  nfcalaway@novo-ordo.com|secrets/email/nfcalaway@novo-ordo.com/smtp` is now
  the **FIRST** row; `burningserenity@gmail.com` remains (it is an iMIP
  organizer principal and must keep its SMTP account). Header comments
  updated: 8 SMTP accounts, order-matters note.
- Because `invites --send`, guest-share emails, calendar-credential emails
  and password-reset emails all use `smtp_accounts.first()`, they now send
  From `burningserenity@novo-ordo.com` with zero code changes.
- No IMAP row was added for it (not an iMIP organizer; inbound replies are
  not polled for it). Add one (with the novo-ordo `ca_file`) only if it ever
  becomes an iMIP organizer principal.
- No new pass entry is required (authenticates as `nfcalaway@novo-ordo.com`,
  mirroring `zero@novo-ordo.com`). If SMTP rejects that identity, the
  fallback is a dedicated `secrets/email/burningserenity@novo-ordo.com/smtp`
  entry + changing the username column.

## 7. Deploy (unchanged flow — see `PLAN_NEXT_AGENT_DEPLOY_SHARING.md`)

1. `cd ~/router-dav && scripts/build-rust.sh` (builds the router binary).
2. Deploy binary + config the way `deploy.sh` does:
   `scripts/render-router-config.sh | ssh router 'cat > /etc/rustical/config.toml'`
   (0600 on the router). The migration (`password_resets` table) runs
   automatically on restart.
3. Restart the service on the router; check the startup log says
   `Scheduling extension enabled (8 SMTP identities, RSVP links enabled)`.

## 8. Live verification checklist (production, https://0115d8cf.duckdns.org:8443)

**Results so far (2026-09-22 session 2 — see also
`PLAN_NEXT_AGENT_CALENDARS_TAB_SHARING.md` §14):**

- [x] `/frontend/login` shows "Forgot your password?" — ✓ live 2026-09-22.
- [~] Submit a real account → **BLOCKED LIVE: the email BOUNCED.**
      `logread`: `SMTP error 554 after RCPT TO:<zero@novo-ordo.com>:
      5.7.1 <omnical.local>: Helo command rejected: Host not found` —
      the hand-rolled SMTP client's `EHLO omnical.local` const is rejected
      by smtp.novo-ordo.com (Postfix `reject_unknown_helo_hostname`);
      Gmail tolerated it before the sender switch. **Fix implemented on
      dev** (config-derived EHLO name from `public_url`, set in the serve
      path + the `invites --send` CLI branch; 4 unit tests, fmt + clippy
      green) — needs rebuild + redeploy, then re-run every email item
      below. The no-enumeration "reset sent" page itself is ✓ (byte-stable
      response for a known account).
- [ ] Open the link → "Set a new password" form; set one; old password
      fails, new one logs in; `needs_password_change` users land normally
      (flag cleared). — `zero@novo-ordo.com` is currently flagged
      `needs_password_change=1` on the router (session-2 setup for exactly
      this test); **restore the pass-stored password afterwards** via
      `rustical principals edit` (stdin, headless).
- [ ] Reuse/re-request: old link now shows the generic invalid page; a fresh
      request supersedes the previous link.
- [ ] Unknown email gets the same "If an account with that email address
      exists…" page (no oracle).
- [ ] Send a guest-share credential email → From
      burningserenity@novo-ordo.com; iMIP invite from an event organized by
      e.g. burningserenity@gmail.com still arrives From that organizer.
- [ ] `rustical invites create --email … --send` arrives From
      burningserenity@novo-ordo.com.

Rate-limiter note: 5 POSTs/h per IP shared by both reset endpoints;
1 spent in session 2. Mailbox access for `zero@novo-ordo.com`:
`mbsync Novo-Ordo-zero` (it's an IMAP folder of the nfcalaway@novo-ordo.com
account; `~/.local/share/novo-ordo.pem` is the pinned intermediate).
Cleanup pattern for the guest/invite/iMIP tests: mint on a throwaway
target, verify, then revoke + cascade-remove (§17.8/§17.10 patterns).

## 9. Optional follow-ups (ask the user)

- Add a PLAN.md section for the password-reset extension (repo convention:
  every Omnical feature is logged there with §17.x numbering — find the next
  free number after §17.10 and mirror the existing section style).
- Pre-existing warnings to potentially clean up separately (NOT this
  effort's scope): `share.rs:480,749` unused `auth_provider`.
- Commit: **only if the user asks** (they have not). Suggested message if so:
  `frontend: forgot/reset password via emailed one-time links; app emails now send from burningserenity@novo-ordo.com`