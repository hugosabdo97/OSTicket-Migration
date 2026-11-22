# osTicket Migration Guide

A comprehensive, step-by-step guide for migrating an **osTicket** instance from an old server (e.g., Atlantic VPS) to a new server (e.g., 8labsid) without requiring a fresh installation.

## 📌 Overview
This migration process covers:
- Database backup & restoration
- Application files backup & extraction
- Apache VirtualHost configuration & port binding
- Nginx Proxy Manager (NPM) integration
- Verification & functional testing

---

## 📦 Step 1: Backup on Old Server (Source VPS)

### 1.1 Backup the Database
```bash
mysqldump -u osticket -p osticket > /root/osticket.sql
```

### 1.2 Backup Application Files & Configurations
```bash
tar czf /root/osticket-files.tar.gz /var/www/osticket
tar czf /root/osticket-vhost.tar.gz /etc/apache2/sites-available/osticket.conf
```

---

## 🔄 Step 2: Transfer Files to New Server (Destination VPS)
Copy the backup archives from the old server to `/root/` on the new server using `scp`:
```bash
scp root@OLD_VPS_IP:/root/osticket*.gz /root/
scp root@OLD_VPS_IP:/root/osticket.sql /root/
```

---

## ⚙️ Step 3: Install Dependencies (Destination VPS)
Install Apache, MariaDB, and all required PHP extensions for osTicket:
```bash
apt update
apt install -y apache2 mariadb-server \
  php php-mysql php-imap php-gd php-mbstring php-xml php-intl php-cli php-curl \
  php-zip php-apcu unzip
systemctl restart apache2
```

---

## 🗄️ Step 4: Restore the Database

### 4.1 Create Database and User
Log into MariaDB/MySQL:
```bash
mysql -u root -p
```

Run the following SQL commands:
```sql
CREATE DATABASE osticket;
CREATE USER 'osticket'@'localhost' IDENTIFIED BY 'StrongPass123!';
GRANT ALL PRIVILEGES ON osticket.* TO 'osticket'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 4.2 Import the SQL Dump
```bash
mysql -u osticket -p osticket < /root/osticket.sql
```

---

## 📁 Step 5: Restore Application Files
Extract the application files to their original directory and set correct ownership permissions for Apache (`www-data`):
```bash
cd /
tar xzf /root/osticket-files.tar.gz
chown -R www-data:www-data /var/www/osticket
```

---

## 🌐 Step 6: Configure Apache VirtualHost

1. Create or place the VirtualHost configuration file at `/etc/apache2/sites-available/osticket.conf`:
```apache
<VirtualHost *:1234>
    ServerName support.yourdomain.com
    DocumentRoot /var/www/osticket/upload

    <Directory /var/www/osticket/upload>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/osticket_error.log
    CustomLog ${APACHE_LOG_DIR}/osticket_access.log combined
</VirtualHost>
```

2. Enable port binding, activate the site, and reload Apache:
```bash
echo "Listen 1234" >> /etc/apache2/ports.conf
a2ensite osticket.conf
systemctl reload apache2
```

3. Test local connectivity:
```bash
curl http://127.0.0.1:1234
```

---

## 🔧 Step 7: Verify osTicket Configuration
Check the database connection settings inside the osTicket configuration file (`/var/www/osticket/upload/include/ost-config.php`):
```bash
cat /var/www/osticket/upload/include/ost-config.php | grep DB
```

Ensure the credentials match your newly restored database:
```php
define('DBHOST', 'localhost');
define('DBNAME', 'osticket');
define('DBUSER', 'osticket');
define('DBPASS', 'StrongPass123!');
```

---

## 🔒 Step 8: Nginx Proxy Manager (NPM) Integration
To expose osTicket securely via HTTPS through Nginx Proxy Manager:

1. Log into your NPM admin dashboard.
2. Go to **Proxy Hosts** -> **Add Proxy Host**.
3. Configure settings:
   - **Domain Names**: `support.yourdomain.com`
   - **Scheme**: `http`
   - **Forward IP / Host**: `<NEW_VPS_IP_OR_HOST>`
   - **Forward Port**: `1234`
   - **Block Common Exploits**: Checked (Enabled)
   - **Websockets Support**: Checked (Enabled)
4. Go to the **SSL** tab:
   - **SSL Certificate**: Request a new Let's Encrypt SSL certificate.
   - **Force SSL**: Checked (Enabled).
5. Save the configuration.

---

## ✅ Step 9: Verification
1. Open your browser and navigate to `https://support.yourdomain.com`.
2. Verify that the osTicket landing page loads correctly via HTTPS.
3. Log in as an administrator to check existing tickets, settings, and attachments.
