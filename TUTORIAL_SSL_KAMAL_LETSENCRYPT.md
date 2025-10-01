# Multi-Service Kamal Deployment with Hybrid SSL: Cloudflare + Let's Encrypt

A complete guide to deploying multiple services with Kamal while solving SSL challenges and file upload limitations.

## Table of Contents

- [Overview](#overview)
- [The Challenge](#the-challenge)
- [Architecture Decision](#architecture-decision)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
- [Common Pitfalls](#common-pitfalls)
- [Verification](#verification)
- [Maintenance](#maintenance)
- [Conclusion](#conclusion)

---

## Overview

This tutorial documents a real-world deployment scenario: running multiple web services (Frontend, API, AI Service) on a single server using Kamal 2.x, with a hybrid SSL approach that combines Cloudflare Origin Certificates and Let's Encrypt to overcome upload size limitations while maintaining security and performance.

### What You'll Learn

- How to deploy multiple Kamal services on one server
- Understanding Kamal-proxy SSL configurations
- Implementing hybrid SSL (Cloudflare + Let's Encrypt)
- Solving path-based routing vs. subdomain routing challenges
- Bypassing Cloudflare's file upload limits
- Certificate management and auto-renewal

### Tech Stack

- **Kamal 2.x** - Deployment tool
- **Kamal-proxy** - Reverse proxy with automatic SSL
- **Rails API** - Backend service
- **Next.js** - Frontend application
- **FastAPI** - AI service (internal only)
- **Cloudflare** - CDN/DNS/DDoS protection
- **Let's Encrypt** - Free SSL certificates
- **DigitalOcean** - Cloud hosting provider

---

## The Challenge

### Initial Setup

We started with a typical multi-service architecture:

```
app.example.com/          → Frontend (Next.js)
app.example.com/api/*     → API (Rails) via path prefix
ai-server:9001            → AI Service (internal)
```

**Initial SSL Strategy:** Cloudflare Origin Certificates for all services.

### Problems Encountered

#### Problem 1: Path-Based Routing with TLS

When trying to deploy the API with a path prefix (`/api`) and TLS:

```yaml
# api/config/deploy.yml
proxy:
  hosts:
    - app.example.com
  path_prefix: /api
  ssl:
    certificate_pem: CLOUDFLARE_CERTIFICATE_PEM
    private_key_pem: CLOUDFLARE_PRIVATE_KEY_PEM
```

**Error:**
```
Error: TLS settings must be specified on the root path service
```

**Root Cause:** Kamal-proxy requires the service serving the root path (`/`) to define SSL settings first. Path-prefixed services cannot independently configure SSL when the root service already has SSL configured.

#### Problem 2: Cloudflare Upload Size Limits

Even after solving the routing issue, we hit Cloudflare's upload limits:

```
HTTP/1.1 413 Payload Too Large
Origin https://app.example.com is not allowed by Access-Control-Allow-Origin
```

**Cloudflare Upload Limits by Plan:**
- Free: 100 MB
- Pro: 100 MB
- Business: 200 MB
- Enterprise: 500 MB

Our use case required uploading files **over 100MB** (audio/video sermon files), making Cloudflare's proxy unsuitable for the API.

#### Problem 3: Wildcard Certificate Scope

We attempted to use a subdomain like `api.app.example.com`, but our Cloudflare Origin Certificate only covered:
- `*.example.com` (one level)
- `app.example.com`
- `example.com`

**Wildcard limitation:** `*.example.com` matches `api.example.com` ✓ but NOT `api.app.example.com` ✗ (two levels).

---

## Architecture Decision

After multiple iterations, we arrived at a **hybrid SSL approach**:

### Final Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     DNS (Cloudflare)                         │
├─────────────────────────────────────────────────────────────┤
│ app.example.com     → Proxied (☁️)  → Frontend              │
│ api.example.com     → DNS Only (🌐) → API (direct)          │
└─────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────┐
│              Server: server.example.com                      │
│                   (203.0.113.10)                             │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Frontend (app.example.com)                           │   │
│  │ - Cloudflare Origin Certificate                      │   │
│  │ - Proxied (CDN, caching, DDoS protection)            │   │
│  │ - Port 443 via Cloudflare                            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ API (api.example.com)                                │   │
│  │ - Let's Encrypt Certificate                          │   │
│  │ - Direct connection (no Cloudflare proxy)            │   │
│  │ - Port 443 direct to server                          │   │
│  │ - No upload limits!                                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ AI Service (internal only)                           │   │
│  │ - No SSL (internal Docker network)                   │   │
│  │ - Port 9001 (not exposed externally)                 │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Design Rationale

| Service | SSL Provider | Cloudflare Proxy | Reason |
|---------|-------------|------------------|---------|
| Frontend | Cloudflare Origin | Yes ☁️ | Benefits from CDN, caching, DDoS protection |
| API | Let's Encrypt | No 🌐 | Bypasses upload limits, direct to server |
| AI Service | None | N/A | Internal only, no external access |

---

## Prerequisites

### DNS Provider (Cloudflare)

1. Domain managed by Cloudflare
2. Ability to control DNS proxy settings (orange/gray cloud)

### Server Requirements

1. Single server with public IP
2. Ports open: 22 (SSH), 80 (HTTP), 443 (HTTPS)
3. Docker installed
4. Kamal 2.x installed

### Cloudflare Origin Certificate (for Frontend)

Generate in Cloudflare Dashboard:
1. Go to SSL/TLS → Origin Server
2. Create Certificate
3. Select hostnames: `*.example.com`, `example.com`, `app.example.com`
4. Download certificate (PEM format)
5. Download private key (PEM format)

---

## Implementation

### Step 1: DNS Configuration

Configure Cloudflare DNS with mixed proxy settings:

| Type | Name | Content | Proxy Status | Purpose |
|------|------|---------|--------------|---------|
| A | app.example.com | 203.0.113.10 | ☁️ Proxied | Frontend via Cloudflare |
| A | api.example.com | 203.0.113.10 | 🌐 DNS Only | API direct to server |
| A | server.example.com | 203.0.113.10 | 🌐 DNS Only | Server management |

**Critical:** `api.example.com` must be **DNS Only** (gray cloud) for Let's Encrypt to work!

**Verify DNS:**
```bash
# Should return Cloudflare IPs (proxied)
dig app.example.com +short
# Output: 104.21.x.x, 172.67.x.x

# Should return your server IP (direct)
dig api.example.com +short
# Output: 203.0.113.10
```

### Step 2: Store Cloudflare Certificates

Create `.kamal/secrets` files for services using Cloudflare certificates:

**`frontend/.kamal/secrets`:**
```bash
CLOUDFLARE_CERTIFICATE_PEM="-----BEGIN CERTIFICATE-----
MIIEpDCCA4ygAwIBAgIUXXXXXXXXXXXXXXXXXXXXXXXXXXX
[... certificate content ...]
-----END CERTIFICATE-----"

CLOUDFLARE_PRIVATE_KEY_PEM="-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC
[... private key content ...]
-----END PRIVATE KEY-----"
```

**Security Note:** 
- Never commit `.kamal/secrets` to version control
- Add to `.gitignore`
- Use environment variables or secret management in production

### Step 3: Frontend Configuration (Cloudflare SSL)

**`frontend/config/deploy.yml`:**
```yaml
service: frontend
image: registry.example.com/my-org/frontend

# Deploy to single server
servers:
  web:
    hosts:
      - server.example.com
    cmd: node server.js

# Cloudflare Origin Certificate
proxy:
  hosts:
    - app.example.com  # Public domain via Cloudflare
  ssl:
    certificate_pem: CLOUDFLARE_CERTIFICATE_PEM
    private_key_pem: CLOUDFLARE_PRIVATE_KEY_PEM
  response_timeout: 1800
  buffering:
    requests: true
    responses: true
    max_request_body: 600000000  # 600MB
    max_response_body: 0
    memory: 5000000
  healthcheck:
    path: /
    interval: 10
    timeout: 10

# Environment variables
env:
  clear:
    NODE_ENV: production
    PORT: 80
    HOSTNAME: 0.0.0.0
    NEXT_PUBLIC_API_URL: https://api.example.com  # Point to API subdomain
```

**Key Points:**
- Custom SSL certificate configuration
- `hosts` is an array (can have multiple)
- Frontend serves root path (`/`)
- API URL points to separate subdomain

### Step 4: API Configuration (Let's Encrypt SSL)

**`api/config/deploy.yml`:**
```yaml
service: api
image: registry.example.com/my-org/api

# Deploy to single server
servers:
  web:
    hosts:
      - server.example.com
    cmd: bin/rails server -b 0.0.0.0 -p 3000
  worker:
    hosts:
      - server.example.com
    cmd: bundle exec sidekiq

# Let's Encrypt automatic SSL
proxy:
  host: api.example.com  # Single host (not array!) for Let's Encrypt
  ssl: true              # Automatic Let's Encrypt
  app_port: 3000
  response_timeout: 1800
  buffering:
    requests: true
    responses: true
    max_request_body: 600000000  # 600MB - no Cloudflare limit!
    max_response_body: 0
    memory: 5000000

# Environment variables
env:
  clear:
    RAILS_ENV: production
    RACK_ENV: production
```

**Critical Differences:**
- `host:` (singular string) not `hosts:` (array)
- `ssl: true` for automatic Let's Encrypt
- No certificate_pem/private_key_pem needed
- No path prefix configuration

### Step 5: CORS Configuration

Since API and Frontend are on different subdomains, configure CORS:

**`api/config/initializers/cors.rb`:**
```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins(
      "http://localhost:8000",        # Local development
      "https://app.example.com",      # Production frontend (Cloudflare)
      "https://api.example.com"       # API subdomain (direct)
    )

    resource "*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true,
      expose: ['authorization'],
      max_age: 600  # Cache preflight for 10 minutes
  end
end
```

**Important:** Add both frontend and API domains to allowed origins!

### Step 6: Deployment Order

**Order matters!** Deploy root path service first.

#### 6a. Deploy Frontend
```bash
cd frontend
kamal deploy
```

**Expected output:**
```
--host="app.example.com" --tls 
--tls-certificate-path="/home/kamal-proxy/.apps-config/frontend/tls/web/cert.pem" 
--tls-private-key-path="/home/kamal-proxy/.apps-config/frontend/tls/web/key.pem"
```

#### 6b. Deploy API
```bash
cd ../api
kamal deploy
```

**Expected output:**
```
--host="api.example.com" --tls
```

Note: No certificate paths shown - Let's Encrypt handles this automatically!

**First deployment:** May take 30-60 seconds while Let's Encrypt performs ACME challenge.

### Step 7: Verification

**Check kamal-proxy status:**
```bash
ssh root@server.example.com "docker exec kamal-proxy kamal-proxy list"
```

**Expected output:**
```
Service       Host                Path  Target          State    TLS  
frontend-web  app.example.com     /     abc123:80       running  yes  
api-web       api.example.com     /     def456:3000     running  yes
```

Both services should show `TLS: yes`!

**Verify SSL certificates:**

Frontend (Cloudflare):
```bash
echo | openssl s_client -connect app.example.com:443 -servername app.example.com 2>/dev/null | grep "issuer"
# Should show: issuer=CloudFlare
```

API (Let's Encrypt):
```bash
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null | grep "issuer"
# Should show: issuer=Let's Encrypt
```

**Test endpoints:**
```bash
# Frontend
curl -I https://app.example.com
# Should return: HTTP/2 200

# API
curl -I https://api.example.com/api/v1/endpoint
# Should return: HTTP/2 (auth redirect or data)
```

---

## Common Pitfalls

### Pitfall 1: Wrong `ssl` Configuration

**❌ Wrong:**
```yaml
proxy:
  host: api.example.com
  ssl: false  # This disables SSL entirely!
```

**✅ Correct:**
```yaml
proxy:
  host: api.example.com
  ssl: true   # Automatic Let's Encrypt
```

### Pitfall 2: Using Array Instead of String

**❌ Wrong (for Let's Encrypt):**
```yaml
proxy:
  hosts:  # Array - won't work with Let's Encrypt
    - api.example.com
  ssl: true
```

**✅ Correct:**
```yaml
proxy:
  host: api.example.com  # Single string - required!
  ssl: true
```

### Pitfall 3: Cloudflare Proxy Still Enabled

**Problem:** DNS set to proxied (orange cloud) for API subdomain.

**Symptom:**
```
Error: Let's Encrypt ACME challenge failed
Connection timeout
```

**Fix:** Set `api.example.com` to DNS Only (gray cloud) in Cloudflare.

### Pitfall 4: Port 80 Not Open

**Problem:** Firewall blocking port 80.

**Symptom:**
```
Error: ACME HTTP-01 challenge failed
```

**Fix:** Ensure port 80 is open for Let's Encrypt validation:
```bash
ufw allow 80/tcp
ufw allow 443/tcp
```

### Pitfall 5: Conflicting SSL Configuration

**❌ Wrong:**
```yaml
proxy:
  host: api.example.com
  ssl:
    certificate_pem: CERT
    private_key_pem: KEY
  ssl: true  # Conflict! Can't have both
```

**✅ Correct (pick one):**

For custom certificate:
```yaml
proxy:
  host: api.example.com
  ssl:
    certificate_pem: CERT
    private_key_pem: KEY
```

For Let's Encrypt:
```yaml
proxy:
  host: api.example.com
  ssl: true
```

### Pitfall 6: Wrong Deployment Order

**❌ Wrong:** Deploy API before Frontend

This causes the root path service error because API tries to claim the root path first.

**✅ Correct:** Always deploy Frontend (root path `/`) first, then API.

### Pitfall 7: Wildcard Certificate Doesn't Cover Multi-Level Subdomains

**❌ Wrong assumption:**
```
Certificate: *.example.com
Covers: api.app.example.com ✗ (two levels)
```

**✅ Correct understanding:**
```
Certificate: *.example.com
Covers:
  - api.example.com ✓ (one level)
  - app.example.com ✓ (one level)
  - anything.example.com ✓ (one level)
Does NOT cover:
  - api.app.example.com ✗ (two levels)
  - cdn.api.example.com ✗ (two levels)
```

**Solution:** Use single-level subdomains or generate certificate with specific multi-level domains.

---

## Maintenance

### Certificate Renewal

#### Let's Encrypt (API)

**Auto-renewal:** Kamal-proxy automatically renews Let's Encrypt certificates when they have < 30 days remaining.

**Monitoring:**
```bash
# Check certificate expiration
ssh root@server.example.com "docker exec kamal-proxy ls -la /home/kamal-proxy/.config/kamal-proxy/state/certificates/"

# View renewal logs
ssh root@server.example.com "docker logs kamal-proxy --tail 100 | grep -i 'renew\|certificate'"
```

**Manual renewal (if needed):**
```bash
cd api
kamal proxy reboot
```

#### Cloudflare Origin Certificate (Frontend)

**Expiration:** Cloudflare Origin Certificates are valid for 15 years.

**Renewal process:**
1. Generate new certificate in Cloudflare Dashboard (before expiration)
2. Update `.kamal/secrets` with new certificate
3. Redeploy frontend:
```bash
cd frontend
kamal deploy
```

### Checking Service Health

**View all services:**
```bash
ssh root@server.example.com "docker exec kamal-proxy kamal-proxy list"
```

**Check specific service logs:**
```bash
# Frontend
kamal app logs -s frontend

# API web
kamal app logs -s api -r web

# API worker
kamal app logs -s api -r worker
```

### Updating Services

**Frontend:**
```bash
cd frontend
git pull
kamal deploy
```

**API:**
```bash
cd api
git pull
kamal deploy
```

**Both (in order):**
```bash
cd frontend && kamal deploy && cd ../api && kamal deploy
```

---

## Troubleshooting

### Issue: "TLS settings must be specified on root path service"

**Cause:** Trying to deploy path-prefixed service with SSL before root service.

**Solution:** 
1. Remove path prefix
2. Use subdomain instead
3. Deploy root service first

### Issue: 413 Payload Too Large (even after Let's Encrypt)

**Possible causes:**

1. **Cloudflare still proxying API:**
   - Check DNS: `dig api.example.com +short`
   - Should return server IP, not Cloudflare IPs
   - Fix: Set to DNS Only in Cloudflare

2. **Nginx/Proxy body size limit:**
   ```yaml
   # Increase in deploy.yml
   proxy:
     buffering:
       max_request_body: 600000000  # 600MB
   ```

3. **Application server limit (Rails/Puma):**
   ```ruby
   # config/puma.rb
   first_data_timeout 300
   persistent_timeout 3600
   ```

4. **Next.js body size limit:**
   ```typescript
   // next.config.ts
   experimental: {
     serverActions: {
       bodySizeLimit: '600mb'
     }
   }
   ```

### Issue: CORS Errors

**Symptom:**
```
Access to XMLHttpRequest has been blocked by CORS policy
```

**Fix:** Add all domains to CORS configuration:
```ruby
origins(
  "https://app.example.com",
  "https://api.example.com"
)
```

### Issue: Let's Encrypt ACME Challenge Fails

**Possible causes:**

1. **Port 80 not accessible:**
```bash
ufw allow 80/tcp
```

2. **DNS not pointing to server:**
```bash
dig api.example.com +short
# Must return server IP
```

3. **Multiple services claiming port 80:**
   - Only one Kamal deployment should handle Let's Encrypt per server
   - Use custom certificates for additional services

4. **Rate limiting:**
   - Let's Encrypt has rate limits (50 certificates per domain per week)
   - Use staging environment for testing: https://letsencrypt.org/docs/staging-environment/

### Issue: SSL Works Locally but Fails in Browser

**Cause:** Mixed content (HTTPS page loading HTTP resources).

**Fix:** Ensure all API calls use HTTPS:
```typescript
// ✅ Correct
const API_URL = 'https://api.example.com'

// ❌ Wrong
const API_URL = 'http://api.example.com'
```

---

## Performance Considerations

### Upload Performance

**Direct to Server (API):**
- ✅ Lower latency (no Cloudflare middle-hop)
- ✅ No size limits (server-limited only)
- ❌ No CDN caching
- ❌ Exposed to direct attacks

**Via Cloudflare (Frontend):**
- ✅ CDN caching (static assets)
- ✅ DDoS protection
- ✅ Geographic distribution
- ❌ 100MB upload limit (Free/Pro)

### Bandwidth Costs

**With Cloudflare proxy:** Bandwidth from Cloudflare is free (unlimited on all plans).

**Without Cloudflare proxy:** Bandwidth costs depend on your hosting provider (DigitalOcean: $0.01/GB outbound).

**Recommendation:** 
- Keep Frontend proxied (static assets benefit from Cloudflare)
- Keep API direct (large uploads benefit from no proxy)

### Caching Strategy

**Frontend (Cloudflare):**
```yaml
# Enable caching for static assets
# Cloudflare automatically caches common static files
# (.js, .css, .jpg, .png, etc.)
```

**API (Direct):**
```ruby
# Use application-level caching
Rails.cache.fetch("key", expires_in: 1.hour) do
  # expensive operation
end
```

---

## Security Best Practices

### 1. Rate Limiting

**Add Rack::Attack to Rails API:**

```ruby
# Gemfile
gem 'rack-attack'

# config/initializers/rack_attack.rb
class Rack::Attack
  # Limit uploads to 5 per hour per IP
  throttle('uploads/ip', limit: 5, period: 1.hour) do |req|
    req.ip if req.path.start_with?('/api/v1/') && req.post?
  end
  
  # Limit general requests to 300 per minute
  throttle('api/ip', limit: 300, period: 1.minute) do |req|
    req.ip if req.path.start_with?('/api/')
  end
end
```

### 2. Firewall Configuration

```bash
# Allow only necessary ports
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp   # SSH
ufw allow 80/tcp   # HTTP (Let's Encrypt)
ufw allow 443/tcp  # HTTPS
ufw enable

# Verify
ufw status
```

### 3. SSL/TLS Configuration

**Kamal-proxy defaults are secure:**
- TLS 1.2+ only
- Strong cipher suites
- HSTS enabled

**Additional hardening (if needed):**
```yaml
proxy:
  ssl_options:
    min_version: "TLSv1.3"
```

### 4. Secret Management

**Never commit secrets:**
```bash
# .gitignore
.kamal/secrets
*.pem
*.key
.env
```

**Use environment variables:**
```bash
# .kamal/secrets
DATABASE_PASSWORD=<%= ENV["DATABASE_PASSWORD"] %>
SECRET_KEY_BASE=<%= ENV["SECRET_KEY_BASE"] %>
```

### 5. Access Control

**Limit SSH access:**
```bash
# /etc/ssh/sshd_config
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
```

**Use fail2ban:**
```bash
apt-get install fail2ban
systemctl enable fail2ban
```

---

## Cost Analysis

### Free Tier Setup

| Component | Provider | Cost |
|-----------|----------|------|
| Domain | Any registrar | ~$12/year |
| DNS/CDN | Cloudflare Free | $0/month |
| SSL (Frontend) | Cloudflare Origin | $0/month |
| SSL (API) | Let's Encrypt | $0/month |
| Server | DigitalOcean Droplet | $6-12/month |
| **Total** | | **$7-13/month** |

### With Cloudflare Pro

| Component | Provider | Cost |
|-----------|----------|------|
| DNS/CDN | Cloudflare Pro | $20/month |
| All other components | Same as above | $6-12/month |
| **Total** | | **$26-32/month** |

**Note:** Cloudflare Pro still has 100MB upload limit, so doesn't solve our upload issue.

---

## Alternative Solutions

### Option 1: All Services via Cloudflare with Chunked Uploads

**Pros:**
- Single SSL source (Cloudflare)
- DDoS protection on all services

**Cons:**
- Complex client-side implementation
- Not all upload libraries support chunking
- Higher latency

### Option 2: Direct S3/R2 Uploads

**Pros:**
- No server upload limits
- Scalable storage
- Cloudflare R2 has zero egress fees

**Cons:**
- More complex flow (presigned URLs)
- Additional service dependency
- Storage costs

### Option 3: Upgrade to Cloudflare Business

**Pros:**
- 200MB upload limit
- Keeps all services proxied

**Cons:**
- $200/month cost
- Still has limits (if you need >200MB)

### Why We Chose Hybrid Approach

✅ **Cost-effective:** Free SSL, minimal infrastructure  
✅ **Simple:** No complex upload flows  
✅ **Unlimited uploads:** Only limited by server  
✅ **Best of both worlds:** Frontend gets Cloudflare benefits  

---

## Conclusion

This hybrid SSL approach with Kamal provides:

1. **Flexibility:** Different SSL strategies for different services
2. **Cost-effectiveness:** Free SSL with Let's Encrypt
3. **Performance:** Frontend via Cloudflare CDN, API direct
4. **Scalability:** No upload size limits on API
5. **Maintainability:** Automatic certificate renewal

### Key Takeaways

- **Path-based routing + TLS** is problematic in Kamal multi-service setups
- **Subdomain routing** is simpler and more flexible
- **Hybrid SSL** (Cloudflare + Let's Encrypt) solves specific use cases
- **DNS proxy settings** matter: proxied vs. DNS-only
- **Deployment order** matters: root path service first
- **Wildcard certificates** only cover one subdomain level

### When to Use This Approach

✅ **Good fit when:**
- Multiple services on one server
- Large file uploads required (>100MB)
- Cost-conscious deployment
- Need some services behind CDN, others direct

❌ **Not ideal when:**
- All services need DDoS protection
- Budget for paid CDN/proxy on all services
- Can use direct-to-storage uploads (S3/R2)
- Multi-server deployment (Let's Encrypt per-host)

---

## Additional Resources

### Documentation
- [Kamal Handbook](https://kamal-deploy.org/)
- [Kamal-proxy Configuration](https://github.com/basecamp/kamal-proxy)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Cloudflare Origin Certificates](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/)

### Community
- [Kamal GitHub Discussions](https://github.com/basecamp/kamal/discussions)
- [Ruby on Rails Forum](https://discuss.rubyonrails.org/)

### Tools
- [SSL Labs Server Test](https://www.ssllabs.com/ssltest/)
- [DNS Checker](https://dnschecker.org/)
- [Qualys SSL Test](https://www.ssllabs.com/ssltest/)

---

## Changelog

**2025-10-01:** Initial publication
- Documented complete SSL journey
- Added troubleshooting section
- Included security best practices

---

## License

This tutorial is released under [MIT License](https://opensource.org/licenses/MIT).

Feel free to use, modify, and share!

---

## Questions?

If you encounter issues or have questions:

1. Check the [Common Pitfalls](#common-pitfalls) section
2. Review the [Troubleshooting](#troubleshooting) guide
3. Search [Kamal GitHub Issues](https://github.com/basecamp/kamal/issues)
4. Ask in [Kamal Discussions](https://github.com/basecamp/kamal/discussions)

**Remember:** Always test in a staging environment before deploying to production!

---

**Happy Deploying! 🚀**

