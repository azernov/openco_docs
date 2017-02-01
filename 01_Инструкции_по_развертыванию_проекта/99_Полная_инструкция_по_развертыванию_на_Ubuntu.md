## Инструкция по развертыванию проекта на ОС Ubuntu

Для удобства, вы можете скопировать содержимое этой страницы и сделать замену в текстовом редакторе следующих строк:
[your_user] - замените на имя пользователя системы

[your_user_group] - замените на основную группу пользователя (например www-data)

[your_site_name] - замените на полное название сайта (например blabla.example.com)

[your_db_name] - замените на имя базы данных

[your_db_user] - замените на имя пользователя БД (не root)

[your_db_user_password] - замените на пароль пользователя БД

***

Все операции выполняем от имени root:

```
su
```

или

```
sudo -s
```

### Добавляем новые репозитории для пакетов системы

Открываем файл /etc/apt/sources.list и добавляем в нем следующие строки:

```
#nginx
deb http://nginx.org/packages/ubuntu/ [your_ubuntu_codename] nginx
deb-src http://nginx.org/packages/ubuntu/ [your_ubuntu_codename] nginx

#php
deb http://ppa.launchpad.net/ondrej/php/ubuntu [your_ubuntu_codename] main
deb-src http://ppa.launchpad.net/ondrej/php/ubuntu [your_ubuntu_codename] main

#mysql-5.7
deb http://ppa.launchpad.net/ondrej/mysql-5.7/ubuntu [your_ubuntu_codename] main
deb-src http://ppa.launchpad.net/ondrej/mysql-5.7/ubuntu [your_ubuntu_codename] main
```

Codename для вашей версии ubuntu можно посмотреть, набрав команду:

```
lsb_release -a
```

### Обновляем все пакеты системы

```
apt-get update
```

Затем

```
apt-get dist-upgrade
```

Дожидаемся завершения операций

### Устанавливаем новые пакеты, необходимые для проекта

```
apt-get install nginx mysql-server-5.7 mysql-client-5.7 libmysqlclient20 ssl-cert php5.6 php5.6-common php5.6-curl php5.6-gd php-imagick php5.6-imap php5.6-mcrypt php5.6-memcache php5.6-mysql php5.6-pspell php5.6-recode php5.6-tidy php5.6-xmlrpc php5.6-xsl php5.6-mbstring php5.6-fpm php-gettext
```

### Добавляем нового пользователя в систему, под которым будет работать наш проект

```
useradd -d /home/[your_user] -g [your_user_group] -m -s /bin/bash [your_user]
usermod -a -G www-data [your_user]
usermod -a -G sudo [your_user]
passwd [your_user]
```

После последней операции укажите пароль для пользователя [your_user]

### Конфигурирование php

Делаем копию файла /etc/php/5.6/fpm/pool.d/www.conf и называем наш новый файл [your_site_name].conf

```
cp /etc/php/5.6/fpm/pool.d/www.conf /etc/php/5.6/fpm/pool.d/[your_site_name].conf
```

Открываем его на редактирование и меняем (или добавляем) в нем следующие настройки:

```
[[your_site_name]]
listen = /run/php/php5.6-[your_user].sock
listen.mode = 0666
user = [your_user]
group = [your_user_group]

php_admin_value[upload_tmp_dir] = /home/[your_user]/tmp
php_admin_value[date.timezone] = Europe/Moscow
php_admin_value[open_basedir] = /home/[your_user]/[your_site_name]
php_admin_value[post_max_size] = 512M
php_admin_value[upload_max_filesize] = 512M
php_admin_value[cgi.fix_pathinfo] = 0
php_admin_value[short_open_tag] = On
php_admin_value[memory_limit] = 512M
php_admin_value[session.gc_probability] = 1
php_admin_value[session.gc_divisor] = 100
php_admin_value[session.gc_maxlifetime] = 28800;

# тут значения можно менять, в зависимости от нагрузки на сайт
pm = dynamic
pm.max_children = 10
pm.start_servers = 2
pm.min_spare_servers = 2
pm.max_spare_servers = 4
```

### Конфигурирование nginx

Открываем файл /etc/nginx/nginx.conf и в секции http дописываем:

```
client_max_body_size 512m;
```

Создаем файл /etc/nginx/conf.d/[your_site_name].conf (не важно, какое имя, главное - расширение .conf).

Пишем в него следующий конфиг (без поддержки https):

```
upstream backend-[your_user] {
    server unix:/run/php/php5.6-[your_user].sock;
}

server {
    listen [::]:80;
    listen 80;
    server_name [your_site_name];
    access_log /home/[your_user]/logs/nginx_access.log;
    error_log /home/[your_user]/logs/nginx_error.log;

    gzip on;
    # Минимальная длина ответа, при которой модуль будет жать, в байтах
    gzip_min_length 1000;
    # Разрешить сжатие для всех проксированных запросов
    gzip_proxied any;
    # MIME-типы которые необходимо жать
    gzip_types text/plain application/xml application/x-javascript text/javascript text/css text/json;
    # Запрещает сжатие ответа методом gzip для IE6 (старый вариант gzip_disable "msie6";)
    gzip_disable "MSIE [1-6]\.(?!.*SV1)";
    # Уровень gzip-компрессии
    gzip_comp_level 6;


    root /home/[your_user]/[your_site_name]/www;
    index index.php;

    location ~ ^/core/.* {
        deny all;
        return 403;
    }

    location / {
        try_files $uri $uri/ @rewrite;
    }
    location @rewrite {
        rewrite ^/(.*)$ /index.php?q=$1;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass backend-[your_user];
    }


    location ~* \.(jpg|jpeg|gif|png|css|zip|tgz|gz|rar|bz2|doc|xls|exe|pdf|ppt|tar|wav|bmp|rtf|swf|ico|flv|txt|docx|xlsx)$ {
        try_files $uri @rewrite;
        access_log off;
        expires 10d;
        break;
    }

    location ~ /\.ht {
        deny all;
    }

}
```

Если мы хотим добавить поддержку ssl с принудительным редиректом на https при обращении по http,
необходимо в файл [your_site_name].conf вписать следующий конфиг:

```
upstream backend-[your_user] {
    server unix:/run/php/php5.6-[your_user].sock;
}

server {
    listen [::]:80;
    listen 80;
    server_name [your_site_name];
    location / {
        #Редиректим на https
        rewrite ^(.*)$ https://[your_site_name]$1 permanent;
    }
}

server {
    listen [::]:443;
    listen 443;
    ssl on;
    server_name [your_site_name];

    # Если ваш приватный ключ защищен паролем, то для автоматического ввода пароля
    # создайте файл /etc/keys/keyfile.pass и поместите в него пароль,
    # и раскомментируйте строку ниже
    #ssl_password_file /etc/keys/keyfile.pass;

    # Предварительно нужно положить файлы сертификата и ключа в папку /home/[your_user]/ssl
    ssl_certificate /home/[your_user]/ssl/ssl-bundle.crt;
    ssl_certificate_key /home/[your_user]/ssl/private.key;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:ECDHE-RSA-AES128-GCM-SHA256:AES256+EECDH:DHE-RSA-AES128-GCM-SHA256:AES256+EDH:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
    ssl_prefer_server_ciphers on;

    access_log /home/[your_user]/logs/nginx_access.log;
    error_log /home/[your_user]/logs/nginx_error.log;

    gzip on;
    # Минимальная длина ответа, при которой модуль будет жать, в байтах
    gzip_min_length 1000;
    # Разрешить сжатие для всех проксированных запросов
    gzip_proxied any;
    # MIME-типы которые необходимо жать
    gzip_types text/plain application/xml application/x-javascript text/javascript text/css text/json;
    # Запрещает сжатие ответа методом gzip для IE6 (старый вариант gzip_disable "msie6";)
    gzip_disable "MSIE [1-6]\.(?!.*SV1)";
    # Уровень gzip-компрессии
    gzip_comp_level 6;


    root /home/[your_user]/[your_site_name]/www;
    index index.php;

    location ~ ^/core/.* {
        deny all;
        return 403;
    }

    location / {
        try_files $uri $uri/ @rewrite;
    }
    location @rewrite {
        rewrite ^/(.*)$ /index.php?q=$1;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass backend-[your_user];
    }


    location ~* \.(jpg|jpeg|gif|png|css|zip|tgz|gz|rar|bz2|doc|xls|exe|pdf|ppt|tar|wav|bmp|rtf|swf|ico|flv|txt|docx|xlsx)$ {
        try_files $uri @rewrite;
        access_log off;
        expires 10d;
        break;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### Конфигурирование mysql

Открываем файл /etc/mysql/mysql.conf.d/mysqld.cnf и в секции [mysqld] в конце дописываем:

```
sql_mode=NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

#### Добавление БД и пользователя для проекта

Подключитесь к серверу mysql:

```
mysql -u root -p
```

Введите пароль для root, указанный при установке пакета mysql

Далее после успешного подключения, выполните следующие команды:

```
CREATE DATABASE IF NOT EXISTS [your_db_name];
CREATE USER '[your_db_user]'@'localhost' IDENTIFIED BY '[your_db_user_password]';
GRANT USAGE ON *.* TO '[your_db_user]'@'localhost' IDENTIFIED BY '[your_db_user_password]' REQUIRE NONE WITH MAX_QUERIES_PER_HOUR 0 MAX_CONNECTIONS_PER_HOUR 0 MAX_UPDATES_PER_HOUR 0 MAX_USER_CONNECTIONS 0;
GRANT ALL PRIVILEGES ON `[your_db_name]`.* TO '[your_db_user]'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

**Запоминанаем имя базы данных, имя пользователя и пароль.**


### Клонирование репозитория и перезапуск сервисов

Все действия ниже выполняются от имени пользователя [your_user]

1. Создайте папку /home/[your_user]/tmp
2. Создайте папку /home/[your_user]/logs
3. Находясь в домашнем каталоге выполните:
    ```
    git clone [пользователь]@[адрес репозитория]:[имя репозитория] [your_site_name]
    cd [your_site_name]
    git checkout production
    ```

4. Перезапускаем сервисы:
    ```
    sudo service php5.6-fpm restart
    sudo service nginx restart
    sudo service mysql restart
    ```

### Загрузка БД

В каталоге проекта есть файл db/dbsample/sample_db.php, который содержит начальное состояние БД
с тестовыми данными. Необходимо импортировать этот файл в нашу БД.

Выполняем действия под пользователем [your_user]:

```
cd ~/[your_site_name]
mysql -u [your_db_user] -p [your_db_name] < db/dbsample/sample_db.sql
```

Далее введите пароль, указанный для пользователя [your_db_user] на этапе конфигурирования mysql

Если никаких ошибок при импорте не будет, команда завершится без каких-либо сообщений

### Инициализация конфигов проекта

Откройте файл ~/[your_site_name]/www/core/config/config.inc.php и поменяйте настройки подключения к БД:

```
...
$database_type = 'mysql';
$database_server = 'localhost';
$database_user = '[your_db_user]';
$database_password = '[your_db_user_password]';
$database_connection_charset = 'utf8';
$dbase = '[your_db_name]';
$table_prefix = 'modx_';
$database_dsn = 'mysql:host=localhost;dbname=[your_db_name];charset=utf8';
...
```

### Проверка

Если все было сделано правильно, сайт должен открываться по привязанному адресу [your_site_name]