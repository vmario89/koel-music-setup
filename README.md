# About
Due to the lack of good description on how to setup Koel (Version 7.2.2) i decided to publish this manual because the installation of Koel required some hours for myself. I hope you will come to a result quicker than me with this cooking recipe ;-)

## Install FFmpeg
```
sudo apt install ffmpeg
```

## Install PostgreSQL
```
sudo apt install postgresql
```

Set it up (create db + user):
```
su - postgres bash -c "createuser --no-superuser --pwprompt koel";
su - postgres bash -c "createdb -O koel -E UTF8 -T template0 koel";
su - postgres bash -c 'psql -d koel -c "GRANT ALL PRIVILEGES ON DATABASE koel TO koel;"'
```

## Instal Redis Caching Server
```
sudo apt install redis
systemctl status redis-server
```

## Install PHP requirements (we explicitely use 8.5)
```
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install php8.5 php8.5-xml php8.5-mbstring php8.5-curl php8.5-zip php8.5-pgsql php8.5-gd php8.5-bcmath php8.5-intl php8.5-sqlite3 php8.5-cli php8.5-redis php8.5-fpm
```

## Adjustments to php.ini (FPM)
```
post_max_size = 50M
upload_max_filesize = 50M
```

## Setup PHP Composer
```
cd /tmp/
wget -qO composer-setup.php https://getcomposer.org/installer
COMPOSER_HASH=`wget -qO - https://composer.github.io/installer.sig`
php -r "if (hash_file('SHA384', 'composer-setup.php') === '$COMPOSER_HASH') { echo 'Installationsskript ist in Ordnung.'; } else { echo 'ACHTUNG: Das Installationsskript ist FEHLERHAFT.'; unlink('composer-setup.php'); } echo PHP_EOL;"

php8.4 composer-setup.php --install-dir=/usr/local/bin/ --filename=composer
```

## Install NPM + pnpm (previously it was yarn) using "n"
```
curl -fsSL https://raw.githubusercontent.com/tj/n/master/bin/n | bash -s install lts
npm install -g n
#npm install --global yarn
npm install --global pnpm

```

## Clone Repository
```
cd /var/www/vhosts/
git clone https://github.com/phanan/koel.git koel.myserver.net
cd koel.myserver.net/
git checkout tags/v7.2.2
```

## Compile the project
```
/usr/bin/php8.4 /usr/local/bin/composer install --optimize-autoloader --no-dev
```

## Init Koel
```
php8.4 artisan koel:init
```

## Configure .env file
```
vim /var/www/vhosts/koel.myserver.net/.env
```

## The default user credentials are:
user: `admin@koel.dev`
password: `KoelIsCool`

## Adjust permissions
```
chown -R www-data:www-data /var/www/vhosts/koel.myserver.net
```

## Adjustments to Manifest
We create that file. See the provided example file in same dir.
```
vim /var/www/vhosts/koel.myserver.net/public/manifest.json
```

## Give a test
Later we use nginx, so forget about this
```
php8.4 artisan serve --host=127.0.0.1 --port=8000
```

## Logs
Logs can be found at
```
less /var/www/vhosts/koel.myserver.net/storage/log/laravel.log
```
## nginx Web Server
Config sample can be found in Koel directory (the official one is not complete). We use:

```
server {
    server_name koel.myserver.net;
    listen 443 ssl;
    error_log /var/log/nginx/koel.myserver.net.error.log;

	ssl_certificate /etc/letsencrypt/live/wildcard.myserver.net/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/wildcard.myserver.net/privkey.pem;
	ssl_session_timeout 5m;
	ssl_session_cache   shared:SSL:10m;
	ssl_session_tickets off;
	ssl_protocols TLSv1.3;
	ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256;
	ssl_prefer_server_ciphers off;
	ssl_dhparam /etc/ssl/dhparamMozilla.pem; # Your web server supports insufficiently secure parameters for Diffie-Hellman key exchange. 

	index index.php;
	root /var/www/vhosts/koel.myserver.net/public;

    gzip            on;
    gzip_types      text/plain text/css application/x-javascript text/xml application/xml application/xml+rss text/javascript application/json;
    gzip_comp_level  9;
    send_timeout    3600;
    client_max_body_size  150M;

	location / {
		try_files $uri $uri/ /index.php?$query_string;
	}

    location /media/ {
        internal;
        alias $upstream_http_x_media_root;
        error_log /var/log/nginx/koel.myserver.net.error.log;
    }
	
    location ~ \.php$ {
        try_files $uri $uri/ /index.php?$args;

        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param PATH_TRANSLATED $document_root$fastcgi_path_info;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

		fastcgi_pass unix:/var/run/php/php8.4-fpm.sock;
        fastcgi_index index.php;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_intercept_errors on;
        include  fastcgi_params;
  }

}
 
server {
    listen 80;
    server_name koel.myserver.net;
    location / {
        return 301 https://koel.myserver.net$request_uri;
    }
}
```

## Give some last config checks
```
php8.4 artisan koel:doctor
```

## Create cache files for performance
See https://laravel.com/docs/10.x/deployment#server-requirements
```
php8.4 artisan config:cache
php8.4 artisan event:cache
php8.4 artisan route:cache
php8.4 artisan view:cache
```

After each .env file change, we have to rerun `php8.4 artisan config:cache`

## Fix "too many attempts" in user interface
```
vim /var/www/vhosts/koel.myserver.net/app/Http/Kernel.php
```
We change 
`'throttle:60,1',` to `'throttle:60000,1',`

```
sed -i "s/throttle:60,1/throttle:60000,1/" /var/www/vhosts/koel.myserver.net/app/Http/Kernel.php
```

## Upload music and listen
If you have a lot of music, you can sync it with rsync
```
rsync --remove-source-files -va /home/myuser/music/ user@myserver:/mnt/music/koel/ --progress
```

# Upgrade
```
#!/bin/bash

LATEST_RELEASE="8.1.0"

read -p "$LATEST_RELEASE korrekt?" -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
	BUP="/backup"
	BUP_POSTGRES="$BUP"/db/postgres
	mkdir -p $BUP_POSTGRES > /dev/null 2>&1

	TGT="/var/www/vhosts/koel.myserver.net"

	#db backup
	echo "Backing up pgsql db"
	KOEL_DB=koel_$(date +"%m-%d-%Y").sql
	sudo -iu postgres bash -c "pg_dump koel > $KOEL_DB" && mv /var/lib/postgresql/$KOEL_DB "$BUP_POSTGRES"/$KOEL_DB

	#backup directory
	echo "Backing up directory"
	rm -rf $TGT.bup > /dev/null 2>&1
	cp -R $TGT $TGT.bup

	#upgrade
	cd $TGT
	git pull
	git stash
	git checkout v$LATEST_RELEASE
	/usr/bin/php8.4 /usr/local/bin/composer install --optimize-autoloader --no-dev
	/usr/bin/php8.4 artisan koel:init

	#update caches
	/usr/bin/php8.4 artisan config:cache
	/usr/bin/php8.4 artisan event:cache
	/usr/bin/php8.4 artisan route:cache
	/usr/bin/php8.4 artisan view:cache

	#fix permissions
	chown -R www-data:www-data $TGT

	#fix throttle
	sed -i "s/throttle:60,1/throttle:60000,1/" $TGT/app/Http/Kernel.php

fi
```


## Stuff
```
find /mnt/storagespace/koel -newermt "2025-10-10"
```
