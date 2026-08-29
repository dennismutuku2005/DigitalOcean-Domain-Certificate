# Wildcard SSL Certificate Setup with Certbot and DigitalOcean DNS

This guide outlines the steps to perform a fresh install and configuration of Let's Encrypt wildcard SSL certificates using Certbot and the DigitalOcean DNS plugin.

---

## Complete Guide for domain.com on DigitalOcean

### Step 1: Update Your System

```bash
# Update package lists
sudo apt update

# Upgrade existing packages
sudo apt upgrade -y
```

---

### Step 2: Install Certbot via Snap

```bash
# Install snapd if not already installed
sudo apt install snapd -y

# Install Certbot (classic for full access)
sudo snap install --classic certbot

# Install the DigitalOcean DNS plugin
sudo snap install certbot-dns-digitalocean

# IMPORTANT: Trust the plugin with root access (required for DNS challenges)
sudo snap set certbot trust-plugin-with-root=ok

# Connect the plugin to Certbot
sudo snap connect certbot:plugin certbot-dns-digitalocean

# Create symlink for easy access
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Verify installation
certbot --version
```

**Expected output**: You should see the Certbot version number.

---

### Step 3: Create DigitalOcean API Token

1. **Log into your DigitalOcean account** (https://cloud.digitalocean.com)
2. Go to **API** section (under "Account" in the top menu)
3. Click **"Generate New Token"**
4. Name it: `certbot-wildcard`
5. **IMPORTANT**: Make sure to check **"Write"** permissions (not just Read)
6. Copy the token immediately - you won't see it again!

---

### Step 4: Store Your API Token Securely

```bash
# Create a directory for credentials
sudo mkdir -p /etc/letsencrypt

# Create and edit the credentials file
sudo nano /etc/letsencrypt/digitalocean.ini
```

**In the nano editor**, add this line (replace with your actual token):
```ini
dns_digitalocean_token = your-digitalocean-api-token-here
```

**Save and exit**: Press `Ctrl+X`, then `Y`, then `Enter`

Secure the file:
```bash
# Restrict permissions so only root can read it
sudo chmod 600 /etc/letsencrypt/digitalocean.ini
```

---

### Step 5: Generate Your Wildcard Certificate

Run this command to get your certificate:

```bash
sudo certbot certonly \
  --dns-digitalocean \
  --dns-digitalocean-credentials /etc/letsencrypt/digitalocean.ini \
  -d domain.com \
  -d *.domain.com \
  --agree-tos \
  --non-interactive \
  --email admin@domain.com
```

**Expected output**: 
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator dns-digitalocean, Installer None
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for domain.com
dns-01 challenge for domain.com
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/domain.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/domain.com/privkey.pem
   Your cert will expire on [date]
```

---

### Step 6: Setup Auto-Renewal with Nginx Reload

This ensures your certificates renew automatically:

```bash
# Create the renewal hook directory
sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy

# Create the reload script
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh > /dev/null << 'EOF'
#!/bin/bash
set -e
if systemctl is-active --quiet nginx; then
    echo "Reloading Nginx to apply new certificate"
    systemctl reload nginx
else
    echo "Nginx is not running, skipping reload"
fi
EOF

# Make it executable
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

---

### Step 7: Install and Configure Nginx (If Not Already Installed)

```bash
# Install Nginx if not already installed
sudo apt install nginx -y

# Create your website directory
sudo mkdir -p /var/www/domain.com/html

# Create a test index page
echo "<h1>Welcome to domain.com</h1>" | sudo tee /var/www/domain.com/html/index.html

# Create and edit your Nginx configuration
sudo nano /etc/nginx/sites-available/domain.com
```

Add this configuration:

```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    server_name domain.com *.domain.com;
    return 301 https://$host$request_uri;
}

# HTTPS configuration
server {
    listen 443 ssl http2;
    server_name domain.com *.domain.com;

    # SSL certificates
    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
    
    # SSL configuration (provided by Certbot)
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Root directory
    root /var/www/domain.com/html;
    index index.html index.htm;

    # Main location block
    location / {
        try_files $uri $uri/ =404;
    }

    # Example: If you have a subdomain app running on port 3000
    # location /app/ {
    #     proxy_pass http://localhost:3000;
    #     proxy_set_header Host $host;
    #     proxy_set_header X-Real-IP $remote_addr;
    #     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #     proxy_set_header X-Forwarded-Proto $scheme;
    # }
}
```

Enable the site and test:

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/domain.com /etc/nginx/sites-enabled/

# Remove default site (optional)
sudo rm /etc/nginx/sites-enabled/default 2>/dev/null || true

# Test the configuration
sudo nginx -t

# If test passes, reload Nginx
sudo systemctl reload nginx
```

---

### Step 8: Test the Auto-Renewal

Test everything works without actually renewing:

```bash
sudo certbot renew --dry-run
```

**Expected output**: 
```
** DRY RUN: simulating 'certbot renew' close to cert expiry
** All renewal failures were logged in: /var/log/letsencrypt/letsencrypt.log
** The following certificates are not due for renewal yet:
**   /etc/letsencrypt/live/domain.com/fullchain.pem (skipped)
** No renewals were attempted.
** Congratulations, all simulated renewals succeeded:
```

---

### Step 9: Verify Everything is Working

```bash
# Check the certificate info
sudo certbot certificates

# Check if the renewal timer is active
sudo systemctl list-timers | grep certbot

# Check the status of the renew timer
sudo systemctl status snap.certbot.renew.timer
```

---

## Quick Reference Commands

| Task | Command |
|------|---------|
| Check certificate status | `sudo certbot certificates` |
| Force immediate renewal | `sudo certbot renew --force-renewal` |
| Test renewal (dry run) | `sudo certbot renew --dry-run` |
| Check Nginx config | `sudo nginx -t` |
| Reload Nginx | `sudo systemctl reload nginx` |
| View Certbot logs | `sudo tail -f /var/log/letsencrypt/letsencrypt.log` |

---

## Certificate Files Location

After successful installation, your certificates will be here:
```
/etc/letsencrypt/live/domain.com/
├── cert.pem          # Your certificate
├── chain.pem         # Let's Encrypt chain
├── fullchain.pem     # cert.pem + chain.pem (use this)
├── privkey.pem       # Your private key (keep secure)
└── README            # Instructions
```

---

## Important Notes

1. **API Token Security**: Your token has write permissions, so keep it secure.
2. **Wildcard Coverage**: `*.domain.com` covers all subdomains (e.g., `app.domain.com`, `api.domain.com`, `www.domain.com`).
3. **Email Notifications**: Check the email you provided for expiry warnings (though auto-renewal handles this).
4. **Rate Limits**: Let's Encrypt has rate limits, but auto-renewal respects them.

---

## Troubleshooting

**If you get "unrecognized arguments: --dns-digitalocean":**
```bash
# Make sure the plugin is installed and connected
sudo snap list | grep certbot
sudo snap connections certbot-dns-digitalocean
sudo snap connect certbot:plugin certbot-dns-digitalocean
```

**If you get "Invalid credentials" error:**
- Ensure your API token has **Write** permissions.
- Verify you copied the full token correctly.

**If DNS propagation fails:**
- DigitalOcean usually propagates within 30-60 seconds.
- Check with: `dig _acme-challenge.domain.com TXT`

**If Nginx won't reload:**
```bash
sudo nginx -t  # Check for errors
sudo systemctl status nginx  # Check service status
```

**To see detailed logs:**
```bash
sudo journalctl -u snap.certbot.renew.service -f
```

---

## Success

Your wildcard SSL certificate for `domain.com` and all subdomains is now set up with full auto-renewal. The system will handle everything automatically every 90 days.

**To verify it's working**: Visit `https://domain.com` in your browser and check that the padlock icon shows the certificate is valid!