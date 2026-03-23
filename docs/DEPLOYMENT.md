# AtlasClaw Enterprise Deployment Guide

> **For Enterprise Customers**: This guide provides comprehensive instructions for deploying AtlasClaw in production environments with high availability and enterprise-grade configuration.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [Docker Deployment](#docker-deployment)
5. [Enterprise Production Deployment](#enterprise-production-deployment)
6. [Database Setup](#database-setup)
7. [High Availability](#high-availability)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### System Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 4 cores | 8+ cores |
| RAM | 8 GB | 16+ GB |
| Disk | 50 GB | 200+ GB SSD |
| Python | 3.11+ | 3.11+ |
| MySQL | 8.5 LTS | 8.5 LTS |

### Required Software

- Python 3.11 or higher
- Docker 20.10+ and Docker Compose 2.0+
- MySQL 8.5 LTS (for enterprise deployments)
- Git

### Network Requirements

- Outbound HTTPS (443) access to LLM provider APIs
- Internal network access to MySQL database
- (Optional) SMTP for notification features

---

## Installation

### 1. Clone the Repository

```bash
git clone <repository-url>
cd AtlasClaw-Core
```

### 2. Create Deployment Directory Structure

```bash
mkdir -p /opt/atlasclaw/{config,data,logs}
cd /opt/atlasclaw
```

### 3. Copy Docker Files

```bash
cp -r AtlasClaw-Core/docker/* /opt/atlasclaw/
```

---

## Configuration

All configuration is managed through a single JSON file: `atlasclaw.json`. No environment variables are required.

### 1. Create Configuration File

Copy the example configuration:

```bash
cp AtlasClaw-Core/atlasclaw.json.example /opt/atlasclaw/config/atlasclaw.json
```

### 2. Configure Database (Enterprise MySQL 8.5)

Edit `atlasclaw.json` to use MySQL:

```json
{
  "database": {
    "type": "mysql",
    "mysql": {
      "host": "mysql",
      "port": 3306,
      "database": "atlasclaw",
      "user": "atlasclaw",
      "password": "your-secure-password-here",
      "charset": "utf8mb4"
    },
    "pool_size": 20,
    "max_overflow": 30,
    "echo": false
  }
}
```

### 3. Configure LLM Provider

```json
{
  "model": {
    "primary": "deepseek-main",
    "fallbacks": [],
    "temperature": 0.2,
    "selection_strategy": "health",
    "tokens": [
      {
        "id": "deepseek-main",
        "provider": "deepseek",
        "model": "deepseek-chat",
        "base_url": "https://api.deepseek.com",
        "api_key": "your-api-key-here",
        "api_type": "openai",
        "priority": 100,
        "weight": 100
      }
    ]
  }
}
```

### 4. Configure Enterprise Service Providers

```json
{
  "service_providers": {
    "jira": {
      "production": {
        "base_url": "https://jira.company.com/",
        "username": "atlasclaw-service-account",
        "password": "your-jira-api-token",
        "api_version": "2",
        "default_project": "PROJ"
      }
    },
    "servicenow": {
      "production": {
        "instance": "your-instance",
        "username": "atlasclaw-user",
        "password": "your-password"
      }
    }
  }
}
```

### 5. Configure Authentication

**OIDC/OAuth2 (Recommended for Enterprise)**:

```json
{
  "auth": {
    "provider": "oidc",
    "cache_ttl_seconds": 300,
    "oidc": {
      "issuer": "https://auth.your-company.com",
      "client_id": "atlasclaw-client-id",
      "client_secret": "your-client-secret",
      "jwks_uri": "https://auth.your-company.com/.well-known/jwks.json",
      "scopes": ["openid", "profile", "email"],
      "authorization_endpoint": "https://auth.your-company.com/oauth2/authorize",
      "token_endpoint": "https://auth.your-company.com/oauth2/token",
      "userinfo_endpoint": "https://auth.your-company.com/oauth2/userinfo",
      "redirect_uri": "https://atlasclaw.your-company.com/api/auth/callback",
      "pkce_enabled": true,
      "pkce_method": "S256"
    }
  }
}
```

**API Key Authentication**:

```json
{
  "auth": {
    "provider": "api_key",
    "api_key": {
      "keys": {
        "sk-production-key-001": {
          "user_id": "service-account",
          "roles": ["admin"]
        }
      }
    }
  }
}
```

### 6. Configure Webhook (Optional)

```json
{
  "webhook": {
    "enabled": true,
    "header_name": "X-AtlasClaw-SK",
    "systems": [
      {
        "system_id": "external-system",
        "enabled": true,
        "sk_env": "your-webhook-secret-key",
        "default_agent_id": "main",
        "allowed_skills": ["skill:allowed-action"]
      }
    ]
  }
}
```

---

## Docker Deployment

### Enterprise Docker Compose (MySQL 8.5)

Create `docker-compose.enterprise.yml`:

```yaml
version: '3.8'

services:
  atlasclaw:
    image: atlasclaw-core:latest
    container_name: atlasclaw-app
    ports:
      - "8000:8000"
    volumes:
      - /opt/atlasclaw/config/atlasclaw.json:/app/atlasclaw.json:ro
      - atlasclaw-data:/app/data
      - /opt/atlasclaw/logs:/app/logs
    depends_on:
      mysql:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G

  mysql:
    image: mysql:8.5
    container_name: atlasclaw-mysql
    environment:
      MYSQL_ROOT_PASSWORD: your-root-password-here
      MYSQL_DATABASE: atlasclaw
      MYSQL_USER: atlasclaw
      MYSQL_PASSWORD: your-secure-password-here
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - /opt/atlasclaw/mysql-init:/docker-entrypoint-initdb.d:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-pyour-root-password-here"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 60s
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-authentication-plugin=mysql_native_password
      - --innodb-buffer-pool-size=2G
      - --innodb-log-file-size=512M
      - --max-connections=200
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 4G
        reservations:
          cpus: '2'
          memory: 2G

  nginx:
    image: nginx:alpine
    container_name: atlasclaw-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /opt/atlasclaw/config/nginx.conf:/etc/nginx/nginx.conf:ro
      - /opt/atlasclaw/ssl:/etc/nginx/ssl:ro
    depends_on:
      - atlasclaw
    restart: unless-stopped

volumes:
  mysql-data:
    driver: local
  atlasclaw-data:
    driver: local
```

### Deploy

```bash
cd /opt/atlasclaw
docker-compose -f docker-compose.enterprise.yml up -d
```

### Verify Deployment

```bash
# Check container status
docker-compose -f docker-compose.enterprise.yml ps

# Health check
curl http://localhost:8000/api/health

# Expected response:
# {"status": "healthy", "timestamp": "2026-03-23T10:00:00+00:00"}
```

---

## Enterprise Production Deployment

### 1. Nginx Configuration with SSL

Create `/opt/atlasclaw/config/nginx.conf`:

```nginx
upstream atlasclaw {
    server atlasclaw:8000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name atlasclaw.your-company.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name atlasclaw.your-company.com;

    ssl_certificate /etc/nginx/ssl/atlasclaw.crt;
    ssl_certificate_key /etc/nginx/ssl/atlasclaw.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    client_max_body_size 50M;

    location / {
        proxy_pass http://atlasclaw;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # SSE endpoint - disable buffering for streaming
    location /api/agent/runs/ {
        proxy_pass http://atlasclaw;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_buffering off;
        proxy_cache off;
        proxy_connect_timeout 600s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s;
    }
}
```

### 2. SSL Certificate Setup

```bash
# Place your SSL certificates
mkdir -p /opt/atlasclaw/ssl
cp your-certificate.crt /opt/atlasclaw/ssl/atlasclaw.crt
cp your-private-key.key /opt/atlasclaw/ssl/atlasclaw.key
chmod 600 /opt/atlasclaw/ssl/atlasclaw.key
```

### 3. Run Database Migrations

On first deployment or after updates:

```bash
docker-compose -f docker-compose.enterprise.yml exec atlasclaw \
  alembic upgrade head
```

---

## Database Setup

### MySQL 8.5 Configuration

Create `/opt/atlasclaw/mysql-init/01-init.sql`:

```sql
-- Create dedicated database user with appropriate privileges
CREATE USER IF NOT EXISTS 'atlasclaw'@'%' IDENTIFIED WITH mysql_native_password BY 'your-secure-password-here';
GRANT ALL PRIVILEGES ON atlasclaw.* TO 'atlasclaw'@'%';
FLUSH PRIVILEGES;
```

### Backup Strategy

Create `/opt/atlasclaw/backup.sh`:

```bash
#!/bin/bash
# AtlasClaw Backup Script

BACKUP_DIR="/opt/atlasclaw/backups"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

# Database backup
docker exec atlasclaw-mysql mysqldump -u root -p'your-root-password-here' atlasclaw | gzip > $BACKUP_DIR/atlasclaw_db_$DATE.sql.gz

# Configuration backup
tar -czf $BACKUP_DIR/atlasclaw_config_$DATE.tar.gz -C /opt/atlasclaw config/

# Data backup
tar -czf $BACKUP_DIR/atlasclaw_data_$DATE.tar.gz -C /opt/atlasclaw data/

# Cleanup old backups (keep 30 days)
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete

echo "Backup completed: $DATE"
```

Make executable and schedule:

```bash
chmod +x /opt/atlasclaw/backup.sh
# Add to crontab for daily backup at 2 AM
echo "0 2 * * * /opt/atlasclaw/backup.sh >> /opt/atlasclaw/logs/backup.log 2>&1" | crontab -
```

---

## High Availability

### Multi-Instance Deployment

For high availability, deploy multiple AtlasClaw instances behind a load balancer:

```yaml
version: '3.8'

services:
  atlasclaw-1:
    image: atlasclaw-core:latest
    environment:
      - INSTANCE_ID=atlasclaw-1
    volumes:
      - /opt/atlasclaw/config/atlasclaw.json:/app/atlasclaw.json:ro
      - shared-data:/app/data
    depends_on:
      - mysql
      - redis

  atlasclaw-2:
    image: atlasclaw-core:latest
    environment:
      - INSTANCE_ID=atlasclaw-2
    volumes:
      - /opt/atlasclaw/config/atlasclaw.json:/app/atlasclaw.json:ro
      - shared-data:/app/data
    depends_on:
      - mysql
      - redis

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    restart: unless-stopped

  haproxy:
    image: haproxy:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /opt/atlasclaw/config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
      - atlasclaw-1
      - atlasclaw-2

volumes:
  shared-data:
  redis-data:
```

---

## Troubleshooting

### Check Service Logs

```bash
# AtlasClaw logs
docker-compose -f docker-compose.enterprise.yml logs -f atlasclaw

# MySQL logs
docker-compose -f docker-compose.enterprise.yml logs -f mysql

# Nginx logs
docker-compose -f docker-compose.enterprise.yml logs -f nginx
```

### Database Connection Issues

```bash
# Test MySQL connectivity from AtlasClaw container
docker-compose -f docker-compose.enterprise.yml exec atlasclaw \
  python -c "import asyncio; from app.atlasclaw.db.database import init_db; asyncio.run(init_db())"
```

### Migration Failures

```bash
# Check migration status
docker-compose -f docker-compose.enterprise.yml exec atlasclaw alembic current

# Rollback to previous version if needed
docker-compose -f docker-compose.enterprise.yml exec atlasclaw alembic downgrade -1

# Re-run migrations
docker-compose -f docker-compose.enterprise.yml exec atlasclaw alembic upgrade head
```

### Health Check Failures

```bash
# Manual health check
curl -v http://localhost:8000/api/health

# Check if all services are running
docker-compose -f docker-compose.enterprise.yml ps

# Restart services
docker-compose -f docker-compose.enterprise.yml restart
```

### Performance Issues

Check resource utilization:

```bash
# Container stats
docker stats atlasclaw-app atlasclaw-mysql

# MySQL slow query log
docker-compose -f docker-compose.enterprise.yml exec mysql \
  mysql -u root -p -e "SHOW VARIABLES LIKE 'slow_query_log';"
```

---

## Support

For enterprise support, contact your AtlasClaw support representative or refer to the [Development Specification](./DEVELOPMENT-SPEC.MD) for detailed technical documentation.

---

## Security Best Practices

1. **Never commit secrets** to version control
2. **Store all configuration** in `atlasclaw.json` with proper file permissions (600)
3. **Enable HTTPS** in production with valid SSL certificates
4. **Use strong passwords** for database and service accounts
5. **Set up firewall rules** to restrict access to MySQL port
6. **Regularly update** AtlasClaw and MySQL images
7. **Enable audit logging** for compliance requirements
8. **Backup regularly** and test restore procedures

---

## Next Steps

- [Configure Additional Skills](../openspec/AGENTS.md)
- [Set up Channel Integrations](./CHANNEL-GUIDE.MD)
- [Customize Provider Plugins](./PROVIDER-GUIDE.MD)
