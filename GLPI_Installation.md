GLPI Installation Documentation
Step-By-Step Process
Important Note: May need to elevate privileges using “sudo” utility for step(s).

1 - Installing the components
Before we start, make sure your server is up to date, using this command:
apt update && apt upgrade

For this, we are using Apache 2, MariaDB Server, PHP and its respective extensions. If your operating system and repositories are updated, the latest stable version of the extensions are already the ones downloaded.
apt install -y apache2 php php-{apcu,cli,common,curl,gd,imap,ldap,mysql,xmlrpc,xml,mbstring,bcmath,intl,zip,redis,bz2} libapache2-mod-php php-soap php-cas

apt install -y mariadb-server

After all the components are installed, we need to follow the steps.

2 - Database Configuration
MariaDB, by default is provided without a default password set to the root user and with some default settings that need to be correctly configured.
Secure MariaDB Installation
mysql\_secure\_installation

By default, it will ask for a password for root password after this command, skip it by pressing ENTRE. You can set password for root user when you select yes for change the root password.


Minimum recommendation (select Y for all these options)
Change the root password
Remove anonymous users
Disallow root login remotely
Remove test database
Reload privilege tables
Furthermore, since GLPI is a global ITSM tool which may be used by people from all around the world at the same time, you would like to activate the possibility to GLPI database service user to read timezone information from your default mysql database.
mysql\_tzinfo\_to\_sql /usr/share/zoneinfo | mysql mysql

Create a user and database dedicated to GLPI
mysql -uroot -pmysql
CREATE DATABASE glpi;
CREATE USER 'glpi'@'localhost' IDENTIFIED BY 'password-1';
GRANT ALL PRIVILEGES ON glpi.* TO 'glpi'@'localhost';
GRANT SELECT ON mysql.* TO 'glpi'@'localhost';
FLUSH PRIVILEGES;
3 - Preparing files to Install GLPI
After installing the components and create database and service user to receive GLPI folders, you will download the *.tgz latest version of GLPI and store it on apache main root folder.
cd /var/www/html
wget tar -xvzf glpi-10.0.18.tgz

Filesystem Hierarchy Standard Breakdown
In this scenario we are storing GLPI information in different folders and following the FHS were, usually.
/etc/glpi : for the files of configuration of GLPI (config\_db.php, config\_db\_slave.php) ;
/var/www/html/glpi : for the source code of GLPI (in reading only), served by Apache;
/var/lib/glpi : for the variable files of GLPI (session, uploaded documents, cache, cron, plugins, …);
/var/log/glpi : for the log files of GLPI.
To make sure GLPI will find those files, we need to indicate in two different files where these folders are on the system:
The Downstream file
The downstream.php file is responsible for instructing GLPI application where the GLPI\_CONFIG\_DIR - the configuration directory of GLPI - is stored. Remember, we must indicate /etc/glpi as the new folder for configuration files. GLPI understands that a file called downstream.php inside the inc folder has these instructions.
Create the downstream.php file

vim /var/www/html/glpi/inc/downstream.php

Declare the new config file folder - you can insert this content in this file you have created

php
define('GLPI\_CONFIG\_DIR', '/etc/glpi/');
if (file\_exists(GLPI\_CONFIG\_DIR . '/local\_define.php')) {
require\_once GLPI\_CONFIG\_DIR . '/local\_define.php';
}

Now you may move the folders from its current directory to the new directories:

mv /var/www/html/glpi/config /etc/glpi
mv /var/www/html/glpi/files /var/lib/glpi
mv /var/lib/glpi/\_log /var/log/glpi

After you declare the new GLPI\_CONFIG\_DIR with the downstream.php, navigate to this new directory /etc/glpi and create a new file called local\_define.php. This file is reponsible for instructing GLPI where the other directories are stored.
We are changing the documents folder ( files ) and the logs folder ( files/\_log ) to their new directory.
Create the local\_define.php file

vim /etc/glpi/local\_define.php

Paste the following in this file

<?php
define('GLPI\_VAR\_DIR', '/var/lib/glpi');
define('GLPI\_DOC\_DIR', GLPI\_VAR\_DIR);
define('GLPI\_CRON\_DIR', GLPI\_VAR\_DIR . '/\_cron');
define('GLPI\_DUMP\_DIR', GLPI\_VAR\_DIR . '/\_dumps');
define('GLPI\_GRAPH\_DIR', GLPI\_VAR\_DIR . '/\_graphs');
define('GLPI\_LOCK\_DIR', GLPI\_VAR\_DIR . '/\_lock');
define('GLPI\_PICTURE\_DIR', GLPI\_VAR\_DIR . '/\_pictures');
define('GLPI\_PLUGIN\_DOC\_DIR', GLPI\_VAR\_DIR . '/\_plugins');
define('GLPI\_RSS\_DIR', GLPI\_VAR\_DIR . '/\_rss');
define('GLPI\_SESSION\_DIR', GLPI\_VAR\_DIR . '/\_sessions');
define('GLPI\_TMP\_DIR', GLPI\_VAR\_DIR . '/\_tmp');
define('GLPI\_UPLOAD\_DIR', GLPI\_VAR\_DIR . '/\_uploads');
define('GLPI\_CACHE\_DIR', GLPI\_VAR\_DIR . '/\_cache');
define('GLPI\_LOG\_DIR', '/var/log/glpi');

4 - Folder and File Permissions
Permissions for your GLPI installation
chown root:root /var/www/html/glpi/ -R
chown www-data:www-data /etc/glpi -R
chown www-data:www-data /var/lib/glpi -R
chown www-data:www-data /var/log/glpi -R
chown www-data:www-data /var/www/html/glpi/marketplace -Rf

5 - Configure the Web Server
To ensure secure communication, we recommend configuring GLPI to use HTTPS, especially when using it within an internal organization. This process includes creating a self-signed certificate, setting up a secure Apache Virtual Host, and enabling the required modules.
Create an HTTPS VirtualHost Dedicated to GLPI
Create a file at /etc/apache2/sites-available/glpi.conf.
In this file, add the following content:
<VirtualHost *:443
 ServerName glpi.test.local

 DocumentRoot /var/www/html/glpi/public


 SSLEngine on
 SSLCertificateFile /etc/ssl/certs/glpi-selfsigned.crt
 SSLCertificateKeyFile /etc/ssl/private/glpi-selfsigned.key

 
 Require all granted

 RewriteEngine On

 RewriteCond %{HTTP:Authorization} ^(.+)$
 RewriteRule .* - [E=HTTP\_AUTHORIZATION:%{HTTP:Authorization}]

 RewriteCond %{REQUEST\_FILENAME} !-f
 RewriteRule ^(.*)$ index.php [QSA,L]

 

 
Generate a Self-Signed SSL Certificate
Run the following command on your web server:
openssl req -x509 -nodes -days 36500 -newkey rsa:2048 \
 -keyout /etc/ssl/private/glpi-selfsigned.key \
 -out /etc/ssl/certs/glpi-selfsigned.crt \
 -subj "/C=CA/ST=Manitoba/L=Winnipeg/O=MITT/OU=IT/CN=glpi.test.local"

Enable Apache Modules and Virtual Host
After creating the Virtual Host file and SSL certificate, run the following commands:
a2dissite 000-default.conf # Disable default Apache site
a2enmod ssl # Enable SSL module
a2enmod rewrite # Enable the rewrite module
a2ensite glpi.conf # Enable the new GLPI Virtual Host
systemctl restart apache2 # Apply all changes
 
Redirect HTTP to HTTPS
Automatically redirected from HTTP to HTTPS:
Create a file: /etc/apache2/sites-available/glpi-redirect.conf
Add this content:

 ServerName glpi.test.local
 Redirect permanent / 
 
Enable the redirect site:
a2ensite glpi-redirect.conf
systemctl reload apache2
 


