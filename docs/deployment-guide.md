# TheSync Deployment Guide

This guide will walk you through deploying TheSync application using Docker Compose with Nginx Proxy Manager and Portainer.

## Prerequisites

- Docker installed on your server
- Docker Compose installed
- Domain name pointed to your server IP
- Ports 80, 443, 4000, 9000, and 81 available

## Quick Start

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd the-sync-deployment
   ```

2. **Create environment file**

   ```bash
   cp .backend.env.example .backend.env
   # Edit .backend.env with your configuration
   ```

3. **Start all services**

   ```bash
   cd docker
   docker-compose up -d
   ```

4. **Access the services**
   - **TheSync Backend**: `http://your-domain:4000`
   - **Nginx Proxy Manager**: `http://your-domain:81`
   - **Portainer**: `http://your-domain:9000`

## Services Overview

### TheSync Backend

- **Container**: `the-sync-backend`
- **Port**: 4000
- **Image**: `ghcr.io/5-logic/the-sync-backend:latest`
- **Configuration**: Uses `.backend.env` file

### Nginx Proxy Manager

- **Container**: `nginx-proxy-manager`
- **Ports**:
  - 80 (HTTP)
  - 443 (HTTPS)
  - 81 (Admin Interface)
- **Features**:
  - Automatic SSL certificate generation with Let's Encrypt
  - Web-based configuration
  - Reverse proxy management

### Portainer Community Edition

- **Container**: `portainer`
- **Ports**:
  - 9000 (HTTP Interface)
  - 9443 (HTTPS Interface)
- **Features**:
  - Docker container management
  - Web-based interface
  - System monitoring

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

### 2. Nginx Proxy Manager Setup

1. **Access Admin Panel**

   - URL: `http://your-server-ip:81`
   - Default credentials:
     - Email: `admin@example.com`
     - Password: `changeme`

2. **Change Default Password**

   - Log in and immediately change the default credentials
   - Go to Users → Admin → Edit

3. **Create Proxy Host**

   - Go to "Proxy Hosts" → "Add Proxy Host"
   - **Domain Names**: `your-domain.com`
   - **Scheme**: `http`
   - **Forward Hostname/IP**: `backend` (container name)
   - **Forward Port**: `4000`
   - **Block Common Exploits**: ✓
   - **Websockets Support**: ✓ (if needed)

4. **SSL Certificate**
   - In the same dialog, go to "SSL" tab
   - **SSL Certificate**: "Request a new SSL Certificate"
   - **Force SSL**: ✓
   - **HTTP/2 Support**: ✓
   - **HSTS Enabled**: ✓
   - **Email**: your-email@domain.com
   - **I Agree to the Let's Encrypt Terms of Service**: ✓

### 3. Portainer Setup

1. **Initial Setup**

   - URL: `http://your-server-ip:9000`
   - Create admin user on first visit
   - Choose "Local" to manage local Docker environment

2. **Connect to Docker**
   - Select "Docker" environment
   - The connection should work automatically via Docker socket

## SSL Certificate Management

Nginx Proxy Manager handles SSL certificates automatically:

- **Automatic Renewal**: Certificates are renewed automatically
- **Let's Encrypt Integration**: Built-in ACME client
- **Wildcard Support**: Supports wildcard certificates
- **Multiple Domains**: Can handle multiple domains per certificate

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
docker logs nginx-proxy-manager
docker logs portainer

# Check network connectivity
docker network ls
docker network inspect the-sync_the-sync-network
```

## Backup Strategy

### Important Data to Backup

1. **Nginx Proxy Manager**
   - Volume: `npm_data`
   - Contains: Proxy configurations, user accounts
2. **SSL Certificates**

   - Volume: `npm_letsencrypt`
   - Contains: Let's Encrypt certificates

3. **Portainer Data**

   - Volume: `portainer_data`
   - Contains: Portainer configurations

4. **Backend Environment**
   - File: `.backend.env`
   - Contains: Application configuration

### Backup Commands

```bash
# Create backup directory
mkdir -p backups/$(date +%Y%m%d)

# Backup volumes
docker run --rm -v the-sync_npm_data:/data -v $(pwd)/backups/$(date +%Y%m%d):/backup alpine tar czf /backup/npm_data.tar.gz -C /data .
docker run --rm -v the-sync_npm_letsencrypt:/data -v $(pwd)/backups/$(date +%Y%m%d):/backup alpine tar czf /backup/npm_letsencrypt.tar.gz -C /data .
docker run --rm -v the-sync_portainer_data:/data -v $(pwd)/backups/$(date +%Y%m%d):/backup alpine tar czf /backup/portainer_data.tar.gz -C /data .

# Backup environment file
cp docker/.backend.env backups/$(date +%Y%m%d)/
```

## Troubleshooting

### Common Issues

1. **Port Already in Use**

   ```bash
   # Check what's using the port
   netstat -tulpn | grep :80
   # Stop the conflicting service or change ports in docker-compose.yml
   ```

2. **SSL Certificate Failed**

   - Ensure domain points to your server
   - Check if ports 80 and 443 are accessible from internet
   - Verify DNS propagation: `nslookup your-domain.com`

3. **Backend Not Accessible**

   - Check if backend container is running: `docker ps`
   - Check backend logs: `docker logs the-sync-backend`
   - Verify network connectivity between containers

4. **Permission Issues**
   - Ensure Docker socket has proper permissions
   - On Linux: `sudo usermod -aG docker $USER`

### Log Locations

- **Application Logs**: `docker logs [container-name]`
- **Nginx Proxy Manager**: Access logs through web interface
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
