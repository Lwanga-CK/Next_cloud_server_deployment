#install updates
always update and upgrade your system to install latest patches and software.
```bash 
# update package index:
sudo apt update -y

# upgrade the system:
sudo apt dist-upgrade
```

#update Hostname
Update the hostname of your server to reflect the domain name you have decided to use.
```bash
# Update the /etc/hostname file to reflect your chosen name:
sudo nano /etc/hostname

# Also, update the /etc/hosts file to reflect your chosen name:
sudo nano /etc/hosts

# For example, you might add an entry that looks similar to this:
127.0.1.1 cloud.example.com cloud
```
#create a non root user account
If you don’t already have a standard user account, be sure to add one (it’s not a good idea to continually use the root account). 
```bash 
adduser username
#Next, add your user to the sudo group to ensure you can run privileged commands
usermod -aG sudo username
```

#Reboot
Reboot the server so that it takes advantage of all of the updates:
```bash
sudo reboot
```

#Setup MariaDB
For this setup, i have used MariaDB to provide the required database layer.
#install MariaDB
```bash
sudo apt install mariadb-server mariadb-client-compat
```
#Check MariaDB's status 
Run this command below to ensure that MariaDB is up and Running.
```bash 
sudo systemctl staus mariadb

# case its not, Run this command to enable and start it immediately.
sudo systemctl enable --now mariadb
```

#Implement basic security 
Since MariaDB is a big target for threat actors, we need to secure it. The following command will prompt you for various things to provide minimum security:
```bash
sudo mysql_secure_installation
```

#Create Database for Next cloud
Next, we’ll create the actual database that Nextcloud will end up using. To create it, enter the MariaDB shell:
```bash
sudo Mariadb

#Then, create the database:
CREATE DATABASE nextcloud;

#Make sure it was actually created (it should appear in the list):
SHOW DATABASES;

#Next, we’ll need to ensure Nextcloud will have access to its database. The following command will create a “user grant” that Nextcloud will use:
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost' IDENTIFIED BY 'mypassword';

#Finally, exit the Mariadb shell
exit
```


#Install Apache
For this build, Apache will serve as the web server for Nextcloud. Install these packages (Apache will be installed automatically as a dependency):
```bash 
sudo apt install imagemagick-7.q16 php php-apcu php-bcmath php-cli php-common php-curl php-gd php-gmp php-imagick php-intl php-mbstring php-mysql php-zip php-xml
```
check and make sure that apache is running.
```bash
sudo systemctl status apache2
```

#Test Apache
Visit your URL in your browser to make sure the default page loads, you should see the default Apache start page,


#Install Nextcloud
At this point, we’ll install some additional dependencies and then copy Nextcloud’s files to the server.

#Install Required packages
The following packages are required for Apache to be able to serve Nextcloud properly:
```bash
sudo phpenmod apcu bcmath gmp imagick intl unzip

# Ignore the warnings.
```

#Download Nextcloud
Next, download Nextcloud’s latest release file to the server (requires the wget package be installed):
```bash 
which wget
#output should look like;
/usr/bin/wget
#if not, install the wget package with:
sudo apt install wget

#Now download Nextcloud
wget https://download.nextcloud.com/server/releases/latest.zip
```

#Unzip Nextcloud
We’ll need to use the unzip comand to extract files from the downloaded archive (the unzip package will need to be installed for this to work):
```bash 
which unzip
# if not installed, install it:
sudo apt install unzip

#Proceed with unziping the Nextcloud zip file:
unzip latest.zip
```

#Remove the Nextcloud zip file
We won’t need latest.zip anymore, so feel free to remove it if you don’t plan on using it again:
```bash 
rm latest.zip
```

#Move the Nextcloud Directory to an appropriate place
Rename the extracted Nextcloud directory to match your server’s FQDN (optional but recommended):
```bash 
mv nextcloud cloud.example.com

#Change ownership for the Nextcloud directory so that Apache will have access to it:
sudo chown -R www-data:www-data cloud.example.com

Next, move your Nextcloud directory into the /var/www directory:
sudo mv cloud.example.com /var/www/
```

#Disable the Apache default sie
Next, you can disable the default Apache start page since we won’t be needing it:
```bash
sudo a2dissite 000-default.conf
```

#Apache virtual Host configuration
Next, we’ll create a config file that will instruct Apache how to serve Nextcloud.

#Create the config file:
```bash
sudo nano /etc/apache2/sites-available/cloud.example.com.conf
```
#Add the following lines to the config file:
```bash
<VirtualHost *:80>
    DocumentRoot "/var/www/cloud.example.com"
    ServerName cloud.example.com

    <Directory "/var/www/cloud.example.com/">
        Options MultiViews FollowSymlinks
        AllowOverride All
        Order allow,deny
        Allow from all
   </Directory>

   TransferLog /var/log/apache2/_cloud.example.com_access.log
   ErrorLog /var/log/apache2/cloud.example.com_error.log

</VirtualHost>sudo a2ensite nc.learnlinux.cloud.conf

```

#Enable the New config file:
```bash
sudo a2ensite cloud.example.com.conf
```

#Set PHP Settings
Edit the PHP config file so we can adjust some settings:
```bash
sudo nano /etc/php/8.4/apache2/php.ini

#Set the following paramenters (use ctrl + F for search); search line per line

memory_limit = 512M
upload_max_filesize = 200M
max_execution_time = 360
post_max_size = 200M
date.timezone = America/Detroit
opcache.enable=1
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.memory_consumption=128
opcache.save_comments=1
opcache.revalidate_freq=1
```

#Enable required modules for Apache:
```bash
sudo a2enmod dir env headers mime rewrite ssl
```

#Enable Caching
Open the apcu.ini file in an editor:
```bash
sudo nano /etc/php/8.4/mods-available/apcu.ini

#Add the following to the end of the file:
apc.enable_cli=1
```

#Restart
```bash
sudo systemctl restart apache2
```

#Set up Nextcloud
To apply the initial configuration, visit your Nextcloud site and answer the questions. 
You’ll be asked for a password for the admin user, database name, and database password.
Add that information, and Nextcloud will be installed!


#Post Install: Tweak Nextcloud Database
Although we just ran through the installer, there’s a few things Nextcloud doesn’t do on a fresh install. 
To implement these tweaks, first mark the occ script executable (be sure to update the path to match yours):

```bash
sudo chmod +x /var/www/cloud.example.com/occ

#Run the following command to add missing databases indices:
sudo /var/www/cloud.example.com/occ db:add-missing-indices

Update mimetypes as well:
sudo /var/www/cloud.example.com/occ maintenance:repair --include-expensive

#Remove execution bit from occ to increase security:
sudo chmod -x /var/www/cloud.example.com/occ
```

#Post Install: Update config file permissions
The following commands will increase security for the config file:
```bash
#Set ownership:
sudo chown root:www-data /var/www/cloud.example.com/config/config.php

#Set permissions:
sudo chown 660 /var/www/cloud.example.com/config/config.php
```

#Post Install: Enable Caching
```bash
#To enable caching, first edit the Nextcloud config file:
sudo nano /var/www/nc.learnlinux.cloud/config/config.php

#Add to the file:
'memcache.local' => '\\OC\\Memcache\\APCu',
'default_phone_region' => 'US',
```

#Post Install: Set up Let’s Encrypt
```bash
#To set up a certificate for encryption, we’ll first install Certbot:
sudo apt install python3-certbot-apache

#Apply a certificate to your site:
sudo certbot --apache -d cloud.example.com

#To fully benefit from our certificate, enable strict transport security:
sudo nano /etc/apache2/sites-available/cloud.example.com-le-ssl.conf

#Add to the config file:
<IfModule mod_headers.c>
    Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
</IfModule>
```

#Post Install : Install Redis
```bash
#Redis is optional, but can add performance benefits. To install it:
sudo apt install redis-server php-redis

#Edit the Redis config file:
sudo nano /etc/redis/redis.conf

#change: Port 6379 to 0

#Uncomment: unixsocket /run/redis/redis-server.sock

#Uncomment: unixsocketperm 700 (change to 770)

#Add the www-data user to the redis group:
sudo usermod -aG redis www-data

#Edit the Nextcloud config file to use Redis:
sudo vim /var/www/nc.learnlinux.cloud/config/config.php

#Add the following to the file:

'filelocking.enabled' => true,
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'redis' =>
    array(
      'host' => '/var/run/redis/redis-server.sock',
      'port' => 0,
      'timeout' => 0.0,
  ),

```

#Restart Apache to apply our final changes:
```bash
sudo systemctl resatart apache2
```

