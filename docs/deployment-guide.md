# TheSync Deployment Guide

This guide will walk you through deploying TheSync application using Docker Compose with Nginx reverse proxy and Portainer.

## Prerequisites

- Docker installed on your server
- Docker Compose installed
- Domain name pointed to your server IP
- Ports 4000 and 9000 available
- SSL certificates (Let's Encrypt or custom)

## Quick Start

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd the-sync-deployment
   ```

2. **Prepare SSL certificates**

   ```bash
   # Create ssl directory
   mkdir -p docker/ssl

   # Copy your SSL certificates to docker/ssl/
   # Required files:
   # - fullchain.pem (certificate + intermediate)
   # - privkey.pem (private key)
   ```

3. **Create environment file**

   ```bash
   cp .backend.env.example .backend.env
   # Edit .backend.env with your configuration
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
- **Internal Port**: 4000
- **Image**: `ghcr.io/5-logic/the-sync-backend:latest`
- **Configuration**: Uses `.backend.env` file
- **Access**: Through Nginx proxy at port 4000

### Nginx Reverse Proxy

- **Container**: `proxy`
- **Ports**:
  - 4000 (Backend proxy)
  - 9000 (Portainer proxy)
- **Features**:
  - SSL termination
  - Reverse proxy for backend and Portainer
  - Custom nginx.conf configuration

### Portainer Community Edition

- **Container**: `portainer`
- **Internal Port**: 9000
- **Features**:
  - Docker container management
  - Web-based interface
  - System monitoring
- **Access**: Through Nginx proxy at port 9000

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

### 2. SSL Certificate Setup

#### Option 1: Using Let's Encrypt with Certbot

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
   sudo systemctl stop nginx

   # Request certificate
   sudo certbot certonly --standalone -d your-domain.com

   # Copy certificates to project directory
   sudo cp /etc/letsencrypt/live/your-domain.com/fullchain.pem docker/ssl/
   sudo cp /etc/letsencrypt/live/your-domain.com/privkey.pem docker/ssl/
   sudo chown $USER:$USER docker/ssl/*.pem
   ```

3. **Setup Auto-Renewal**

   ```bash
   # Add to crontab
   sudo crontab -e

   # Add this line for renewal every 2 months
   0 3 1 */2 * certbot renew --quiet --post-hook "cp /etc/letsencrypt/live/your-domain.com/fullchain.pem /path/to/your/project/docker/ssl/ && cp /etc/letsencrypt/live/your-domain.com/privkey.pem /path/to/your/project/docker/ssl/ && docker-compose -f /path/to/your/project/docker/docker-compose.yml restart proxy"
   ```

#### Option 2: Using Custom SSL Certificates

If you have custom SSL certificates:

```bash
# Copy your certificates to the ssl directory
cp your-certificate.crt docker/ssl/fullchain.pem
cp your-private-key.key docker/ssl/privkey.pem
```

### 3. Nginx Configuration

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

### 4. Portainer Setup

1. **Initial Setup**

   - URL: `https://your-domain:9000`
   - Create admin user on first visit
   - Choose "Local" to manage local Docker environment

2. **Connect to Docker**
   - Select "Docker" environment
   - The connection should work automatically via Docker socket

## SSL Certificate Management

### Manual Certificate Management

Since we're using Nginx with custom SSL certificates:

- **Manual Renewal**: You need to renew certificates manually or via cron job
- **Let's Encrypt**: Use Certbot as shown in setup section
- **Custom Certificates**: Replace files in `docker/ssl/` directory
- **Restart Required**: Restart nginx container after certificate updates

### Certificate Update Process

```bash
# Update certificates
sudo cp /etc/letsencrypt/live/your-domain.com/fullchain.pem docker/ssl/
sudo cp /etc/letsencrypt/live/your-domain.com/privkey.pem docker/ssl/

# Restart nginx container
docker-compose restart proxy
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

# Check network connectivity
docker network ls
docker network inspect the-sync_the-sync-network
```

## Backup Strategy

### Important Data to Backup

1. **SSL Certificates**

   - Directory: `docker/ssl/`
   - Contains: fullchain.pem, privkey.pem

2. **Nginx Configuration**

   - File: `docker/nginx.conf`
   - Contains: Proxy configurations

3. **Portainer Data**

   - Volume: `portainer_data`
   - Contains: Portainer configurations

4. **Backend Environment**
   - File: `docker/.backend.env`
   - Contains: Application configuration

### Backup Commands

```bash
# Create backup directory
mkdir -p backups/$(date +%Y%m%d)

# Backup SSL certificates
cp -r docker/ssl/ backups/$(date +%Y%m%d)/

# Backup nginx configuration
cp docker/nginx.conf backups/$(date +%Y%m%d)/

# Backup portainer volume
docker run --rm -v the-sync_portainer_data:/data -v $(pwd)/backups/$(date +%Y%m%d):/backup alpine tar czf /backup/portainer_data.tar.gz -C /data .

# Backup environment file
cp docker/.backend.env backups/$(date +%Y%m%d)/
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

   - Ensure certificates exist in `docker/ssl/` directory
   - Check certificate permissions: `ls -la docker/ssl/`
   - Verify certificate validity: `openssl x509 -in docker/ssl/fullchain.pem -text -noout`
   - Check domain in nginx.conf matches your actual domain

3. **Nginx Configuration Issues**

   - Test nginx configuration: `docker exec proxy nginx -t`
   - Check nginx logs: `docker logs proxy`
   - Verify backend container name in nginx.conf matches docker-compose.yml

4. **Backend Not Accessible**

   - Check if backend container is running: `docker ps`
   - Check backend logs: `docker logs the-sync-backend`
   - Verify network connectivity between containers
   - Test direct backend access: `docker exec proxy curl http://backend:4000`

5. **Permission Issues**
   - Ensure Docker socket has proper permissions
   - On Linux: `sudo usermod -aG docker $USER`
   - Check SSL certificate file permissions

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
