# Nextcloud on AWS — Ubuntu Server Setup Documentation

**Generated:** May 2026 | **Platform:** AWS EC2 — `us-east-1` (N. Virginia)

This repository contains the complete terminal command reference and explanations for deploying a private, highly secure Nextcloud Hub instance on AWS using Ubuntu Server, Nginx, MariaDB, and Tailscale for VPN-only access.

---

## 📦 Installed Versions

| Component | Version | Purpose |
| :--- | :--- | :--- |
| **Ubuntu (OS)** | 24.04.4 LTS (Noble) | Server operating system |
| **Nginx** | 1.24.0 | Web server & reverse proxy |
| **PHP-FPM** | 8.3.6 | PHP application runtime |
| **MariaDB** | 10.11.14 LTS | Relational database server |
| **Nextcloud** | 33.0.3 (Hub 26 Winter) | Private cloud application |
| **Tailscale** | 1.96.4 | VPN mesh network |
| **Redis** | 7.0.15 | In-memory cache & file locking |

> ⚠️ **Security Notice:** Sensitive information including IP addresses, hostnames, usernames, passwords, key pair names, and account identifiers has been intentionally omitted from this document.

---

## 🚦 Critical Deployment Timing: When to Initialize Nextcloud

A major cause of "Code Integrity" errors, "Missing Certificate Keys," and HTTP routing failures is initializing Nextcloud before the underlying infrastructure is strictly configured. 

**Do NOT extract the Nextcloud archive or hit the web installer URL until the following "Pre-Flight" conditions are met:**

1. **Database is Locked:** MariaDB is secured (`mysql_secure_installation`), and the dedicated `nextcloud` database and user are actively mapped.
2. **PHP is Tuned:** The `/etc/php/8.3/fpm/php.ini` file has been updated to support `512M` memory limits and OPcache parameters. If Nextcloud initializes on default PHP settings, background tasks will immediately time out.
3. **SSL is Active:** The Tailscale MagicDNS certificates are generated and correctly placed in `/etc/ssl/certs/`. Initializing Nextcloud over plain HTTP will permanently bake unsecured paths into the database, causing redirect loops later.
4. **Nginx Headers are Strict:** The Nginx virtual host is actively enforcing Strict-Transport-Security (HSTS) and OCM/CalDAV rewrite rules *before* Nextcloud runs its first internal systems check.

**The Golden Rule:** Extract the Nextcloud `.zip` and change the folder permissions (`chown -R www-data:www-data`) as the **absolute final step** before opening your web browser to finish the setup.

---

## 🖥️ Instance Specs

| Field | Value |
| :--- | :--- |
| **Instance Type** | `t3.medium` (2 vCPU, 4 GB RAM) |
| **Storage** | 20 GB EBS gp3 |
| **Region** | `us-east-1` (N. Virginia) |
| **Access Method** | Tailscale MagicDNS over WireGuard VPN |

---

## PHASE 1 — EC2 Instance & SSH Access

### Step 1 — Prepare SSH Key Permissions
Before connecting to EC2, the private key file must have strict permissions. `chmod 400` sets the file to read-only for the owner only. SSH refuses to use keys that are accessible by others as a security measure.

```bash
chmod 400 ~/.ssh/[your-key].pem
```
*ℹ️ The key must be copied into WSL’s own filesystem (`~/.ssh/`) first. `chmod` does not work on Windows-mounted paths (`/mnt/c/...`).*

### Step 2 — Connect to EC2 via SSH
Establishes an encrypted SSH connection to the Ubuntu EC2 instance. The `-i` flag specifies the private key file. All subsequent commands run inside this remote session.
```bash
ssh -i ~/.ssh/[your-key].pem ubuntu@[your-ec2-ip]
```
*ℹ️ Ubuntu 24.04 AMI uses `ubuntu` as the login user, not `admin` or `ec2-user`.*

---

## PHASE 2 — System Update & Reboot

### Step 3 — Update Package Lists & Upgrade System
`apt update` refreshes the local package index. `apt upgrade -y` installs all available security patches and software updates. Always the first step on a fresh server.
```bash
sudo apt update && sudo apt upgrade -y
```
*ℹ️ A new kernel was installed during this step, requiring a reboot.*

### Step 4 — Reboot to Apply New Kernel
After a kernel upgrade, the server must reboot to load the new kernel into memory. 
```bash
sudo reboot
```
*ℹ️ After reboot, reconnect using the same SSH command from Step 2.*

---

## PHASE 3 — Nginx Web Server

### Step 5 — Install Nginx
Nginx handles all incoming HTTP/HTTPS requests. It serves Nextcloud’s static files directly and forwards PHP requests to PHP-FPM. Chosen over Apache for lower memory usage.
```bash
sudo apt install nginx -y
```

### Step 6 — Start & Enable Nginx
Starts Nginx immediately and registers it to auto-start on every reboot.
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

---

## PHASE 4 — PHP 8.3-FPM & Extensions

### Step 7 — Install PHP 8.3-FPM & All Required Extensions
PHP-FPM executes Nextcloud’s PHP application code.
```bash
sudo apt install php8.3-fpm php8.3-mysql php8.3-gd php8.3-curl php8.3-xml \
php8.3-zip php8.3-mbstring php8.3-intl php8.3-bcmath php8.3-gmp \
php8.3-imagick php8.3-redis php8.3-apcu -y
```

### Step 8 — Start & Enable PHP-FPM
```bash
sudo systemctl start php8.3-fpm
sudo systemctl enable php8.3-fpm
sudo systemctl status php8.3-fpm
```

### Step 9 — Tune PHP Configuration
Five key values must be increased from defaults for Nextcloud to function properly.
```bash
sudo nano /etc/php/8.3/fpm/php.ini
```
**Modifications:**
* `memory_limit = 512M`
* `upload_max_filesize = 512M`
* `post_max_size = 512M`
* `max_execution_time = 300`
* `output_buffering = Off`

Restart the service to apply changes:
```bash
sudo systemctl restart php8.3-fpm
```

---

## PHASE 5 — MariaDB Database Server

### Step 10 — Install MariaDB Server & Client
MariaDB stores all Nextcloud metadata. It is a fully open-source MySQL-compatible database.
```bash
sudo apt install mariadb-server mariadb-client -y
```

### Step 11 — Start & Enable MariaDB
```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo systemctl status mariadb
```

### Step 12 — Secure MariaDB Installation
Runs an interactive security hardening script.
```bash
sudo mysql_secure_installation
```
*(Accept defaults to remove anonymous users, disable remote root login, and remove the test database).*

### Step 13 — Create Nextcloud Database & User
Creates a dedicated database with `utf8mb4` character set (required for full Unicode/emoji support).
```bash
sudo mysql -u root -p
```
```sql
CREATE DATABASE [dbname] CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER '[dbuser]'@'localhost' IDENTIFIED BY '[password]';
GRANT ALL PRIVILEGES ON [dbname].* TO '[dbuser]'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## PHASE 6 — Tailscale VPN & MagicDNS

### Step 14 — Install Tailscale
Installs Tailscale — a WireGuard-based mesh VPN. Makes the server completely invisible to the public internet.
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### Step 15 — Authenticate Tailscale
```bash
sudo tailscale up
```
*ℹ️ MagicDNS and HTTPS Certificates must be enabled in the Tailscale admin console under DNS settings before proceeding.*

### Step 16 — Generate Tailscale HTTPS Certificate
Provisions a browser-trusted TLS certificate for the server’s MagicDNS hostname.
```bash
sudo tailscale cert [your-magicdns-hostname]
```

### Step 17 — Move Certificates to System Locations
Moves the certificate and private key to standard system SSL directories and enforces secure access rights.
```bash
sudo mv [hostname].crt /etc/ssl/certs/
sudo mv [hostname].key /etc/ssl/private/
sudo chmod 644 /etc/ssl/certs/[hostname].crt
sudo chmod 600 /etc/ssl/private/[hostname].key
```

---

## PHASE 7 — Nginx Configuration for Nextcloud

### Step 18 — Create Nginx Site Configuration
Creates the production Nginx config including HTTP-to-HTTPS redirect, SSL certificate paths, PHP-FPM upstream, and security headers.
```bash
sudo nano /etc/nginx/sites-available/nextcloud
```

```nginx
upstream php-handler {
    server unix:/run/php/php8.3-fpm.sock;
}

map $arg_v $asset_immutable {
    "" "";
    default ", immutable";
}

server {
    listen 80;
    listen [::]:80;
    server_name [YOUR_TAILSCALE_IP].ts.net;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name [YOUR_TAILSCALE_IP].ts.net;
    server_tokens off;

    ssl_certificate     /etc/ssl/certs/[YOUR_TAILSCALE_IP].ts.net.crt;
    ssl_certificate_key /etc/ssl/private/[YOUR_TAILSCALE_IP].ts.net.key;

    root /var/www/nextcloud;
    index index.php /index.php$request_uri;

    client_max_body_size 512M;
    client_body_timeout 300s;
    client_body_buffer_size 512k;

    fastcgi_buffers 64 4K;
    fastcgi_hide_header X-Powered-By;

    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml text/javascript application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/xhtml+xml application/xml font/opentype image/svg+xml text/css text/plain;

    add_header Referrer-Policy                   "no-referrer"       always;
    add_header X-Content-Type-Options            "nosniff"           always;
    add_header X-Frame-Options                   "SAMEORIGIN"        always;
    add_header X-Permitted-Cross-Domain-Policies "none"              always;
    add_header X-Robots-Tag                      "noindex, nofollow" always;
    add_header X-XSS-Protection                  "1; mode=block"     always;

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location ^~ /.well-known {
        location = /.well-known/carddav { return 301 /remote.php/dav/; }
        location = /.well-known/caldav  { return 301 /remote.php/dav/; }
        location /.well-known/acme-challenge { try_files $uri $uri/ =404; }
        return 301 /index.php$request_uri;
    }

    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) { return 404; }

    location ~ \.php(?:$|/) {
        rewrite ^/(?!index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|ocs-provider\/.+) /index.php$request_uri;
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;
        try_files $fastcgi_script_name =404;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        fastcgi_param HTTPS on;
        fastcgi_param modHeadersAvailable true;
        fastcgi_param front_controller_active true;
        fastcgi_pass php-handler;
        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
        fastcgi_max_temp_file_size 0;
    }

    location ~ \.(?:css|js|mjs|svg|gif|png|jpg|ico|wasm|tflite|map|ogg|flac)$ {
        try_files $uri /index.php$request_uri;
        add_header Cache-Control "public, max-age=15778463$asset_immutable";
        access_log off;
    }

    location ~ \.woff2?$ {
        try_files $uri /index.php$request_uri;
        expires 7d;
        access_log off;
    }

    location /remote {
        return 301 /remote.php$request_uri;
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }

    access_log /var/log/nginx/nextcloud_access.log;
    error_log /var/log/nginx/nextcloud_error.log;
}
```

Enable the site and reload Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

## PHASE 8 — Nextcloud Application Extraction & Deployment

### Step 19 — Install unzip Utility
```bash
sudo apt install unzip -y
```

### Step 20 — Download Nextcloud
Downloads the latest stable Nextcloud release.
```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.zip
```

### Step 21 — Extract & Deploy Nextcloud
Extracts the zip, moves Nextcloud to `/var/www/nextcloud`, and sets the proper web-server ownership and directory permissions. **(This triggers the actual application deployment now that the infrastructure is ready).**
```bash
unzip latest.zip
sudo mv nextcloud /var/www/nextcloud
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chmod -R 755 /var/www/nextcloud
```

*Note: You can now safely navigate to your Tailscale MagicDNS URL in a browser to run the Web Installer.*

---

## PHASE 9 — Post-Installation: Cron, Firewall & Redis

### Step 22 — Set Up Nextcloud Cron Job
Nextcloud requires background tasks to run regularly.
```bash
sudo crontab -u www-data -e
```
Add this line:
```cron
*/5 * * * * php -f /var/www/nextcloud/cron.php
```

### Step 23 — Configure UFW Firewall
Allows SSH and Tailscale traffic, blocking all standard public web traffic for a highly secure perimeter.
```bash
sudo ufw allow ssh
sudo ufw allow in on tailscale0
sudo ufw enable
sudo ufw status
```

### Step 24 — Install Redis & Configure Memcache
Redis provides memory caching (`memcache.local`) and transactional file locking (`memcache.locking`).
```bash
sudo apt install redis-server -y
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

### Step 25 — Add Redis & Performance Settings
```bash
sudo nano /var/www/nextcloud/config/config.php
```
Add the caching parameters:
```php
'memcache.local' => '\OC\Memcache\Redis',
'memcache.locking' => '\OC\Memcache\Redis',
'redis' => [
    'host' => 'localhost',
    'port' => 6379,
],
'maintenance_window_start' => 1,
'default_phone_region' => 'US',
```

### Step 26 — Fix PHP Environment Variables & OPcache

**Fix 1: PHP environment variables**
```bash
sudo nano /etc/php/8.3/fpm/pool.d/www.conf
# Change to: clear_env = no
```

**Fix 2: OPcache interned strings buffer**
```bash
sudo nano /etc/php/8.3/fpm/php.ini
# Change to: opcache.interned_strings_buffer = 16
```

Apply fixes and Run Nextcloud Maintenance Repair:
```bash
sudo systemctl restart php8.3-fpm
sudo -u www-data php /var/www/nextcloud/occ maintenance:repair --include-expensive
```

---

## PHASE 10 — Final Verification

### Step 27 — Verify All Services Are Running
Final health check to confirm all services are active and enabled.
```bash
sudo systemctl status nginx
sudo systemctl status php8.3-fpm
sudo systemctl status mariadb
sudo systemctl status tailscaled
sudo systemctl status redis-server
sudo ufw status
```

---

## Nextcloud Screenshots

**Setting up the Nextcloud admin account**:
![Admin Setup](assets/screenshots/Account%20Admin%20Setup.png)

**Recommended Apps from Nextcloud display**:
![Apps](assets/screenshots/Nextcloud%20Recommended%20Apps.png)

**Recommended Files from Nextcloud display**:
![Files](assets/screenshots/Recommend%20Files.png)

**Security and Setup Warnings from Nextcloud**:
![Warnings](assets/screenshots/Security%20and%20Setup%20Warnings.png)

**Tailscale private certificate secure status display**:
![SSL](assets/screenshots/Tailscale%20private%20certificate.png)
