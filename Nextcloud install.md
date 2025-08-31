# Nextcloud guide

## Nextcloud install

### Установить UbuntuServer 

### Обновить и очистить пакеты
```bash
sudo apt update && sudo apt upgrade -y && sudo apt autoremove
```

### Настроить hostname и добавим имя в hosts файл _(в данном примере "cloud")_
```bash
echo "cloud" > /etc/hostname
sudo sed -i 's/^127\\.0\\.1\\.1.\*/127.0.1.1 cloud/g' /etc/hosts
```

### Настроим timezone (вот несколько на выбор)
```
sudo timedatectl set-timezone Europe/Moscow
sudo timedatectl set-timezone Europe/Kyiv
sudo timedatectl set-timezone Europe/Minsk
sudo timedatectl set-timezone Asia/Almaty
```

### Настроить сетевые интерфейсы
Делаем бекап конфига
```bash
sudo cp -a /etc/netplan/50-cloud-init.yaml{,.orig}
```
Смотрим имя интерфейса
```
ip -c -br a
```
Редактируем конфиг
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
Тестово применяем конфиг и проверяем соединение за 15 секунд теста
```bash
sudo netplan try --timeout 15
```
Если тест прошел успешно, то применяем конфиг
```bash
sudo netplan apply
```
При необходимости добавляем ip в /etc/hosts (делаем это только если у вас есть понимание с какой целью)
```
192.168.1.100 cloud.mydom.com collabora.mydom.com
```

### Создаем и настраиваем пользователя
```bash
sudo adduser vol
usermod -aG sudo vol
usermod -aG www-data vol
```

### Настраиваем ssh по ключу
#### Если у вас windows установить openssh командой
Установить openssh командой
```powershell
winget install openssh
```
или командой
```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```
Настроить запуск ssh агента
вручную (более безопасно)
```powershell
Get-Service -Name ssh-agent | Set-Service -StartupType Manual
```
автоматически при запуске ОС
```powershell
Get-Service -Name ssh-agent | Set-Service -StartupType Automatic
```
проверить статус агента
```powershell
Get-Service ssh-agent | Select Name, Status, StartType, DisplayName
```

#### Если у вас Linux устанавливаем openssh командой
```bash
sudo apt install openssh
```
активировать и ключить демона ssh
```bash
sudo systemctl enable ssh --now
```
проверить статус демона
```bash
sudo systemctl status ssh
```

#### далее инструкция подходит и для windows и для linux
Создаем папку .ssh и переходим в неё
```
mkdir ~/.ssh
cd ~/.ssh
```
Создаем открытый и закрытый ключ
```
ssh-keygen -t rsa -b 4096
```
Смотрим названия файлов
```
ls
```
Добавить ключ в SSH агент (НЕ тот что .pub, а тот что без расширения)
```
ssh-add c:/Users/admin/.ssh/key
```

создание папки ssh на удаленном сервере по шаблону:
ssh <remote_username>@<server_ip_address> mkdir -p .ssh
```bash
ssh vol@12.34.56.100 mkdir -p .ssh
```
скопировать публичный ключ на удаленный сервер
scp .\<key.pub> <remote_username>@<server_ip_address>:/home/vol/.ssh/<key.pub>
```bash
scp ~/.ssh/key.pub vol@12.34.56.100:/home/vol/.ssh/key.pub
```
подключится к удаленному серверу по ssh
```
ssh vol@12.34.56.100
```
экспорт публичного ключа в uthorized_keys
```
cat ~/.ssh/key.pub >> ~/.ssh/authorized_keys
```
отключиться и проверить что подключение происходит без пароля
```
exit
```
```
ssh vol@12.34.56.100
```
Если запрашивает пароль, то что-то не так, возможно дело в правах на authorized_keys. Права на на сервере на директорию .ssh и файл authorized_keys должны быть "600"

### Создаем доменную запись
Необходимо создать доменную "A" запись на доменном хостинге который вы приобрели или на локальном доменном сервере если не планируете использовать https. В данной инструкции это будет **cloud.mydomain.com**

https://www.google.com/search?q=купить+домен

### Установка и настройка базы данных
Устанавливаем mariadb-server
```bash
sudo apt install mariadb-server -y && systemctl status mariadb
```
Если вы планируете расположить базу данных отдельно от web сервера то на web сервер достаточно установить клиент.
```bash
sudo apt install mariadb-client -y
```

### Настройки безопасности MariaDB
Ubuntu 24.04
```bash
sudo mysql_secure_installation
```
Ubuntu 25.04
```bash
sudo mariadb-secure-installation
```
Далее выбираем следующие параметры:
- Enter current password for root (enter for none): none
- Switch to unix_socket authentication [Y/n] : n
- Change the root password? [Y/n]: y
- Remove anonymous users? [Y/n]: y
- Disallow root login remotely? [Y/n]: y
- Remove test database and access to it? [Y/n]: y
- Reload privilege tables now? [Y/n]: y

### Настройки конфига MariaDB
Открываем файл конфига mariadb
```bash
sudo vim /etc/mysql/my.cnf
```
И добавляем строки ниже, которые отсутствуют в файле
```
[mysqld]
log_error = /var/log/mysql/error.log
general_log = 0 # 0-disable log, 1-enable log
general_log_file = /var/log/mysql/general.log
slow_query_log = 0 # 0-disable log, 1-enable log
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
max_connections = 500
wait_timeout = 300
interactive_timeout = 300

innodb_buffer_pool_size = 2G
# For stable connections
wait_timeout = 28800
interactive_timeout = 28800
# For larger queries/uploads
max_allowed_packet = 64M
```
Создаем директорию с логами и лог файлы и меняем владельца
```bash
sudo mkdir /var/log/mysql
sudo touch /var/log/mysql/error.log
sudo touch /var/log/mysql/general.log
sudo touch /var/log/mysql/slow.log

sudo chown -R mysql:mysql /var/log/mysql
```

### Создание базы nextcloud
Заходим в базу mariadb
```bash
sudo mariadb
```
Создаем базу
```sql
CREATE DATABASE nextcloud;
```
Смотрим что база создалась
```sql
SHOW DATABASES;
```
Создаем пользователя и выдаем ему полные права на базу nextcloud при обращении с localhost. Не забываем выдать свой пароль, важно чтобы он был сложный и желательно из случайных символов. Он будет хранится в /var/www/nextcloud/config/config.php
```sql
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost' IDENTIFIED BY 'mypassword';
```
Если забыли поменять пароль, то это можно сделать следующей командой
```sql
ALTER USER 'nextcloud'@'localhost' IDENTIFIED BY 'new_password';
```
Обновляем права
```sql
FLUSH PRIVILEGES;
```
Выходим из mariadb
```sql
exit
```

### Установка WEB сервера
Устанавливаем необходимые пакеты
```bash
sudo apt install vim php php-apcu php-bcmath php-cli php-common php-curl php-gd php-gmp php-imagick php-intl php-mbstring php-mysql php-zip php-xml -y 
```
Проверяем работу apache, он должен установится как зависимость к пакетам выше
```bash
sudo systemctl status apache2
```

### Активируем PHP расширения
```bash
sudo phpenmod bcmath gmp imagick intl
```

### Скачиваем пакет nextcloud
Последнюю версию
```bash
wget https://download.nextcloud.com/server/releases/latest.zip
```
Или если вам необходима предыдущая версия мо можно скачать с сайта
https://nextcloud.com/changelog/
например
```bash
https://download.nextcloud.com/server/releases/nextcloud-31.0.7.zip

mv nextcloud-31.0.7.zip latest.zip
```

### Установка nextcloud
Установить unzip если не установлен
```bash
sudo apt install unzip
```
Распаковать архив и переместить в нужную директорию
```bash
unzip latest.zip
sudo mv nextcloud /var/www/
```
Освобождаем место удаляя архив
```bash
rm latest.zip
```

### Редактируем права и владельца всей директории nextcloud
Смена владельца на пользователя www-data и группу www-data (в которую мы ранее для удобства добавили нашего пользователя)
```bash
sudo chown -R www-data:www-data /var/www/nextcloud/
```
Смена прав на директории
```bash
find /var/www/nextcloud/ -type d -exec chmod 750 {} \;
```
Смена прав на файлы
```bash
find /var/www/nextcloud/ -type f -exec chmod 640 {} \;
```

### Отключаем дефолтный apache сайт
```bash
sudo a2dissite 000-default.conf
```

### Настроиваем сайт nextcloud
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

### Активируем сайт nextcloud
```bash
sudo a2ensite nextcloud.conf
```

### Правим конфиг apache
#### Ubuntu 24.04
Смотрим текущие параметры важных настроек
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
Заменяем параметры по умолчанию на требуемые для nextcloud
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
Настраиваем memory_limit. Для стандартного конфига достаточно 512M:
```bash
sudo sed -i 's/memory_limit = 128M/memory_limit = 512M/g' /etc/php/8.3/apache2/php.ini
```
А если у вас есть достаточно оперативной памяти(указанный объем будет выделяться на каждого пользователя) и вы собираетесь просматривать RAW изображения то можете установить 2048M:
```bash
sudo sed -i 's/memory_limit = 128M/memory_limit = 2048M/g' /etc/php/8.3/apache2/php.ini
```
Снова проводим проверку параметров первой командой из этого пункта, вывод должен быть следующим:
```
memory_limit = 512M #(или 2048M)
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

#### Ubuntu 25.04
Смотрим текущие параметры важных настроек
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
Заменяем параметры по умолчанию на требуемые для nextcloud
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
Настраиваем memory_limit. Для стандартного конфига достаточно 512M:
```bash
sudo sed -i 's/memory_limit = 128M/memory_limit = 512M/g' /etc/php/8.4/apache2/php.ini
```
А если у вас есть достаточно оперативной памяти(указанный объем будет выделяться на каждого пользователя) и вы собираетесь просматривать RAW изображения то можете установить 2048M:
```bash
sudo sed -i 's/memory_limit = 128M/memory_limit = 2048M/g' /etc/php/8.4/apache2/php.ini
```
Снова проводим проверку параметров первой командой из этого пункта, вывод должен быть следующим:
```
memory_limit = 512M #(или 2048M)
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

### Перезапускаем apache
Ещё раз убеждаемся что модификаторы apache включены и исправны и перезапускаем apache
```bash
sudo a2enmod dir env headers mime rewrite ssl
sudo systemctl restart apache2
```

### Переходим на сайт и завершаем первичную установку
http://12.34.56.100

### Устанавливаем дополнительный софт
Здесь собран софт для функционирования nextcloud, популярных nextcloud плагинов и базового дебага linux
```bash
sudo apt install libapache2-mod-php imagemagick ffmpeg php-bz2 php-redis redis-server redis-server php-redis cron ncdu lnav net-tools iotop htop snap -y
```

### Правим trusted_domains и overwrite.cli.url
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

### Дополнительные настройки
Добавляем эти параметры в конфиг
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

### Делаем url ссылок короче
```bash
sudo vim /var/www/nextcloud/config/config.php
```
Удаляем "index.php" из url
```
  'htaccess.RewriteBase' => '/',
```
Или можно просто выполнить occ команду
```bash
sudo php /var/www/nextcloud/occ maintenance:update:htaccess
```

### Убираем ошибку Image Magick error
```bash
sudo apt install libmagickcore-7.q16-10-extra
```
Если установка не успешна то вбиваете команду ниже и нажимаете **Tab**. После этого выбираете пакет из доступных с пометкой extra
```bash
sudo apt install --simulate libmagickcore-
```

### Настроить сертификат
#### Настройка через certbot
Действуем по инструкции для apache через snap с сайта
https://certbot.eff.org/instructions

После выполнения инструкции и успешного получения сертификата в ssl конфиг файл добавляем строку:
```
<IfModule mod_ssl.c>
<VirtualHost *:443>
  ............
  Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
  ............
</VirtualHost>
</IfModule>
```

#### Настройка вручную
На сервере создаем директорию ssl
```bash
sudo mkdir /etc/apache2/ssl
```
Меняем владельца на пользователя, через которого бум копировать файлы
```bash
sudo chown vol: /etc/apache2/ssl
```
Отключаемся от сервера, в терминале на локальном пк переходи в папку с сертификатом и ключом и копируем их на сервер.
```
scp ./ssl/mydomain.* vol@cloud.mydomain.ru:/etc/apache2/ssl
```
Снова подключаемся на сервер и возвращаем права в норму
```bash
sudo chown -R root: /etc/apache2/ssl
```
Далее создаем ssl конфиг apache
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
Активируем модуль ssl в apache
```bash
sudo a2enmod ssl
```
Активируем только что созданный конфиг https
```bash
sudo a2ensite nextcloud-ssl.conf
```
Перезапускаем apache
```bash
sudo systemctl reload apache2
```

### Редирект с 80 порта на 443 порт
Создаем конфиг
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
Перезапускаем apache
```
sudo systemctl restart apache2
```

### Настрока cron
Файл cron это основной файл отвечающий за автоматизацию всех процессов: атоматические операции с файлами, корзиной, базой данных и т.п. 
**Когда в /var/www/nextcloud/config/config.php стоит 'maintenance' => true, cron файл пропускает выполнение по расписанию.**

Установим cron если ещё не установлен
```bash
sudo apt install cron
```
Ативируем автоматический запуск вместе с системой
```bash
sudo systemctl enable cron --now
```
Отредактируем cron файл от имени пользователя www-data т.к. только ему выданы все необходимые cron файлу права
```bash
sudo -u www-data EDITOR=vim crontab -e
```
Все строки начинающиеся с # можно удалить и по итогу получится следующее:
_Рекомендуется ставить перенос строки в конце cron файла._
```
*/5  *  *  *  * php -f /var/www/nextcloud/cron.php

```
Проверяем что файл успешно записан
```bash
sudo -u www-data crontab -l
```

### Настраиваем Redis
Redis — это хранящаяся в оперативной памяти  база данных которая работает как очень быстрая «временная память», которуя используется для кеша и очередей.
В Nextcloud Redis нужен, чтобы хранить сессии пользователей и кешировать разные данные. Так сайт работает быстрее, а одновременные запросы обрабатываются без заметных задержек и конфликтов.

Устанавливаем Redis
```bash
sudo apt install redis-server php-redis
```
Даем пользователю www-data права на упраление redis
```bash
sudo usermod -aG redis www-data
```
Перезапускаем apache
```bash
sudo systemctl restart apache2
```
Настраиваем в конфиге redis следующие строки
```bash
sudo vim /etc/redis/redis.conf
```
```
unixsocket /var/run/redis/redis-server.sock
unixsocketperm 770
port 0
```
Перезапускае Redis
```bash
sudo systemctl restart redis
```
Настраиваем конфиг nextcloud под Redis
```bash
sudo vim /var/www/nextcloud/config/config.php
```
Удаляем строку
```
  'memcache.local' => '\\OC\\Memcache\\APCu',
```
Добавляем строки
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

### Настраиваем fail2ban
Устанавливаем fail2ban
```bash
sudo apt install fail2ban
```
Копируем конфиги чтобы они не стерлись при обновлении
```bash
sudo cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Раскомментируем строку ignoreip
```bash
sudo vim /etc/fail2ban/jail.local
```
```
ignoreip = 127.0.0.1/8 ::1
```
Создаем конфиг nextcloud. Актуальный конфиг можно найти на [офф сайте](https://docs.nextcloud.com/server/19/admin_manual/installation/harden_server.html?highlight=fail2ban#setup-fail2ban)
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
Создаем jail конфиг для nextcloud
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
Создаем jail конфиг для ssh
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
Создаем файлы логов
```bash
sudo touch /var/log/nextcloud.log 
sudo chown www-data:www-data /var/log/nextcloud.log
sudo chmod 660 /var/log/nextcloud.log
```
Перезапускаем fail2ban
```bash
sudo systemctl enable fail2ban && sudo systemctl restart fail2ban && sudo systemctl status fail2ban
```

### Настраиваем базовый ufw фаервол
Устанавливаем ufw
```bash
sudo apt-get install ufw 
```
Проверяем чтобы IPV6 тоже фильтровался
```bash
sudo vim /etc/default/ufw

```
```
IPV6=yes
```
Настраиваем правила ufw
```bash
sudo ufw default deny incoming
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 0.0.0.0/0 to any port 22 proto tcp
sudo ufw allow from 192.168.0.0/16 to any port 22 proto tcp
```
Проверяем правила ufw
```bash
sudo ufw status numbered
```
При необходимости
удаляем ненужное правило
```bash
sudo ufw delete <номер>
```
добавляем правило на нужную позицию
```bash
#sudo ufw insert <позиция> <правило>
#например
sudo ufw insert 1 allow from 8.8.8.8 to any
```

### Настройка логов
Добавляем необходимые строки в конфиг
```bash
sudo vim /var/www/nextcloud/config/config.php
```
```
  'log_type' => "file",
  'logfile' => '/var/log/nextcloud.log',
  'loglevel' => 1,
  'logdateformat' => "F d, Y H:i:s",
```
Создаем файлы логов
```bash
sudo touch /var/log/nextcloud.log
sudo chown -R root:www-data /var/log/nextcloud.log
sudo chmod 660 /var/log/nextcloud.log
```
Один из вариантов просмотра логов - с помощью программы lnav
```bash
sudo tail -n 100 /var/log/nextcloud.log | lnav
sudo tail -n 100 /var/log/apache2/nextcloud.log | lnav
```

### Skeleton directory
Когда создается новый пользователь, у него в nextcloud уже есть некоторый набор файлов. Они копируются из Skeleton directory по умолчанию. Если хотите настроить свой набор файлов по умолчанию или чтобы их вообще не было - нужно настроить свою Skeleton directory. **Внимание, не редактируте ничего в Skeleton directory по умолчанию т.к. неминуемо встретите проблемы при обновлении.**

Создаем директорию под Skeleton directory
```bash
sudo mkdir /opt/nextcloud_skeleton
sudo chown -R www-data:www-data /opt/nextcloud_skeleton
```
Добавляем сроку в конфиг
```bash
sudo vim /var/www/nextcloud/config/config.php
```
```
  'skeletondirectory' => '/opt/nextcloud_skeleton',
```

### Настройка автоматической очистки корзины
По умолчанию в nextcloud под когзину отводится половина свободного места, выделенного пользователю и автоматическое удаление самых старых удаленных файлов при нехватке места. При таком раскладе если вы единовременно загружаете файлы общим объемом больше половины свободного места то параллельно в загрузкой начинает происходить удаление что может замедлить процесс загрузки и нагрузить сервер. Так же это может значительно увеличивать размер бекапов.

Чтобы настроить автоматическую очистку корзины от файлов, удаленных более 30 дней назад, добавьте в конфиг эту строку:
```bash
sudo vim /var/www/nextcloud/config/config.php
```
```
  'trashbin_retention_obligation' => 'auto, 30',
```

### Починка индексов в базе
Иногда после установки или обновления требуется починить индексы в базе данных
```bash
sudo -u www-data php /var/www/nextcloud/occ maintenance:repair --include-expensive
sudo -u www-data php /var/www/nextcloud/occ db:add-missing-indices
```

### Увеличить лимит памяти
Если ранее в конфиге /etc/php/8.*/apache2/php.ini вы установили memory_limit = 2048M то нужно добавить строку в конфиг. Это необходимо например для корректного отборажения превью крупных изображений в приложении Memories.
```bash
sudo vim  /var/www/nextcloud/config/config.php
```
```
  'preview_max_memory' => 2048,
```

## Collabora install

### Создать dns запись
Создаем доменную "A" запись на том-же хостинге где делали cloud.mydomain.com
В данном примере это будет **collabora.mydomain.com**

### Установка Nextcloud Office
В web интерефейсе идем в Apps и устнавливаем там приложение "Nextcloud Office"

### Импорт ключа официального репозитория
Установим gnupg
```bash
sudo apt install gnupg
```
Скачаем ключи репозитория collabora ([ссылка](https://sdk.collaboraonline.com/docs/installation/Installation_from_packages.html))
```bash
cd /usr/share/keyrings
sudo wget https://collaboraoffice.com/downloads/gpg/collaboraonline-release-keyring.gpg
```

### Добавим репозиторий в /etc/apt/sources.list.d
```bash
sudo cat << EOF > /etc/apt/sources.list.d/collaboraonline.sources
Types: deb
URIs: https://www.collaboraoffice.com/repos/CollaboraOnline/24.04/customer-deb-$customer_hash
Suites: ./
Signed-By: /usr/share/keyrings/collaboraonline-release-keyring.gpg
EOF
```

### Установка Collabora
```bash
sudo apt update && sudo apt install coolwsd collabora-online-brand
```

### Настройка Apache для Collabora
Создаем шаблон конфига
```bash
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/collabora.conf
```
Редактируем файл (закомментированные строки при желании можно удалить).
```bash
sudo vim /etc/apache2/sites-available/collabora.conf
```
```
        ServerName collabora.mydomain.com
```
```
        # Удалить или закомментировать строки:
        ServerAdmin
        DocumentRoot
```
Активируем конфиг collabora в apache
```bash
sudo a2ensite collabora && sudo systemctl reload apache2
```

### Включаем модули apache
```bash
sudo a2enmod proxy proxy_wstunnel proxy_http proxy_connect
```

### Получить SSL сертификат
#### Через certbot
Используем команду certbot
```bash
sudo certbot --apache
```
В конфиг файл, созданны certbot'ом добавляем следующие строки:
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

#### Настройка сертификатов вручную
Создаем конфиг файл. В данной инструкции предполагается что у вас сертификаты подходят на весь домен (*.mydoamin.com). Если у вас отдельные ключ и сертификат для collabora, вам необходимо подгрузить их на сервер и ипоменять к ним пути в конфиге ниже и выдать права как и у сертификатов cloud.
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
Добавим в этот файл редирект на пустую страницу чтобы не отобразалась стартовая страниц apache при входе на сайт
```bash
sudo vim /etc/apache2/sites-enabled/collabora-le-ssl.conf
```
Ативируем конфиг в apache
```bash
sudo a2ensite collabora-ssl.conf
```

#### Для обоих вариантов
Добавим в этот файл редирект на пустую страницу чтобы не отобразалась стартовая страниц apache при входе на сайт
```bash
sudo vim /etc/apache2/sites-enabled/collabora-le-ssl.conf
```
```
        Redirect permanent / https://collabora.mydomain.com/
```
Активируем модуль ssl если не активирован
```bash
sudo a2enmod ssl
```
Перезапускаем apache
```bash
sudo systemctl reload apache2
```

### Настройка логов
Создадим файлы логов
```bash
sudo touch /var/log/apache2/collabora-error.log
sudo touch /var/log/apache2/collabora-access.log
```
Выдадим права на файлы логов
```bash
sudo chown root:cool /var/log/apache2/collabora-error.log
sudo chown root:cool /var/log/apache2/collabora-access.log
```

### Настроить конфиг collabora
В Collabora языки используются для оформления интерфейса collabora и для проверки орфографии. В больших текстовых файлах проверка орфографии большого кол-ва языков может замедлять производительность. Рекомендуется оставить только нужные языки.

Посметреть какие языки сейчас в конфиге
```bash
sudo grep en_US.*ru /etc/coolwsd/coolwsd.xml
```
Оставить только нужные языки (в данном примере это en_US ru)
```bash
sudo sed -i 's/de_DE en_GB en_US es_ES fr_FR it nl pt_BR pt_PT ru/en_US ru/g' /etc/coolwsd/coolwsd.xml
```
Убедиться что строка изменилась корректно
```bash
sudo grep en_US.*ru /etc/coolwsd/coolwsd.xml
```
Отключаем шифрование трафика collabora т.к. при таком /etc/redis/redis.conf обмен данными будет идти через сокет локально
```bash
sudo coolconfig set ssl.enable false
sudo coolconfig set ssl.termination true
```

### Перезапускаем collabora и apache
```bash
sudo systemctl restart apache2
sudo systemctl status apache2
sudo systemctl restart coolwsd
sudo systemctl status coolwsd
```

### Включить Collabora в Nextcloud
На сайте переходим в меню "Администрирование" и находим там вкладку "Office".
Выбираем меню "Использовать свой сервер" и вводим 
```
https://collabora.mydomain.com
```
Ниже находим меню "Allow list for WOPI requests" и вводим ip адрес сервера (в данной инструкции он знаком вам как 12.34.56.100)

### Траблшут collabora
https://sdk.collaboraonline.com/docs/installation/Collabora_Online_Troubleshooting_Guide.html
```bash
journalctl -e -u coolwsd | lnav
journalctl -r -u coolwsd | lnav
```

## Настройка LDAP

## Обновление сервера

## Перенос сервера

## Смена DNS имени







