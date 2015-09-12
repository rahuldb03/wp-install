#!/bin/bash

# Installation script for the latest WordPress on Ubuntu 14.04
#

##### Functions

# Create new database
function install_nginx {
mkdir -p /var/www/html/
echo "please enter domain name required :"
read domain
bash -c "cat > /etc/nginx/sites-available/example.com" << _EOF_
server {
    listen   80;
    server_name  $domain;
    access_log  /var/log/nginx/localhost.access.log;

## Default location
    location / {
        root   /var/www/html;
        index  index.php;
    }

## Images and static content is treated different
    location ~* ^.+.(jpg|jpeg|gif|css|png|js|ico|xml)$ {
      access_log        off;
      expires           30d;
      root /var/www/html;
    }

## Parse all .php file in the /var/www directory
    location ~ .php$ {
        try_files \$uri =404;
        fastcgi_split_path_info ^(.+\\.php)(.*)$;
        fastcgi_pass   backend;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /var/www\$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_param  QUERY_STRING     \$query_string;
        fastcgi_param  REQUEST_METHOD   \$request_method;
        fastcgi_param  CONTENT_TYPE     \$content_type;
        fastcgi_param  CONTENT_LENGTH   \$content_length;
        fastcgi_intercept_errors        on;
        fastcgi_ignore_client_abort     off;
        fastcgi_connect_timeout 60;
        fastcgi_send_timeout 180;
        fastcgi_read_timeout 180;
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;
    }

## Disable viewing .htaccess & .htpassword
    location ~ /\\.ht {
        deny  all;
    }
}
upstream backend {
        server 127.0.0.1:9000;
}
_EOF_

ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
service nginx restart
service php5-fpm restart
}

function check_package {
nginx_chk=`dpkg -s nginx|wc -l`
php_ck=`dpkg -s php5-fpm php5 php5-mysql php5-mysqlnd|wc -l`
mysql_ck=`dpkg -s mysql-server|wc -l`

if [ $nginx_chk -gt 0 ]; then
	echo ""
else 
	apt-get install -y nginx;
fi

if [ $php_ck -gt 0 ]; then
        echo ""
else
        apt-get install -y php5-fpm php5 php5-mysql php5-mysqlnd;
fi

if [ $mysql_ck -gt 0 ]; then
        echo ""
else
        apt-get install -y mysql-server;
fi

}

function create_new_db {
  echo -n "Database root password is NEWPASSWORD: "
  echo ""
  Q00="CREATE DATABASE $dbname;"
  Q01="USE $dbname;"
  Q02="CREATE USER $dbuser@localhost IDENTIFIED BY '$dbpass';"
  Q03="GRANT ALL PRIVILEGES ON $dbname.* TO $dbuser@localhost;"
  Q04="FLUSH PRIVILEGES;"
  SQL0="${Q00}${Q01}${Q02}${Q03}${Q04}"
  mysql -v -u "root" -pNEWPASSWORD -e"$SQL0"
}

# Download WordPress, modify wp-config.php, set permissions
function install_wp {
  cd /var/www/html/
  wget http://wordpress.org/latest.tar.gz
  tar xzvf latest.tar.gz
  cp -rf wordpress/** ./
  rm -R wordpress
  cp wp-config-sample.php wp-config.php
  sed -i "s/database_name_here/$dbname/g" wp-config.php
  sed -i "s/username_here/$dbuser/g" wp-config.php
  sed -i "s/password_here/$dbpass/g" wp-config.php
  wget -O wp.keys https://api.wordpress.org/secret-key/1.1/salt/
  sed -i '/#@-/r wp.keys' wp-config.php
  sed -i "/#@+/,/#@-/d" wp-config.php
  mkdir wp-content/uploads
  find . -type d -exec chmod 755 {} \;
  find . -type f -exec chmod 644 {} \;
  chown -R :www-data wp-content/uploads
  chown -R $USER:www-data *
  chmod 640 wp-config.php
  rm -f latest.tar.gz
  rm -f wp.keys
}

# Create .htaccess file
function generate_htaccess {
 cd /var/www/html/
  touch .htaccess
  chown :www-data .htaccess
  chmod 644 .htaccess
  bash -c "cat > .htaccess" << _EOF_
# Block the include-only files.
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^wp-admin/includes/ - [F,L]
RewriteRule !^wp-includes/ - [S=3]
RewriteRule ^wp-includes/[^/]+\.php$ - [F,L]
RewriteRule ^wp-includes/js/tinymce/langs/.+\.php - [F,L]
RewriteRule ^wp-includes/theme-compat/ - [F,L]
</IfModule>

# BEGIN WordPress
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
# END WordPress

# Prevent viewing of .htaccess file
<Files .htaccess>
order allow,deny
deny from all
</Files>
# Prevent viewing of wp-config.php file
<files wp-config.php>
order allow,deny
deny from all
</files>
# Prevent directory listings
Options All -Indexes
_EOF_
}

# Create robots.txt file
function generate_robots {
 cd /var/www/html/
  touch robots.txt
  bash -c "cat > robots.txt" << _EOF_
# Sitemap: absolute url
User-agent: *
Disallow: /cgi-bin/
Disallow: /wp-admin/
Disallow: /wp-includes/
Disallow: /wp-content/plugins/
Disallow: /wp-content/cache/
Disallow: /wp-content/themes/
Disallow: /trackback/
Disallow: /comments/
Disallow: */trackback/
Disallow: */comments/
Disallow: wp-login.php
Disallow: wp-signup.php
_EOF_
}

# Download WordPress plugins
function download_plugins {
  cd /var/www/html/
  cd wp-content/plugins/
  # W3 Total Cache
  plugin_url=$(curl -s https://wordpress.org/plugins/w3-total-cache/ | egrep -o "https://downloads.wordpress.org/plugin/[^']+")
  wget $plugin_url
  # Theme Test Drive
  plugin_url=$(curl -s https://wordpress.org/plugins/theme-test-drive/ | egrep -o "https://downloads.wordpress.org/plugin/[^']+")
  wget $plugin_url
  # Login LockDown
  plugin_url=$(curl -s https://wordpress.org/plugins/login-lockdown/ | egrep -o "https://downloads.wordpress.org/plugin/[^']+")
  wget $plugin_url
  # Easy Theme and Plugin Upgrades
  plugin_url=$(curl -s https://wordpress.org/plugins/easy-theme-and-plugin-upgrades/ | egrep -o "https://downloads.wordpress.org/plugin/[^']+")
  wget $plugin_url
  # Install unzip package
  apt-get install unzip
  # Unzip all zip files
  unzip \*.zip
  # Remove all zip files
  rm -f *.zip
  echo ""
  cd ../..
}


##### User inputs

echo -n "WordPress database name: "
read dbname
echo -n "WordPress database user: "
read dbuser
echo -n "WordPress database password: "
read -s dbpass
echo ""
echo -n "Install Wordpress? [Y/n] "
read instwp
echo -n "Create a NEW database with entered info? [Y/n] "
read newdb


##### Main

if [ "$newdb" = y ] || [ "$newdb" = Y ]
then
  mkdir -p /var/www/html/
  check_package
  install_nginx
  create_new_db
  install_wp
  generate_htaccess
  generate_robots
  download_plugins
else
  if [ "$instwp" = y ] || [ "$instwp" = Y ]
  then
  mkdir -p /var/www/html/
    install_wp
    check_package
    install_nginx
    generate_htaccess
    generate_robots
    download_plugins
  fi
fi

echo -n "Now, go to your WordPress site to finish installation!"
echo ""
