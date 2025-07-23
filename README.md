# Scaling Laravel App with CI/CD and Docker

A complete Laravel application setup with Docker containerization and automated CI/CD pipeline using GitHub Actions. This project demonstrates how to scale a Laravel application using modern DevOps practices.

## 🚀 Features

- **Laravel 12** - Latest version of the Laravel framework
- **Docker Containerization** - Multi-container setup with PHP, Nginx, MySQL, Redis, and PHPMyAdmin
- **CI/CD Pipeline** - Automated deployment using GitHub Actions
- **SSL Support** - HTTPS configuration with custom certificates
- **Database Management** - MySQL with PHPMyAdmin interface
- **Caching** - Redis for improved performance
- **Production Ready** - Optimized configurations for production deployment

## 📋 Prerequisites

Before you begin, ensure you have the following installed on your system:

- [Docker](https://www.docker.com/get-started) (v20.10 or higher)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2.0 or higher)
- [Git](https://git-scm.com/downloads)
- [Composer](https://getcomposer.org/download/) (for local development)

## 🛠️ Installation & Setup

### 1. Clone the Repository

```bash
git clone https://github.com/ruhulamin63/Scaling-Laravel-App-with-CI-CD-and-Docker.git
cd Scaling-Laravel-App-with-CI-CD-and-Docker
```

### 2. Environment Configuration

Create your environment file from the example:

```bash
cp .env.example .env
```

Update the `.env` file with your configuration:

```env
# Application
APP_NAME="Laravel CI/CD App"
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

# Database Configuration
DB_CONNECTION=mysql
DB_HOST=ci_cd_db
DB_PORT=3306
DB_DATABASE=laravel_cicd
DB_USERNAME=laravel_user
DB_PASSWORD=secure_password

# Redis Configuration
REDIS_HOST=ci_cd_redis_cache
REDIS_PASSWORD=redis_password
REDIS_PORT=6379

# Docker Ports
DOCKER_APP_PORT=3692
DOCKER_APP_SSL_PORT=3693
DOCKER_DB_PORT=3694
DOCKER_PHPMYADMIN_PORT=3695
DOCKER_REDIS_PORT=3696
```

### 3. Build and Start the Application

```bash
# Build and start all containers
docker-compose up -d --build

# Install Composer dependencies
docker-compose exec ci_cd composer install

# Generate application key
docker-compose exec ci_cd php artisan key:generate

# Run database migrations
docker-compose exec ci_cd php artisan migrate

# Seed the database (optional)
docker-compose exec ci_cd php artisan db:seed
```

### 4. Access the Application

- **Laravel App**: http://localhost:3692 or https://localhost:3693
- **PHPMyAdmin**: http://localhost:3695
- **Redis**: localhost:3696

## 🐳 Docker Architecture

### Services Overview

| Service | Container Name | Purpose | Port |
|---------|---------------|---------|------|
| `ci_cd` | ci_cd | PHP 8.2-FPM Laravel Application | 9000 |
| `ci_cd_nginx` | ci_cd_nginx | Nginx Web Server | 80, 443 |
| `ci_cd_db` | ci_cd_db | MySQL 8.0 Database | 3306 |
| `ci_cd_phpmyadmin` | ci_cd_phpmyadmin | Database Management | 80 |
| `ci_cd_redis_cache` | ci_cd_redis_cache | Redis Cache | 6379 |

### Docker Files Structure

```
.docker/
├── Dockerfile              # PHP-FPM container configuration
├── nginx/
│   ├── conf.d/
│   │   └── app.conf        # Nginx virtual host configuration
│   └── certs/
│       ├── cert.crt        # SSL certificate
│       └── key.pem         # SSL private key
├── php/
│   └── local.ini           # PHP configuration
└── mysql/
    └── my.cnf              # MySQL configuration
```

### Key Docker Features

- **Multi-stage builds** for optimized production images
- **Custom PHP extensions** (GD, PDO, BCMath, etc.)
- **SSL/TLS support** with custom certificates
- **Volume mounting** for persistent data
- **Network isolation** with custom bridge network
- **Health checks** for service monitoring

## 🔄 CI/CD Pipeline

### GitHub Actions Workflow

The project includes an automated CI/CD pipeline using GitHub Actions (`.github/workflows/ci-cd-docker.yml`):

#### Pipeline Stages

1. **Checkout Code** - Retrieves the latest code from the repository
2. **Docker Setup** - Configures Docker Buildx for multi-platform builds
3. **Build Image** - Creates optimized Docker image
4. **Image Packaging** - Compresses Docker image for transfer
5. **Deploy to Server** - Transfers and deploys to production server
6. **Post-deployment** - Runs migrations and optimizations

#### Required Secrets

Configure these secrets in your GitHub repository settings:

```env
SERVER_IP=your.server.ip.address
SSH_USER=your_ssh_username
SSH_KEY=your_private_ssh_key
```

#### Workflow Triggers

- **Push to main branch** - Automatic deployment
- **Manual trigger** - Via GitHub Actions interface

### Production Deployment

#### Server Requirements

- Ubuntu 20.04+ or similar Linux distribution
- Docker and Docker Compose installed
- SSH access configured
- Minimum 2GB RAM, 2 CPU cores
- 20GB+ storage space

#### Deployment Steps

1. **Prepare Server**:
   ```bash
   # Install Docker
   curl -fsSL https://get.docker.com -o get-docker.sh
   sh get-docker.sh
   
   # Install Docker Compose
   sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   sudo chmod +x /usr/local/bin/docker-compose
   ```

2. **Configure Environment**:
   ```bash
   # Create project directory
   mkdir -p /home/ubuntu/laravel-app
   cd /home/ubuntu/laravel-app
   
   # Create production environment file
   nano .env.production
   ```

3. **Deploy Application**:
   ```bash
   # The CI/CD pipeline automatically handles:
   # - Image transfer
   # - Container startup
   # - Database migrations
   # - Cache optimization
   ```

## 🛡️ Security Considerations

### SSL/TLS Configuration

- Custom SSL certificates in `.docker/nginx/certs/`
- HTTPS redirect configuration
- Secure headers implementation

### Database Security

- Non-root MySQL user configuration
- Password-protected Redis instance
- Database connection encryption

### Application Security

- Environment-based configuration
- Secure session handling
- CSRF protection enabled
- XSS protection headers

## 📊 Performance Optimization

### Application Level

```bash
# Optimize for production
docker-compose exec ci_cd php artisan config:cache
docker-compose exec ci_cd php artisan route:cache
docker-compose exec ci_cd php artisan view:cache
docker-compose exec ci_cd php artisan event:cache
```

### Database Optimization

- MySQL 8.0 with optimized `my.cnf`
- Connection pooling
- Query optimization
- Index strategies

### Caching Strategy

- **Redis** for session and cache storage
- **OPcache** for PHP bytecode caching
- **Nginx** static file caching
- **Application** cache for views and routes

## 🔧 Development Workflow

### Local Development

```bash
# Start development environment
docker-compose up -d

# Watch for file changes (if using Vite)
docker-compose exec ci_cd npm run dev

# Run tests
docker-compose exec ci_cd php artisan test

# Access container shell
docker-compose exec ci_cd bash
```

### Database Management

```bash
# Create migration
docker-compose exec ci_cd php artisan make:migration create_example_table

# Run migrations
docker-compose exec ci_cd php artisan migrate

# Rollback migrations
docker-compose exec ci_cd php artisan migrate:rollback

# Database seeding
docker-compose exec ci_cd php artisan db:seed
```

### Debugging

```bash
# View application logs
docker-compose logs ci_cd

# View Nginx logs
docker-compose logs ci_cd_nginx

# View database logs
docker-compose logs ci_cd_db

# Monitor containers
docker-compose ps
docker stats
```

## 📝 Useful Commands

### Container Management

```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# Rebuild specific service
docker-compose build ci_cd

# View container status
docker-compose ps

# Remove all containers and volumes
docker-compose down -v --remove-orphans
```

### Laravel Artisan Commands

```bash
# Generate application key
docker-compose exec ci_cd php artisan key:generate

# Create controller
docker-compose exec ci_cd php artisan make:controller ExampleController

# Create model
docker-compose exec ci_cd php artisan make:model Example

# Clear caches
docker-compose exec ci_cd php artisan cache:clear
docker-compose exec ci_cd php artisan config:clear
docker-compose exec ci_cd php artisan route:clear
docker-compose exec ci_cd php artisan view:clear
```

### Database Operations

```bash
# Access MySQL CLI
docker-compose exec ci_cd_db mysql -u laravel_user -p laravel_cicd

# Import SQL dump
docker-compose exec -T ci_cd_db mysql -u laravel_user -p laravel_cicd < backup.sql

# Export database
docker-compose exec ci_cd_db mysqldump -u laravel_user -p laravel_cicd > backup.sql
```

## 🔍 Troubleshooting

### Common Issues

1. **Port Already in Use**:
   ```bash
   # Change ports in .env file
   DOCKER_APP_PORT=8080
   DOCKER_DB_PORT=3307
   ```

2. **Permission Issues**:
   ```bash
   # Fix file permissions
   sudo chown -R $USER:$USER .
   chmod -R 755 storage bootstrap/cache
   ```

3. **Database Connection Failed**:
   ```bash
   # Ensure database service is running
   docker-compose ps ci_cd_db
   
   # Check database logs
   docker-compose logs ci_cd_db
   ```

4. **Composer Install Issues**:
   ```bash
   # Clear composer cache
   docker-compose exec ci_cd composer clear-cache
   
   # Install with verbose output
   docker-compose exec ci_cd composer install -v
   ```

### Health Checks

```bash
# Check service health
curl -f http://localhost:3692/health || echo "Service down"

# Database connectivity test
docker-compose exec ci_cd php artisan tinker
# >>> DB::connection()->getPdo();
```

## 📚 Additional Resources

- [Laravel Documentation](https://laravel.com/docs)
- [Docker Documentation](https://docs.docker.com/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Nginx Configuration Guide](https://nginx.org/en/docs/)
- [MySQL 8.0 Reference](https://dev.mysql.com/doc/refman/8.0/en/)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 👨‍💻 Author

**Ruhul Amin**
- GitHub: [@ruhulamin63](https://github.com/ruhulamin63)

## 🙏 Acknowledgments

- Laravel Framework Team
- Docker Community
- GitHub Actions Team
- Open Source Contributors