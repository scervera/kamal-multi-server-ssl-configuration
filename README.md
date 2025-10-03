# Deploying Rails API with Next.js Frontend using Kamal

A comprehensive guide to deploying a full-stack application with Rails API backend and Next.js frontend using Kamal 2.7.0, including PostgreSQL, Redis, and proper networking configuration.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Configuration Files](#configuration-files)
- [Deployment Process](#deployment-process)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

## Overview

This guide demonstrates how to deploy a monorepo containing:
- **Rails API** (Ruby on Rails 8) - Backend API service
- **Next.js Frontend** (React 18) - Frontend web application
- **PostgreSQL** - Primary database
- **Redis** - Caching and job queue
- **Kamal Proxy** - Load balancing and SSL termination

All services are containerized and orchestrated using Kamal 2.7.0, with two subdomains and full SSL via Cloudflare Origin Certificates:
- Frontend at `https://assets.yourdomain.com`
- API at `https://assets-api.yourdomain.com`

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
            DNS: assets.yourdomain.com
                      │
            DNS: assets-api.yourdomain.com
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                   Kamal Proxy (Traefik)                     │
│                Terminating SSL (443/80)                     │
└───────────────┬────────────────────────┬────────────────────┘
                │                        │
        ┌───────▼──────┐           ┌─────▼───────┐
        │  Frontend    │           │    API      │
        │ (Next.js)    │           │ (Rails)     │
        │   :80        │           │   :3000     │
        └──────────────┘           └─────────────┘
                 │                         │
                 └──────────┬──────────────┘
                            │
                     ┌──────▼──────┐   ┌──────────┐
                     │ PostgreSQL  │   │  Redis   │
                     │   :5432     │   │  :6379   │
                     └─────────────┘   └──────────┘
```

### Routing Configuration

- **Frontend**: `https://assets.yourdomain.com/` → Next.js application (root)
- **API**: `https://assets-api.yourdomain.com/*` → Rails API (root)

## Prerequisites

- **Kamal 2.7.0+** installed locally
- **Docker** installed on deployment server
- **SSH access** to deployment server
- **Domain name** with DNS pointing to server
- **Container registry** (DigitalOcean, AWS ECR, etc.)

## Project Structure

```
asset_management_system/
├── apps/
│   ├── api/                    # Rails API
│   │   ├── config/
│   │   │   ├── deploy.yml      # Kamal configuration
│   │   │   └── storage.yml     # Active Storage config
│   │   ├── .kamal/
│   │   │   └── secrets         # Encrypted secrets
│   │   ├── Dockerfile          # API container definition
│   │   └── ...
│   └── web/                    # Next.js Frontend
│       ├── config/
│       │   └── deploy.yml      # Frontend Kamal config
│       ├── .kamal/
│       │   └── secrets         # Frontend secrets
│       ├── Dockerfile          # Frontend container definition
│       └── ...
├── bin/
│   └── setup_kamal            # Automated setup script
└── DEVELOPER_DOCUMENTATION/
    └── kamal-rails-nextjs-deployment-guide.md
```

## Configuration Files

### 1. Rails API Configuration (`apps/api/config/deploy.yml`)

```yaml
# Kamal 2.7.0 Configuration for Rails API
service: api
image: your-registry/your-app-api

# Deploy to these servers
servers:
  web:
    hosts:
      - your-server.com
    cmd: bash -lc "until pg_isready -h api-postgres -p 5432 -U postgres; do echo 'Waiting for database...'; sleep 2; done && bin/rails db:prepare && bin/rails server -b 0.0.0.0 -p 3000"

# API accessible via its own subdomain with Cloudflare SSL
proxy:
  host: assets-api.yourdomain.com
  ssl:
    certificate_pem: CLOUDFLARE_CERTIFICATE_PEM
    private_key_pem: CLOUDFLARE_PRIVATE_KEY_PEM
  app_port: 3000

# Container registry configuration
registry:
  server: registry.digitalocean.com
  username: do_access_token
  password:
    - KAMAL_REGISTRY_TOKEN

# Build configuration
builder:
  arch: amd64

# Environment variables
env:
  secret:
    - RAILS_MASTER_KEY
    - JWT_SECRET_KEY
    - POSTGRES_PASSWORD
  clear:
    RAILS_ENV: production
    RACK_ENV: production
    RAILS_LOG_TO_STDOUT: "1"
    RAILS_SERVE_STATIC_FILES: "true"
    ACTIVE_STORAGE_SERVICE: local
    ACTIVE_STORAGE_HOST: https://assets-api.yourdomain.com
    FRONTEND_URL: https://assets.yourdomain.com
    DATABASE_URL: postgresql://postgres:$POSTGRES_PASSWORD@api-postgres:5432/your_database
    REDIS_URL: redis://api-redis:6379/0

# Database and Redis accessories
accessories:
  postgres:
    image: postgres:16
    host: your-server.com
    env:
      clear:
        POSTGRES_DB: your_database
        POSTGRES_USER: postgres
      secret:
        - POSTGRES_PASSWORD
    directories:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    host: your-server.com
    cmd: redis-server --appendonly yes
    directories:
      - redis_data:/data
```

### 2. Next.js Frontend Configuration (`apps/web/config/deploy.yml`)

```yaml
# Kamal 2.7.0 Configuration for Next.js Frontend
service: frontend
image: your-registry/your-app-web

# Deploy to these servers
servers:
  web:
    hosts:
      - your-server.com
    cmd: node server.js

# Frontend accessible through proxy on its own subdomain with Cloudflare SSL
proxy:
  host: assets.yourdomain.com
  ssl:
    certificate_pem: CLOUDFLARE_CERTIFICATE_PEM
    private_key_pem: CLOUDFLARE_PRIVATE_KEY_PEM
  healthcheck:
    path: /
    interval: 10
    timeout: 10

# Container registry configuration
registry:
  server: registry.digitalocean.com
  username: do_access_token
  password:
    - KAMAL_REGISTRY_PASSWORD

# Build configuration with API URL
builder:
  arch: amd64
  context: .
  dockerfile: Dockerfile
  args:
    NEXT_PUBLIC_API_URL: https://assets-api.yourdomain.com

# Environment variables
env:
  clear:
    NODE_ENV: production
    PORT: 80
    HOSTNAME: 0.0.0.0
    NEXT_PUBLIC_API_URL: https://assets-api.yourdomain.com

# Logging configuration
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

### 3. Rails API Dockerfile (`apps/api/Dockerfile`)

```dockerfile
# Multi-stage build for Rails API
FROM ruby:3.2-slim AS base

# Install system dependencies
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    curl \
    libjemalloc2 \
    libvips \
    postgresql-client && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Gems stage
FROM base AS gems

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    build-essential \
    git \
    libpq-dev \
    libyaml-dev \
    pkg-config && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

WORKDIR /rails

COPY Gemfile Gemfile.lock ./
RUN bundle config set --local deployment 'true' && \
    bundle config set --local without 'development test' && \
    bundle install && \
    rm -rf ~/.bundle/ "/usr/local/bundle"/ruby/*/cache "/usr/local/bundle"/ruby/*/bundler/gems/*/.git && \
    bundle exec bootsnap precompile --gemfile

# Build stage
FROM gems AS build

COPY . .
RUN bundle exec bootsnap precompile app/ lib/

# Production stage
FROM base AS production

RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash

COPY --from=build /rails/vendor/bundle /rails/vendor/bundle
COPY --from=build /rails /rails

RUN mkdir -p /rails/db /rails/log /rails/storage /rails/tmp && \
    mkdir -p /app/public/assets && \
    chown -R rails:rails /rails /app

WORKDIR /rails

RUN bundle config set --local deployment 'true' && \
    bundle config set --local without 'development test'

USER rails

EXPOSE 3000

CMD ["bin/rails", "server", "-b", "0.0.0.0", "-p", "3000"]
```

### 4. Next.js Frontend Dockerfile (`apps/web/Dockerfile`)

```dockerfile
# Multi-stage build for Next.js frontend
FROM node:20-alpine AS build

# Set working directory
WORKDIR /app

# Copy package files
COPY package.json package-lock.json* ./

# Install dependencies
RUN npm ci

# Copy source code
COPY . .

# Define build argument for API URL
ARG NEXT_PUBLIC_API_URL=/api

# Build the application
RUN NEXT_TELEMETRY_DISABLED=1 NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL} npm run build

# Production stage
FROM node:20-alpine AS production

# Set working directory
WORKDIR /app

# Copy built application from build stage
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/public ./public
COPY --from=build /app/.next/static ./.next/static

# Create app user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Set ownership
RUN chown -R nextjs:nodejs /app
USER nextjs

# Expose port
EXPOSE 3000

# Set environment
ENV NODE_ENV=production
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

# Start the application
CMD ["node", "server.js"]
```

### 5. Secrets Configuration (`apps/api/.kamal/secrets`)

```bash
# Kamal secrets file
# Generated on $(date)

RAILS_MASTER_KEY=your_rails_master_key_here
JWT_SECRET_KEY=your_jwt_secret_key_here
POSTGRES_PASSWORD=your_secure_postgres_password
KAMAL_REGISTRY_TOKEN=your_registry_token_here

# Database and Redis URLs
DATABASE_URL=postgresql://postgres:$POSTGRES_PASSWORD@api-postgres:5432/your_database
REDIS_URL=redis://api-redis:6379/0

# Local storage configuration
ACTIVE_STORAGE_SERVICE=local

# Cloudflare SSL certificates (quoted multi-line PEM)
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

### 6. Frontend Secrets (`apps/web/.kamal/secrets`)

```bash
# Kamal secrets file for web frontend
KAMAL_REGISTRY_PASSWORD=your_registry_token_here

# Cloudflare SSL certificates (quoted multi-line PEM)
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

## Deployment Process

### 1. Initial Setup

```bash
# Install Kamal
gem install kamal

# Generate Rails master key (if not exists)
cd apps/api
rails credentials:edit

# Generate JWT secret
openssl rand -hex 32
```

### 2. Configure DNS

Create DNS A records pointing to your server:
- `yourdomain.com` → `YOUR_SERVER_IP`

### 3. Deploy with setup script (recommended)

```bash
# From repo root: generates secrets (from ssl-cert.pem/ssl-key.pem),
# validates, then runs kamal setup + deploy for API and web
./bin/setup_kamal --dev -p
```

### 4. Manual deploy (alternative)

```bash
cd apps/api

# Setup and deploy API (creates database and Redis)
kamal setup -c config/deploy.yml
kamal deploy -c config/deploy.yml
```

### 5. Deploy Frontend

```bash
cd apps/web

# Setup and deploy frontend
kamal setup -c config/deploy.yml
kamal deploy -c config/deploy.yml
```

### 6. Verify Deployment

```bash
# Check API health
curl -I http://yourdomain.com/api

# Check frontend
curl -I http://yourdomain.com

# Check containers
kamal app details -c config/deploy.yml
```

## Troubleshooting

### Common Issues

#### 1. Build Arguments Not Working

**Problem**: Next.js environment variables not being set correctly.

**Solution**: Ensure Dockerfile properly defines build arguments:

```dockerfile
# Define build argument
ARG NEXT_PUBLIC_API_URL=/api

# Use in build process
RUN NEXT_TELEMETRY_DISABLED=1 NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL} npm run build
```

#### 2. Database Connection Issues

**Problem**: Rails can't connect to PostgreSQL.

**Solution**: Ensure database URL uses correct container name:

```yaml
DATABASE_URL: postgresql://postgres:$POSTGRES_PASSWORD@api-postgres:5432/your_database
```

#### 3. Proxy/SSL Configuration

**Problem**: `Secret 'CLOUDFLARE_CERTIFICATE_PEM' not found` or `unable to load certificate`.

**Solution**:
- Ensure `apps/api/.kamal/secrets` (and web) contain quoted multi-line `CLOUDFLARE_*` entries.
- Don’t store certs in `.env.setup_kamal` with `\n` escapes. Use `ssl-cert.pem`/`ssl-key.pem` and the setup script.
- Confirm `deploy.yml` references:
  - `certificate_pem: CLOUDFLARE_CERTIFICATE_PEM`
  - `private_key_pem: CLOUDFLARE_PRIVATE_KEY_PEM`

#### 4. Health Check Failures

**Problem**: Containers failing health checks.

**Solution**: Ensure proper health check configuration:

```yaml
proxy:
  healthcheck:
    path: /
    interval: 10
    timeout: 10
```

### Debugging Commands

```bash
# View application logs
kamal app logs -c config/deploy.yml

# Check container status
kamal app details -c config/deploy.yml

# SSH into server
kamal app exec -i -c config/deploy.yml

# Check proxy configuration
kamal proxy logs

# Remove deployment
kamal remove -c config/deploy.yml
```

## Best Practices

### 1. Security

- **Never commit secrets**: Use `.kamal/secrets` files
- **Use strong passwords**: Generate secure passwords for database
- **Enable SSL**: Configure Let's Encrypt for production
- **Network isolation**: Use Docker networks for service communication

### 2. Performance

- **Multi-stage builds**: Optimize Docker images
- **Resource limits**: Set appropriate memory/CPU limits
- **Health checks**: Implement proper health monitoring
- **Logging**: Configure log rotation and retention

### 3. Monitoring

- **Application logs**: Monitor Rails and Next.js logs
- **Database monitoring**: Track PostgreSQL performance
- **Redis monitoring**: Monitor cache hit rates
- **Proxy logs**: Monitor request patterns

### 4. Maintenance

- **Regular updates**: Keep dependencies updated
- **Backup strategy**: Implement database backups
- **Rollback plan**: Test deployment rollback procedures
- **Documentation**: Keep deployment docs current

## Sample Deployment Script

Here's a sample automated deployment script:

```bash
#!/bin/bash
set -e

echo "🚀 Starting deployment process..."

# Deploy API first
echo "📦 Deploying Rails API..."
cd apps/api
kamal setup -c config/deploy.yml
kamal deploy -c config/deploy.yml
cd ../..

# Deploy frontend
echo "🌐 Deploying Next.js Frontend..."
cd apps/web
kamal setup -c config/deploy.yml
kamal deploy -c config/deploy.yml
cd ../..

echo "✅ Deployment completed successfully!"
echo "🌐 Frontend: http://yourdomain.com"
echo "🔧 API: http://yourdomain.com/api"
```

## Conclusion

This guide provides a complete solution for deploying Rails API with Next.js frontend using Kamal. The key components include:

- **Path-based routing** for single domain deployment
- **Proper build argument handling** for Next.js environment variables
- **Database and Redis accessories** for data persistence
- **Health checks and monitoring** for production reliability
- **Security best practices** for secrets management

For more information, refer to the [Kamal documentation](https://kamal-deploy.org/) and [Next.js deployment guide](https://nextjs.org/docs/deployment).
