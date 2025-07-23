# CI/CD Pipeline Documentation

This document provides comprehensive information about the Continuous Integration and Continuous Deployment (CI/CD) pipeline implemented for the Laravel application using GitHub Actions.

## 🔄 Pipeline Overview

The CI/CD pipeline automates the entire deployment process from code commit to production deployment, ensuring consistent and reliable releases.

### Pipeline Flow

```mermaid
graph LR
    A[Code Push] --> B[GitHub Actions Trigger]
    B --> C[Checkout Code]
    C --> D[Setup Docker]
    D --> E[Build Application]
    E --> F[Run Tests]
    F --> G[Build Docker Image]
    G --> H[Deploy to Server]
    H --> I[Post-deployment Tasks]
    I --> J[Health Check]
```

## 📁 Pipeline Configuration

### GitHub Actions Workflow File

Location: `.github/workflows/ci-cd-docker.yml`

```yaml
name: Deploy Laravel with Docker

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:  # Manual trigger

env:
  DOCKER_IMAGE_NAME: laravel-cicd-app
  REGISTRY: ghcr.io

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3
      
      redis:
        image: redis:alpine
        ports:
          - 6379:6379
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: 8.2
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, soap, intl, gd, exif, iconv
          coverage: xdebug
      
      - name: Cache Composer dependencies
        uses: actions/cache@v3
        with:
          path: vendor
          key: composer-${{ hashFiles('composer.lock') }}
          restore-keys: composer-
      
      - name: Install Composer dependencies
        run: composer install --prefer-dist --no-interaction --no-progress
      
      - name: Copy environment file
        run: cp .env.testing .env
      
      - name: Generate application key
        run: php artisan key:generate
      
      - name: Create SQLite database
        run: touch database/database.sqlite
      
      - name: Run database migrations
        run: php artisan migrate --force
      
      - name: Run PHPUnit tests
        run: vendor/bin/phpunit --coverage-clover coverage.xml
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml

  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: .docker/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            user=laravel
            uid=1000

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [test, build]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master'
    environment: production
    
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          port: ${{ secrets.SERVER_PORT }}
          script: |
            # Navigate to application directory
            cd ${{ secrets.DEPLOY_PATH }}
            
            # Pull latest changes
            git pull origin main
            
            # Log in to container registry
            echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            
            # Pull latest Docker image
            docker pull ${{ env.REGISTRY }}/${{ github.repository }}:latest
            
            # Stop existing containers
            docker-compose down
            
            # Start updated containers
            docker-compose up -d
            
            # Run post-deployment commands
            docker-compose exec -T ci_cd php artisan migrate --force
            docker-compose exec -T ci_cd php artisan config:cache
            docker-compose exec -T ci_cd php artisan route:cache
            docker-compose exec -T ci_cd php artisan view:cache
            docker-compose exec -T ci_cd php artisan queue:restart
            
            # Health check
            sleep 30
            curl -f http://localhost:3692/health || exit 1
            
            # Clean up old images
            docker image prune -f

  notify:
    name: Send Notifications
    runs-on: ubuntu-latest
    needs: [deploy]
    if: always()
    
    steps:
      - name: Notify Slack on Success
        if: success()
        uses: 8398a7/action-slack@v3
        with:
          status: success
          text: '🚀 Deployment successful!'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      
      - name: Notify Slack on Failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          text: '❌ Deployment failed!'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## 🔐 Required Secrets

Configure these secrets in your GitHub repository settings (`Settings > Secrets and variables > Actions`):

### Server Configuration
```
SERVER_HOST=your.production.server.ip
SERVER_USER=deploy_user
SERVER_SSH_KEY=-----BEGIN OPENSSH PRIVATE KEY-----...
SERVER_PORT=22
DEPLOY_PATH=/home/deploy/laravel-app
```

### Application Configuration
```
APP_KEY=base64:your_32_character_secret_key
DB_PASSWORD=your_database_password
REDIS_PASSWORD=your_redis_password
```

### External Services
```
SLACK_WEBHOOK=https://hooks.slack.com/services/...
CODECOV_TOKEN=your_codecov_token
SENTRY_DSN=https://your_sentry_dsn
```

## 🏗️ Pipeline Stages Detailed

### 1. Test Stage

**Purpose**: Validate code quality and functionality

**Steps**:
- **Code Checkout**: Download source code from repository
- **PHP Setup**: Install PHP 8.2 with required extensions
- **Dependency Installation**: Install Composer packages
- **Environment Setup**: Configure testing environment
- **Database Setup**: Run migrations on test database
- **Test Execution**: Run PHPUnit test suite
- **Coverage Report**: Generate and upload coverage data

**Test Configuration** (`.env.testing`):
```env
APP_ENV=testing
APP_DEBUG=true
DB_CONNECTION=sqlite
DB_DATABASE=:memory:
CACHE_DRIVER=array
SESSION_DRIVER=array
QUEUE_CONNECTION=sync
MAIL_MAILER=array
```

### 2. Build Stage

**Purpose**: Create production-ready Docker images

**Steps**:
- **Docker Setup**: Configure Docker Buildx for multi-platform builds
- **Registry Login**: Authenticate with GitHub Container Registry
- **Metadata Extraction**: Generate image tags and labels
- **Image Building**: Build optimized Docker image
- **Registry Push**: Upload image to container registry
- **Cache Management**: Utilize build cache for faster builds

**Docker Build Arguments**:
```dockerfile
ARG user=laravel
ARG uid=1000
ARG BUILD_ENV=production
```

### 3. Deploy Stage

**Purpose**: Deploy application to production server

**Pre-deployment Checks**:
- Verify all tests passed
- Confirm Docker image built successfully
- Check deployment environment availability

**Deployment Steps**:
1. **Server Connection**: SSH into production server
2. **Code Update**: Pull latest code changes
3. **Image Pull**: Download latest Docker image
4. **Service Stop**: Gracefully stop existing containers
5. **Service Start**: Start updated containers
6. **Database Migration**: Run pending migrations
7. **Cache Optimization**: Clear and rebuild caches
8. **Health Check**: Verify application functionality
9. **Cleanup**: Remove old Docker images

### 4. Notification Stage

**Purpose**: Inform team about deployment status

**Notifications Sent**:
- Slack channel updates
- Email notifications (optional)
- GitHub deployment status
- Monitoring system alerts

## 🎯 Deployment Strategies

### Blue-Green Deployment

```yaml
# docker-compose.blue.yml
version: '3.7'
services:
  ci_cd_blue:
    image: ghcr.io/ruhulamin63/scaling-laravel-app:latest
    container_name: ci_cd_blue
    # ... configuration

# docker-compose.green.yml
version: '3.7'
services:
  ci_cd_green:
    image: ghcr.io/ruhulamin63/scaling-laravel-app:latest
    container_name: ci_cd_green
    # ... configuration
```

**Deployment Script**:
```bash
#!/bin/bash
CURRENT_ENV=$(docker ps --format "table {{.Names}}" | grep ci_cd | head -1)

if [[ $CURRENT_ENV == *"blue"* ]]; then
    NEW_ENV="green"
    OLD_ENV="blue"
else
    NEW_ENV="blue"
    OLD_ENV="green"
fi

# Deploy to new environment
docker-compose -f docker-compose.$NEW_ENV.yml up -d

# Health check
sleep 30
curl -f http://localhost:3692/health

# Switch traffic (update load balancer)
# Update nginx upstream configuration

# Stop old environment
docker-compose -f docker-compose.$OLD_ENV.yml down
```

### Rolling Deployment

```yaml
# docker-compose.rolling.yml
version: '3.7'
services:
  ci_cd:
    image: ghcr.io/ruhulamin63/scaling-laravel-app:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 30s
        failure_action: rollback
    # ... configuration
```

## 📊 Monitoring and Observability

### Application Performance Monitoring

```yaml
# Add to docker-compose.yml
services:
  apm:
    image: elastic/apm-server:8.0.0
    environment:
      - output.elasticsearch.hosts=["elasticsearch:9200"]
    ports:
      - "8200:8200"
```

### Log Aggregation

```yaml
# Fluentd configuration
services:
  fluentd:
    image: fluent/fluentd:v1.14
    volumes:
      - ./fluentd:/fluentd/etc
      - /var/log:/var/log:ro
```

### Health Checks

**Application Health Endpoint** (`routes/web.php`):
```php
Route::get('/health', function () {
    return response()->json([
        'status' => 'healthy',
        'timestamp' => now(),
        'services' => [
            'database' => DB::connection()->getPdo() ? 'up' : 'down',
            'redis' => Redis::ping() ? 'up' : 'down',
            'storage' => Storage::disk('local')->exists('test') ? 'up' : 'down',
        ]
    ]);
});
```

## 🔧 Environment Management

### Environment-specific Configurations

**Development** (`.env.development`):
```env
APP_ENV=local
APP_DEBUG=true
LOG_LEVEL=debug
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
```

**Staging** (`.env.staging`):
```env
APP_ENV=staging
APP_DEBUG=true
LOG_LEVEL=info
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
```

**Production** (`.env.production`):
```env
APP_ENV=production
APP_DEBUG=false
LOG_LEVEL=error
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_SECURE_COOKIE=true
```

### Configuration Management

```bash
# Encrypt sensitive configuration
php artisan env:encrypt --env=production

# Decrypt in CI/CD pipeline
php artisan env:decrypt --env=production --key=$ENCRYPTION_KEY
```

## 🚨 Rollback Procedures

### Automatic Rollback

```yaml
# In GitHub Actions workflow
- name: Health Check and Rollback
  run: |
    sleep 30
    if ! curl -f http://localhost:3692/health; then
      echo "Health check failed, rolling back..."
      docker-compose down
      docker pull ${{ env.REGISTRY }}/${{ github.repository }}:previous
      docker-compose up -d
      exit 1
    fi
```

### Manual Rollback

```bash
# List available image tags
docker images ghcr.io/ruhulamin63/scaling-laravel-app

# Rollback to previous version
docker-compose down
docker run -d --name ci_cd_rollback ghcr.io/ruhulamin63/scaling-laravel-app:previous-tag
docker-compose up -d

# Verify rollback
curl -f http://localhost:3692/health
```

## 📈 Performance Optimization

### Pipeline Optimization

1. **Parallel Job Execution**:
   ```yaml
   jobs:
     test-unit:
       # Unit tests
     test-integration:
       # Integration tests
     test-browser:
       # Browser tests
   ```

2. **Docker Layer Caching**:
   ```dockerfile
   # Cache dependencies separately
   COPY composer.json composer.lock ./
   RUN composer install --no-dev --no-scripts
   
   # Copy application code
   COPY . .
   ```

3. **Build Cache Optimization**:
   ```yaml
   - name: Build with cache
     uses: docker/build-push-action@v5
     with:
       cache-from: type=gha
       cache-to: type=gha,mode=max
   ```

### Deployment Optimization

1. **Multi-stage Builds**:
   ```dockerfile
   FROM node:18 AS frontend-build
   COPY package*.json ./
   RUN npm ci
   COPY . .
   RUN npm run build
   
   FROM php:8.2-fpm AS backend
   COPY --from=frontend-build /app/public /var/www/public
   ```

2. **Dependency Optimization**:
   ```bash
   # Production-only dependencies
   composer install --no-dev --optimize-autoloader --classmap-authoritative
   ```

## 🔍 Troubleshooting

### Common Pipeline Issues

1. **Test Failures**:
   ```bash
   # Debug test environment
   php artisan env
   php artisan config:show database
   ```

2. **Docker Build Failures**:
   ```bash
   # Build with verbose output
   docker build --progress=plain --no-cache -f .docker/Dockerfile .
   ```

3. **Deployment Failures**:
   ```bash
   # Check server connectivity
   ssh -o ConnectTimeout=10 user@server 'echo "Connection successful"'
   
   # Verify Docker service
   docker --version
   docker-compose --version
   ```

4. **Health Check Failures**:
   ```bash
   # Debug application status
   docker-compose logs ci_cd
   docker-compose exec ci_cd php artisan tinker
   ```

### Pipeline Monitoring

```bash
# Monitor GitHub Actions usage
gh api /repos/ruhulamin63/Scaling-Laravel-App-with-CI-CD-and-Docker/actions/runs

# Check deployment status
curl -f http://localhost:3692/health

# Monitor resource usage
docker stats --no-stream
```

This CI/CD pipeline provides automated, reliable, and scalable deployment for the Laravel application with comprehensive testing, monitoring, and rollback capabilities.
