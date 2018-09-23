# About
Due to the lack of good description on how to setup Koel i decided to publish this manual because the installation of Koel required some hours for myself. I hope you will come to a result quicker than me with this cooking recipe ;-)

Some details of my platform:
Operating system: Raspbian GNU/Linux 9.4 (stretch)
Kernel: 4.14.69-v7+
Architecture: armv7l
Model: Raspberry Pi 3 Model B Rev 1.2

## Clone Repository
```
cd /opt
git clone https://github.com/phanan/koel.git
cd koel
git checkout v3.7.2
```

## Install Software Basics
```
apt-get install php php-xml php-mbstring php-curl php-zip php-pgsql postgresql apache2
```

## Install NodeJS
```
curl -sL https://deb.nodesource.com/setup_10.x | bash -
apt-get install -y nodejs
```

## Install Yarn
```
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
apt-get update
apt-get install yarn
```

## Get Composer
```
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('SHA384', 'composer-setup.php') === '544e09ee996cdf60ece3804abc52599c22b1f40f4323403c44d44fdfdd586475ca9813a858088ffbc1f233e9b180f061') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php
php -r "unlink('composer-setup.php');"
```

## Use Composer
```
php composer.phar install
```

## Make Postgres Database
```
sudo - postgres
createuser --no-superuser --pwprompt koel
psql
CREATE database koel WITH OWNER koel ENCODING 'utf8' LC_COLLATE = 'en_US.UTF-8' LC_CTYPE = 'en_US.UTF-8';
GRANT ALL PRIVILEGES ON DATABASE koel TO koel;
\q
```

## Edit php config
```
vim /etc/php/7.0/apache2/php.ini
#enable pdo_pgsql and pgsql extensions - it's questionable if this is really required
extension=php_pdo_pgsql.dll
extension=php_pfsql.dll
memory_limit=512M
```

## Run npm/yarn
```
vim /opt/koel/packages.json #replace ^4.7.2 with ~4.7.2 for node-sass
 
npm update
npm install --unsafe-perm=true --allow-root
```

## Setup Koel
```
# Populate credentials during the process
php artisan koel:init
 
Koel cannot connect to the database. Let's set it up.
 
 Your DB driver of choice [MySQL/MariaDB]:
  [mysql     ] MySQL/MariaDB
  [pgsql     ] PostgreSQL
  [sqlsrv    ] SQL Server
  [sqlite-e2e] SQLite
 > pgsql
 
 DB host:
 > localhost
 
 DB port (leave empty for default) []:
 > 5432
 
 DB name:
 > koel
 
 DB user:
 > koel
 
 DB password []:
 > yourPassword
 
Migrating database
Let's create the admin account.
 
 Your name:
 > Administrator
 
 Your email address:
 > admin@admin.de
 
 Your desired password:
 >
 
 Again, just to make sure:
 >
 
Seeding initial data
The absolute path to your media directory. If this is skipped (left blank) now, you can set it later via the web interface.
 
 Media path []:
 > /mnt/shared_media
 
Compiling front-end stuff
sh: 1: yarn: not found
 
🎆  Success! Koel can now be run from localhost with `php artisan serve`.
You can also scan for media with `php artisan koel:sync`.
Again, for more configuration guidance, refer to
📙  https://koel.phanan.net/docs
or open the .env file in the root installation folder.
Thanks for using Koel. You rock!
```

## Apache Reverse Proxy
```
<VirtualHost *:80>
 RewriteEngine on
 RewriteCond %{HTTPS} !^on$ [NC]
 RewriteRule . https://%{HTTP_HOST}%{REQUEST_URI} [L]
</VirtualHost>
<VirtualHost *:443>
 ServerName music.yourdomain.de
 Header set Access-Control-Allow-Origin "https://music.yourdomain.de"
 Header set Access-Control-Allow-Headers "x-requested-with, Content-Type, origin, authorization, accept, client-security-token"
 SSLEngine on
 SSLVerifyClient none
 SSLCertificateFile /etc/ssl/yourdomain.de.pem
 
 ProxyPass / http://127.0.0.1:8000/
 ProxyPassReverse / http://127.0.0.1:8000/
</VirtualHost>
```

```
a2enmod alias  rewrite proxy proxy_html proxy_http ssl vhost_alias xml2enc
a2enmod music.yourdomain.de.conf
apachectl configtest
service apache2 restart
```

## Dirty hack to enable https
```
vim /opt/koel/routes/web.php
 
#add the URL:: lines to make it work
 
<?php
 
URL::forceRootUrl('https://sub.yourdomain.de:5443');
URL::forceScheme('https');
 
Route::get('/', function () {
    return view('index');
});
 
// Some backward compatibilities.
Route::get('/♫', function () {
    return redirect('/');
});
 
Route::get('/remote', function () {
    return view('remote');
});
```

## Start Server
```
#php artisan serve
cd /opt/koel/;nohup php artisan serve --host 0.0.0.0 > koel-artisan-server.log &
```

## Version overview
```
php -V #7.0
npm --version #6.4.1
nodejs v10.11.0
yarn --version #1.9.4
```
