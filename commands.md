# Reacher Self-Hosting Commands - Bulk API Setup

This document contains all the commands and environment variable configurations needed to run Reacher with bulk API support, PostgreSQL, and Nginx on your server.

## Prerequisites

- Docker and Docker Compose installed on your server
- Ports 80 (HTTP), 443 (HTTPS, optional), and 5432 (PostgreSQL, optional) available
- Basic knowledge of Docker and command line

## Quick Start Guide

### 1. Set up environment variables

Create a `.env` file in the project root with your sensitive configuration:

```bash
# Create .env file (copy from example below or create manually)
cat > .env << 'EOF'
# PostgreSQL Configuration
POSTGRES_USER=reacher
POSTGRES_PASSWORD=your_secure_password_here
POSTGRES_DB=reacher_db
POSTGRES_PORT=5432

# Backend Configuration
RCH__BACKEND_NAME=reacher-backend
RUST_LOG=reacher=info

# SMTP Configuration
RCH__HELLO_NAME=your-domain.com
RCH__FROM_EMAIL=hello@your-domain.com
RCH__SMTP_TIMEOUT=45

# Security (IMPORTANT: Generate secure values)
RCH__HEADER_SECRET=your_secret_key_here

# Optional: Throttle Configuration
RCH__THROTTLE__MAX_REQUESTS_PER_SECOND=20
RCH__THROTTLE__MAX_REQUESTS_PER_MINUTE=60
RCH__THROTTLE__MAX_REQUESTS_PER_HOUR=1000
RCH__THROTTLE__MAX_REQUESTS_PER_DAY=10000

# Nginx Configuration
NGINX_HTTP_PORT=80
NGINX_HTTPS_PORT=443
EOF

# Edit with your actual values
nano .env
```

**Important**: 
- The `.env` file is gitignored and should contain your actual passwords and sensitive configuration
- Generate secure passwords: `openssl rand -base64 32` for POSTGRES_PASSWORD
- Generate secure secret: `openssl rand -hex 32` for RCH__HEADER_SECRET

### 2. Set up local commands file (optional)

For convenience, you can copy the example commands file:

```bash
# Copy the example file with setup steps
cp localcommands.txt.example localcommands.txt
```

The `localcommands.txt` file contains step-by-step commands to run the setup. It's also gitignored.

### 3. Start all services

```bash
# Start all services in detached mode
# Docker Compose automatically reads .env file
docker-compose up -d
```

### 3. Verify services are running

```bash
# Check status of all services
docker-compose ps

# You should see:
# - reacher-postgres (healthy)
# - reacher-backend (running)
# - reacher-nginx (running)
```

### 4. Check logs

```bash
# View logs from all services
docker-compose logs -f

# View logs from specific service
docker-compose logs -f backend
docker-compose logs -f postgres
docker-compose logs -f nginx
```

### 5. Test the API

```bash
# Test single email verification
curl -X POST http://localhost/v0/check_email \
  -H "Content-Type: application/json" \
  -d '{"to_email": "test@example.com"}'

# Test bulk API - Submit a job
curl -X POST http://localhost/v0/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "input_type": "array",
    "input": [
      "test1@example.com",
      "test2@example.com",
      "test3@example.com"
    ]
  }'
```

## Server Deployment Steps

### Initial Setup on Server

1. **Clone or upload the project to your server**

```bash
# If using git
git clone <your-repo-url>
cd check-if-email-exists

# Or upload files via SCP/SFTP
```

2. **Create and configure .env file**

```bash
# Create .env file with your sensitive configuration
# See environment variables section below for the template
nano .env  # Edit with your actual values
```

3. **Start the services**

```bash
docker-compose up -d
```

4. **Verify everything is working**

```bash
# Check if backend is accessible
curl http://localhost/v0/check_email -X POST \
  -H "Content-Type: application/json" \
  -d '{"to_email":"test@example.com"}'

# Check nginx health
curl http://localhost/health
```

## Docker Compose Commands

### Basic Operations

```bash
# Start services (automatically reads .env file)
docker-compose up -d

# Stop services (keeps containers)
docker-compose stop

# Stop and remove containers
docker-compose down

# Stop and remove containers + volumes (WARNING: deletes database)
docker-compose down -v

# Restart a specific service
docker-compose restart backend

# View logs
docker-compose logs -f [service_name]

# Rebuild and restart
docker-compose up -d --build
```

### Service-Specific Commands

#### PostgreSQL

```bash
# Connect to PostgreSQL database
docker exec -it reacher-postgres psql -U reacher -d reacher_db

# List all tables
docker exec -it reacher-postgres psql -U reacher -d reacher_db -c "\dt"

# Backup database
docker exec reacher-postgres pg_dump -U reacher reacher_db > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore database from backup
docker exec -i reacher-postgres psql -U reacher reacher_db < backup.sql

# Check database size
docker exec reacher-postgres psql -U reacher -d reacher_db -c "SELECT pg_size_pretty(pg_database_size('reacher_db'));"
```

#### Backend

```bash
# View backend logs
docker logs -f reacher-backend

# Restart backend
docker-compose restart backend

# Execute command in backend container
docker exec -it reacher-backend sh

# Check backend environment variables
docker exec reacher-backend env | grep RCH
```

#### Nginx

```bash
# Test Nginx configuration
docker exec reacher-nginx nginx -t

# Reload Nginx configuration (after editing nginx.conf)
docker exec reacher-nginx nginx -s reload

# View Nginx logs
docker logs -f reacher-nginx

# View Nginx access logs
docker exec reacher-nginx tail -f /var/log/nginx/access.log

# View Nginx error logs
docker exec reacher-nginx tail -f /var/log/nginx/error.log
```

## Bulk API Usage

### API Endpoints

The bulk API is available at:
- **v0/bulk** (PostgreSQL-based, simpler setup)
- **v1/bulk** (RabbitMQ-based, requires additional setup - see advanced section)

### Using v0/bulk API

#### 1. Submit a bulk verification job

```bash
curl -X POST http://localhost/v0/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "input_type": "array",
    "input": [
      "email1@example.com",
      "email2@example.com",
      "email3@example.com"
    ],
    "hello_name": "your-domain.com",
    "from_email": "hello@your-domain.com",
    "smtp_ports": [25, 587]
  }'
```

Response:
```json
{
  "job_id": 1
}
```

#### 2. Check job status

```bash
# Replace 1 with your actual job_id
curl http://localhost/v0/bulk/1
```

Response (while running):
```json
{
  "job_id": 1,
  "created_at": "2024-01-01T00:00:00Z",
  "finished_at": null,
  "total_records": 3,
  "total_processed": 2,
  "summary": {
    "total_safe": 1,
    "total_invalid": 1,
    "total_risky": 0,
    "total_unknown": 0
  },
  "job_status": "Running"
}
```

Response (when completed):
```json
{
  "job_id": 1,
  "created_at": "2024-01-01T00:00:00Z",
  "finished_at": "2024-01-01T00:05:00Z",
  "total_records": 3,
  "total_processed": 3,
  "summary": {
    "total_safe": 2,
    "total_invalid": 1,
    "total_risky": 0,
    "total_unknown": 0
  },
  "job_status": "Completed"
}
```

#### 3. Get job results

```bash
# Get first 50 results (default)
curl http://localhost/v0/bulk/1/results

# Get results with pagination (offset and limit)
curl "http://localhost/v0/bulk/1/results?offset=0&limit=10"

# Get results as CSV
curl "http://localhost/v0/bulk/1/results?format=csv" > results.csv

# Get all results (use with caution for large jobs)
curl "http://localhost/v0/bulk/1/results?offset=0&limit=10000"
```

### Using v1/bulk API (Advanced - Requires RabbitMQ)

The v1/bulk API requires RabbitMQ setup. See the `rabbitmq/docker-compose.yaml` file for an example configuration.

## Environment Variables Reference

### Creating .env File

Create a `.env` file in the project root with the following structure:

```bash
# PostgreSQL Configuration
POSTGRES_USER=reacher
POSTGRES_PASSWORD=your_secure_password_here
POSTGRES_DB=reacher_db
POSTGRES_PORT=5432

# Backend Configuration
RCH__BACKEND_NAME=reacher-backend
RUST_LOG=reacher=info

# SMTP Configuration
RCH__HELLO_NAME=your-domain.com
RCH__FROM_EMAIL=hello@your-domain.com
RCH__SMTP_TIMEOUT=45

# Security Configuration
RCH__HEADER_SECRET=your_secret_key_here

# Optional: Proxy Configuration
# RCH__PROXY__HOST=your-proxy-host.com
# RCH__PROXY__PORT=1080
# RCH__PROXY__USERNAME=proxy_username
# RCH__PROXY__PASSWORD=proxy_password

# Optional: Throttle Configuration
RCH__THROTTLE__MAX_REQUESTS_PER_SECOND=20
RCH__THROTTLE__MAX_REQUESTS_PER_MINUTE=60
RCH__THROTTLE__MAX_REQUESTS_PER_HOUR=1000
RCH__THROTTLE__MAX_REQUESTS_PER_DAY=10000

# Nginx Configuration
NGINX_HTTP_PORT=80
NGINX_HTTPS_PORT=443
```

**Note**: The `.env` file is automatically read by Docker Compose. Make sure it's in your `.gitignore` (it already is).

### PostgreSQL Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `POSTGRES_USER` | `reacher` | PostgreSQL username |
| `POSTGRES_PASSWORD` | `changeme` | **CHANGE THIS** - PostgreSQL password |
| `POSTGRES_DB` | `reacher_db` | PostgreSQL database name |
| `POSTGRES_PORT` | `5432` | PostgreSQL port (host mapping) |

### Backend Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `RCH__BACKEND_NAME` | `reacher-backend` | Backend identifier |
| `RCH__HTTP_HOST` | `0.0.0.0` | HTTP bind address (internal) |
| `RCH__HTTP_PORT` | `8020` | HTTP port (internal) |
| `RCH_ENABLE_BULK` | `1` | Enable bulk API (required for bulk endpoints) |
| `RCH__STORAGE__POSTGRES__DB_URL` | Auto-generated | PostgreSQL connection string |
| `RCH__HELLO_NAME` | `localhost` | SMTP EHLO name |
| `RCH__FROM_EMAIL` | `hello@localhost` | SMTP FROM email |
| `RCH__SMTP_TIMEOUT` | `45` | SMTP timeout in seconds |
| `RCH__HEADER_SECRET` | (empty) | Secret for x-reacher-secret header (recommended for production) |
| `RUST_LOG` | `reacher=info` | Log level (debug, info, warn, error) |

### Proxy Variables (Optional)

| Variable | Description |
|----------|-------------|
| `RCH__PROXY__HOST` | SOCKS5 proxy host |
| `RCH__PROXY__PORT` | SOCKS5 proxy port |
| `RCH__PROXY__USERNAME` | Proxy username (if required) |
| `RCH__PROXY__PASSWORD` | Proxy password (if required) |

### Throttle Variables (Optional)

| Variable | Description |
|----------|-------------|
| `RCH__THROTTLE__MAX_REQUESTS_PER_SECOND` | Max requests per second |
| `RCH__THROTTLE__MAX_REQUESTS_PER_MINUTE` | Max requests per minute |
| `RCH__THROTTLE__MAX_REQUESTS_PER_HOUR` | Max requests per hour |
| `RCH__THROTTLE__MAX_REQUESTS_PER_DAY` | Max requests per day |

### Nginx Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `NGINX_HTTP_PORT` | `80` | HTTP port (host mapping) |
| `NGINX_HTTPS_PORT` | `443` | HTTPS port (host mapping) |

## Security Considerations

### 1. Change Default Passwords

**CRITICAL**: Always change default passwords in production:

```bash
# Generate a secure password
openssl rand -base64 32

# Update in .env file
POSTGRES_PASSWORD=your_generated_password_here
```

### 2. Set Header Secret

Protect your API with a secret header:

```bash
# Generate a secure secret
openssl rand -hex 32

# Update in .env file
RCH__HEADER_SECRET=your_generated_secret_here

# Then use in API requests
curl -X POST http://localhost/v0/bulk \
  -H "Content-Type: application/json" \
  -H "x-reacher-secret: your_generated_secret_here" \
  -d '{"input_type": "array", "input": ["test@example.com"]}'
```

### 3. Configure HTTPS

1. Place your SSL certificates in `nginx/ssl/`:
   - `cert.pem` (SSL certificate)
   - `key.pem` (SSL private key)

2. Uncomment the HTTPS server block in `nginx/nginx.conf`

3. Update `.env` file:
   ```bash
   NGINX_HTTP_PORT=80
   NGINX_HTTPS_PORT=443
   ```

### 4. Firewall Configuration

Only expose necessary ports:
- **80** (HTTP) - Public
- **443** (HTTPS) - Public
- **5432** (PostgreSQL) - **DO NOT EXPOSE PUBLICLY** - Only for internal access

### 5. Database Access

- PostgreSQL is only accessible within the Docker network
- If you need external access, use SSH tunneling or a VPN
- Never expose PostgreSQL port directly to the internet

## Troubleshooting

### Backend not starting

```bash
# Check backend logs
docker-compose logs backend

# Verify PostgreSQL is healthy
docker-compose ps postgres

# Test database connection
docker exec reacher-postgres psql -U reacher -d reacher_db -c "SELECT 1;"

# Check if backend can connect to database
docker exec reacher-backend sh -c 'echo $RCH__STORAGE__POSTGRES__DB_URL'
```

### Nginx 502 Bad Gateway

```bash
# Check if backend is running
docker-compose ps backend

# Check backend logs
docker-compose logs backend

# Test backend directly (bypass nginx)
curl http://localhost:8020/v0/check_email -X POST \
  -H "Content-Type: application/json" \
  -d '{"to_email":"test@example.com"}'

# Check nginx logs
docker logs reacher-nginx
```

### Database connection errors

```bash
# Check PostgreSQL logs
docker-compose logs postgres

# Verify environment variables
docker-compose config | grep POSTGRES

# Test connection string manually
docker exec reacher-postgres psql -U reacher -d reacher_db -c "SELECT version();"
```

### Bulk API not working

```bash
# Verify bulk is enabled
docker exec reacher-backend env | grep RCH_ENABLE_BULK

# Check if database tables exist
docker exec reacher-postgres psql -U reacher -d reacher_db -c "\dt"

# Check backend logs for bulk-related errors
docker logs reacher-backend | grep -i bulk
```

### Port already in use

```bash
# Check what's using port 80
sudo lsof -i :80
# or
sudo netstat -tulpn | grep :80

# Change port in .env file
NGINX_HTTP_PORT=8020
```

## Production Deployment Checklist

- [ ] Changed `POSTGRES_PASSWORD` to a secure value
- [ ] Set `RCH__HEADER_SECRET` for API authentication
- [ ] Configured `RCH__HELLO_NAME` with your domain
- [ ] Configured `RCH__FROM_EMAIL` with your email
- [ ] Set up SSL certificates for HTTPS
- [ ] Configured firewall rules
- [ ] Set up database backups
- [ ] Configured log rotation
- [ ] Set up monitoring/alerting
- [ ] Tested bulk API with sample data
- [ ] Documented your specific configuration

## Backup and Restore

### Backup Database

```bash
# Create backup
docker exec reacher-postgres pg_dump -U reacher reacher_db > backup_$(date +%Y%m%d_%H%M%S).sql

# Or backup with compression
docker exec reacher-postgres pg_dump -U reacher reacher_db | gzip > backup_$(date +%Y%m%d_%H%M%S).sql.gz
```

### Restore Database

```bash
# Restore from backup
docker exec -i reacher-postgres psql -U reacher reacher_db < backup.sql

# Or restore from compressed backup
gunzip < backup.sql.gz | docker exec -i reacher-postgres psql -U reacher reacher_db
```

### Automated Backups

Create a cron job for automated backups:

```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * cd /path/to/check-if-email-exists && docker exec reacher-postgres pg_dump -U reacher reacher_db | gzip > backups/backup_$(date +\%Y\%m\%d).sql.gz
```

## Monitoring

### Check Service Health

```bash
# All services status
docker-compose ps

# Check resource usage
docker stats

# Check disk usage
docker system df
```

### View Recent Logs

```bash
# Last 100 lines from backend
docker logs --tail 100 reacher-backend

# Follow logs with timestamps
docker logs -f -t reacher-backend
```

## Additional Resources

- [Reacher Documentation](https://docs.reacher.email)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Nginx Documentation](https://nginx.org/en/docs/)
