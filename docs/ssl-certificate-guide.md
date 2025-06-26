# SSL Certificate Setup Guide with Let's Encrypt

This guide walks you through the process of obtaining and installing SSL certificates from Let's Encrypt using Certbot and Nginx.

## Prerequisites

- Ubuntu/Debian server with Nginx installed
- Domain name pointing to your server
- Root or sudo access to the server

## Step-by-Step Instructions

### Step 1: Install Certbot & Nginx Plugin

First, update your package list and install Certbot with the Nginx plugin:

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
```

**Description:** This installs Certbot and the Nginx plugin needed to issue and install SSL certificates automatically.

### Step 2: Obtain SSL Certificate

Run the following command to obtain and install the SSL certificate for your domain:

```bash
sudo certbot --nginx -d thesync-server.eastus.cloudapp.azure.com
```

**What this does:**

- Validates domain ownership
- Generates certificate files in `/etc/letsencrypt/live/[your-domain]/`
- Automatically configures Nginx to use the certificates
- Sets up HTTPS redirect from HTTP

## Important Notes

⚠️ **Certificate Management:**

- No automatic renewal is configured with this manual setup
- Certificate validity lasts up to 90 days
- To renew later, manually re-run the certbot command above to replace the certificate

## Certificate Files Location

After successful installation, your certificates will be located at:

```
/etc/letsencrypt/live/thesync-server.eastus.cloudapp.azure.com/
├── cert.pem
├── chain.pem
├── fullchain.pem
└── privkey.pem
```

## Manual Renewal Process

When your certificate is close to expiration (recommended: 30 days before), run:

```bash
sudo certbot --nginx -d thesync-server.eastus.cloudapp.azure.com
```

## Verification

After installation, verify your SSL certificate by:

1. Visiting `https://thesync-server.eastus.cloudapp.azure.com` in your browser
2. Check that the connection is secure (lock icon in address bar)
3. Test using online SSL checkers like SSL Labs

## Troubleshooting

**Common Issues:**

- **Domain not reachable:** Ensure DNS is properly configured and pointing to your server
- **Nginx configuration error:** Check nginx configuration with `sudo nginx -t`
- **Port 80/443 not accessible:** Verify firewall settings allow HTTP and HTTPS traffic

**Check certificate status:**

```bash
sudo certbot certificates
```

**Test Nginx configuration:**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

_Note: This guide covers manual certificate management. For production environments, consider setting up automatic renewal using cron jobs or systemd timers._
