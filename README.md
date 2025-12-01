# OSTicket-Migration
Migrating osTicket from Another VPS

## 📌 Overview
Dokumentasi ini menjelaskan proses migrasi osTicket dari server lama (Atlantic) ke server baru (8labsid) tanpa instalasi ulang.  
Migrasi mencakup:

- Backup database  
- Backup file aplikasi  
- Restore ke server baru  
- Konfigurasi Apache VirtualHost  
- Integrasi dengan Nginx Proxy Manager  
- Uji fungsionalitas  

---

## 📦 1. Backup on Old VPS (Atlantic)

### 1.1 Backup Database

```bash
mysqldump -u osticket -p osticket > /root/osticket.sql


### 1.2 Backup Application Files
tar czf /root/osticket-files.tar.gz /var/www/osticket
tar czf /root/osticket-vhost.tar.gz /etc/apache2/sites-available/osticket.conf


🔄 2. Transfer to New VPS
scp root@OLD_VPS_IP:/root/osticket*.gz /root/
scp root@OLD_VPS_IP:/root/osticket.sql /root/


⚙️ 3. Install Dependencies (New VPS)
apt update
apt install apache2 mariadb-server \
  php php-mysql php-imap php-gd php-mbstring php-xml php-intl php-cli php-curl \
  php-zip php-apcu unzip
systemctl restart apache2


🗄️ 4. Restore Database
4.1 Create Database & User
mysql -u root -p

CREATE DATABASE osticket;
CREATE USER 'osticket'@'localhost' IDENTIFIED BY 'StrongPass123!';
GRANT ALL PRIVILEGES ON osticket.* TO 'osticket'@'localhost';
FLUSH PRIVILEGES;
EXIT;


4.2 Import .sql
mysql -u osticket -p osticket < /root/osticket.sql


📁 5. Restore Application Files
cd /
tar xzf /root/osticket-files.tar.gz
chown -R www-data:www-data /var/www/osticket


🌐 6. Configure Apache VirtualHost
File: /etc/apache2/sites-available/osticket.conf

<VirtualHost *:1234>
    ServerName support.pandoyo.cloud
    DocumentRoot /var/www/osticket/upload

    <Directory /var/www/osticket/upload>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/osticket_error.log
    CustomLog ${APACHE_LOG_DIR}/osticket_access.log combined
</VirtualHost>


Aktifkan site & port:

echo "Listen 1234" >> /etc/apache2/ports.conf
a2ensite osticket.conf
systemctl reload apache2


Test
curl http://127.0.0.1:1234

🔧 7. Verify osTicket Configuration
Check file : /var/www/osticket/upload/include/ost-config.php
can use cat /var/www/osticket/upload/include/ost-config.php | grep DB

define('DBHOST', 'localhost');
define('DBNAME', 'osticket');
define('DBUSER', 'osticket');
define('DBPASS', 'StrongPass123!');


🌍 8. Configure Nginx Proxy Manager (NPM)
Domain: support.pandoyo.cloud
Scheme: http
Forward IP: NEW_VPS_IP
Forward Port: 1234


☑️ 9. Post-Migration Checklist
✔ System Information

Pastikan semua modul PHP aktif:

gd, imap, json, mbstring, intl, zip, APCu, dsb.

✔ Functional Tests

Login admin & agent

Kirim tiket via email (support@, noc@, alerts@)

Balas tiket (SMTP bekerja)

Upload/download attachment

✔ Cron Job
crontab -e


Tambah:

*/5 * * * * /usr/bin/php /var/www/osticket/upload/api/cron.php

📴 10. Disable osTicket on Old VPS
a2dissite osticket.conf
systemctl reload apache2


Opsional:

systemctl stop apache2
systemctl disable apache2

📚 Notes

Pastikan DNS sudah mengarah ke IP VPS baru.

Backup rutin disarankan:

/var/www/osticket/

Database osticket

Warning cgi.fix_pathinfo tidak wajib diubah.

© Pandoyo Cloud Documentation

This guide is maintained as part of the internal documentation.
