# Firecrawl Promoflix - Fly.io Deployment Guide

This document provides comprehensive guidelines for deploying the Firecrawl Promoflix application on Fly.io.

## Prerequisites

1. **Fly.io Account**: Sign up at [fly.io](https://fly.io)
2. **Fly CLI**: Install the Fly CLI tool
   ```bash
   # macOS
   brew install flyctl
   
   # Linux/WSL
   curl -L https://fly.io/install.sh | sh
   
   # Windows
   iwr https://fly.io/install.ps1 -useb | iex
   ```
3. **Docker**: Ensure Docker is installed and running
4. **Git**: For version control and deployment

## Initial Setup

### 1. Login to Fly.io
```bash
fly auth login
```

### 2. Initialize Fly.io App
```bash
# In your project root directory
fly launch
```

This will:
- Create a `fly.toml` configuration file
- Set up your app name and region
- Configure basic settings

### 3. Configure fly.toml
Ensure your `fly.toml` file includes the following configuration:

```toml
app = "firecrawl-promoflix"
primary_region = "bos"

[build]

[http_service]
  internal_port = 3002
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 0
  processes = ["app"]

[[vm]]
  memory = "1gb"
  cpu_kind = "shared"
  cpus = 1

[[http_service.checks]]
  interval = "15s"
  grace_period = "5s"
  method = "GET"
  path = "/"
  protocol = "http"
  timeout = "10s"
  tls_skip_verify = false
```

## Redis Setup (Optional)

If you need Redis for rate limiting and caching:

### 1. Create Upstash Redis Instance
1. Go to [Upstash Console](https://console.upstash.com/)
2. Create a new Redis database
3. Note down the connection details

### 2. Set Redis Environment Variables
```bash
# Replace with your actual Upstash Redis credentials
fly secrets set REDIS_URL="rediss://default:YOUR_PASSWORD@your-redis-host.upstash.io:6379"
fly secrets set REDIS_RATE_LIMIT_URL="rediss://default:YOUR_PASSWORD@your-redis-host.upstash.io:6379"
```

**Important**: Use `rediss://` (with TLS) for secure connections to Upstash Redis.

## Environment Variables

Set up your environment variables using Fly secrets:

```bash
# Required environment variables
fly secrets set NODE_ENV="production"
fly secrets set PORT="3002"

# Optional: API keys and other configuration
fly secrets set OPENAI_API_KEY="your-openai-key"
fly secrets set ANTHROPIC_API_KEY="your-anthropic-key"

# Database configuration (if using)
fly secrets set DATABASE_URL="your-database-url"
```

### View Current Secrets
```bash
fly secrets list
```

### Remove Secrets
```bash
fly secrets unset SECRET_NAME
```

## Dockerfile Requirements

Ensure your `Dockerfile` is optimized for production:

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Expose port
EXPOSE 3002

# Start the application
CMD ["npm", "start"]
```

## Deployment Process

### 1. Deploy the Application
```bash
fly deploy
```

### 2. Monitor Deployment
```bash
# View logs
fly logs

# Check app status
fly status

# View machines
fly machine list
```

### 3. Scale if Needed
```bash
# Scale to specific number of machines
fly scale count 2

# Scale memory
fly scale memory 2gb
```

## Post-Deployment

### 1. Verify Application
```bash
# Open your app in browser
fly open

# Check health
fly status
```

### 2. Set up Custom Domain (Optional)
```bash
# Add custom domain
fly certs create your-domain.com

# Add DNS records as instructed by Fly.io
```

## Troubleshooting

### Common Issues

#### 1. Redis Connection Errors
- **Problem**: `ENOTFOUND` or `ECONNRESET` errors
- **Solution**: 
  - Ensure you're using `rediss://` for TLS connections
  - Verify Redis credentials are correct
  - Check Upstash connection limits

#### 2. Application Won't Start
- **Problem**: App crashes on startup
- **Solution**:
  ```bash
  # Check logs for errors
  fly logs
  
  # Verify environment variables
  fly secrets list
  
  # Check machine status
  fly machine list
  ```

#### 3. Memory Issues
- **Problem**: Out of memory errors
- **Solution**:
  ```bash
  # Increase memory allocation
  fly scale memory 2gb
  ```

#### 4. Port Configuration
- **Problem**: Service unreachable
- **Solution**: Ensure `internal_port` in `fly.toml` matches your app's port (3002)

### Debugging Commands

```bash
# SSH into your machine
fly ssh console

# View detailed logs
fly logs --no-tail

# Restart application
fly apps restart

# Check machine health
fly checks list
```

## Best Practices

### 1. Environment Management
- Use Fly secrets for sensitive data
- Keep development and production environments separate
- Document all required environment variables

### 2. Monitoring
- Regularly check application logs
- Set up health checks in `fly.toml`
- Monitor resource usage

### 3. Security
- Always use TLS for external connections
- Keep dependencies updated
- Use minimal Docker images

### 4. Performance
- Optimize Docker image size
- Configure appropriate memory/CPU allocation
- Use connection pooling for databases

### 5. Backup and Recovery
- Regularly backup your data
- Test deployment process in staging
- Keep deployment documentation updated

## Useful Commands Reference

```bash
# Deployment
fly deploy                    # Deploy application
fly deploy --no-cache        # Deploy without Docker cache

# Management
fly apps list                # List all apps
fly apps destroy APP_NAME    # Delete an app
fly apps restart             # Restart application

# Monitoring
fly logs                     # View logs (live)
fly logs --no-tail          # View recent logs
fly status                  # Check app status
fly machine list           # List machines

# Scaling
fly scale count 2           # Scale to 2 machines
fly scale memory 1gb        # Set memory to 1GB
fly scale show              # Show current scaling

# Secrets
fly secrets list            # List all secrets
fly secrets set KEY=value   # Set secret
fly secrets unset KEY       # Remove secret

# Networking
fly ips list                # List IP addresses
fly certs list              # List certificates
fly open                    # Open app in browser
```

## Support

- **Fly.io Documentation**: [https://fly.io/docs/](https://fly.io/docs/)
- **Fly.io Community**: [https://community.fly.io/](https://community.fly.io/)
- **Project Issues**: Create an issue in the project repository

---

**Note**: This deployment guide is specific to the Firecrawl Promoflix application. Adjust configurations based on your specific requirements and use case.
