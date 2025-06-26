# TheSync Deployment Guide

This guide will walk you through deploying TheSync application using Docker Compose with Nginx reverse proxy, Portainer, and Diun for container monitoring.

## Prerequisites

- Docker installed on your server
- Docker Compose installed
- Domain name pointed to your server IP
- Ports 4000 and 9000 available
- SSL certificates from Let's Encrypt or custom certificates

## Quick Start

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd the-sync-deployment
   ```

2. **Prepare SSL certificates (using Let's Encrypt live directory)**

   ```bash
   # Ensure Let's Encrypt certificates exist
   sudo certbot certonly --standalone -d your-domain.com

   # Verify certificates are in place
   ls -la /etc/letsencrypt/live/your-domain.com/
   ```

3. **Create environment files**

   ```bash
   # Backend configuration
   cp .backend.env.example .backend.env
   # Edit .backend.env with your configuration

   # Diun Discord notification configuration
   cp .diun.env.example .diun.env
   # Edit .diun.env with your Discord webhook URL
   ```

4. **Update Nginx configuration**

   ```bash
   # Edit docker/nginx.conf
   # Replace [your-domain] with your actual domain name
   ```

5. **Start all services**

   ```bash
   cd docker
   docker-compose up -d
   ```

6. **Access the services**
   - **TheSync Backend**: `https://your-domain:4000`
   - **Portainer**: `https://your-domain:9000`

## Services Overview

### TheSync Backend

- **Container**: `the-sync-backend`
- **Internal Port**: 4000 (not exposed, accessed via Nginx proxy)
- **Image**: `ghcr.io/5-logic/the-sync-backend:latest`
- **Configuration**: Uses `.backend.env` file
- **Monitoring**: Monitored by Diun for updates
- **Access**: Through Nginx proxy at port 4000

### Nginx Reverse Proxy

- **Container**: `proxy`
- **Ports**:
  - 4000 (Backend proxy with SSL)
  - 9000 (Portainer proxy with SSL)
- **Features**:
  - SSL termination using Let's Encrypt certificates
  - Reverse proxy for backend and Portainer
  - Custom nginx.conf configuration
- **SSL Mount**: Direct mount from `/etc/letsencrypt/live/[yourdomain]`

### Portainer Community Edition

- **Container**: `portainer`
- **Internal Port**: 9000 (not exposed, accessed via Nginx proxy)
- **Features**:
  - Docker container management
  - Web-based interface
  - System monitoring
- **Access**: Through Nginx proxy at port 9000

### Diun (Docker Image Update Notifier)

- **Container**: `diun`
- **Purpose**: Monitor Docker images for updates
- **Schedule**: Checks every 10 minutes
- **Notifications**: Discord webhook notifications
- **Configuration**: Uses `.diun.env` file for Discord settings
- **Target**: Monitors only `the-sync-backend` container

## Configuration Steps

### 1. Environment Variables

Create `.backend.env` file in the docker directory:

```env
# Database Configuration
DATABASE_URL=your_database_url
DB_HOST=your_db_host
DB_PORT=5432
DB_NAME=your_db_name
DB_USER=your_db_user
DB_PASSWORD=your_db_password

# Application Configuration
NODE_ENV=production
PORT=4000
JWT_SECRET=your_jwt_secret
API_KEY=your_api_key

# Other configurations
REDIS_URL=your_redis_url
```

### 2. Diun Configuration

Create `.diun.env` file in the docker directory for Discord notifications:

```env
# Diun Discord Notification Configuration
DIUN_NOTIF_DISCORD_WEBHOOKURL=https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN

# Optional: Additional Discord settings
# DIUN_NOTIF_DISCORD_MENTIONS=@everyone
# DIUN_NOTIF_DISCORD_TIMEOUT=10s
# DIUN_NOTIF_DISCORD_RENDERFIELDS=true
```

**Setting up Discord Webhook:**

1. Go to your Discord server settings
2. Navigate to Integrations → Webhooks
3. Create New Webhook
4. Copy the webhook URL and paste it in `.diun.env`

### 3. SSL Certificate Setup (Let's Encrypt Direct Mount)

The application is configured to use Let's Encrypt certificates directly from the host system.

#### Using Let's Encrypt with Certbot

1. **Install Certbot**

   ```bash
   # Ubuntu/Debian
   sudo apt update
   sudo apt install certbot

   # CentOS/RHEL
   sudo yum install certbot
   ```

2. **Obtain SSL Certificate**

   ```bash
   # Stop any service using port 80
   sudo systemctl stop nginx apache2

   # Request certificate
   sudo certbot certonly --standalone -d your-domain.com

   # Verify certificates are created
   sudo ls -la /etc/letsencrypt/live/your-domain.com/
   ```

3. **Update docker-compose.yml**

   ```bash
   # Edit docker/docker-compose.yml
   # Replace [yourdomain] with your actual domain name in the SSL mount path:
   # - /etc/letsencrypt/live/your-domain.com:/etc/ssl:ro
   ```

4. **Setup Auto-Renewal**

   ```bash
   # Add to crontab
   sudo crontab -e

   # Add this line for renewal every 2 months with container restart
   0 3 1 */2 * certbot renew --quiet --post-hook "docker-compose -f /path/to/your/project/docker/docker-compose.yml restart proxy"
   ```

### 4. Nginx Configuration

Edit `docker/nginx.conf` and replace `[your-domain]` with your actual domain:

```nginx
events {
    worker_connections 1024;
}

http {
    server {
        listen 4000 ssl;
        server_name your-actual-domain.com;

        ssl_certificate /etc/ssl/fullchain.pem;
        ssl_certificate_key /etc/ssl/privkey.pem;

        location / {
            proxy_pass http://backend:4000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 9000 ssl;
        server_name your-actual-domain.com;

        ssl_certificate /etc/ssl/fullchain.pem;
        ssl_certificate_key /etc/ssl/privkey.pem;

        location / {
            proxy_pass http://portainer:9000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### 5. Portainer Setup

1. **Initial Setup**

   - URL: `https://your-domain:9000`
   - Create admin user on first visit
   - Choose "Local" to manage local Docker environment

2. **Connect to Docker**
   - Select "Docker" environment
   - The connection should work automatically via Docker socket

## Container Monitoring with Diun

### Diun Configuration

Diun monitors your containers for image updates and sends notifications:

- **Monitoring Schedule**: Every 10 minutes (`*/10 * * * *`)
- **Target Containers**: Only containers with `diun.enable=true` label
- **Notifications**: Discord webhook notifications
- **Configuration File**: `.diun.env`

### Viewing Diun Logs

```bash
# View Diun logs
docker logs diun

# Follow logs in real-time
docker logs -f diun

# Check if Diun detected any updates
docker logs diun | grep -i "update"
```

### Discord Notifications

When a new image version is available, you'll receive Discord notifications containing:

- Container name
- Current image version
- New image version available
- Registry information
- Timestamp

## SSL Certificate Management

### Let's Encrypt Direct Mount

The application uses direct mount from Let's Encrypt certificates:

- **Certificate Path**: `/etc/letsencrypt/live/[yourdomain]/`
- **Mount Point**: `/etc/ssl` inside nginx container
- **Auto-Renewal**: Via crontab with container restart
- **No Manual Copy**: Certificates are used directly from Let's Encrypt directory

### Certificate Update Process

```bash
# Renew certificates (if not using auto-renewal)
sudo certbot renew

# Restart nginx container to use new certificates
docker-compose restart proxy

# Verify certificate expiration
sudo certbot certificates
```

## Monitoring and Maintenance

### Docker Commands

```bash
# View running containers
docker-compose ps

# View logs
docker-compose logs -f [service-name]

# Restart services
docker-compose restart [service-name]

# Update services
docker-compose pull
docker-compose up -d

# Stop all services
docker-compose down

# Stop and remove volumes (CAUTION: This will delete data)
docker-compose down -v
```

### Health Checks

Check service health:

```bash
# Check if containers are running
docker ps

# Check container logs
docker logs the-sync-backend
docker logs proxy
docker logs portainer
docker logs diun

# Check network connectivity
docker network ls
docker network inspect the-sync_the-sync-network
```

## Backup Strategy

### Important Data to Backup

1. **Let's Encrypt Certificates**

   - Directory: `/etc/letsencrypt/`
   - Contains: All Let's Encrypt certificates and renewal configuration

2. **Nginx Configuration**

   - File: `docker/nginx.conf`
   - Contains: Proxy configurations

3. **Environment Files**

   - File: `docker/.backend.env` - Backend configuration
   - File: `docker/.diun.env` - Discord notification settings

4. **Portainer Data**

   - Volume: `portainer_data`
   - Contains: Portainer configurations

5. **Diun Data**
   - Volume: `diun_data`
   - Contains: Diun monitoring data and history

### Backup Commands

```bash
# Create backup directory
mkdir -p backups/$(date +%Y%m%d)

# Backup Let's Encrypt certificates (requires sudo)
sudo tar czf backups/$(date +%Y%m%d)/letsencrypt_backup.tar.gz -C /etc letsencrypt

# Backup nginx configuration
cp docker/nginx.conf backups/$(date +%Y%m%d)/

# Backup environment files
cp docker/.backend.env backups/$(date +%Y%m%d)/
cp docker/.diun.env backups/$(date +%Y%m%d)/

# Backup docker volumes
docker run --rm -v the-sync_portainer_data:/data -v $(pwd)/backups/$(date +%Y%m%d):/backup alpine tar czf /backup/portainer_data.tar.gz -C /data .
docker run --rm -v the-sync_diun_data:/data -v $(pwd)/backups/$(date +%Y%m%d):/backup alpine tar czf /backup/diun_data.tar.gz -C /data .
```

## Troubleshooting

### Common Issues

1. **Port Already in Use**

   ```bash
   # Check what's using the port
   netstat -tulpn | grep :4000
   netstat -tulpn | grep :9000
   # Stop the conflicting service or change ports in docker-compose.yml
   ```

2. **SSL Certificate Issues**

   - Ensure Let's Encrypt certificates exist: `sudo ls -la /etc/letsencrypt/live/your-domain.com/`
   - Check certificate validity: `sudo openssl x509 -in /etc/letsencrypt/live/your-domain.com/fullchain.pem -text -noout`
   - Verify domain in docker-compose.yml matches your actual domain
   - Check nginx configuration: `docker exec proxy nginx -t`

3. **Discord Notification Issues**

   - Verify Discord webhook URL in `.diun.env`
   - Check Diun logs: `docker logs diun`
   - Test webhook URL manually with curl
   - Ensure Discord webhook permissions are correct

4. **Nginx Configuration Issues**

   - Test nginx configuration: `docker exec proxy nginx -t`
   - Check nginx logs: `docker logs proxy`
   - Verify backend container name in nginx.conf matches docker-compose.yml
   - Ensure SSL certificate paths are correct in nginx.conf

5. **Backend Not Accessible**

   - Check if backend container is running: `docker ps`
   - Check backend logs: `docker logs the-sync-backend`
   - Verify network connectivity between containers
   - Test direct backend access: `docker exec proxy curl http://backend:4000`

6. **Diun Not Working**

   - Check Diun container status: `docker ps | grep diun`
   - Verify `.diun.env` file exists and is properly formatted
   - Check if backend has `diun.enable=true` label
   - Monitor Diun logs for errors: `docker logs -f diun`

7. **Permission Issues**
   - Ensure Docker socket has proper permissions
   - On Linux: `sudo usermod -aG docker $USER`
   - Check Let's Encrypt certificate access permissions

### Log Locations

- **Application Logs**: `docker logs [container-name]`
- **Nginx Access/Error Logs**: `docker logs proxy`
- **System Logs**: `/var/log/` on host system

## Security Considerations

1. **Change Default Passwords**: Always change default credentials
2. **Firewall Configuration**: Only expose necessary ports
3. **Regular Updates**: Keep Docker images updated
4. **SSL Only**: Force HTTPS for all public endpoints
5. **Access Control**: Use strong passwords and consider 2FA
6. **Network Isolation**: Keep backend services in internal network

## Performance Optimization

1. **Resource Limits**: Set appropriate memory/CPU limits
2. **Log Rotation**: Configure log rotation to prevent disk issues
3. **Monitoring**: Use Portainer to monitor resource usage
4. **Cleanup**: Regularly clean up unused images and volumes

```bash
# Clean up Docker system
docker system prune -a
```

## Support

For issues and questions:

- Check container logs first
- Review this documentation
- Check official documentation for each service
- Create an issue in the project repository

---

**Last Updated**: June 26, 2025
