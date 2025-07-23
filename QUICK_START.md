# Laravel Docker Quick Start

This is a quick reference guide for getting the Laravel application up and running with Docker.

## 🚀 Quick Setup (5 minutes)

### Prerequisites
- Docker & Docker Compose installed
- Git installed

### 1. Clone & Setup
```bash
git clone https://github.com/ruhulamin63/Scaling-Laravel-App-with-CI-CD-and-Docker.git
cd Scaling-Laravel-App-with-CI-CD-and-Docker
cp .env.example .env
```

### 2. Configure Environment
Edit `.env` file with these essential settings:
```env
# Database
DB_DATABASE=laravel_cicd
DB_USERNAME=laravel_user  
DB_PASSWORD=secure_password

# Ports (change if conflicts)
DOCKER_APP_PORT=3692
DOCKER_DB_PORT=3694
DOCKER_PHPMYADMIN_PORT=3695
```

### 3. Start Application
```bash
docker-compose up -d --build
docker-compose exec ci_cd composer install
docker-compose exec ci_cd php artisan key:generate
docker-compose exec ci_cd php artisan migrate
```

### 4. Access Your App
- **Laravel App**: http://localhost:3692
- **PHPMyAdmin**: http://localhost:3695

## 📋 Common Commands

### Container Management
```bash
# Start all services
docker-compose up -d

# Stop all services  
docker-compose down

# View logs
docker-compose logs ci_cd

# Access container shell
docker-compose exec ci_cd bash
```

### Laravel Commands
```bash
# Artisan commands
docker-compose exec ci_cd php artisan migrate
docker-compose exec ci_cd php artisan cache:clear
docker-compose exec ci_cd php artisan config:cache

# Composer
docker-compose exec ci_cd composer install
docker-compose exec ci_cd composer update
```

### Database Operations
```bash
# Access MySQL CLI
docker-compose exec ci_cd_db mysql -u laravel_user -p

# Run migrations
docker-compose exec ci_cd php artisan migrate

# Seed database  
docker-compose exec ci_cd php artisan db:seed
```

## 🛠️ Development Workflow

### File Changes
- Source code changes are automatically reflected (volume mounted)
- For config changes: `docker-compose exec ci_cd php artisan config:clear`

### Database Changes
```bash
# Create migration
docker-compose exec ci_cd php artisan make:migration create_example_table

# Run migrations
docker-compose exec ci_cd php artisan migrate

# Fresh migration (destructive)
docker-compose exec ci_cd php artisan migrate:fresh --seed
```

### Frontend Assets
```bash
# Install NPM dependencies
docker-compose exec ci_cd npm install

# Build assets
docker-compose exec ci_cd npm run build

# Watch for changes
docker-compose exec ci_cd npm run dev
```

## 🔧 Troubleshooting

### Port Conflicts
If ports are already in use, change in `.env`:
```env
DOCKER_APP_PORT=8080
DOCKER_DB_PORT=3307
DOCKER_PHPMYADMIN_PORT=8081
```

### Permission Issues
```bash
# Fix Laravel permissions
docker-compose exec ci_cd chown -R www-data:www-data storage bootstrap/cache
docker-compose exec ci_cd chmod -R 755 storage bootstrap/cache
```

### Container Issues
```bash
# Rebuild containers
docker-compose down
docker-compose up -d --build

# Check container status
docker-compose ps

# View container logs
docker-compose logs [service_name]
```

### Database Connection Issues
```bash
# Test database connection
docker-compose exec ci_cd php artisan tinker
# In Tinker: DB::connection()->getPdo();

# Check database logs
docker-compose logs ci_cd_db
```

## 📊 Service Overview

| Service | Port | Purpose |
|---------|------|---------|
| Laravel App | 3692 | Main application |
| MySQL | 3694 | Database |
| PHPMyAdmin | 3695 | Database management |
| Redis | 3696 | Cache/Sessions |
| Nginx | 80/443 | Web server |

## 🔄 CI/CD Quick Setup

### 1. GitHub Secrets
Add these secrets to your GitHub repository:
- `SERVER_HOST` - Your server IP
- `SERVER_USER` - SSH username
- `SERVER_SSH_KEY` - Private SSH key
- `DEPLOY_PATH` - Application path on server

### 2. Auto-deployment
Push to `main` branch triggers automatic deployment via GitHub Actions.

## 📚 Full Documentation

For detailed setup, production deployment, and advanced configuration:
- [Complete README](README.md)
- [Docker Setup Guide](DOCKER_SETUP.md)
- [CI/CD Pipeline Documentation](CICD_PIPELINE.md)
- [Production Deployment Guide](DEPLOYMENT_GUIDE.md)

## 💡 Tips

1. **Development**: Use `docker-compose logs -f` to watch logs in real-time
2. **Performance**: Enable OPcache in production for better PHP performance
3. **Security**: Always use strong passwords in production
4. **Backup**: Regular database backups are automated in production setup
5. **Monitoring**: Health check endpoint available at `/health`

Need help? Check the full documentation or open an issue on GitHub!
