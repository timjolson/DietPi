# Agent: dietpi-letsencrypt — README

Purpose
- Provide background, development & testing plan, and modification guidance
  for adding IP-address (and mixed IP+domain) certificate support to
  the `dietpi-letsencrypt` flow. This file collects relevant findings from
  Certbot and Let’s Encrypt docs and gives step-by-step developer instructions.

Status update (2026-06-07)
- **Dev-only constraint:** All further development MUST be confined to `/usr/local/bin/dietpi` (this is mounted to `/boot/dietpi`). Do not modify `/boot/dietpi` or system Certbot directories without explicit confirmation (for example: creating `/etc/letsencrypt-dev` or `/var/lib/letsencrypt-dev`).
- **Current artifacts created:**
  - `/usr/local/bin/dietpi/dietpi/dietpi-letsencrypt-dev` — dev script copy exists (development canonical path).
  - `/usr/local/bin/dietpi/dietpi/.dietpi-letsencrypt-dev` — dev settings file (project-local config).
  - Project-local Certbot runtime dirs: `/usr/local/bin/dietpi/letsencrypt-dev/config`, `/usr/local/bin/dietpi/letsencrypt-dev/work`, `/usr/local/bin/dietpi/letsencrypt-dev/logs` (created and used by the dev script).
  - Dev systemd renewal directory used by some dev hooks: `/etc/systemd/system/certbot-dev.service.d` (dev-only, avoid changing system timers without confirmation).
- **Next immediate actions (awaiting confirmation):**
  - Confirm you want any remaining `/boot/dietpi` copies removed or copied into `/usr/local/bin/dietpi` (I can consolidate them now).
  - Implement IP vs Domain vs Both logic inside the dev script and test against Let's Encrypt staging (will use project-local Certbot dirs and staging by default).

Key findings (authoritative sources)
- Certbot supports IP address requests using the `--ip-address` flag (added in
  Certbot v5.3.0). See Certbot release notes and `certbot --help` for options.
- Webroot plugin IP issuance improvements landed in Certbot v5.4.0.
- Let’s Encrypt supports short-lived (6-day) certificates and provides a
  `shortlived` profile; when requesting IP certificates from Let's Encrypt you
  must request the short-lived profile (`--preferred-profile shortlived`).
- X.509 SANs allow `iPAddress` name forms (RFC 5280) — but actual issuance is
  governed by CA policy (Let's Encrypt / other ACME CAs). Use the ACME staging
  server for safe testing.

Compatibility requirements
- Certbot: use **>= 5.3.0** for `--ip-address`; prefer **>= 5.4.0** for better
  webroot IP handling.
- Use Let's Encrypt staging for tests: `https://acme-staging-v02.api.letsencrypt.org/directory`

Quick reference commands
- Generate CSR with both IP and DNS SANs (example):
```bash
openssl req -new -nodes -out /tmp/test.csr -newkey rsa:2048 -keyout /tmp/test.key \
  -subj "/CN=example.com" \
  -config <(cat /etc/ssl/openssl.cnf <(printf "\n[req_ext]\nsubjectAltName=IP:203.0.113.5,DNS:example.com"))
```

- Certbot using CSR against Let’s Encrypt staging (short-lived profile):
```bash
certbot certonly --cert-name test-ip-domain --csr /tmp/test.csr \
  --server https://acme-staging-v02.api.letsencrypt.org/directory \
  --preferred-profile shortlived --config-dir /usr/local/bin/dietpi/letsencrypt-dev/config \
  --work-dir /usr/local/bin/dietpi/letsencrypt-dev/work --logs-dir /usr/local/bin/dietpi/letsencrypt-dev/logs --dry-run
```

- Direct Certbot request including IP (standalone/manual) with staging:
```bash
certbot certonly --standalone -d example.com --ip-address 203.0.113.5 \
  --preferred-profile shortlived --server https://acme-staging-v02.api.letsencrypt.org/directory \
  --config-dir /usr/local/bin/dietpi/letsencrypt-dev/config --work-dir /usr/local/bin/dietpi/letsencrypt-dev/work --logs-dir /usr/local/bin/dietpi/letsencrypt-dev/logs --dry-run
```

Notes about flags & behavior
- `--ip-address`: add one or more IPs (multiple flags allowed). Added in
  Certbot v5.3.0.
- `--preferred-profile shortlived`: request LE's short-lived profile for IP certs.
- `--csr`: allowed with `certonly` to submit a CSR (Certbot will not rewrite
  the CSR; use when you need full control over SANs).
- Use `--config-dir`, `--work-dir`, and `--logs-dir` to run a dev Certbot
  instance in parallel with a production instance (avoids Certbot lockfile
  conflicts).
- `--dry-run` uses the staging server by default; include `--server` to target
  a specific ACME server.

Development & testing plan (high level)
1. Create a safe development copy of the script:
   - Copy `dietpi-letsencrypt` → `dietpi-letsencrypt-dev` in `/usr/local/bin/dietpi`.
   - Copy `/boot/dietpi/.dietpi-letsencrypt` → `/boot/dietpi/.dietpi-letsencrypt-dev`.
   - Ensure the dev script reads its settings file path and never overwrites
     the production config by default.

2. Make minimal changes in the dev copy first:
   - Add new settings keys in the dev settings file: `LETSENCRYPT_IP`,
     `LETSENCRYPT_SHORTTERM` (boolean), and `LETSENCRYPT_CA_SERVER` (optional).
   - Use project-local certbot dirs and staging usage by default:
     `--config-dir /usr/local/bin/dietpi/letsencrypt-dev/config --work-dir /usr/local/bin/dietpi/letsencrypt-dev/work --logs-dir /usr/local/bin/dietpi/letsencrypt-dev/logs`.

3. Implement cert request logic in the dev script:
   - Flows:
     - IP-only: request IP SAN(s); use `--ip-address` or CSR flow; mark
       short-term behavior.
     - Domain-only: existing behavior (no change).
     - IP+domain: include both SAN types; treat as short-term for LE.
   - Validation: add `Validate_IP()` helper to ensure provided IPs are valid
     IPv4/IPv6 addresses.
   - CSR path (optional): implement `Generate_CSR_With_IP_SAN()` if you need
     to craft CSRs rather than using `--ip-address` flags.

4. Testing against staging:
   - Start with `--dry-run` and staging server. Validate issuance of test
     certs (they will be invalid/TEST_CERT but confirm flow).
   - Test all three flows (IP-only, domain-only, both).
   - Verify renewals: for short-lived certs, test renew cadence and that
     `certbot renew` behaves as expected (Certbot 4.0+ uses lifetime-based
     thresholds; certs <=10 days use 1/2 lifetime threshold).

5. Automation & scheduling:
   - Use randomized schedule for renewals; for 6-day certs, renew every ~3 days.
   - Use separate cron/systemd timers for dev script to avoid impacting
     production.

6. Documentation & PR:
   - Update `CONTRIBUTING.md` and add a testing checklist that uses the dev
     script and staging server.

Files to create/modify and guidance
-- `dietpi-letsencrypt-dev` (new): copy of `dietpi-letsencrypt` modified to:
  - Read the dev settings file under the project tree, by default:
    `/usr/local/bin/dietpi/dietpi/.dietpi-letsencrypt-dev`.
  - Use separate project-local Certbot dirs via `--config-dir`, `--work-dir`, `--logs-dir`.
  - Default to `--dry-run` and Let's Encrypt staging for safety; provide a flag to run live tests.

-- `/usr/local/bin/dietpi/dietpi/.dietpi-letsencrypt-dev` (new): config variables for dev runs.

- `func/dietpi-globals` (edit): add helper functions (put in the dev script if
  you prefer minimal intrusion):
  - `Validate_IP()` — simple IPv4/IPv6 validation using `grep`/`awk` or
    `python -c` quick check.
  - `Generate_CSR_With_IP_SAN()` — generate CSR with SANs when needed.

- `dietpi-letsencrypt` (production): do not modify yet. After dev testing,
  prepare a small patch that mirrors the tested changes and follow PR review.

Implementation tips
- Keep the dev script changes limited and reversible.
- Use `--server` to test alternate ACME CAs if Let's Encrypt refuses a flow.
- Respect rate limits during testing; do not hit production LE repeatedly.
- Use `--preferred-profile shortlived` when targeting Let's Encrypt for IP
  issuance.

Renewal timing guidance
- For 6-day short-lived certificates: renew every 3 days (Certbot 4.0+ uses
  1/2 lifetime threshold for certs <=10 days; adjust scheduling accordingly).
- Randomize run times (splay) to avoid renewal spikes; Certbot docs suggest
  adding a randomized sleep in cron or using systemd timer with jitter.

Security & operations notes
- Keep private keys secure; Certbot defaults to restrictive perms for
  `/etc/letsencrypt/live/*/privkey.pem`.
- Never commit keys or `.dietpi-letsencrypt` files with secrets to source
  control.

Troubleshooting
- If cert issuance fails for IP SANs, try CSR-based issuance (submit CSR
  that encodes SANs) or an alternate CA that explicitly supports IP SANs.
- Use the Let’s Encrypt community forum and Certbot logs (`/var/log/letsencrypt`) for
  detailed error messages.

References
- Certbot docs & `--help` (Certbot options, `--ip-address`, `--csr`, `--server`)
- Certbot release notes (v5.3.0 - 5.4.0) for IP-related features.
- Let’s Encrypt FAQ and short-lived cert announcement (policy on short-lived certs).
- RFC 5280 for X.509 SAN `iPAddress` form.

Maintainer checklist (when merging into production)
- Verify all tests pass in staging and renewal scheduling works.
- Keep production `dietpi-letsencrypt` changes minimal and documented.
- Add migration notes for users about Certbot version requirements.

-- End
