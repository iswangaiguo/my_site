---
title: 使用Docker安装WordPress
date: 2021-08-29T14:43:01+00:00
categories:
  - Docker
  - Nginx
  - 编程
tags:
  - docker
  - Nginx
  - wordpress

---
一般而言，如果服务器只需要部署Wordpress服务，采用Docker Compose是比较好的安装方式，写好Yml文件后，直接启动即可，但是如果需要在单台服务器上部署多个服务，并且进行个性化配置，则手动安装配置各个容器是比较好的选择。

## 一、创建网络

使用Docker命令创建网络：

```shell
docker network create my-net
```

使用Docker自带的网络有一个明显的缺点，即无法通过容器名访问其他容器，如果容器网络互联，则要查看具体的IP，有些繁琐。所以，创建好Docker网络是首先要做的。

## 二、安装MySQL

使用Docker安装MYSQL并配置UTF8编码

```shell
docker run --name mysql -e MYSQL_ROOT_PASSWORD=password -dp 3306:3306 mysql --net my-net --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --skip-character-set-client-handshake
```

```sql
CREATE DATABASE wordpress;
CREATE USER 'wordpress'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON `wordpress`.* TO 'wordpress'@'%';  # 授权wordpress用户所有wordpress数据库权限
FLUSH PRIVILEGES;  # 刷新权限
```

## 三、安装WordPress

使用Docker安装Wordpress

```shell
docker run --name wordpress -d --net my-net -e WORDPRESS_DB_USER=wordpress -e WORDPRESS_DB_PASSWORD="passowrd" -e WORDPRESS_DB_NAME=wordpress -e WORDPRESS_DB_HOST=mysql -v ~/wordpress/html:/var/www/html wordpress:5.8-php7.4-fpm
```

将Wordpress内容映射到～/wordpress/html下，方便Nginx直接访问静态资源文件。

这里还需要对Wordpress默认的上传文件大小进行修改，主要涉及两个部分，一个是php配置文件的修改，另一个是Nginx配置的修改。

wordpress容器内的php配置修改比较容易，首先在主机目录下新建upload.ini文件，填写上传文件的一系列配置。

```ini
upload_max_filesize = 1024M
post_max_size = 1024M
max_execution_time = 300
```

## 四、安装Nginx

最后一步就是Nginx的安装及配置

```shell
docker run --name nginx -d -p80:80 -p443:443 -v ~/nginx:/etc/nginx -v ~/wordpress/html:/var/www/wordpress  --network my-net  nginx
```

这里将wordpress映射的主机目录映射到Nginx容器中，然后对nginx.conf进行修改

```nginx
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;
    client_max_body_size 1024m;  # nginx上传大小配置

    include /etc/nginx/conf.d/*.conf;
}

server {
    listen 80;
    listen [::]:80;
    server_name localhost;

    root /var/www/wordpress;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }

    location ~ \.php$ {
        root /var/www/html;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires max;
        log_not_found off;
    }

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }
}
```

需要注意的地方是

`location ~ \.php$`下的**root**位置，它指的是wordpress容器中程序的位置，server下root的位置是wordpress挂载点对应的nginx中挂载的位置。
