# Deployment Guide

This guide provides step-by-step instructions for deploying the Laravel application to production using Docker and CI/CD pipeline.

## 🎯 Deployment Overview

The deployment process involves several stages:
1. **Local Development Setup**
2. **Production Server Preparation**
3. **CI/CD Pipeline Configuration**
4. **Initial Deployment**
5. **Post-deployment Verification**
6. **Monitoring and Maintenance**

## 🖥️ Production Server Requirements

### Minimum System Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **CPU** | 2 cores | 4+ cores |
| **RAM** | 4GB | 8GB+ |
| **Storage** | 20GB SSD | 50GB+ SSD |
| **OS** | Ubuntu 20.04+ | Ubuntu 22.04 LTS |
| **Network** | 100Mbps | 1Gbps |

### Required Software

```bash
# Docker Engine (v20.10+)
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
sudo usermod -aG docker $USER

# Docker Compose (v2.0+)
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Git
sudo apt update
sudo apt install git -y

# Nginx (for reverse proxy)
sudo apt install nginx -y

# Fail2ban (security)
sudo apt install fail2ban -y
```

## 🔧 Server Preparation

### 1. User Setup

```bash
# Create deployment user
sudo adduser deploy
sudo usermod -aG docker deploy
sudo usermod -aG www-data deploy

# Setup SSH key authentication
sudo mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
echo "your-ssh-public-key" | sudo tee /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh

# Configure sudo access
echo "deploy ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/deploy
```

### 2. Directory Structure

```bash
# Switch to deploy user
sudo su - deploy

# Create application directories
mkdir -p /home/deploy/laravel-app
mkdir -p /home/deploy/backups
mkdir -p /home/deploy/logs
mkdir -p /home/deploy/ssl

# Set permissions
chmod 755 /home/deploy/laravel-app
chmod 755 /home/deploy/backups
chmod 755 /home/deploy/logs
```

### 3. Security Configuration

**Firewall Setup**:
```bash
# Configure UFW
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# Configure fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

**SSH Hardening** (`/etc/ssh/sshd_config`):
```
Port 22
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

### 4. SSL Certificate Setup

**Using Let's Encrypt**:
```bash
# Install Certbot
sudo apt install snapd
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Generate SSL certificate
sudo certbot certonly --standalone -d your-domain.com

# Copy certificates for Docker
sudo cp /etc/letsencrypt/live/your-domain.com/fullchain.pem /home/deploy/ssl/cert.crt
sudo cp /etc/letsencrypt/live/your-domain.com/privkey.pem /home/deploy/ssl/key.pem
sudo chown -R deploy:deploy /home/deploy/ssl
```

## 🚀 Initial Deployment

### 1. Clone Repository

```bash
# Clone to server
cd /home/deploy
git clone https://github.com/ruhulamin63/Scaling-Laravel-App-with-CI-CD-and-Docker.git laravel-app
cd laravel-app

# Set correct ownership
sudo chown -R deploy:deploy /home/deploy/laravel-app
```

### 2. Environment Configuration

**Production Environment File** (`.env.production`):
```env
# Application
APP_NAME="Laravel Production App"
APP_ENV=production
APP_KEY=base64:your-32-character-secret-key-here
APP_DEBUG=false
APP_URL=https://your-domain.com

# Database
DB_CONNECTION=mysql
DB_HOST=ci_cd_db
DB_PORT=3306
DB_DATABASE=laravel_prod
DB_USERNAME=laravel_user
DB_PASSWORD=super-secure-database-password

# Redis
REDIS_HOST=ci_cd_redis_cache
REDIS_PASSWORD=super-secure-redis-password
REDIS_PORT=6379

# Cache & Sessions
CACHE_DRIVER=redis
SESSION_DRIVER=redis
SESSION_LIFETIME=120
SESSION_SECURE_COOKIE=true

# Queue
QUEUE_CONNECTION=redis

# Mail
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailgun.org
MAIL_PORT=587
MAIL_USERNAME=your-mailgun-username
MAIL_PASSWORD=your-mailgun-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@your-domain.com
MAIL_FROM_NAME="${APP_NAME}"

# Docker Ports
DOCKER_APP_PORT=80
DOCKER_APP_SSL_PORT=443
DOCKER_DB_PORT=3306
DOCKER_PHPMYADMIN_PORT=8080
DOCKER_REDIS_PORT=6379

# Logging
LOG_CHANNEL=daily
LOG_LEVEL=error
LOG_DAYS=14

# Security
SESSION_SECURE_COOKIE=true
SANCTUM_STATEFUL_DOMAINS=your-domain.com
```

**Copy environment file**:
```bash
cp .env.production .env
```

### 3. SSL Certificate Integration

**Update Nginx configuration** (`.docker/nginx/conf.d/app.conf`):
```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    root /var/www/public;
    index index.php index.html;
    
    # SSL Configuration
    ssl_certificate /etc/nginx/certs/cert.crt;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied expired no-cache no-store private must-revalidate auth;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss;
    
    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=login:10m rate=10r/m;
    
    client_max_body_size 100M;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass ci_cd:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_read_timeout 300;
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
    }
    
    location ~ /\.ht {
        deny all;
    }
    
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### 4. Deploy Application

```bash
# Copy SSL certificates
cp /home/deploy/ssl/* .docker/nginx/certs/

# Build and start containers
docker-compose up -d --build

# Install dependencies
docker-compose exec ci_cd composer install --no-dev --optimize-autoloader

# Generate application key (if not set)
docker-compose exec ci_cd php artisan key:generate

# Run database migrations
docker-compose exec ci_cd php artisan migrate --force

# Optimize for production
docker-compose exec ci_cd php artisan config:cache
docker-compose exec ci_cd php artisan route:cache
docker-compose exec ci_cd php artisan view:cache
docker-compose exec ci_cd php artisan event:cache

# Create storage symlink
docker-compose exec ci_cd php artisan storage:link

# Set correct permissions
docker-compose exec ci_cd chown -R www-data:www-data /var/www/storage
docker-compose exec ci_cd chown -R www-data:www-data /var/www/bootstrap/cache
```

## 🔄 CI/CD Setup

### 1. GitHub Secrets Configuration

Navigate to `GitHub Repository > Settings > Secrets and variables > Actions` and add:

```
# Server Configuration
SERVER_HOST=your.server.ip.address
SERVER_USER=deploy
SERVER_SSH_KEY=-----BEGIN OPENSSH PRIVATE KEY-----
(your complete private key)
-----END OPENSSH PRIVATE KEY-----
SERVER_PORT=22
DEPLOY_PATH=/home/deploy/laravel-app

# Application Secrets
APP_KEY=base64:your-32-character-secret-key
DB_PASSWORD=super-secure-database-password
REDIS_PASSWORD=super-secure-redis-password

# External Services (Optional)
SLACK_WEBHOOK=https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
SENTRY_DSN=https://your-sentry-dsn@sentry.io/project-id
```

### 2. Deploy Script

Create deployment script (`deploy.sh`):
```bash
#!/bin/bash
set -e

echo "🚀 Starting deployment..."

# Navigate to application directory
cd /home/deploy/laravel-app

# Pull latest changes
echo "📥 Pulling latest code..."
git pull origin main

# Login to GitHub Container Registry
echo "🔐 Logging into container registry..."
echo $GITHUB_TOKEN | docker login ghcr.io -u $GITHUB_ACTOR --password-stdin

# Pull latest Docker image
echo "📦 Pulling latest Docker image..."
docker pull ghcr.io/ruhulamin63/scaling-laravel-app-with-ci-cd-and-docker:latest

# Stop existing containers
echo "⏹️ Stopping existing containers..."
docker-compose down

# Start updated containers
echo "🔄 Starting updated containers..."
docker-compose up -d

# Wait for containers to be ready
echo "⏳ Waiting for containers to be ready..."
sleep 30

# Run database migrations
echo "🗄️ Running database migrations..."
docker-compose exec -T ci_cd php artisan migrate --force

# Clear and rebuild caches
echo "🧹 Clearing caches..."
docker-compose exec -T ci_cd php artisan config:cache
docker-compose exec -T ci_cd php artisan route:cache
docker-compose exec -T ci_cd php artisan view:cache

# Restart queue workers
echo "👷 Restarting queue workers..."
docker-compose exec -T ci_cd php artisan queue:restart

# Health check
echo "🏥 Performing health check..."
sleep 10
if curl -f https://your-domain.com/health; then
    echo "✅ Deployment successful!"
else
    echo "❌ Health check failed!"
    exit 1
fi

# Clean up old Docker images
echo "🧽 Cleaning up old images..."
docker image prune -f

echo "🎉 Deployment completed successfully!"
```

Make script executable:
```bash
chmod +x deploy.sh
```

## 📊 Monitoring Setup

### 1. Application Monitoring

**Health Check Endpoint** (`routes/web.php`):
```php
Route::get('/health', function () {
    $checks = [
        'app' => 'ok',
        'database' => 'ok',
        'redis' => 'ok',
        'storage' => 'ok'
    ];
    
    try {
        // Database check
        DB::connection()->getPdo();
    } catch (Exception $e) {
        $checks['database'] = 'failed';
    }
    
    try {
        // Redis check
        Redis::ping();
    } catch (Exception $e) {
        $checks['redis'] = 'failed';
    }
    
    try {
        // Storage check
        Storage::disk('local')->put('health-check', 'ok');
        Storage::disk('local')->delete('health-check');
    } catch (Exception $e) {
        $checks['storage'] = 'failed';
    }
    
    $status = in_array('failed', $checks) ? 500 : 200;
    
    return response()->json([
        'status' => $status === 200 ? 'healthy' : 'unhealthy',
        'timestamp' => now(),
        'checks' => $checks
    ], $status);
});
```

### 2. Server Monitoring

**System monitoring script** (`monitor.sh`):
```bash
#!/bin/bash

# System metrics
echo "=== System Metrics ==="
echo "CPU Usage: $(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)"
echo "Memory Usage: $(free | grep Mem | awk '{printf("%.2f%%", $3/$2 * 100.0)}')"
echo "Disk Usage: $(df -h / | awk 'NR==2{printf "%s", $5}')"

# Docker metrics
echo -e "\n=== Docker Metrics ==="
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Application health
echo -e "\n=== Application Health ==="
curl -s https://your-domain.com/health | jq .

# Log errors
echo -e "\n=== Recent Errors ==="
docker-compose logs --tail=10 ci_cd | grep -i error
```

### 3. Log Management

**Logrotate configuration** (`/etc/logrotate.d/laravel-app`):
```
/home/deploy/laravel-app/storage/logs/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 644 www-data www-data
    postrotate
        docker-compose exec ci_cd php artisan log:clear
    endscript
}
```

## 🔒 Security Hardening

### 1. Application Security

**Security middleware** (`app/Http/Middleware/SecurityHeaders.php`):
```php
<?php

namespace App\Http\Middleware;

use Closure;

class SecurityHeaders
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);
        
        $response->headers->set('X-Frame-Options', 'SAMEORIGIN');
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-XSS-Protection', '1; mode=block');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
        $response->headers->set('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
        
        return $response;
    }
}
```

### 2. Database Security

**MySQL security configuration** (`.docker/mysql/secure.cnf`):
```ini
[mysqld]
# Security settings
local-infile=0
skip-show-database
skip-symbolic-links

# Performance settings
innodb_buffer_pool_size=1G
innodb_log_file_size=256M
innodb_flush_log_at_trx_commit=2
innodb_flush_method=O_DIRECT

# Binary logging
log-bin=mysql-bin
binlog_format=ROW
expire_logs_days=7
```

### 3. Network Security

**Docker network security**:
```yaml
# docker-compose.prod.yml
networks:
  ci_cd_network:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: laravel_bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

## 📋 Maintenance Procedures

### 1. Backup Strategy

**Database backup script** (`backup.sh`):
```bash
#!/bin/bash
BACKUP_DIR="/home/deploy/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Database backup
docker-compose exec -T ci_cd_db mysqldump -u root -p${DB_PASSWORD} ${DB_DATABASE} > ${BACKUP_DIR}/db_${TIMESTAMP}.sql

# Files backup
tar -czf ${BACKUP_DIR}/files_${TIMESTAMP}.tar.gz storage/app/public

# Keep only last 7 days of backups
find ${BACKUP_DIR} -name "*.sql" -type f -mtime +7 -delete
find ${BACKUP_DIR} -name "*.tar.gz" -type f -mtime +7 -delete

echo "Backup completed: ${TIMESTAMP}"
```

**Automated backup cron** (`crontab -e`):
```bash
# Daily backup at 2 AM
0 2 * * * /home/deploy/laravel-app/backup.sh >> /home/deploy/logs/backup.log 2>&1

# Weekly system cleanup
0 3 * * 0 docker system prune -f
```

### 2. Update Procedures

**Application updates**:
```bash
# Pull latest code
git pull origin main

# Update dependencies
docker-compose exec ci_cd composer update

# Run migrations
docker-compose exec ci_cd php artisan migrate

# Clear caches
docker-compose exec ci_cd php artisan optimize:clear
docker-compose exec ci_cd php artisan optimize
```

**Security updates**:
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Update Docker images
docker-compose pull
docker-compose up -d

# Update SSL certificates
sudo certbot renew
```

## 🚨 Troubleshooting

### Common Deployment Issues

1. **Container startup failures**:
   ```bash
   # Check container logs
   docker-compose logs ci_cd
   
   # Check system resources
   df -h
   free -m
   ```

2. **Database connection issues**:
   ```bash
   # Test database connectivity
   docker-compose exec ci_cd php artisan tinker
   # >>> DB::connection()->getPdo();
   ```

3. **SSL certificate problems**:
   ```bash
   # Check certificate validity
   openssl x509 -in /home/deploy/ssl/cert.crt -text -noout
   
   # Test SSL configuration
   curl -I https://your-domain.com
   ```

4. **Performance issues**:
   ```bash
   # Monitor resource usage
   docker stats
   
   # Check slow queries
   docker-compose exec ci_cd_db mysql -u root -p -e "SHOW PROCESSLIST;"
   ```

This deployment guide provides a comprehensive approach to deploying and maintaining the Laravel application in a production environment with proper security, monitoring, and maintenance procedures.
