# Условные обозначения

⚠️ - необходимо **обязательно** заменить значения в командах или коде на собственные

```
информация для ввода в терминал
```

> информация, которую нужно внести в текстовый файл или команды в базе данных

# Установка и подготовка Ubuntu Server

#### 0\. Создать DNS запись

Cоздать DNS A-запись   
**cloud.mydomain.ru = IP вашего nextcloud**  
Если собираетесь устанавливать Collabora Office, сразу создайте А-запись и для Collabora **collabora.mydomain.ru = IP вашего nextcloud**

#### 1\. Установить UbuntuServer

https://ubuntu.com/download/server

#### 2\. При установке:

Разметить диски

*/dev/sda
  2G ext4 /boot

/dev/sdb
  root-vg (lvm)
    ext4 / root-lv (lvm)*

После установки извлечь установочный диск и перезагрузить

#### 3\. Обновить и очистить пакеты

```
sudo apt update
sudo apt upgrade
sudo apt autoremove
```

#### 4\. Настроить hostname

Не используйте полное доменное имя, в будущем это вызовет проблемы при настройке collabora на этом сервере

```
sudo vim /etc/hostname
```

> cloud

```
sudo vim /etc/hosts     
```

> 127\.0.1.1 cloud

#### 5\. Настроить timezone ⚠️

```
sudo timedatectl set-timezone Europe/Moscow
```

#### 6\. Настроить сетевые интерфейсы

Убедиться что  *“network: {config: disabled}”* запись существует в данном файле. Если нет, то её небходимо создать.

```
sudo cat /etc/cloud/cloud.cfg.d/subiquity-disable-cloudinit-networking.cfg 
```

Команда должна вывести следующую строку: *network: {config: disabled}*

Если данная строка не была выведена, следует добавить её в файл */etc/cloud/cloud.cfg.d/subiquity-disable-cloudinit-networking.cfg*

Получить информацию о текущем состоянии сетевых интерфейсов и сделать бекап конфига

```
sudo cat /etc/netplan/00-installer-config.yaml
sudo cp -a /etc/netplan/00-installer-config.yaml{,.orig}
ip a | grep -E "inet |<.*>"
```

Далее настроить сетевые интерфейсы ⚠️

```
sudo vim /etc/netplan/00-installer-config.yaml
```

>network:
> 
>  version: 2
> 
>  renderer: networkd
> 
>  ethernets:
> 
>    ens18:
> 
>      addresses:
> 
>        - 192.168.1.11/24
> 
>      nameservers:
> 
>        addresses: [192.168.1.1, 8.8.8.8]
> 
>    ens19:
> 
>      addresses:
> 
>        - 123.123.123.123/24
> 
>      nameservers:
> 
>        addresses: [8.8.8.8, 1.1.1.1]
> 
>      routes:
> 
>        - to: default
> 
>          via: 123.123.123.1


Применить изменения (если в конфиге вы сменили IP, следующую команду необходимо выполнить залогинившись непосредственно в консоль сервера, а **не** по SSH и т.п.)

```
sudo netplan try 
```

#### 7\. Добавить пользователя ⚠️

```
adduser myuser
usermod -aG sudo myuser
```

#### 8\. Настроить SSH по ключу

##### 💻 На локальном пк

Установить OpenSSH

| Система | Команда |
|---------|---------|
| linux bash | sudo apt install openssh |
| windows powershell | winget install openssh |

Перейти в папку .ssh на локальной машине

```
cd ~/.ssh
```

Сгенерировать ключ на локальной машине и дать ему любое удобное имя, например *key*

```
ssh-keygen -t rsa -b 4096
```

Посмотреть созданные ключи

```
ls
```

Создать папку .ssh на будущем nextcloud сервере⚠️

*ssh \[remote_username\]@\[server_ip_address\] mkdir -p .ssh* 

```
ssh myuser@mydomain.ru mkdir -p .ssh
```

Скопировать публичный ключ на удаленный сервер⚠️

*scp .\\\[key.pub\] \[remote_username\]@\[server_ip_address\]:/home/myuser/.ssh/\[key.pub\]*

```
scp C:\Users\windowsuser\.ssh\key.pub myuser@mydomain.ru:/home/myuser/.ssh/key.pub
```

Включить ssh агент (**windows**)

```
Start-Service ssh-agent 
Get-Service ssh-agent
Get-Service ssh-agent | Select StartType
Get-Service -Name ssh-agent | Set-Service -StartupType Manual
```

Включить ssh агент (**linux**)

```
sudo systemctl start sshd
sudo systemctl enable sshd.service
```

Добавить ключ в SSH агент (**windows**)⚠️

*ssh-add <path to new private key file>*

```
ssh-add c:/Users/windowsuser/.ssh/key
```

Добавить ключ в SSH агент (**linux**)⚠️

```
ssh-add /home/linuxuser/.ssh/key
```

##### 💻 Подключится к удаленному серверу по ssh

Экспорт публичного ключа в uthorized_keys

```
cat ~/.ssh/key.pub >> ~/.ssh/authorized_keys
```

Отключится от сервера и проверить подключение без ключа⚠️

```
exit
ssh myuser@mydomain.ru
```

Отредактировать *sshd_config* на сервере

(В строке *Port* можно установить любой порт, не занятый популярным сервисом) ⚠️

```
sudo vim /etc/ssh/sshd_config 
```

> Port 2822
>
> UsePAM yes
>
> PasswordAuthentication no
>
> PermitRootLogin no

Перезагрузить sshd сервис

```
sudo systemctl restart sshd
```

Теперь при подключении необходимо указывать порт⚠️

```
ssh myuser@mydomain.ru -p 2822
```

# Установка и настройка NextCloud

#### 1\. Установка и настройка базы данных

```
sudo apt install mariadb-server
systemctl status mariadb
```

#### 2\. Настройки безопасности MariaDB

```
sudo mysql_secure_installation
```

- Enter current password for root (enter for none): none
- Switch to unix_socket authentication [Y/n] : n
- Change the root password? [Y/n]: y
- Remove anonymous users? [Y/n]: y
- Disallow root login remotely? [Y/n]: y
- Remove test database and access to it? [Y/n]: y
- Reload privilege tables now? [Y/n]: y

#### 3\. Создание базы nextcloud

Установите свой сложный пароль⚠️

```
sudo mariadb
```

> CREATE DATABASE nextcloud;
> SHOW DATABASES;
> GRANT ALL PRIVILEGES ON nextcloud.\* TO 'nextcloud'@'localhost' IDENTIFIED BY 'mypassword';
> FLUSH PRIVILEGES;
> exit

сменить пароль пользователя в maiadb (если забыли сменить)

> ALTER USER 'nextcloud'@'localhost' IDENTIFIED BY 'new_password';
> FLUSH PRIVILEGES;
> exit

#### 4\. Установка WEB сервера

apache2 должен установится как зависимость вместе со всеми пакетами

```
sudo apt install php php-apcu php-bcmath php-cli php-common php-curl php-gd php-gmp php-imagick php-intl php-mbstring php-mysql php-zip php-xml

systemctl status apache2
```

#### 5\. Включить PHP расширения

```
sudo phpenmod bcmath gmp imagick intl
```

#### 6\. Скачать дистрибутив Nextcloud

https://nextcloud.com/

```
wget https://download.nextcloud.com/server/releases/latest.zip
```

К сожалению *latest* версия не всегда идеальна и в ней могут быть баги. В таком случае можно скачать последнюю версию с предыдущей первой цифрой (если latest 28.0.2 то нам нужна 27.1.5) и обновится до последней стабильной версии, если сервер предложит.
https://nextcloud.com/changelog/

```
wget https://download.nextcloud.com/server/releases/nextcloud-27.1.5.zip
```

#### 7\. Распаковать и установить Nextcloud

```
sudo apt install unzip
unzip latest.zip
sudo mv nextcloud /var/www/
rm latest.zip
```

#### 8\. Сменить владельца /var/www/nextcloud/

```
sudo chown -R www-data:www-data /var/www/nextcloud/
```

#### 9\. Отключаем дефолтный сайт apache

```
sudo a2dissite 000-default.conf
```

#### 10\. Настроить сайт nextcloud

```
sudo vim /etc/apache2/sites-available/nextcloud.conf
```

> <VirtualHost \*:80>  
>     DocumentRoot "/var/www/nextcloud"  
>     ServerName cloud  
>   
>     <Directory "/var/www/nextcloud/">  
>         Options MultiViews FollowSymlinks  
>         AllowOverride All  
>         Order allow,deny  
>         Allow from all  
>    </Directory>  
>   
>    TransferLog /var/log/apache2/nextcloud.log  
>    ErrorLog /var/log/apache2/nextcloud.log  
>   
> </VirtualHost>

#### 11\. Включить nextcloud сайт в apache

```
sudo a2ensite nextcloud.conf
```

#### 12\. Конфигурирование appache сервера

провести проверку текущих настроек в файле /etc/php/8.1/apache2/php.ini через **grep**

```
PHPINI="/etc/php/8.1/apache2/php.ini"

grep 'memory_limit = ' $PHPINI
grep 'memory_limit = ' $PHPINI
grep 'upload_max_filesize =' $PHPINI
grep 'max_execution_time =' $PHPINI
grep 'post_max_size =' $PHPINI
grep 'date.timezone =' $PHPINI
grep 'opcache.enable=' $PHPINI
grep 'opcache.interned_strings_buffer=' $PHPINI
grep 'opcache.max_accelerated_files=' $PHPINI
grep 'opcache.memory_consumption=' $PHPINI
grep 'opcache.save_comments=' $PHPINI
grep 'opcache.revalidate_freq=' $PHPINI
```

применить требуемые строки настроек к файлу /etc/php/8.1/apache2/php.ini (⚠️обратить внимание на date.timezone)

```
PHPINI="/etc/php/8.1/apache2/php.ini"

sudo sed -i 's/memory_limit = 128M/memory_limit = 512M/g' $PHPINI
sudo sed -i 's/upload_max_filesize = 2M/upload_max_filesize = 200M/g' $PHPINI
sudo sed -i 's/max_execution_time = 30/max_execution_time = 360/g' $PHPINI
sudo sed -i 's/post_max_size = 8M/post_max_size = 200M/g' $PHPINI

sudo sed -i 's/;opcache.enable=1/opcache.enable=1/g' $PHPINI
sudo sed -i 's/;opcache.interned_strings_buffer=8/opcache.interned_strings_buffer=8/g' $PHPINI
sudo sed -i 's/;opcache.max_accelerated_files=10000/opcache.max_accelerated_files=10000/g' $PHPINI
sudo sed -i 's/;opcache.memory_consumption=128/opcache.memory_consumption=128/g' $PHPINI
sudo sed -i 's/;opcache.save_comments=1/opcache.save_comments=1/g' $PHPINI
sudo sed -i 's/;opcache.revalidate_freq=2/opcache.revalidate_freq=1/g' $PHPINI
```

```
sudo sed -i 's/;date.timezone =/date.timezone = Europe\/Moscow/g' $PHPINI
```

провести проверку через **grep** снова, результат должен быть следующим:

*memory_limit = 512M  
upload_max_filesize = 200M  
max_execution_time = 360  
post_max_size = 200M  
date.timezone = Europe/Moscow  
opcache.enable=1  
opcache.interned_strings_buffer=8  
opcache.max_accelerated_files=10000  
opcache.memory_consumption=128  
opcache.save_comments=1  
opcache.revalidate_freq=1*

#### 13\. Перепроверить что модификаторы appache в порядке и перезагрузить appache

```
sudo a2enmod dir env headers mime rewrite ssl
sudo systemctl restart apache2
```

#### 14\. Перейти на сайт и завершить установку nextcloud

*http://your_server_ip*

# Дополнительные настройки и сервисы

#### 1\. Дополнительный софт

```
sudo apt install libapache2-mod-php php-bz2 php-redis redis-server php-redis cron net-tools iotop htop imagemagick ffmpeg ncdu lnav 
```

#### 2\. Добавить в 'trusted_domains' все адреса, к которым будет осуществляться подключение⚠️

```
sudo vim /var/www/nextcloud/config/config.php
```

> 'trusted_domains' =>
> array (
>   0 => 'cloud.mydomain.ru',
> ),

#### 3\. Дополнительные настройки в config.php

```
sudo vim /var/www/nextcloud/config/config.php
```

>   'memcache.local' => '\\\\OC\\\\Memcache\\\\APCu',  
>   'default_phone_region' => 'RU',  
>   'simpleSignUpLink.shown' => false,  
>   'config_is_read_only' => true,  
>   'maintenance' => false,

#### 4\. Убрать ошибку "Image Magick error"

```
sudo apt install libmagickcore-6.q16-6-extra
```

#### 5\. Настроить certbot

- переходим на сайт
- выбираем свою операционную систему
- устанавливаем certbot согласно иструкции

*https://certbot.eff.org/instructions*

#### 6\. Включить Strict Transport Security

HTTP Strict Transport Security (HSTS) — механизм, активирующий форсированное защищённое соединение по HTTPS. Данная политика безопасности позволяет сразу же устанавливать безопасное соединение, вместо использования HTTP.

```
sudo vim /etc/apache2/sites-available/nextcloud-le-ssl.conf
```

добавить **выделенную** строку 

> <IfModule mod_ssl.c>  
> <VirtualHost \*:443>  
>   ............  
>   **Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"**   
>   ............  
> </VirtualHost>  
> </IfModule>

#### 7\. Перенаправляем запросы с 80 порта на 443

```
sudo vim /etc/apache2/sites-available/nc-redir.conf 
```

> <VirtualHost \*:80>  
>    ServerName nc.domain.org
>
>    RewriteEngine On  
>    RewriteCond %{HTTPS} off  
>    RewriteRule ^(.\*)$ https://%{HTTP_HOST}$1 \[R=301,L\]  
> </VirtualHost>

```
sudo systemctl restart apache2
```

#### 8\. Настроить cron

В web интерфейсе *ЛКМ аватар - Параметры сервера - Основные параметры - Фоновые задания - Cron .* Или ввести команду:

```
sudo -u www-data php /var/www/nextcloud/occ background:cron
```

Установка и настройка

```
sudo apt install cron 
sudo systemctl enable cron
```

```
sudo crontab -e 
```

в конце строки crontab поставить перенос на новую строку    

> \*/5  \*  \*  \*  \* php -f /var/www/nextcloud/cron.php  sudo crontab -l

```
sudo crontab -l
```

#### 9\. Настроить redis

Redis нужен для корректной работы cron задачи.

Установка и настройка

```
sudo apt install redis-server php-redis
sudo usermod -a -G redis www-data
sudo systemctl restart apache2
```

```
sudo vim /etc/redis/redis.conf
```

>   unixsocket /var/run/redis/redis-server.sock  
>   unixsocketperm 770  
>   port 0

```
sudo systemctl restart redis
```

Удалить строку из конфига nextcloud

```
sudo vim /var/www/nextcloud/config/config.php
```

 удалить строку    

> 'memcache.local' => '\\\\OC\\\\Memcache\\\\APCu',

добавить строки

>   'filelocking.enabled' => true,  
>   'memcache.distributed' => '\\\\OC\\\\Memcache\\\\Redis',  
>   'memcache.local' => '\\\\OC\\\\Memcache\\\\Redis',  
>   'memcache.locking' => '\\\\OC\\\\Memcache\\\\Redis',  
>   'redis' =>   
>   array (  
>         'host' => '/var/run/redis/redis-server.sock',  
>         'port' => 0,  
>   ),

#### 10\. Настроить fail2ban

Установка

```
sudo apt install fail2ban
sudo cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Добавляем ip, которые fail2ban будет игнорировать

```
sudo vim /etc/fail2ban/jail.local
```

> ignoreip = 127.0.0.1/8 ::1

Создаем фильтр файл fail2ban для nextcloud (код взят из официальной документации <https://docs.nextcloud.com/server/19/admin_manual/installation/harden_server.html?highlight=fail2ban> )

```
sudo vim /etc/fail2ban/filter.d/nextcloud.conf
```

> \[Definition\]  
> \_groupsre = (?:(?:,?\\s\*"\\w+":(?:"\[^"\]+"|\\w+))\*)  
> failregex = ^\\{%(\_groupsre)s,?\\s\*"remoteAddr":"<HOST>"%(\_groupsre)s,?\\s\*"message":"Login failed:  
>             ^\\{%(\_groupsre)s,?\\s\*"remoteAddr":"<HOST>"%(\_groupsre)s,?\\s\*"message":"Trusted domain error.  
> datepattern = ,?\\s\*"time"\\s\*:\\s\*"%%Y-%%m-%%d\[T \]%%H:%%M:%%S(%%z)?"

Создаем конфиг файл fail2ban для nextcloud (⚠️ можете изменять port , maxretry , bantime,  findtime,  по своему усмотрению)

```
sudo vim /etc/fail2ban/jail.d/nextcloud.local
```

> \[nextcloud\]  
> backend = auto  
> enabled = true  
> port = 80,443  
> protocol = tcp  
> filter = nextcloud  
> maxretry = 3  
> bantime = 86400  
> findtime = 3600  
> logpath = /var/log/nextcloud.log

Создаем конфиг файл fail2ban для ssh (⚠️ можете изменять port , maxretry , bantime,  findtime,  по своему усмотрению)

```
sudo vim /etc/fail2ban/jail.d/sshd.local
```

> \[sshd\]  
> backend = systemd  
> enabled = true  
> port = ssh  
> protocol = tcp  
> filter = sshd  
> maxretry = 1  
> bantime = 1d  
> findtime = 60m  
> logpath = %(sshd_log)s

Создать файл логов

```
sudo touch /var/log/nextcloud.log 
sudo chown www-data:www-data /var/log/nextcloud.log
sudo chmod 660 /var/log/nextcloud.log

sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
sudo systemctl status fail2ban
```

#### 11\. Настройка ufw файрвола

```
sudo apt-get install ufw 
```

```
sudo vim /etc/default/ufw 
```

>   IPV6=yes

не забудьте поменять порт ssh на ваш порт ⚠️

```
sudo ufw default deny

sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443

sudo ufw enable
```

#### 12\. Настройка логов nextcloud

```
sudo vim /var/www/nextcloud/config/config.php
```

>   'log_type' => "file",  
>   'logfile' => '/var/log/nextcloud.log',  
>   'loglevel' => 1,  
>   'logdateformat' => "F d, Y H:i:s",

```
sudo touch /var/log/nextcloud.log
sudo chown -R root:www-data /var/log/nextcloud.log

sudo systemctl restart apache2
```

Для просмотра логов можно использовать программу *lnav*

```
sudo tail -n 100 /var/log/nextcloud.log
```

```
sudo tail -n 100 /var/log/apache2/nextcloud.log
```

#### 13\. Удалить лишние символы index.php из URL всех ссылок

```
sudo vim /var/www/nextcloud/config/config.php
```

>   'htaccess.RewriteBase' => '/',

```
sudo -u www-data php /var/www/nextcloud/occ maintenance:update:htaccess
```

#### 14\. Проверить и поправить, если изменились права доступа к config.php

```
sudo chmod 660 /var/www/nextcloud/config/config.php
sudo chown root:www-data /var/www/nextcloud/config/config.php
```

# Установка Collabora office

#### 0\. Создать DNS запись

Cоздать DNS A-запись   
**collabora.mydomain.ru = IP вашего nextcloud**

#### 1\. Установка

В web интерфейсе *ЛКМ аватар - Приложения*  и установить приложение *Nextcloud Office*

#### 2\. Импорт ключа официального репозитория

```
sudo apt install gnupg
cd /usr/share/keyrings
sudo wget https://collaboraoffice.com/downloads/gpg/collaboraonline-release-keyring.gpg
```

#### 3\. Добавить репозиторий

```
sudo vim /etc/apt/sources.list.d/collaboraonline.sources

Types: deb
URIs: https://www.collaboraoffice.com/repos/CollaboraOnline/CODE-ubuntu2204
Suites: ./
Signed-By: /usr/share/keyrings/collaboraonline-release-keyring.gpg
```

#### 4\. Установить необходимые пакеты

```
sudo apt update
sudo apt install coolwsd code-brand hunspell collaboraoffice*
```

#### 5\. Настройка Apache для Collabora

```
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/collabora.conf
```

```
sudo vim /etc/apache2/sites-available/collabora.conf
```

\- настроить **ServerName** (collabora.mydomain.ru)  
\- удалить **ServerAdmin** и **DocumentRoot**

```
sudo a2ensite collabora
sudo systemctl reload apache2
```

#### 6\. Получить SSL сертификат с помощью certbot

```
sudo certbot --apache
```

#### 7\. Включить модули apache

```
sudo a2enmod proxy proxy_wstunnel proxy_http proxy_connect
```

#### 8\. Настроить SSL конфиг Collabora на локальную работу

```
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

#### 9\. Настройка файлов логов

```
sudo touch /var/log/apache2/collabora-error.log
sudo touch /var/log/apache2/collabora-access.log

sudo chown root:cool /var/log/apache2/collabora-error.log
sudo chown root:cool /var/log/apache2/collabora-access.log
```

#### 10\. Перезапустить apache

```
sudo systemctl restart apache2
```

#### 11\. Настроить конфиг Collabora

```
sudo vim /etc/coolwsd/coolwsd.xml
```

  Оставить только нужные языки из **"de_DE en_GB en_US es_ES fr_FR it nl pt_BR pt_PT ru"** например **"ru en_US"**

```
sudo coolconfig set ssl.enable false
sudo coolconfig set ssl.termination true
```

#### 12\. Перезапустить Collabora

```
sudo systemctl restart coolwsd
sudo systemctl status coolwsd
```

#### 13\. Включить Collabora в Nextcloud

В web интерфейсе *ЛКМ аватар - Параметры сервера - Офис* и в окне *URL-адрес (и порт) сервера документов Collabora Online* прописываем адрес нажего сервера:

**https://collabora.mydomain.ru/**

В опцию *Allow list for WOPI requests* нужно вписать IP вашего сервера nextcloud/collabora.

#### 14\. Возможные проблемы и их решение

Много полезной информации можно найти по этой ссылке:

[https://sdk.collaboraonline.com/docs/installation/Collabora_Online_Troubleshooting_Guide.html](https://sdk.collaboraonline.com/docs/installation/Collabora_Online_Troubleshooting_Guide.html￼journalctl)

Так же можно почитать логи при помощи этих команд:

```
journalctl -e -u coolwsd | lnav
```

```
journalctl -r -u coolwsd | lnav
```
