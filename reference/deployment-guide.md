# MessageForge Deployment Guide

**Last Updated:** 2026-03-29

This guide covers deploying MessageForge Backend to production on a Hetzner VPS with PostgreSQL and Caddy reverse proxy with automatic TLS.

**Architecture:** Go middleware server (port 8080) + admin/metrics server (port 9090) + PostgreSQL 17 + Caddy (TLS termination).

## Table of Contents

1. [Infrastructure Setup](#infrastructure-setup)
2. [Server Provisioning](#server-provisioning)
3. [PostgreSQL Configuration](#postgresql-configuration)
4. [Go Application Setup](#go-application-setup)
5. [Caddy Reverse Proxy](#caddy-reverse-proxy)
6. [Environment Configuration](#environment-configuration)
7. [First-Run Checklist](#first-run-checklist)
8. [Monitoring & Maintenance](#monitoring--maintenance)

## Infrastructure Setup

### Hetzner VPS Selection

Choose based on message volume and channel count:

| Tier | VM Type | vCPU | RAM | Storage | Monthly Cost | Use Case |
|------|---------|------|-----|---------|--------------|----------|
| MVP | CAX11 | 2 | 4GB | 40GB NVMe | €4.50 | <10 channels, <1K msg/day |
| Growth | CAX21 | 4 | 8GB | 80GB NVMe | €8 | 10-100 channels, <10K msg/day |
| Scale | CAX31 | 8 | 16GB | 160GB NVMe | €16 | 100+ channels, 10K+ msg/day |

**Architecture:** ARM Ampere (CAX series preferred for cost efficiency)

**Storage:** NVMe SSD (all Hetzner Cloud offerings include this)

**Region:** Choose geographically closest to your user base

### Firewall Rules

Configure Hetzner Cloud Firewall:

| Protocol | Port | Source | Purpose |
|----------|------|--------|---------|
| TCP | 22 | Your IP / Team IPs | SSH access |
| TCP | 80 | 0.0.0.0/0 | HTTP (Caddy auto-redirect) |
| TCP | 443 | 0.0.0.0/0 | HTTPS (Caddy TLS) |
| TCP | 8080 | localhost only | Go backend (internal) |
| TCP | 9090 | localhost only | Admin/metrics server (internal) |

## Server Provisioning

### 1. Create and Connect to VPS

```bash
# On your local machine, note the IPv4 address from Hetzner console
# SSH into the server
ssh root@<HETZNER_IP>

# Update system packages
apt update && apt upgrade -y
apt install -y wget curl git build-essential

# Configure hostname
hostnamectl set-hostname messageforge
echo "messageforge" > /etc/hostname
```

### 2. Create Application User

```bash
# Create non-root user for running services
useradd -m -s /bin/bash messageforge
usermod -aG sudo messageforge

# Configure passwordless sudo for messageforge (optional but recommended for deployment automation)
echo "messageforge ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/messageforge
```

### 3. Set Up SSH Keys (Optional but Recommended)

```bash
# On your local machine
ssh-copy-id -i ~/.ssh/id_rsa messageforge@<HETZNER_IP>

# Test passwordless SSH
ssh messageforge@<HETZNER_IP> "echo 'SSH works!'"
```

## PostgreSQL Configuration

### Option A: Self-Hosted on VPS (Recommended for MVP/Growth)

#### 1. Install PostgreSQL 17

```bash
# Add PostgreSQL repository
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Install PostgreSQL 17
sudo apt update
sudo apt install -y postgresql-17 postgresql-contrib-17

# Start and enable PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

#### 2. Create Application Database and User

```bash
# Connect as postgres superuser
sudo -u postgres psql

-- Inside psql:
CREATE DATABASE messenger;
CREATE USER messenger WITH PASSWORD 'your-secure-password-here';
ALTER ROLE messenger SET client_encoding TO 'utf8';
ALTER ROLE messenger SET default_transaction_isolation TO 'read committed';
ALTER ROLE messenger SET default_transaction_deferrable TO off;
ALTER ROLE messenger SET default_transaction_read_only TO off;
ALTER ROLE messenger SET statement_timeout TO 0;
GRANT ALL PRIVILEGES ON DATABASE messenger TO messenger;
\q
```

#### 3. Configure PostgreSQL for Production

Edit `/etc/postgresql/17/main/postgresql.conf`:

```ini
# Connection settings
max_connections = 200
superuser_reserved_connections = 3

# Memory
shared_buffers = 256MB                    # 25% of RAM for CAX11, scale up with larger VMs
effective_cache_size = 1GB                # 25% of RAM
work_mem = 4MB                            # shared_buffers / max_connections
maintenance_work_mem = 64MB

# WAL (Write-Ahead Logging)
wal_level = replica                       # Required for replication/backups
max_wal_senders = 3
wal_keep_size = 1GB

# Logging
log_statement = 'all'                     # Change to 'mod' or 'ddl' for high-traffic
log_duration = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on

# Performance
random_page_cost = 1.1                    # Lower for SSD
synchronous_commit = on

# Autovacuum
autovacuum = on
autovacuum_naptime = 10s
```

Restart PostgreSQL:

```bash
sudo systemctl restart postgresql
```

#### 4. Set Up Automated Backups

```bash
# Create backup directory
sudo mkdir -p /backups/postgres
sudo chown postgres:postgres /backups/postgres
sudo chmod 700 /backups/postgres

# Create backup script
sudo tee /usr/local/bin/backup-postgres.sh > /dev/null <<'EOF'
#!/bin/bash
BACKUP_DIR="/backups/postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="messenger"

sudo -u postgres pg_dump "$DB_NAME" | gzip > "$BACKUP_DIR/messenger_$TIMESTAMP.sql.gz"

# Keep last 7 days of backups
find "$BACKUP_DIR" -name "messenger_*.sql.gz" -mtime +7 -delete
EOF

sudo chmod +x /usr/local/bin/backup-postgres.sh

# Add to crontab to run daily at 2 AM
echo "0 2 * * * /usr/local/bin/backup-postgres.sh" | sudo tee /etc/cron.d/postgres-backup
```

### Option B: Hetzner Managed PostgreSQL (Recommended for Scale)

For production with guaranteed uptime SLA:

1. Create managed database cluster in Hetzner console
2. Note connection string: `postgresql://user:pass@hostname:5432/messenger`
3. Follow "Create Application Database" step above with the provided credentials
4. Backups are automatic (7-day retention standard)

## Go Application Setup

### 1. Install Go Runtime

```bash
# Install Go 1.23+ (check latest stable version)
cd /tmp
wget https://go.dev/dl/go1.23.3.linux-arm64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.23.3.linux-arm64.tar.gz

# Add Go to PATH
echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee -a /etc/profile.d/go.sh
source /etc/profile.d/go.sh

# Verify installation
go version
```

### 2. Clone Repository and Build

```bash
# Clone the repository
cd /home/messageforge
sudo -u messageforge git clone https://github.com/your-org/messageforge.git
cd messageforge/MessageForge.Backend

# Build the application
sudo -u messageforge go build -o /home/messageforge/messageforge-backend ./cmd/messenger
```

### 3. Create systemd Service File

Create `/etc/systemd/system/messageforge-backend.service`:

```ini
[Unit]
Description=MessageForge Backend Service
Documentation=https://github.com/your-org/messageforge
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=messageforge
WorkingDirectory=/home/messageforge
ExecStart=/home/messageforge/messageforge-backend

# Environment variables loaded from .env file
EnvironmentFile=/home/messageforge/.env

# Process management
Restart=on-failure
RestartSec=5s
KillMode=process

# Security and logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=messageforge-backend
NoNewPrivileges=true

# Resource limits
LimitNOFILE=65536
LimitNPROC=512

[Install]
WantedBy=multi-user.target
```

Enable and test the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable messageforge-backend
sudo systemctl start messageforge-backend
sudo systemctl status messageforge-backend

# View logs
sudo journalctl -u messageforge-backend -f
```

### 4. Set Up Automatic Deployments (Optional)

Create `/home/messageforge/deploy.sh`:

```bash
#!/bin/bash
set -e

cd /home/messageforge/messageforge
git pull origin main

cd MessageForge.Backend
go build -o /home/messageforge/messageforge-backend ./cmd/messenger

sudo systemctl restart messageforge-backend
echo "Deployment complete"
```

Make it executable and add to crontab or use a webhook trigger.

## Caddy Reverse Proxy

Caddy automatically manages TLS certificates via Let's Encrypt and handles reverse proxying to the Go backend.

### 1. Install Caddy

```bash
# Using Debian package (simplest method)
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl https://dl.caddy.community/linux/caddy_linux_arm64.deb -o /tmp/caddy.deb
sudo dpkg -i /tmp/caddy.deb

# Verify installation
caddy version
```

### 2. Create Caddyfile Configuration

Create `/etc/caddy/Caddyfile`:

```
yourdomain.com {
  # API endpoints
  @api_paths {
    path /api/*
    path /webhooks/*
  }
  handle @api_paths {
    reverse_proxy localhost:8080 {
      header_up X-Forwarded-For {remote_host}
      header_up X-Forwarded-Proto {scheme}
      header_up X-Forwarded-Host {host}
    }
  }

  # Health check endpoint (public)
  @health {
    path /health
  }
  handle @health {
    reverse_proxy localhost:8080
  }

  # TLS settings
  tls {
    protocols tls1.2 tls1.3
  }

  # Gzip compression
  encode gzip

  # Security headers
  header {
    X-Content-Type-Options "nosniff"
    X-Frame-Options "SAMEORIGIN"
    X-XSS-Protection "1; mode=block"
    Referrer-Policy "strict-origin-when-cross-origin"
    Permissions-Policy "geolocation=()"
  }

  # HSTS (only enable after confirming HTTPS works)
  header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

  # Logging
  log {
    level info
    format console
  }
}
```

**Change `yourdomain.com` to your actual domain.**

### 3. Enable Caddy Service

```bash
# Verify Caddyfile syntax
sudo caddy validate --config /etc/caddy/Caddyfile

# Reload Caddy (picks up new config)
sudo systemctl reload caddy
sudo systemctl enable caddy

# Check status
sudo systemctl status caddy
sudo journalctl -u caddy -f
```

### 4. DNS Configuration

Point your domain to the Hetzner VPS IP in your DNS provider:

```
yourdomain.com    A  <HETZNER_IP>
www.yourdomain.com CNAME yourdomain.com
```

Wait for DNS propagation (typically 5-30 minutes), then Caddy will automatically obtain a Let's Encrypt certificate.

Verify:

```bash
curl https://yourdomain.com/health
```

## Environment Configuration

### Create `.env` File

Create `/home/messageforge/.env` with all required variables:

```bash
# Server
HTTP_HOST=0.0.0.0
HTTP_PORT=8080
ADMIN_HOST=127.0.0.1
ADMIN_PORT=9090

# PostgreSQL
DATABASE_URL=postgresql://messenger:your-password@localhost:5432/messenger

# Salesforce (not required for MVP, set dummy values)
SF_INSTANCE_URL=https://your-instance.salesforce.com
SF_CLIENT_ID=your-client-id
SF_USERNAME=your-username
SF_PRIVATE_KEY=your-private-key

# Telegram
TELEGRAM_API_ID=your-api-id
TELEGRAM_API_HASH=your-api-hash
TELEGRAM_BOT_TOKEN=your-bot-token

# Webhook Authentication
WEBHOOK_SECRET=your-hmac-secret-for-salesforce-webhooks

# Encryption
ENCRYPTION_KEY=your-32-byte-aes-256-key-in-hex
```

**Security notes:**
- Restrict file permissions: `sudo chmod 600 /home/messageforge/.env`
- Use strong random values for all secrets
- Rotate secrets regularly
- Never commit `.env` to git (use `.env.example` instead)

### Create `.env.example`

Create `/home/messageforge/messageforge/MessageForge.Backend/.env.example`:

```bash
# Server Configuration
HTTP_HOST=0.0.0.0
HTTP_PORT=8080
ADMIN_HOST=127.0.0.1
ADMIN_PORT=9090

# PostgreSQL Connection
# Format: postgresql://user:password@host:port/database
DATABASE_URL=postgresql://messenger:password@localhost:5432/messenger

# Salesforce Configuration (OAuth 2.0 JWT Bearer Flow)
SF_INSTANCE_URL=https://your-instance.salesforce.com
SF_CLIENT_ID=your-salesforce-connected-app-client-id
SF_USERNAME=your-integration-user@salesforce.com
SF_PRIVATE_KEY=-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----

# Telegram Configuration (obtain from https://my.telegram.org)
TELEGRAM_API_ID=your-api-id
TELEGRAM_API_HASH=your-api-hash

# Telegram Bot Token (obtain from @BotFather)
TELEGRAM_BOT_TOKEN=your-bot-token-from-botfather

# Webhook Authentication (for Salesforce platform event triggers)
WEBHOOK_SECRET=your-hmac-secret-for-webhooks

# Encryption (AES-256 key in hex format, generate with: openssl rand -hex 32)
ENCRYPTION_KEY=your-32-byte-hex-encoded-aes256-key
```

## First-Run Checklist

Follow this checklist to bring up a complete production environment from scratch:

### Pre-Deployment (Local Machine)

- [ ] Domain registered and DNS provider access available
- [ ] Hetzner account created
- [ ] Telegram API credentials obtained
- [ ] Salesforce OAuth credentials prepared
- [ ] SSH key pair generated (`ssh-keygen -t ed25519`)
- [ ] All secrets generated and stored in secure password manager

### Server Setup

- [ ] VPS created (CAX11 or larger)
- [ ] Firewall rules configured (22, 80, 443, 8080, 9090)
- [ ] System packages updated (`apt update && apt upgrade`)
- [ ] Non-root user created (`messageforge`)
- [ ] SSH keys installed for passwordless access

### Database Setup

- [ ] PostgreSQL 17 installed
- [ ] `messenger` database and user created
- [ ] PostgreSQL config optimized (`/etc/postgresql/17/main/postgresql.conf`)
- [ ] Automated backup script created and tested
- [ ] Backup cron job enabled

### Application Deployment

- [ ] Go runtime installed (1.23+)
- [ ] Repository cloned to `/home/messageforge/messageforge`
- [ ] Application built: `go build -o /home/messageforge/messageforge-backend ./cmd/messenger`
- [ ] `.env` file created with all secrets
- [ ] Permissions set: `chmod 600 /home/messageforge/.env`
- [ ] systemd service created and enabled
- [ ] Service started and tested: `systemctl status messageforge-backend`

### Reverse Proxy & TLS

- [ ] Caddy installed
- [ ] Caddyfile created with correct domain
- [ ] Caddyfile syntax validated
- [ ] DNS records pointing to VPS IP
- [ ] Caddy service enabled and started
- [ ] HTTPS working: `curl https://yourdomain.com/health`
- [ ] Let's Encrypt certificate auto-renewed (check in 3 months)

### Final Verification

- [ ] Go backend responding: `curl https://yourdomain.com/health`
- [ ] PostgreSQL connected and accepting queries
- [ ] Logs clean (no errors): `journalctl -u messageforge-backend -f`
- [ ] Load test passed (simulate expected message volume)
- [ ] Backup tested (restore to test database)

### Post-Deployment

- [ ] Monitoring configured (see section below)
- [ ] Alert rules set up
- [ ] Runbook created for on-call engineers
- [ ] Documentation updated with actual domain/IPs
- [ ] Team notified of deployment
- [ ] Status page updated

## Monitoring & Maintenance

### Health Checks

Add external uptime monitoring (e.g., Uptime.com, Better Stack):

```bash
# Health endpoint for backend
GET https://yourdomain.com/health

# Response (healthy):
# 200 OK with JSON body
```

### Log Aggregation

View service logs:

```bash
# MessageForge backend
sudo journalctl -u messageforge-backend -f

# Caddy
sudo journalctl -u caddy -f

# PostgreSQL
sudo tail -f /var/log/postgresql/postgresql-17-main.log
```

Configure centralized logging (optional):

```bash
# Install rsyslog for remote log forwarding
sudo apt install rsyslog

# Configure to forward to your log aggregator (e.g., Datadog, Sentry)
# Edit /etc/rsyslog.conf and add:
# *.* @@logserver.example.com:514
```

### Regular Maintenance

**Weekly:**
- Review error logs for anomalies
- Check disk usage: `df -h`
- Verify all systemd services running: `systemctl status`

**Monthly:**
- Update system packages: `sudo apt update && sudo apt upgrade`
- Test backup restoration on test database
- Review PostgreSQL slow query logs

**Quarterly:**
- Update Go runtime to latest stable version
- Review and rotate all secrets
- Conduct security audit

### Performance Tuning

If experiencing issues:

```bash
# Check system resources
top -b -n 1 | head -20
free -h
iostat -x 1 5

# Check database connections
sudo -u postgres psql -d messenger -c "SELECT datname, count(*) FROM pg_stat_activity GROUP BY datname;"

# Check database size
sudo -u postgres psql -d messenger -c "SELECT pg_size_pretty(pg_database_size('messenger'));"

# Analyze slow queries
sudo -u postgres psql -d messenger -c "SELECT query, calls, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"
```

### Scaling Considerations

When message volume grows:

1. **Upgrade VPS** to next tier (CAX21, then CAX31)
2. **Increase PostgreSQL resources:**
   - `shared_buffers` (25-40% of RAM)
   - `work_mem` (RAM / max_connections)
3. **Consider managed PostgreSQL** for automated failover
4. **Add load balancing** (Hetzner Load Balancer) for multiple backend instances
5. **Enable Redis** for distributed rate limiting and session caching

---

## Additional Resources

- [Hetzner Cloud Documentation](https://docs.hetzner.cloud/)
- [PostgreSQL 17 Documentation](https://www.postgresql.org/docs/current/)
- [Caddy Documentation](https://caddyserver.com/docs/)

## Support

For deployment assistance:
- Check logs with `journalctl` and grep for errors
- Review configuration files for typos
- Verify environment variables are loaded: `ps aux | grep messageforge-backend`
- Test network connectivity: `curl -v https://yourdomain.com/health`
