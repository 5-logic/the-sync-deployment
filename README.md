# TheSync Deployment

> Deployment for TheSync application using Docker Compose with Nginx Proxy Manager and Portainer

## Quick Start

```bash
cd docker
docker-compose up -d
```

Access your services:

- **TheSync Backend**: `http://localhost:4000`
- **Nginx Proxy Manager**: `http://localhost:81`
- **Portainer**: `http://localhost:9000`

## Documentation

- [🚀 Deployment Guide](docs/deployment-guide.md) - Complete deployment instructions and configuration
- [🔒 SSL Certificate Setup Guide](docs/ssl-certificate-guide.md) - Guide for obtaining SSL certificates from Let's Encrypt

## Services Included

- **TheSync Backend** - Main application backend
- **Nginx Proxy Manager** - Reverse proxy with automatic SSL
- **Portainer Community** - Docker container management interface
