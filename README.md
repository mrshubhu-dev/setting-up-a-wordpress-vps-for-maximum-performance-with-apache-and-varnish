# Setting Up a WordPress VPS for Maximum Performance with Apache and Varnish

## Step 1: Update System and Install Dependencies

Update the system and install essential packages.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apache2 mysql-server php php-mysql php-fpm php-curl php-gd php-mbstring php-xml php-zip curl unzip
```

## Step 2: Install and Configure Varnish

Varnish will act as a caching layer to reduce server load and improve response times.

```bash
sudo apt install -y varnish
```

Edit the Varnish configuration to listen on port 80 and point to Apache on port 8080.

```bash
sudo nano /etc/varnish/default.vcl
```

Update the backend configuration:

```vcl
vcl 4.1;

backend default {
    .host = "127.0.0.1";
    .port = "8080";
}
```

Modify the Varnish service to listen on port 80:

```bash
sudo nano /etc/systemd/system/varnish.service
```

Update the `ExecStart` line:

```ini
ExecStart=/usr/sbin/varnishd -a :80 -f /etc/varnish/default.vcl -s malloc,256m
```

Reload systemd and restart Varnish:

```bash
sudo systemctl daemon-reload
sudo systemctl restart varnish
```

## Step 3: Configure Apache

Apache will run on port 8080 to work with Varnish.

Edit the Apache ports configuration:

```bash
sudo nano /etc/apache2/ports.conf
```

Update the `Listen` directive:

```apache
Listen 8080
```

Update the default virtual host:

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

Ensure it listens on port 8080:

```apache
<VirtualHost *:8080>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ...
</VirtualHost>
```

Enable Apache modules for performance:

```bash
sudo a2enmod rewrite expires headers
```

Enable PHP-FPM for better performance:

```bash
sudo apt install -y libapache2-mod-fcgid
sudo a2enmod proxy_fcgi setenvif
sudo a2enconf php8.1-fpm
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

## Step 4: Install and Configure MySQL

Secure MySQL and create a database for WordPress.

```bash
sudo mysql_secure_installation
```

Log in to MySQL and create a database and user:

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE wordpress;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'secure_password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Step 5: Install WordPress

Download and configure WordPress.

```bash
cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xvzf latest.tar.gz
sudo mv wordpress/* .
sudo rm -rf wordpress latest.tar.gz
```

Set permissions:

```bash
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
```

Create a `wp-config.php` file:

```bash
sudo mv wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

Update the database settings:

```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'wp_user');
define('DB_PASSWORD', 'secure_password');
define('DB_HOST', 'localhost');
```

## Step 6: Optimize WordPress for Performance

As i saw in your website "UPSC Folder" WP rocket is installed

- In  WP Rocket Addon you can enable varnish cache module (if not found then you can also move with other plugin for purging varnish cache)
- Alternatively, you can use a WordPress plugin: one of the most installed (and better maintained) is **<u>Proxy Cache Purge.</u>**

Add Varnish-specific rules to `default.vcl` to handle WordPress cookies and dynamic content:

```vcl
sub vcl_recv {
    if (req.url ~ "^/wp-(admin|login|signup|activate)") {
        return (pass);
    }
    if (req.http.Cookie ~ "wp-admin|wordpress_logged_in") {
        return (pass);
    }
    unset req.http.Cookie;
    return (hash);
}

sub vcl_backend_response {
    if (bereq.url !~ "^/wp-(admin|login|signup|activate)") {
        set beresp.ttl = 24h;
        set beresp.grace = 1h;
    }
}
```

Restart Varnish:

```bash
sudo systemctl restart varnish
```

## Step 7: Additional Optimizations

1. **Enable Gzip Compression** in Apache:

```bash
sudo nano /etc/apache2/mods-available/deflate.conf
```

Add:

```apache
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/plain text/xml text/css text/javascript application/javascript
</IfModule>
```

Enable the module and restart Apache:

```bash
sudo a2enmod deflate
sudo systemctl restart apache2
```

2. **Optimize PHP**: Edit the PHP-FPM configuration:

```bash
sudo nano /etc/php/8.1/fpm/php.ini
```

Update:

```ini
memory_limit = 512M
upload_max_filesize = 256M
post_max_size = 256M
max_execution_time = 90
opcache.enable=1
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
```

Restart PHP-FPM:

```bash
sudo systemctl restart php8.1-fpm
```

3. **Install Redis for Object Caching**:

```bash
sudo apt install -y redis-server php-redis
```

Enable Redis and install a plugin like **Redis Object Cache** in WordPress.

4. **Use a CDN**: Integrate a CDN like Cloudflare to cache static assets and reduce latency.

5. **Tune MySQL**: Edit the MySQL configuration:

```bash
sudo nano /etc/mysql/my.cnf
```

Add under `[mysqld]`:

```ini
innodb_buffer_pool_size = 512M
query_cache_type = 1
query_cache_size = 64M
tmp_table_size = 32M
max_connections = 100
```

Restart MySQL:

```bash
sudo systemctl restart mysql
```

## Step 8: Secure the VPS

- Install and configure a firewall (e.g., UFW):

```bash
sudo apt install -y ufw
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

- Install an SSL certificate using Let’s Encrypt:

```bash
sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache
```

- Harden WordPress by adding security plugins like **Wordfence** or **iThemes Security**.
