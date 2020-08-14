How to actually set up WordPress on Ubuntu 20 on EC2
1. Launch an EC2 instance with Ubuntu, set up your ssh config, security group, etc. making sure TCP 80, 443, and 22 are exposed.
2. Get an ENI and associate it with the instance, make a DNS A record pointing to it.
3. `ssh -i yourkey.pem ubuntu@yourdomainname.com`
4. Change hostname, install dependencies, enable services after reboot:
```
sudo nano /etc/hostname [edit this to match your domain name]
sudo apt update
sudo apt install mysql-server mysql-client libapache2-mod-php php-cli php-cgi php-mysql php-gd php-bcmath php-curl php-imagick php-mbstring certbot python3-certbot-apache
sudo systemctl enable mysql
sudo systemctl enable apache2
```

5. `sudo mysql_secure_installation`, select the recommended defaults.
6. `sudo mysql -u root`, then in MySQL:
```
create database wordpress;
create user 'wordpress'@'localhost' identified by 'goodpasswordherethatyouwilluseinwp-config.php';
grant all privileges on wordpress.* to wordpress;
flush privileges;
exit;
```
7. Download/extract/configure WordPress:
```
cd ~; wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz 
cd wordpress/
mv wp-config-sample.php wp-config.php
nano wp-config.php 

    - edit DB connection and auth/nonce/salt sections, using link in there for easily-generated values

    - add/edit these lines:
        define( 'WP_DEBUG', true );
        define( 'WP_HTTP_BLOCK_EXTERNAL', false );
        define( 'WP_ACCESSIBLE_HOSTS', 'api.wordpress.org' );
```

8. Move WordPress to its new home and set permissions
```
sudo rm /var/www/html/index.html
sudo cp -r ~/wordpress/* /var/www/html
sudo chown -R www-data:www-data /var/www/html/*
sudo service apache2 restart
```
9. Set up HTTPS
```
sudo apt install certbot
sudo certbot --apache -d yourdomainname.com -d www.yourdomainname.com

    - change `upload_max_filesize` to `1G` in
        /etc/php/[version]/apache2/php.ini
        /etc/php/[version]/cgi/php.ini

sudo service apache2 restart
```
10. Edit these two files by uncommenting the `ServerName` directive and replacing the example with your domain name and adding a `www` alias which will be redirected to the bare domain:
```
/etc/apache2/sites-available/000-default.conf
/etc/apache2/sites-available/000-default-le-ssl.conf

    ServerName yourdomainname.com
    ServerAlias www.yourdomainname.com
```
then edit `/etc/apache2/apache2.conf`, find the block that starts `<Directory /var/www/>`, and change the line inside from `AllowOverride None` to `AllowOverride All`.

12. Set up weekly patches and monthly cert renewals with `sudo crontab -e` and adding these lines to the bottom:
```
10 0  1 * * certbot renew --force-renewal
20 0  * * 1 apt update && apt upgrade -y
```

13. Go to yourdomainname.com/wp-admin and set up WordPress. At some point in there, go to Settings->Permalinkx and change "Common Settings" to "Post name" (to take out the `/index.php/` on every page). Then in your ssh session, `sudo nano /var/www/html/.htaccess` and add contents:
```
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /
    RewriteRule ^index\.php$ - [L]
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule . /index.php [L]
</IfModule>
```

14. Later, once everything's working, go back into `/var/www/html/wp-config.php` and change the line to be `define( 'WP_DEBUG', false );`
