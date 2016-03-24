---
layout: post
title:  "Настройка nginx и php-fpm в Debian"
date:   2016-03-23 02:10:17 +0300
categories: devops
---
## Настрока окружения
Для тех, кто захочет повторить на практике, можете повторять за мной на виртуальной машине.

Создаем пустую виртуальную машину с образом 64-х битного debian jessie.
`vagrant init debian/jessie64`

Запускаем:
`vagrant up`

Если используете vagrant перный раз, возможно придется подождать некоторое время, пока образ скачается.

Заходим на машину: `vagrant ssh`

## Обновляем список пакетов:
`sudo apt-get update`

## Удаляем ненужное
Первым делом необходимо удалить пакет apache, чтобы в дальнейшем избежать проблем с занятостью портов. Даже если вы его не устанавливали, он мог быть подтянут как зависимость к какому-нибудь пакету использующему веб-интерфейс.
`sudo apt-get remove --purge apache2`

## Ставим php и php-fpm
Так как пакет php5
 имеет в зависимостях `libapache2-mod-php5`, `libapache2-mod-php5filter`, `php5-cgi`, или `php5-fpm` нужно устанавливать `php5-fpm` первым иначе как зависимость снова подтянется апач.

 `sudo apt-get install php5-fpm php5`
 
## Устанавливаем nginx
В стандартном репозитории nginx старый, поэтому заранее подключаем официальный репозиторий nginx:

1. скачиваем ключ для проверки подписи `wget http://nginx.org/keys/nginx_signing.key`

2. импортируем `sudo apt-key add nginx_signing.key`

3. добавляем репы `echo 'deb http://nginx.org/packages/debian/ jessie nginx' | sudo tee -a /etc/apt/sources.list`
`echo 'deb-src http://nginx.org/packages/debian/ jessie nginx' | sudo tee -a /etc/apt/sources.list`

4. снова обновляем список пакетов
`sudo apt-get update`

5. ставим сам nginx
`sudo apt-get install nginx`

## Создаем тестовый сайт
Размещать проекты будем в папке /srv/ так, что создаем папку для проекта

`sudo mkdir /srv/demo/`

и папку для логов

`sudo mkdir /var/log/nginx/demo/`

Даем доступ пользователю и группе www-data

`sudo chown -R www-data:www-data /srv/demo`

`sudo chmod -R g+w  /srv/demo`

Чтобы самому иметь доступ к проекту, добавляем свой аккаунт в группу www-data

`sudo gpasswd -a $(whoami) www-data`

После этого **обязательно необходимо перелогиниться в системе**.
 
Затем создаем тестовый скрипт с единственной строчкой:
    
    <?php phpinfo();
   

`echo '<?php phpinfo();' > /srv/demo/index.php`
 
Создаем директорию и файлы для логов

`sudo mkdir /var/log/nginx/demo/`
 
`sudo touch /var/log/nginx/demo/access.log`
 
`sudo touch /var/log/nginx/demo/error.log`
 
 Удаляем ненужные стандартные конфиги
 
`sudo rm /etc/nginx/conf.d/example_ssl.conf`
 
`sudo rm /etc/nginx/conf.d/default.conf`
 
 
 Теперь необходимо изменить пользоваетеля, от которого запускается nginx, чтобы он мог без проблем работать с php-fpm.
 Приводим /etc/nginx/nginx.conf к такому виду
{% highlight nginx %}
user  www-data;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
{% endhighlight %}
 
 Создаем конфиг для сайта
 /etc/nginx/conf.d/demo.conf
 
{% highlight nginx %}
server {
    listen 80;
    #server_name demo.com;
    access_log /var/log/nginx/demo/access.log;
    error_log /var/log/nginx/demo/error.log;
    index index.php index.html index.htm;
    root /srv/demo;

    location / {
        try_files $uri $uri/ /index.php$uri?$args;
    }

    location ~ \.php {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        if (!-f $document_root$fastcgi_script_name) {
            return 404;
        }
        fastcgi_index  index.php;
        fastcgi_pass   'unix:/var/run/php5-fpm.sock';
        include        fastcgi_params;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        expires max;
        access_log off;
    }
}
    
{% endhighlight %}
Директива `server_name` закомментирована, чтобы можно было обрашаться к сайту по айпишнику, на продакшене обязательно расскоментируйте её, и установите имя домена.

Приводим /etc/nginx/fastcgi_params к виду

{% highlight nginx %}
    fastcgi_param  QUERY_STRING       $query_string;
    fastcgi_param  REQUEST_METHOD     $request_method;
    fastcgi_param  CONTENT_TYPE       $content_type;
    fastcgi_param  CONTENT_LENGTH     $content_length;
    
    fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
    fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
    fastcgi_param  PATH_INFO          $fastcgi_path_info;
    fastcgi_param  PATH_TRANSLATED    $document_root$fastcgi_path_info;
    fastcgi_param  REQUEST_URI        $request_uri;
    fastcgi_param  DOCUMENT_URI       $document_uri;
    fastcgi_param  DOCUMENT_ROOT      $document_root;
    fastcgi_param  SERVER_PROTOCOL    $server_protocol;
    fastcgi_param  HTTPS              $https if_not_empty;
    
    fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
    fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;
    
    fastcgi_param  REMOTE_ADDR        $remote_addr;
    fastcgi_param  REMOTE_PORT        $remote_port;
    fastcgi_param  SERVER_ADDR        $server_addr;
    fastcgi_param  SERVER_PORT        $server_port;
    fastcgi_param  SERVER_NAME        $server_name;
    
    # PHP only, required if PHP was built with --enable-force-cgi-redirect
    fastcgi_param  REDIRECT_STATUS    200;
{% endhighlight %}

Рестартуем сервисы и проверяем, что все работает

    sudo service php5-fpm restart
    sudo service nginx restart
    
Открываем в браузере ip вашего сервера или localhost если настраивали локально
![Настрока nginx+php-fpm](/img/nginx+php-fpm.png)
