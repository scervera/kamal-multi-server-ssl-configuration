## Using the setup_kamal Script

This guide explains how to deploy the Asset Management System using `bin/setup_kamal`.

### Overview
- Automates configuration and deployment for both API (Rails) and Web (Next.js)
- Supports two modes:
  - Interactive (guided prompts)
  - Developer preset mode (`--dev -p`) that reads `.env.setup_kamal`
- Handles Cloudflare Origin Certificates and writes Kamal‑parsable secrets

### Prerequisites
- Kamal 2.7+ installed locally (`gem install kamal`)
- SSH access to the target server
- DNS A records pointing to the server for both:
  - `assets.yourdomain.com` (frontend)
  - `assets-api.yourdomain.com` (API)
- Certificate files in the project root:
  - `ssl-cert.pem` (Cloudflare Origin Certificate)
  - `ssl-key.pem` (Cloudflare Private Key)

### Files
- `.env.setup_kamal` — preset config (server, domains, tokens, etc.)
- `ssl-cert.pem` / `ssl-key.pem` — multi-line PEMs (not committed)
- `apps/api/.kamal/secrets` — generated; includes quoted multi-line `CLOUDFLARE_*`
- `apps/api/config/deploy.yml` and `apps/web/config/deploy.yml` — generated

### Running in Developer Preset Mode (recommended)
```bash
./bin/setup_kamal --dev -p
```
What happens:
1) Loads `.env.setup_kamal`. If `CLOUDFLARE_*` are not present, reads `ssl-cert.pem` / `ssl-key.pem`.
2) Generates deploy configs and secrets.
3) Writes certs to `apps/api/.kamal/secrets` as quoted multi-line values.
4) Validates cert keys exist, previews secret keys (names only).
5) Runs `kamal setup` and `kamal deploy` for API, then Web.

### Running Interactively
```bash
./bin/setup_kamal
```
- Select setup type (customer, developer, developer preset, or nuclear wipe)
- Answer prompts for domains, server, registry, passwords, certs, etc.
- The script validates DNS (if tools available) and SSH access
- Same generation/validation/deploy steps as preset

### Cleanup Options
If prior containers/volumes exist, the script offers:
- 1) Remove Asset Management System containers (recommended)
- 2) Skip cleanup
- 3) Remove all Docker objects (keeps Docker installed)
- 4) Nuclear option (wipe Docker and data)

### SSL Handling
- Certs are loaded from files when not present in `.env.setup_kamal`
- Secrets are written as:
```
CLOUDFLARE_CERTIFICATE_PEM="
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
"
CLOUDFLARE_PRIVATE_KEY_PEM="
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
"
```
- Deploy yml references:
```
proxy:
  ssl:
    certificate_pem: CLOUDFLARE_CERTIFICATE_PEM
    private_key_pem: CLOUDFLARE_PRIVATE_KEY_PEM
```

### Rotate Certificates
1) Replace `ssl-cert.pem` and/or `ssl-key.pem`
2) Rerun: `./bin/setup_kamal --dev -p`

### Troubleshooting
- Error: `Secret 'CLOUDFLARE_CERTIFICATE_PEM' not found`
  - Ensure `apps/api/.kamal/secrets` contains the quoted multi-line `CLOUDFLARE_*` entries
  - Ensure you ran the script from repo root and cert files exist
- Error: `unable to load certificate`
  - Verify PEM formatting (BEGIN/END lines, no escaped `\n`)
- DB auth errors on fresh deploy
  - Use cleanup option 1; if persisted volumes remain, choose option 3 to reset

### Manual Commands (if needed)
```bash
# API
cd apps/api
kamal setup -c config/deploy.yml
kamal deploy -c config/deploy.yml

# Web
cd ../web
kamal setup -c config/deploy.yml
kamal deploy -c config/deploy.yml
```
