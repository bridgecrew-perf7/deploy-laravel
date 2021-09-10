# Install nginx

```
$>apt update -y
$>apt upgrade -y
$>apt install nginx
```

# Install MySQL 8

```
$>cd /tmp
$>curl -LO http://repo.mysql.com/mysql-apt-config_0.8.17-1_all.deb
$>dpkg -i mysql-apt-config_0.8.17-1_all.deb
$>apt-get update
$>apt install -y mysql-server
$>mysql_secure_installation
```

# Remove MySQL

```
apt remove --purge "mysql*" -y
apt autoremove -y
apt autoclean -y
rm -rf /etc/mysql
```

# Create user and enable remote

Add ```bind-address = 0.0.0.0``` to ```/etc/mysql/mysql.conf.d/mysqld.cnf```

```
$>mysql -uroot -p
```

```
CREATE USER 'codebase'@'%' IDENTIFIED WITH mysql_native_password BY '**********'; #C@d3B@s3
GRANT ALL ON *.* TO 'codebase'@'%';
FLUSH PRIVILEGES;
```

```
$>systemctl restart mysql
```

Change password

```
ALTER USER 'codebase'@'%' IDENTIFIED WITH mysql_native_password BY 'C@d3B@s3';
```

Create Database

```
CREATE DATABASE codebase CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```


Config vhost

```
server {
    listen 80;
    server_name example.com;
    root /srv/example.com/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```
