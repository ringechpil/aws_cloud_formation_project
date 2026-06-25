# Deployment Guide — Installing LAMP and WordPress on EC2

This guide walks through manually installing the LAMP stack and WordPress on an Amazon Linux EC2 instance. These steps are also automated in the `UserData` section of `templates/wordpress-server.yaml`.

---

## Prerequisites

- An EC2 instance running Amazon Linux 2023
- SSH access to the instance
- Root or sudo privileges

---

## 1. Update the System

```bash
sudo dnf upgrade -y
```

---

## 2. Install LAMP Stack Packages

```bash
sudo dnf install -y httpd wget php php-devel php-fpm php-mysqlnd php-mysqli \
  php-json php-gd php-mbstring php-xml mariadb105-server
```

This installs:
- **Apache HTTP Server** (`httpd`)
- **PHP** and common extensions
- **MariaDB 10.5** (MySQL-compatible database)
- **wget** and **tar** for downloading WordPress

---

## 3. Start and Enable Services

```bash
sudo systemctl start httpd && sudo systemctl enable httpd
sudo systemctl start mariadb && sudo systemctl enable mariadb
```

---

## 4. Secure MariaDB

Run the security script (interactive — follow prompts):

```bash
sudo mysql_secure_installation
```

Typical prompts:
- Enter current password for root (press Enter if unset)
- Set root password? [Y/n] — **Y**
- Remove anonymous users? — **Y**
- Disallow root login remotely? — **Y**
- Remove test database? — **Y**
- Reload privilege tables? — **Y**

---

## 5. Create the WordPress Database and User

```bash
sudo mysql -u root -p <<EOF
CREATE DATABASE wordpress_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'YourStrongPassword123!';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EOF
```

Replace `YourStrongPassword123!` with a secure password. You'll need it later for `wp-config.php`.

---

## 6. Download and Extract WordPress

```bash
cd ~
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
```

---

## 7. Configure WordPress

```bash
cp wordpress/wp-config-sample.php wordpress/wp-config.php
```

Edit `wp-config.php` with your database details:

```bash
nano wordpress/wp-config.php
```

Change these lines:

```php
define( 'DB_NAME', 'database_name_here' );
define( 'DB_USER', 'username_here' );
define( 'DB_PASSWORD', 'password_here' );
define( 'DB_HOST', 'localhost' );
```

To:

```php
define( 'DB_NAME', 'wordpress_db' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'YourStrongPassword123!' );
define( 'DB_HOST', 'localhost' );
```

Then add the [secret keys](https://api.wordpress.org/secret-key/1.1/salt/) — scroll to the "Authentication Unique Keys and Salts" section, copy the values, and replace the placeholder lines in `wp-config.php`.

---

## 8. Deploy to Apache Web Root

```bash
sudo cp -r wordpress/* /var/www/html/
sudo chown -R apache:apache /var/www/html/
sudo chmod 2775 /var/www
find /var/www -type d -exec sudo chmod 2775 {} \;
find /var/www -type f -exec sudo chmod 0644 {} \;
```

---

## 9. Enable `.htaccess` Overrides

```bash
sudo sed -i '/<Directory "\/var\/www\/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf
```

---

## 10. Restart Apache

```bash
sudo systemctl restart httpd
```

---

## 11. Clean Up

```bash
rm -f ~/latest.tar.gz && rm -rf ~/wordpress/
```

---

## 12. Verify WordPress

Open a browser and navigate to:

```
http://<ec2-instance-public-ip>
```

You should see the WordPress setup wizard. Complete the site title, admin username, password, and email to finish installation.

To verify the automated installation (via UserData), visit:

```
http://<ec2-instance-public-ip>/health.html
```

You should see: **WordPress Installation Completed Successfully**
