# Install GLPI on Debian

This README.md is an operating mode to correctly install GLPI. I advise you to log in as root with sudo -i.

## Installation of Apache2, PHP and MariaDB

First, you can install Apache2, PHP and MariaDB to host your website.

### Download Apache2

```bash
apt update

apt install apache2

systemctl status apache2
```

### Download and installation PHP 8.3
```bash
apt-get update

apt-get -y install lsb-release ca-certificates curl

curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb

dpkg -i /tmp/debsuryorg-archive-keyring.deb

sh -c 'echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'

apt-get update

apt-get install php8.3
```

### Download Extensions PHP 8.3

```bash
apt-get install php-xml php-common php-json php-mysql php-mbstring php-curl php-gd php-intl php-zip php-bz2 php-imap php-apcu

apt-get install php-ldap
```

### Installation MariaDB

```bash
apt install mariadb-server

mysql_secure_installation
```

Now, you need to connect to mysql to create a databse and a user for GLPI :

```bash
mysql -u root -p
```

Write your password and write these lines :

```sql
CREATE DATABASE db_glpi;
GRANT ALL PRIVILEGES ON db_glpi.* TO glpi_adm@localhost IDENTIFIED BY "your_password";
FLUSH PRIVILEGES;
EXIT
```

## Download GLPI

Go in /tmp to download GLPI :

```bash
cd /tmp

wget https://github.com/glpi-project/glpi/releases/download/10.0.X/glpi-10.0.X.tgz #replace 10.0.X with the latest version

tar -xzvf glpi-10.0.X.tgz -C /var/www/html # replace X by your version

chown www-data:www-data /var/www/html/glpi/ -R
```

## Configuration GLPI

To configure correctly GLPI, you need to move the configuration directories out of the public folder :

```bash
mkdir /etc/glpi

mv /var/www/html/glpi/config /etc/glpi

chown www-data:www-data /etc/glpi/ -R

mkdir /var/lib/glpi

mv /var/www/html/glpi/files /var/lib/glpi

chown www-data:www-data /var/lib/glpi/ -R

mkdir /var/log/glpi

chown www-data:www-data /var/log/glpi -R
```

Now, you must create files to indicate new path :

```bash
nano /var/www/html/glpi/inc/downstream.php
```

And write this :

```php
<?php
    define('GLPI_CONFIG_DIR', '/etc/glpi/');
    if (file_exists(GLPI_CONFIG_DIR . '/local_define.php')) {
        require_once GLPI_CONFIG_DIR . '/local_define.php';
}
```

And you must create this file :

```bash
nano /etc/glpi/local_define.php
```

And write this :

```php
<?php
    define('GLPI_VAR_DIR', '/var/lib/glpi/files');
    define('GLPI_LOG_DIR', '/var/log/glpi');
```

## Configuration Apache2

To configurate apache2, you need to create a configuration file (.conf) with your url of your website (example : support.gpsea.fr)

```bash
nano /etc/apache2/sites-available/glpi.support.fr.conf
```
```conf
<VirtualHost *:80>
    ServerName glpi
    ServerAlias glpi.support.fr
    ServerAdmin alertecluster@support.fr
    Redirect / https://glpi.support.fr/
    DocumentRoot "/var/www/html/glpi/public"
    Alias "/glpi" "/var/www/html/glpi/public"

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    # If you want to place GLPI in a subfolder of your site (e.g. your virtual host is serving multiple applications),
    # you can use an Alias directive. If you do this, the DocumentRoot directive MUST NOT target the GLPI directory itself.
    # Alias "/glpi" "/var/www/glpi/public"

    <Directory "/var/www/html/glpi/public">
        Options FollowSymLinks MultiViews
        AllowOverride All
        Require all granted
        RewriteEngine On

        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php [QSA,L]
    </Directory>

    <Directory "/var/www/html/glpi">
        Options FollowSymLinks
        AllowOverride None
        Require all denied
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>

<VirtualHost *:443>
    ServerName glpi
    ServerAdmin alertecluster@support.fr
    DocumentRoot "/var/www/html/glpi/public"

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    SSLEngine on
    SSLCertificateFile      /etc/apache2/SSL2024/certif2024.pem
    SSLCertificateKeyFile /etc/apache2/SSL2024/clecertif2024.key

    ServerSignature Off
#   ServerTokens Prod

    # If you want to place GLPI in a subfolder of your site (e.g. your virtual host is serving multiple applications),
    # you can use an Alias directive. If you do this, the DocumentRoot directive MUST NOT target the GLPI directory itself.
    # Alias "/glpi" "/var/www/glpi/public"

    <Directory "/var/www/html/glpi/public">
        Options FollowSymLinks MultiViews
        AllowOverride All
        Require all granted
        RewriteEngine On

        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php [QSA,L]
    </Directory>

    <Directory "/var/www/html/glpi">
        Options FollowSymLinks
        AllowOverride None
        Require all denied
    </Directory>

    <FilesMatch "\.(cgi|shtml|phtml|php)$">
        SSLOptions +StdEnvVars
    </FilesMatch>

    <Directory /usr/lib/cgi-bin>
        SSLOptions +StdEnvVars
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
```

Now, you can activate your website

```bash
a2ensite glpi.support.fr.conf

a2dissite 000-default.conf

a2enmod rewrite

systemctl restart apache2
```

## Use PHP 8.3-FPM with Apache2

```bash
apt-get install php8.3-fpm

a2enmod proxy_fcgi setenvif

a2enconf php8.3-fpm

systemctl reload apache2
```

You must activate "*session.cookie_httponly*" indicating "on" in this file : 

```bash
nano /etc/php/8.3/fpm/php.ini
```

And restart php8.3-fpm :

```bash
systemctl restart php8.3-fpm.service
```

## Installation GLPI

To finish, you can go in your website and follow installation. After that, you can connect with the account "glpi" and the password "glpi".

You can delete the file of installation :

```bash
rm /var/www/html/glpi/install/install.php
```
