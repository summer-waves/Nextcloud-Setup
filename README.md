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

## PHASE 6 — Nextcloud Application

### Step 14 — Install unzip Utility
```bash
sudo apt install unzip -y
```

### Step 15 — Download Nextcloud
Downloads the latest stable Nextcloud release.
```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.zip
```

### Step 16 — Extract & Deploy Nextcloud
Extracts the zip, moves Nextcloud to `/var/www/nextcloud`, and sets the proper web-server ownership and directory permissions.
```bash
unzip latest.zip
sudo mv nextcloud /var/www/nextcloud
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chmod -R 755 /var/www/nextcloud
```

---

## PHASE 7 — Tailscale VPN & MagicDNS

### Step 17 — Install Tailscale
Installs Tailscale — a WireGuard-based mesh VPN. Makes the server completely invisible to the public internet.
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### Step 18 — Authenticate Tailscale
```bash
sudo tailscale up
```
*ℹ️ MagicDNS and HTTPS Certificates must be enabled in the Tailscale admin console under DNS settings before proceeding.*

### Step 19 — Generate Tailscale HTTPS Certificate
Provisions a browser-trusted TLS certificate for the server’s MagicDNS hostname.
```bash
sudo tailscale cert [your-magicdns-hostname]
```

### Step 20 — Move Certificates to System Locations
Moves the certificate and private key to standard system SSL directories and enforces secure access rights.
```bash
sudo mv [hostname].crt /etc/ssl/certs/
sudo mv [hostname].key /etc/ssl/private/
sudo chmod 644 /etc/ssl/certs/[hostname].crt
sudo chmod 600 /etc/ssl/private/[hostname].key
```

---

## PHASE 8 — Nginx Configuration for Nextcloud

### Step 21 — Create Nginx Site Configuration
Creates the production Nginx config including HTTP-to-HTTPS redirect, SSL certificate paths, PHP-FPM upstream, and security headers.
```bash
sudo nano /etc/nginx/sites-available/nextcloud
```

*(Insert the full optimized Nginx server block provided in the documentation here).*

Enable the site and reload Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

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

### Step 24 — Run Nextcloud Maintenance Repair
Fixes database collation issues, repairs invalid shares, and clears caches.
```bash
sudo -u www-data php /var/www/nextcloud/occ maintenance:repair --include-expensive
```

### Step 25 — Install Redis
Redis provides memory caching (`memcache.local`) and transactional file locking (`memcache.locking`).
```bash
sudo apt install redis-server -y
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

### Step 26 — Add Redis & Performance Settings
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

### Step 27 — Fix PHP Environment Variables & OPcache

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

Apply fixes:
```bash
sudo systemctl restart php8.3-fpm
```

---

## PHASE 10 — Final Verification

### Step 28 — Verify All Services Are Running
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

## ✅ Final Stack Summary

| # | Component | Version | Status | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **1** | Ubuntu LTS | 24.04.4 | ✅ Active | Server OS |
| **2** | Nginx | 1.24.0 | ✅ Running | Web server / reverse proxy |
| **3** | PHP-FPM | 8.3.6 | ✅ Running | PHP application runtime |
| **4** | MariaDB | 10.11.14 | ✅ Running | Relational database |
| **5** | Nextcloud | 33.0.3 | ✅ Installed | Private cloud application |
| **6** | Tailscale | 1.96.4 | ✅ Connected | VPN / MagicDNS / HTTPS certs |
| **7** | Redis | 7.0.15 | ✅ Running | Memcache & file locking |
| **8** | UFW Firewall | Built-in | ✅ Active | Network traffic control |
| **9** | Cron Job | System | ✅ Active | Background task scheduler |

---

### 🔒 Security Notes
* This Nextcloud instance is accessible exclusively through Tailscale VPN.
* All public ports (80/443) are blocked by UFW.
* Access requires Tailscale to be installed and authenticated on the client device.
* The HTTPS certificate is issued by Tailscale and valid for the MagicDNS hostname only.
* Sensitive identifiers have been intentionally omitted from this document.
