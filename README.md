
# INSTALL VARNISH FOR WORDPRESS IN DIGITAL OCEAN / UBUNTU 16.04.2 x64

Frustration with old tutorial, and not working documentation? try this!
In this tutorial we will install latest Wordpress with server spec:
 - PHP 7.0
 - Mysql
 - Varnish
 - NginX

Requirement:
  - Fresh droplet (Ubuntu 16.04.2 x64)


### STEP 1

Install Nginx, php, Mysql, and Varnish

```sh
sudo apt-get update
```

```sh
sudo apt-get install -y nginx mysql-server php php-fpm php-mysql varnish
```

when it ask for the mysql root password, you can type new password.

### STEP 2
Backup default virtualhost file

```sh
mv /etc/nginx/sites-available/default /etc/nginx/sites-available/default_org
```

create new default virtualhost file:
```sh
nano /etc/nginx/sites-available/default
```

copy and paste this, don't forget to change yourdomain.com www.yourdomain.com with your domain.

```sh
server {
        listen 127.0.0.1:8080 default_server;
        listen [::]:8080 default_server;
         root /var/www/html/wordpress;
        index index.php index.html index.htm;
        server_name yourdomain.com www.yourdomain.com;
        location / {
                        try_files $uri $uri/ /index.php?$args;
                }

         location ~ \.php$ {
                                fastcgi_pass unix:/run/php/php7.0-fpm.sock;
                                fastcgi_index index.php;
                                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                                fastcgi_param PATH_INFO $fastcgi_path_info;
                                include fastcgi_params;
                            }

}
```

Restart NginX server

```sh
sudo systemctl restart nginx
```

Check the NginX is listening on port 8080

```sh
netstat -ntulp
```
Output:
``
tcp  -      0    -  0 127.0.0.1:8080      -    0.0.0.0:*       -        LISTEN  -    nginx -g daemo                            
``
### STEP 3
Setting Varnish

```sh
nano /etc/default/varnish
```

Change -a:6081 to -a:80 on Alternative 2, like this:

```sh
DAEMON_OPTS="-a :80 \
             -T localhost:6082 \
             -f /etc/varnish/default.vcl \
             -S /etc/varnish/secret \
             -s malloc,256m"
```

Change -a:6081 to -a:80 in this file too

```sh
nano /lib/systemd/system/varnish.service 
```
like this

```sh
[Unit]
Description=Varnish HTTP accelerator
Documentation=https://www.varnish-cache.org/docs/4.1/ man:varnishd

[Service]
Type=simple
LimitNOFILE=131072
LimitMEMLOCK=82000
ExecStart=/usr/sbin/varnishd -j unix,user=vcache -F -a :80 -T localhost:6082 -f /etc/varnish/default.vcl -S /etc/varnish/secret -s malloc,256m
ExecReload=/usr/share/varnish/reload-vcl
ProtectSystem=full
ProtectHome=true
PrivateTmp=true
PrivateDevices=true

[Install]
WantedBy=multi-user.target
```

Restart nginx, varnish server, and reload daemon.

```sh
sudo /etc/init.d/nginx restart
systemctl daemon-reload
sudo /etc/init.d/varnish restart
```

check varnish listening port 80
```sh
netstat -ntulp
```
Output:
``
tcp       - 0   -   0 0.0.0.0:80       -       0.0.0.0:*       -        LISTEN  -    1510/varnishd 
``

**Nginx must listen port 127.0.0.1:8080 and varnish must port 0.0.0.0:80**


### STEP 4
Download and install Wordpress

```sh
wget http://wordpress.org/latest.tar.gz
sudo tar -xvf latest.tar.gz -C /var/www/html/
```

Change permission of the Wordpress

```sh
sudo chmod -R 755 /var/www/html/wordpress
chown -R www-data:www-data /var/www/html/wordpress
```

### STEP 5
Create database ex. "wordpress_db"
```sh
mysql -u root -p'mysql_root_password' -e "CREATE DATABASE wordpress_db"
```
Create user ex. "wpuser"

```sh
mysql -u root -p'mysql_root_password' -e "CREATE USER wpuser@localhost IDENTIFIED BY 'password';"
```

add user to database
```sh
mysql -u root -p'mysql_root_password' -e "GRANT ALL PRIVILEGES ON wordpress.* TO wpuser@localhost;"
```

flush previleges
```sh
mysql -u root -p'mysql_root_password' -e "FLUSH PRIVILEGES;"
```

### STEP 6
If there is no error, you can access your website proxified by Varnish.
- Open your domain.com and run wordpress installation wizard.
- Install plugins for purge varnish cache (choose one: Varnish HTTP Purge / W3 Total Cache / WPBase Cache)


### STEP 7 
Currently varnish will cache all pages, including wp-admin. To prevent this, we can configuring Varnish Default Configuration.

Backup your default.vcl
```sh
mv /etc/varnish/default.vcl /etc/varnish/default_bak.vcl
```

Create new Default Configuration
```sh
nano /etc/varnish/default.vcl
```

and copy paste this configuration:
``
https://github.com/erickvasilev/wordpress_varnish/blob/master/default.vcl
``

restart to affect changes

```sh
sudo /etc/init.d/nginx restart
systemctl daemon-reload
sudo /etc/init.d/varnish restart
```

If your site is not working properly after this change, that's mean this default.vcl configuration is not suitable for your website, you may modified by yourself. 


