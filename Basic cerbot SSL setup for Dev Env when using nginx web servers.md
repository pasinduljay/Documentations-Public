# Manual SSL Setup & Renewal for Dockerized NGINX Using Certbot (Ubuntu)

This document outlines how to:

1. Perform a fresh SSL setup using Certbot with a standalone web server.  
2. Manually renew certificates when needed.  
3. Set up automated certificate renewal for NGINX running in Docker, using a safe cron job method.

---

## ✅ 1. Fresh Installation and SSL Certificate Setup

This method is best when NGINX runs inside a Docker container and ports 80/443 are occupied.

### Step 1: Install Certbot

```bash
sudo apt update
sudo apt install certbot
```

### Step 2: Stop the Docker NGINX Container

Certbot needs temporary access to port 80.

```bash
docker stop your-nginx-container-name
```

### Step 3: Issue SSL Certificate (Standalone Mode)

Replace `yourdomain.com` with your actual domain:

```bash
sudo certbot certonly --standalone -d yourdomain.com
```

This will create certificates in:

```bash
/etc/letsencrypt/live/yourdomain.com/
```

### Step 4: Restart Your NGINX Docker Container

```bash
docker start your-nginx-container-name
```

### Step 5: Mount SSL Certificates in Docker Compose

In your `docker-compose.yml`:

```yaml
volumes:
  - /etc/letsencrypt:/etc/letsencrypt:ro
```

Update `nginx.conf` or your site config:

```nginx
ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
```

---

## 🔁 2. Manual SSL Renewal (Without Auto-Renew)

When close to expiration:

### Step 1: Stop NGINX Container

```bash
docker stop your-nginx-container-name
```

### Step 2: Renew Certificate Manually

```bash
sudo certbot renew --force-renewal
```

### Step 3: Restart the Container

```bash
docker start your-nginx-container-name
```

### Optional: Check Certificate Validity

```bash
openssl x509 -in /etc/letsencrypt/live/yourdomain.com/fullchain.pem -noout -dates
```

---

## 🔂 3. Auto-Renew SSL with Docker-Aware Cronjob

This method handles renewal with minimal downtime (<30 seconds) using a script + cronjob.

### Step 1: Create Renewal Script

```bash
sudo nano /usr/local/bin/ssl-renew.sh
```

Paste this (replace `your-nginx-container-name`):

```bash
#!/bin/bash

# Configuration
NGINX_CONTAINER="your-nginx-container-name"
LOG_FILE="/var/log/ssl-renew.log"
EMAIL="your-email@domain.com"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Function to send notification (optional)
send_notification() {
    # Uncomment and configure if you want email notifications
    # echo "$1" | mail -s "SSL Renewal Notification" "$EMAIL"
    log_message "$1"
}

log_message "Starting SSL renewal process"

# Check if container is running
if ! docker ps --format "table {{.Names}}" | grep -q "^${NGINX_CONTAINER}$"; then
    log_message "Warning: NGINX container is not running"
    exit 1
fi

# Stop NGINX container
log_message "Stopping NGINX container: $NGINX_CONTAINER"
if ! docker stop "$NGINX_CONTAINER"; then
    log_message "Error: Failed to stop NGINX container"
    exit 1
fi

# Attempt certificate renewal
log_message "Attempting certificate renewal"
if certbot renew --quiet; then
    log_message "Certificate renewal successful"
    RENEWAL_STATUS="SUCCESS"
else
    log_message "Certificate renewal failed"
    RENEWAL_STATUS="FAILED"
fi

# Start NGINX container
log_message "Starting NGINX container: $NGINX_CONTAINER"
if docker start "$NGINX_CONTAINER"; then
    log_message "NGINX container started successfully"
else
    log_message "Error: Failed to start NGINX container"
    send_notification "CRITICAL: Failed to restart NGINX after SSL renewal"
    exit 1
fi

# Verify container is running
sleep 5
if docker ps --format "table {{.Names}}" | grep -q "^${NGINX_CONTAINER}$"; then
    log_message "NGINX container is running properly"
else
    log_message "Warning: NGINX container may not be running properly"
fi

if [ "$RENEWAL_STATUS" = "SUCCESS" ]; then
    log_message "SSL renewal process completed successfully"
    send_notification "SSL certificates renewed successfully"
else
    log_message "SSL renewal process completed with errors"
    send_notification "SSL renewal failed - manual intervention required"
fi

log_message "SSL renewal process finished"
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/ssl-renew.sh
```

### Step 2: Add Cronjob for Auto-Renewal

Edit the crontab:

```bash
sudo crontab -e
```

Add this line (runs daily at 3 AM):

```cron
0 3 * * * /usr/local/bin/ssl-renew.sh >> /var/log/ssl-renew.log 2>&1
```

### Step 3: Optional – Test Manually

```bash
sudo /usr/local/bin/ssl-renew.sh
```

---

## 🧠 Tips

- Certbot **won’t renew** unless the cert expires in 30 days or less.
- Use `--force-renewal` only for testing, **not in production**.
- The script gracefully stops and starts the container to avoid port conflicts.
- You can monitor the log at: `/var/log/ssl-renew.log`