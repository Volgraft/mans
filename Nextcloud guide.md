# Nextcloud install

## 1 Install UbuntuServer 
There are countless instructions on the Internet on how to install an Ubuntu server. And if you are using a VDS, the installation will usually be done for you.

## 2 Install openssh on your computer
### 2.1 If you have Windows
#### Install openssh with a command in PowerShell run as administrator
```powershell
winget install openssh
```
or with a command
```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```
#### Configure ssh agent startup
manually (more secure)
```powershell
Get-Service -Name ssh-agent | Set-Service -StartupType Manual
```
automatically at OS startup
```powershell
Get-Service -Name ssh-agent | Set-Service -StartupType Automatic
```
#### check agent status
```powershell
Get-Service ssh-agent | Select Name, Status, StartType, DisplayName
```
#### Connect to the server
```
ssh user@server_ip
```

### 2.2 If you have Debian based Linux
#### Install openssh
```bash
sudo apt install openssh
```
#### activate and start the ssh daemon
```bash
sudo systemctl enable ssh --now
```
#### check daemon status
```bash
sudo systemctl status ssh
```
#### Connect to the server
```
ssh user@server_ip
```

## 3 Update and clean packages
```bash
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
```

## 4 Configure hostname and add a name to the hosts file _(in this example "cloud")_
```bash
echo "cloud" > /etc/hostname
sudo sed -i 's/^127\.0\.1\.1.*/127.0.1.1 cloud/g' /etc/hosts
```
Reboot Ubuntu
```bash
sudo reboot
```

## 5 Configure timezone
You can see the list of timezones with the command
```bash
timedatectl list-timezones # all timezones
timedatectl list-timezones | grep Europe  # Europe timezones
```
Timezone setting command
```bash
sudo timedatectl set-timezone Europe/Helsinki
```

## 6 Configure network interfaces
If you rented a global IP from a provider specifically for this server, then you need to configure it according to the instructions below. If you have a cloud server, then probably the way of configuring the IP will differ, or the IP is already set during deployment.
You can check the current server IP with the command
```bash
ip -c a
```
Make a config backup
```bash
sudo cp -a /etc/netplan/50-cloud-init.yaml{,.orig}
```
Check the interface name
```
ip -c -br a
```
Edit the config
```
sudo vim /etc/netplan/50-cloud-init.yaml
```
```
network:
  version: 2
  renderer: networkd
  ethernets:
    ens160:
      dhcp4: no
      addresses:
        - 12.34.56.100/24
      routes:
        - to: 0.0.0.0/0
          via: 12.34.56.1
      nameservers:
        addresses:
          - 12.34.56.4
          - 8.8.8.8
```
Test apply the config and check the connection with a 15-second test and check ping to new ip in these 15 sec
```bash
sudo netplan try --timeout 15
```
If the test was successful, apply the config
```bash
sudo netplan apply
```
If necessary, add the IP to /etc/hosts (do this only if you understand the purpose)
```
192.168.1.100 cloud.mydom.com collabora.mydom.com
```

## 7 Create and configure a user
```bash
sudo adduser vol
```
```
usermod -aG sudo vol
usermod -aG www-data vol
```

## 8 Configure ssh with a key
### 8.1 Disconnect from the server
```bash
exit
```
### 8.2 Create a .ssh folder and swich into it
```
mkdir ~/.ssh
cd ~/.ssh
```
### 8.3 Create public and private key
```
ssh-keygen -t rsa -b 4096
```
### 8.4 Check file names
```
ls
```
### 8.5 Add key to SSH agent
NOT the one with .pub, but the one without extension!
```
ssh-add c:/Users/admin/.ssh/key
```
### 8.6 create ssh directory on remote server with this command
ssh <remote_username>@<server_ip_address> mkdir -p .ssh
```bash
ssh vol@12.34.56.100 mkdir -p .ssh
```
### 8.7 copy public key to remote server
scp .<key.pub> <remote_username>@<server_ip_address>:/home/vol/.ssh/<key.pub>
```bash
scp ~/.ssh/key.pub vol@12.34.56.100:/home/vol/.ssh/key.pub
```
### 8.8 connect to remote server
```
ssh vol@12.34.56.100
```
### 8.9 export public key to authorized_keys
```
cat ~/.ssh/key.pub >> ~/.ssh/authorized_keys
```
### 8.10 check that the connection is passwordless
disconnect
```
exit
```
```
ssh vol@12.34.56.100
```
If it asks for a password, something is wrong, possibly the permissions on authorized_keys. The permissions on the server for the .ssh directory and authorized_keys file should be "600"

### 8.11 set sshd_config
If the connection is successful and passwordless, configure the sshd_config file
```bash
sudo vim /etc/ssh/sshd_config
```
Configure which users can connect. For example, ```AllowUsers *@192.168.*``` will allow any user to connect from a PC whose IP starts with 192.168. This is a useful extra level of protection.
In this instruction, I will only allow my newly created user to connect.
```
AllowUsers vol@*
```
If desired, you can change the default port, don’t forget about it when configuring the firewall.
```
Port 2822
```
Change other settings
```
UsePAM yes
PasswordAuthentication no
PermitRootLogin no
```
### 8.12 restart ssh daemon
```bash
sudo systemctl enable ssh
sudo systemctl restart ssh
```

## 9 Create a domain record
You need to create a domain "A" record on the domain hosting you purchased or on a local domain server if you do not plan to use https. In this instruction it will be **cloud.mydomain.com**

https://www.google.com/search?q=buy+domain

Or if you don’t want to buy a domain right now, there is an alternative way. You can use [https://freemyip.com](https://freemyip.com?utm_source=chatgpt.com) and get a domain name for free, but then instead of "cloud" you will have some other name. Follow the instructions carefully so as not to forget to change the name where necessary.

## 10 Install and configure database
Install mariadb-server
```bash
sudo apt install mariadb-server -y && systemctl status mariadb
```
If you plan to place the database separately from the web server, then it is enough to install the client on the web server.
```bash
sudo apt install mariadb-client -y
```

## 11 MariaDB security settings
Ubuntu 24.04
```bash
sudo mysql_secure_installation
```
Ubuntu 25.04
```bash
sudo mariadb-secure-installation
```
Then select the following parameters:
- Enter current password for root (enter for none): <Enter>
- Switch to unix_socket authentication [Y/n] : n
- Change the root password? [Y/n]: y
- Remove anonymous users? [Y/n]: y
- Disallow root login remotely? [Y/n]: y
- Remove test database and access to it? [Y/n]: y
- Reload privilege tables now? [Y/n]: y

## 12 Set the mariadb config file
Open the mariadb config file
```bash
sudo vim /etc/mysql/my.cnf
```
And add the following lines at the end of the file, which are missing in the file
```c
[mysqld]
log_error = /var/log/mysql/error.log

# 0-disable log, 1-enable log
general_log = 0
slow_query_log = 0
general_log_file = /var/log/mysql/general.log
slow_query_log_file = /var/log/mysql/slow.log

long_query_time = 2
max_connections = 500
wait_timeout = 300
interactive_timeout = 300

innodb_buffer_pool_size = 1G
# For stable connections
wait_timeout = 28800
interactive_timeout = 28800
# For larger queries/uploads
max_allowed_packet = 64M
```
Edit log files
```bash
sudo vim /etc/logrotate.d/mariadb
```
instead of
```
  monthly
  rotate 6
```
write
```
	daily
	rotate 14
```
and you can also edit the log file sizes
```
	maxsize 500M
	minsize 50M
```
Restart mariadb
```bash
sudo systemctl restart mariadb
```
Create a directory with logs and log files and change the owner
```bash
sudo install -o mysql -g mysql -m 0750 -d /var/log/mysql
sudo install -o mysql -g mysql -m 0640 /dev/null /var/log/mysql/error.log
sudo install -o mysql -g mysql -m 0640 /dev/null /var/log/mysql/general.log
sudo install -o mysql -g mysql -m 0640 /dev/null /var/log/mysql/slow.log
```

## 13 Create nextcloud database
Log in to mariadb
```bash
sudo mariadb
```
Create a database
```sql
CREATE DATABASE nextcloud;
```
Check that the database was created
```sql
SHOW DATABASES;
```
Create a user and grant full rights on the nextcloud database when accessed from localhost. Don’t forget to set your own password, it is important that it is strong and preferably random characters. It will be stored in /var/www/nextcloud/config/config.php
```sql
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost' IDENTIFIED BY 'mypassword';
```
If you forgot to change the password, you can do it with the following command
```sql
ALTER USER 'nextcloud'@'localhost' IDENTIFIED BY 'new_password';
```
Update privileges
```sql
FLUSH PRIVILEGES;
```
Exit mariadb
```sql
exit
```

## 14 Install WEB server
Install necessary packages
```bash
sudo apt install vim apache2 php php-apcu php-bcmath php-cli php-common php-curl php-gd php-gmp php-imagick php-intl php-mbstring php-mysql php-zip php-xml -y 
```
Check apache operation
```bash
sudo systemctl status apache2
```

## 15 Activate PHP extensions
```bash
sudo phpenmod bcmath gmp imagick intl
```

## 16 Download nextcloud package
Latest version
```bash
wget https://download.nextcloud.com/server/releases/latest.zip
```
Or if you need the previous version you can download from
https://nextcloud.com/changelog/
for example
```bash
https://download.nextcloud.com/server/releases/nextcloud-31.0.7.zip

mv nextcloud-31.0.7.zip latest.zip
```

## 17 Install nextcloud
Install unzip if not installed
```bash
sudo apt install unzip
```
Unzip archive and move to the desired directory
```bash
unzip latest.zip
sudo mv nextcloud /var/www/
```
Free space by deleting the archive
```bash
rm latest.zip
```

## 18 Edit permissions and owner of nextcloud directory
Change owner to user www-data and group www-data (to which we previously added our user for convenience)
```bash
sudo chown -R www-data:www-data /var/www/nextcloud/
```

## 19 Disable default apache site
```bash
sudo a2dissite 000-default.conf
```

## 20 Configure nextcloud site
```bash
sudo vim /etc/apache2/sites-available/nextcloud.conf
```
```bash
<VirtualHost *:80>
    DocumentRoot "/var/www/nextcloud"
    ServerName cloud

    <Directory "/var/www/nextcloud/">
        Options MultiViews FollowSymlinks
        AllowOverride All
        Order allow,deny
        Allow from all
   </Directory>

   TransferLog /var/log/apache2/nextcloud.log
   ErrorLog /var/log/apache2/nextcloud.log

</VirtualHost>
```

## 21 Activate nextcloud site
```bash
sudo a2ensite nextcloud.conf
```

## 22 Edit apache config
### 22.1 Ubuntu 24.04
Check current important settings
```bash
cat /etc/php/8.3/apache2/php.ini | grep 'memory_limit = '
cat /etc/php/8.3/apache2/php.ini | grep 'upload_max_filesize ='
cat /etc/php/8.3/apache2/php.ini | grep 'max_execution_time ='
cat /etc/php/8.3/apache2/php.ini | grep 'post_max_size ='
cat /etc/php/8.3/apache2/php.ini | grep 'date.timezone ='
cat /etc/php/8.3/apache2/php.ini | grep 'opcache.enable='
cat /etc/php/8.3/apache2/php.ini | grep 'opcache.interned_strings_buffer='
cat /etc/php/8.3/apache2/php.ini | grep 'opcache.max_accelerated_files='
cat /etc/php/8.3/apache2/php.ini | grep 'opcache.memory_consumption='
cat /etc/php/8.3/apache2/php.ini | grep 'opcache.save_comments='
cat /etc/php/8.3/apache2/php.ini | grep 'opcache.revalidate_freq='
```
Replace default settings with those required for nextcloud
```bash
sudo sed -i 's/upload_max_filesize = 2M/upload_max_filesize = 200M/g' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/max_execution_time = 30/max_execution_time = 360/g' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/post_max_size = 8M/post_max_size = 200M/g' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/;date.timezone =/date.timezone = Europe\/Moscow/g' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/;opcache.enable=1/opcache.enable=1/g' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/;opcache.interned_strings_buffer=8/opcache.interned_strings_buffer=16/g' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/;opcache.max_accelerated_files=10000/opcache.max_accelerated_files=10000/g' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/;opcache.memory_consumption=128/opcache.memory_consumption=128/g' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/;opcache.save_comments=1/opcache.save_comments=1/g' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/;opcache.revalidate_freq=2/opcache.revalidate_freq=1/g' /etc/php/8.3/apache2/php.ini
```
Set memory_limit. For standard config, 512M is enough:
```bash
sudo sed -i 's/memory_limit = 128M/memory_limit = 512M/g' /etc/php/8.3/apache2/php.ini
```
If you have enough RAM (the specified amount will be allocated per user) and you plan to view RAW images, you can set 2048M:
```bash
sudo sed -i 's/memory_limit = 128M/memory_limit = 2048M/g' /etc/php/8.3/apache2/php.ini
```
Check the settings again with the first command from this section, the output should be as follows:
```
memory_limit = 512M #(or 2048M)
upload_max_filesize = 200M
max_execution_time = 360
post_max_size = 200M
date.timezone = Europe/Moscow
opcache.enable=1
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.memory_consumption=128
opcache.save_comments=1
opcache.revalidate_freq=1
```

### 22.1 Ubuntu 25.04
Check current important settings
```bash
cat /etc/php/8.4/apache2/php.ini | grep 'memory_limit = '
cat /etc/php/8.4/apache2/php.ini | grep 'upload_max_filesize ='
cat /etc/php/8.4/apache2/php.ini | grep 'max_execution_time ='
cat /etc/php/8.4/apache2/php.ini | grep 'post_max_size ='
cat /etc/php/8.4/apache2/php.ini | grep 'date.timezone ='
cat /etc/php/8.4/apache2/php.ini | grep 'opcache.enable='
cat /etc/php/8.4/apache2/php.ini | grep 'opcache.interned_strings_buffer='
cat /etc/php/8.4/apache2/php.ini | grep 'opcache.max_accelerated_files='
cat /etc/php/8.4/apache2/php.ini | grep 'opcache.memory_consumption='
cat /etc/php/8.4/apache2/php.ini | grep 'opcache.save_comments='
cat /etc/php/8.4/apache2/php.ini | grep 'opcache.revalidate_freq='
```
Replace default settings with those required for nextcloud
```bash
sudo sed -i 's/upload_max_filesize = 2M/upload_max_filesize = 200M/g' /etc/php/8.4/apache2/php.ini
sudo sed -i 's/max_execution_time = 30/max_execution_time = 360/g' /etc/php/8.4/apache2/php.ini
sudo sed -i 's/post_max_size = 8M/post_max_size = 200M/g' /etc/php/8.4/apache2/php.ini
sudo sed -i 's/;date.timezone =/date.timezone = Europe\/Moscow/g' /etc/php/8.4/apache2/php.ini
sudo sed -i 's/;opcache.enable=1/opcache.enable=1/g' /etc/php/8.4/apache2/php.ini
sudo sed -i 's/;opcache.interned_strings_buffer=8/opcache.interned_strings_buffer=16/g' /etc/php/8.4/apache2/php.ini
sudo sed -i 's/;opcache.max_accelerated_files=10000/opcache.max_accelerated_files=10000/g' /etc/php/8.4/apache2/php.ini
sudo sed -i 's/;opcache.memory_consumption=128/opcache.memory_consumption=128/g' /etc/php/8.4/apache2/php.ini
sudo sed -i 's/;opcache.save_comments=1/opcache.save_comments=1/g' /etc/php/8.4/apache2/php.ini
sudo sed -i 's/;opcache.revalidate_freq=2/opcache.revalidate_freq=1/g' /etc/php/8.4/apache2/php.ini
```
Set memory_limit. For standard config, 512M is enough:
```bash
sudo sed -i 's/memory_limit = 128M/memory_limit = 512M/g' /etc/php/8.4/apache2/php.ini
```
If you have enough RAM (the specified amount will be allocated per user) and you plan to view RAW images, you can set 2048M:
```bash
sudo sed -i 's/memory_limit = 128M/memory_limit = 2048M/g' /etc/php/8.4/apache2/php.ini
```
Check the settings again with the first command from this section, the output should be as follows:
```
memory_limit = 512M #(or 2048M)
upload_max_filesize = 200M
max_execution_time = 360
post_max_size = 200M
date.timezone = Europe/Moscow
opcache.enable=1
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.memory_consumption=128
opcache.save_comments=1
opcache.revalidate_freq=1
```

## 23 Restart apache
Make sure apache modules are enabled and working, then restart apache
```bash
sudo a2enmod dir env headers mime rewrite ssl
sudo systemctl restart apache2
```

## 24 Go to the site and finish the initial installation
http://12.34.56.100

## 25 Install additional software
Here is software for nextcloud functionality, popular nextcloud plugins and basic linux debugging
```bash
sudo apt install libapache2-mod-php imagemagick ffmpeg php-bz2 php-redis redis-server redis-server php-redis cron ncdu lnav net-tools iotop htop snapd -y
```

## 26 Edit trusted_domains and overwrite.cli.url
```bash
sudo vim /var/www/nextcloud/config/config.php
```
```
'trusted_domains' =>
  array (
    0 => 'cloud.mydomain.com',
  ),

'overwrite.cli.url' => 'https://cloud.mydomain.com',
```

## 27 Additional settings
Add these parameters to the config
```bash
sudo vim /var/www/nextcloud/config/config.php
```
```
'maintenance' => false,
  'default_phone_region' => 'RU',
  'simpleSignUpLink.shown' => false,
  'config_is_read_only' => false,
  'default_phone_region' => 'RU',
  'maintenance_window_start' => 1,
  'memcache.local' => '\\OC\\Memcache\\APCu',
```

## 28 Make URLs shorter
```bash
sudo vim /var/www/nextcloud/config/config.php
```
Remove "index.php" from url
```
'htaccess.RewriteBase' => '/',
```
Or just run occ command
```bash
sudo php /var/www/nextcloud/occ maintenance:update:htaccess
```

## 29 Fix Image Magick error
Ubuntu 24.04
```bash
sudo apt install libmagickcore-6.q16-6-extra
```

Ubuntu 25.04
```bash
sudo apt install libmagickcore-7.q16-10-extra
```

If installation fails, enter the command below and press Tab. After that select the package with _extra_ tag
```bash
sudo apt install --simulate libmagickcore-
```

## 30 Configure certificate
### 30.1 Setup via certbot
Install snap
```bash
sudo apt install snapd
```

Follow instructions for apache via snap from the site
https://certbot.eff.org/instructions

After successful certificate acquisition, add the line to ssl config file:
```bash
sudo vim /etc/apache2/sites-available/nextcloud-le-ssl.conf
```
```
<IfModule mod_ssl.c>
<VirtualHost *:443>
............
Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
............
</VirtualHost>
</IfModule>
```

Restart apache
```
sudo systemctl restart apache2
```

### 30.1 Manual setup
On the server, create ssl directory
```bash
sudo mkdir /etc/apache2/ssl
```
Change owner to user who will copy the files
```bash
sudo chown vol: /etc/apache2/ssl
```
Disconnect from server, in terminal on local PC go to folder with certificate and key and copy them to the server.
```bash
exit
```
```
scp ./ssl/mydomain.* vol@cloud.mydomain.ru:/etc/apache2/ssl
```
Reconnect to server and restore normal permissions
```bash
sudo chown -R root: /etc/apache2/ssl
```
Then create ssl config for apache
```bash
sudo vim /etc/apache2/sites-available/nextcloud-ssl.conf
```
```
<IfModule mod_ssl.c>
<VirtualHost *:443>
    DocumentRoot "/var/www/nextcloud"
    ServerName cloud.mydomain.com
    ServerAlias cloud.mydomain.com

    <Directory "/var/www/nextcloud/">
        Options MultiViews FollowSymlinks
        AllowOverride All
        Require all granted
   </Directory>

   ErrorLog /var/log/apache2/nextcloud_error.log
   CustomLog /var/log/apache2/nextcloud_access.log combined

   Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"

   SSLEngine on
   SSLCertificateFile /etc/apache2/ssl/mydomain.com.crt
   SSLCertificateKeyFile /etc/apache2/ssl/mydomain.com.key
</VirtualHost>
</IfModule>
```
### 30.2 Enable ssl module in apache
```bash
sudo a2enmod ssl
```
### 30.3 Enable newly created https config
```bash
sudo a2ensite nextcloud-ssl.conf
```
### 30.4 Restart apache
```bash
sudo systemctl reload apache2
```

## 31 Redirect from port 80 to port 443
Create config
```bash
sudo vim /etc/apache2/sites-available/nc-redir.conf
```
```
<VirtualHost *:80>
   ServerName cloud.mydomain.com

   RewriteEngine On
   RewriteCond %{HTTPS} off
   RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]
</VirtualHost>
```
Restart apache
```
sudo systemctl restart apache2
```

## 32 Configure cron
Cron file is the main file responsible for automation of all processes: automatic file operations, trash, database, etc.
**When in /var/www/nextcloud/config/config.php there is 'maintenance' => true, nextcloud cron script  skips execution.**

Install cron if not yet installed
```bash
sudo apt install cron
```
Enable autostart with system
```bash
sudo systemctl enable cron --now
```
Edit cron file as user www-data since only it has all necessary permissions for nextcloud cron file
```bash
sudo -u www-data EDITOR=vim crontab -e
```
All lines starting with # can be deleted and the result will be:
_It is recommended to add a newline at the end of the cron file._
```
*/5  *  *  *  * php -f /var/www/nextcloud/cron.php

```
Check that the file was successfully written
```bash
sudo -u www-data crontab -l
```

## 33 Setup Redis
Redis is an in-memory database that works as very fast "temporary memory" used for cache and queues.
In Nextcloud Redis is needed to store user sessions and cache various data. This makes the site faster, and simultaneous requests are handled without noticeable delays and conflicts.

Install Redis
```bash
sudo apt install redis-server php-redis
```
Give www-data user rights to manage redis
```bash
sudo usermod -aG redis www-data
```
Restart apache
```bash
sudo systemctl restart apache2
```
Configure the following lines in redis config
```bash
sudo vim /etc/redis/redis.conf
```
```
unixsocket /var/run/redis/redis-server.sock
unixsocketperm 770
port 0
```
Restart Redis
```bash
sudo systemctl restart redis
```
Configure nextcloud config for Redis
```bash
sudo vim /var/www/nextcloud/config/config.php
```
Add lines
```
'memcache.local' => '\\OC\\Memcache\\APCu',
```
Add rows
```
'filelocking.enabled' => true,
  'memcache.distributed' => '\\OC\\Memcache\\Redis',
  'memcache.local' => '\\OC\\Memcache\\Redis',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'redis' => 
  array (
        'host' => '/var/run/redis/redis-server.sock',
        'port' => 0,
  ),
```

## 34 Configure logs
Add necessary lines to config
```bash
sudo vim /var/www/nextcloud/config/config.php
```
```
'log_type' => "file",
  'logfile' => '/var/log/nextcloud.log',
  'loglevel' => 1,
  'logdateformat' => "F d, Y H:i:s",
```
Create log files
```bash
sudo install -o www-data -g www-data -m 0640 /dev/null /var/log/nextcloud.log
```
One way to view logs - using lnav program
```bash
sudo tail -n 100 /var/log/nextcloud.log | lnav
sudo tail -n 100 /var/log/apache2/nextcloud.log | lnav
```

## 35 Configure fail2ban
Install fail2ban
```bash
sudo apt install fail2ban
```
Copy configs so they won’t be erased during update
```bash
sudo cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Uncomment ignoreip line
```bash
sudo vim /etc/fail2ban/jail.local
```
```
ignoreip = 127.0.0.1/8 ::1
```
Create nextcloud config. Current config can be found on [official site](https://docs.nextcloud.com/server/19/admin_manual/installation/harden_server.html?highlight=fail2ban#setup-fail2ban)
```bash
sudo vim /etc/fail2ban/filter.d/nextcloud.conf
```
```
[Definition]
_groupsre = (?:(?:,?\s*"\w+":(?:"[^"]+"|\w+))*)
failregex = ^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Login failed:
            ^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Two-factor challenge failed:
            ^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Trusted domain error.
datepattern = ,?\s*"time"\s*:\s*"%%Y-%%m-%%d[T ]%%H:%%M:%%S(%%z)?"
```
Create jail config for nextcloud
```bash
sudo vim /etc/fail2ban/jail.d/nextcloud.local
```
```
[nextcloud]
backend = auto
enabled = true
port = 80,443
protocol = tcp
filter = nextcloud
maxretry = 3
bantime = 86400
findtime = 3600
logpath = /var/log/nextcloud.log
```
Create jail config for ssh
```bash
sudo vim /etc/fail2ban/jail.d/sshd.local
```
```
[sshd]
backend = systemd
enabled = true
port = ssh
protocol = tcp
filter = sshd
maxretry = 2
bantime = 1d
findtime = 60m
logpath = %(sshd_log)s
```
Restart fail2ban
```bash
sudo systemctl enable fail2ban && sudo systemctl restart fail2ban && sudo systemctl status fail2ban
```

## 36 UFW basic firewall
Install ufw
```bash
sudo apt install ufw 
```
Check that IPV6 is also filtered
```bash
sudo vim /etc/default/ufw
```
```
IPV6=yes
```
Configure ufw rules
If you changed ssh port then replace port 22 with yours.
```bash
sudo ufw default deny incoming
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 0.0.0.0/0 to any port 22 proto tcp
sudo ufw allow from 192.168.0.0/16 to any port 22 proto tcp
```
Check rules with ufw disabled
```bash
sudo cat /etc/ufw/user*.rules | grep tuple
```
Enable ufw
```bash
sudo ufw enable
```
Check ufw rules
```bash
sudo ufw status numbered
```

If necessary
delete unnecessary rule
```bash
sudo ufw delete <number>
```
add rule at desired position ```sudo ufw insert <position> <rule>``` for example
```bash
sudo ufw insert 1 allow from 8.8.8.8 to any
```

## 37 Skeleton directory
When a new user is created, nextcloud already has some files. They are copied from the Skeleton directory by default (/var/www/nextcloud/core/skeleton). If you want to configure your own default files set or none at all – configure your own Skeleton directory. **Attention, do not edit anything in default Skeleton directory as you will inevitably encounter issues during update.**

Create directory for Skeleton directory
```bash
sudo install -o www-data -g www-data -m 0750 -d /opt/nextcloud_skeleton
```
Add line to config
```bash
sudo vim /var/www/nextcloud/config/config.php
```
```
'skeletondirectory' => '/opt/nextcloud_skeleton',
```

## 38 Set auto trashbin cleanup
By default, nextcloud allocates half of the free space to trash and automatically deletes the oldest deleted files when space runs out. With this setup, if you upload files larger than half of free space, deletion begins simultaneously with upload, slowing down process and loading the server. This can also greatly increase backup size.

To configure auto cleanup of files deleted more than 30 days ago, add this line to config:
```bash
sudo vim /var/www/nextcloud/config/config.php
```
```
'trashbin_retention_obligation' => 'auto, 30',
```

## 39 Fix database indexes
Sometimes after installation or update you need to fix database indexes
```bash
sudo -u www-data php /var/www/nextcloud/occ maintenance:repair --include-expensive
sudo -u www-data php /var/www/nextcloud/occ db:add-missing-indices
```

## 40 Increase memory limit
If earlier in config /etc/php/8.*/apache2/php.ini you set memory_limit = 2048M then you need to add a line to config. This is necessary, for example, for correct preview of large images in Memories app.
```bash
sudo vim  /var/www/nextcloud/config/config.php
```
```
'preview_max_memory' => 2048,
```

# Collabora install

## 1 Create dns record
Create an "A" domain record at the same hosting where you did cloud.mydomain.com\
In this example it will be **collabora.mydomain.com**

## 2 Install Nextcloud Office
In web interface go to Apps and install "Nextcloud Office" app

## 3 Import official repository key
Install gnupg
```bash
sudo apt install gnupg
```
Download collabora repository keys ([link](https://sdk.collaboraonline.com/docs/installation/Installation_from_packages.html))
```bash
cd /usr/share/keyrings
sudo wget https://collaboraoffice.com/downloads/gpg/collaboraonline-release-keyring.gpg
```

## 4 Add repository to /etc/apt/sources.list.d
```bash
sudo vim /etc/apt/sources.list.d/collaboraonline.sources
```
```bash
Types: deb
URIs: https://www.collaboraoffice.com/repos/CollaboraOnline/CODE-ubuntu2204
Suites: ./
Signed-By: /usr/share/keyrings/collaboraonline-release-keyring.gpg
```

## 5 Install Collabora
```bash
sudo apt update
```
```bash
sudo apt install coolwsd code-brand hunspell collaboraoffice*
```

## 6 Configure Apache for Collabora
Create config template
```bash
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/collabora.conf
```
Disable default apache page
```
sudo mv /var/www/html/index.html /var/www/html/index.html.bak
```
Edit file (commented lines can be deleted if desired).
```bash
sudo vim /etc/apache2/sites-available/collabora.conf
```
```
        ServerName collabora.mydomain.com
```
```
        # Delete or comment out lines:  
        ServerAdmin
        DocumentRoot
```
Enable collabora config in apache
```bash
sudo a2ensite collabora && sudo systemctl reload apache2
```

## 7 Enable apache modules
```bash
sudo a2enmod proxy proxy_wstunnel proxy_http proxy_connect
```

## 8 Get SSL certificate
### 8.1 Via certbot
Use certbot command
```bash
sudo certbot --apache
```
In config file created by certbot add the following lines:
```bash
sudo vim /etc/apache2/sites-enabled/collabora-le-ssl.conf
```
```
        Options -Indexes

        AllowEncodedSlashes NoDecode
        ProxyPreserveHost On


        # static html, js, images, etc. served from coolwsd
        # browser is the client part of Collabora Online
        ProxyPass           /browser http://127.0.0.1:9980/browser retry=0
        ProxyPassReverse    /browser http://127.0.0.1:9980/browser


        # WOPI discovery URL
        ProxyPass           /hosting/discovery http://127.0.0.1:9980/hosting/discovery retry=0
        ProxyPassReverse    /hosting/discovery http://127.0.0.1:9980/hosting/discovery


        # Capabilities
        ProxyPass           /hosting/capabilities http://127.0.0.1:9980/hosting/capabilities retry=0
        ProxyPassReverse    /hosting/capabilities http://127.0.0.1:9980/hosting/capabilities


        # Main websocket
        ProxyPassMatch      "/cool/(.*)/ws$"      ws://127.0.0.1:9980/cool/$1/ws nocanon


        # Admin Console websocket
        ProxyPass           /cool/adminws ws://127.0.0.1:9980/cool/adminws


        # Download as, Fullscreen presentation and Image upload operations
        ProxyPass           /cool http://127.0.0.1:9980/cool
        ProxyPassReverse    /cool http://127.0.0.1:9980/cool
        # Compatibility with integrations that use the /lool/convert-to endpoint
        ProxyPass           /lool http://127.0.0.1:9980/cool
        ProxyPassReverse    /lool http://127.0.0.1:9980/cool


        ErrorLog /var/log/apache2/collabora-error.log
        CustomLog /var/log/apache2/collabora-access.log combined
```

### 8.1 Manual certificate setup
Create config file. In this instruction it is assumed you have certificates valid for the entire domain (*.mydomain.com). If you have a separate key and certificate for collabora, you need to upload them to server, change paths in config below and set permissions same as cloud certificates.
```bash
sudo vim /etc/apache2/sites-available/collabora-ssl.conf
```
```
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName collabora.mydomain.com

    Options -Indexes

    AllowEncodedSlashes NoDecode
    ProxyPreserveHost On


    # static html, js, images, etc. served from coolwsd
    # browser is the client part of Collabora Online
    ProxyPass           /browser http://127.0.0.1:9980/browser retry=0
    ProxyPassReverse    /browser http://127.0.0.1:9980/browser


    # WOPI discovery URL
    ProxyPass           /hosting/discovery http://127.0.0.1:9980/hosting/discovery retry=0
    ProxyPassReverse    /hosting/discovery http://127.0.0.1:9980/hosting/discovery


    # Capabilities
    ProxyPass           /hosting/capabilities http://127.0.0.1:9980/hosting/capabilities retry=0
    ProxyPassReverse    /hosting/capabilities http://127.0.0.1:9980/hosting/capabilities


    # Main websocket
    ProxyPassMatch      "/cool/(.*)/ws$"      ws://127.0.0.1:9980/cool/$1/ws nocanon


    # Admin Console websocket
    ProxyPass           /cool/adminws ws://127.0.0.1:9980/cool/adminws


    # Download as, Fullscreen presentation and Image upload operations
    ProxyPass           /cool http://127.0.0.1:9980/cool
    ProxyPassReverse    /cool http://127.0.0.1:9980/cool
    # Compatibility with integrations that use the /lool/convert-to endpoint
    ProxyPass           /lool http://127.0.0.1:9980/cool
    ProxyPassReverse    /lool http://127.0.0.1:9980/cool

    Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
    SSLEngine on

    ErrorLog /var/log/apache2/collabora-error.log
    CustomLog /var/log/apache2/collabora-access.log combined

    SSLCertificateFile /etc/apache2/ssl/mydomain.com.crt
    SSLCertificateKeyFile /etc/apache2/ssl/mydomain.com.key
</VirtualHost>
</IfModule>
```

Enable config in apache
```bash
sudo a2ensite collabora-ssl.conf
```

### 8.2 For both options
Enable ssl module if not enabled
```bash
sudo a2enmod ssl
```
Restart apache
```bash
sudo systemctl reload apache2
```

## 9 Configure logs
Create log files
```bash
sudo install -o root -g adm -m 0640 /dev/null /var/log/apache2/collabora-error.log
sudo install -o root -g adm -m 0640 /dev/null /var/log/apache2/collabora-access.log
```

## 10 Configure collabora config
In Collabora languages are used for interface layout and spell check. In large text files spell checking many languages may slow performance. It is recommended to keep only needed languages.

Check which languages are currently in config
```bash
sudo grep en_US.*ru /etc/coolwsd/coolwsd.xml
```
Keep only needed languages (in this example "en_US ru")
```bash
sudo sed -i 's/de_DE en_GB en_US es_ES fr_FR it nl pt_BR pt_PT ru/en_US ru/g' /etc/coolwsd/coolwsd.xml
```
Make sure the line changed correctly
```bash
sudo grep en_US.*ru /etc/coolwsd/coolwsd.xml
```
Disable collabora traffic encryption since with such /etc/redis/redis.conf data exchange will go through local socket
```bash
sudo coolconfig set ssl.enable false
sudo coolconfig set ssl.termination true
```

## 11 Restart collabora and apache
```bash
sudo systemctl restart apache2
sudo systemctl status apache2
```
```bash
sudo systemctl restart coolwsd
sudo systemctl status coolwsd
```

## 12 Enable Collabora in Nextcloud
On the site go to "Administration" menu and find "Office" tab.
Choose "Use your own server" and enter
```
https://collabora.mydomain.com
```
Below find "Allow list for WOPI requests" and enter the server IP (in this instruction it is known to you as 12.34.56.100)

## 13 Troubleshoot collabora
https://sdk.collaboraonline.com/docs/installation/Collabora_Online_Troubleshooting_Guide.html
```bash
journalctl -e -u coolwsd | lnav
journalctl -r -u coolwsd | lnav
```

# LDAP setup (ActiveDirectory authentication)

# Update Nextcloud version

# User deletion

# Move nextcloud to another server

# Change Nextcloud DNS name







