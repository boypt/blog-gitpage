---
title: 用Nginx作为Webdav服务器
date: 2021-09-30 16:26:23
tags:
    - networking
    - linux
    - webdav
categories:
---

提供Webdav服务，本来有rclone 、 go-webdav等使用golang的服务端，跑起来是非常方便的，但是如果设备是CPU性能较低内存较小的比如MTK7621，就分分钟卡死整个系统。

研究后决定用nginx提供webdav。
<!--more-->

Nginx的Webdav支持是比较分裂的，一个是官方自带的(ngx_http_dav_module)[http://nginx.org/en/docs/http/ngx_http_dav_module.html]但是功能不齐全，需要一个第三方模块(nginx-dav-ext-module)[https://github.com/arut/nginx-dav-ext-module]。不过这些不管啦，Openwrt里面的`nginx-all-module`包，是全带了适合的模块了。

直接上配置。

```
dav_ext_lock_zone zone=davlock:10m;

server {
    listen       8081 default_server;
    listen       [::]:8081 default_server;
    listen       8083 default_server ssl;
    listen       [::]:8083 default_server ssl;
    ssl_certificate /etc/ssl/my.domain/fullchain.cer;
    ssl_certificate_key /etc/ssl/my.domain/my.domain.key;
    ssl_protocols TLSv1.3 TLSv1.2;
    ssl_ciphers EECDH+ECDSA+AESGCM:EECDH+aRSA+AESGCM:EECDH+ECDSA+SHA512:EECDH+ECDSA+SHA384:EECDH+ECDSA+SHA256:ECDH+AESGCM:ECDH+AES256:DH+AESGCM:DH+AES256:RSA+AESGCM:!aNULL:!eNULL:!LOW:!RC4:!3DES:!MD5:!EXP:!PSK:!SRP:!DSS;
    ssl_session_cache shared:TLS:2m;
    ssl_buffer_size 4k;
    ssl_prefer_server_ciphers on;

    root /mnt;

    location / {

        # enable creating directories without trailing slash
        set $x $uri$request_method;
        if ($x ~ [^/]MKCOL$) {
            rewrite ^(.*)$ $1/;
        }

        client_body_temp_path /mnt/sda1/.nginxtemp 2;
        autoindex on;
        dav_methods PUT DELETE MKCOL COPY MOVE;
        dav_ext_methods PROPFIND OPTIONS LOCK UNLOCK;
        dav_ext_lock zone=davlock;
        dav_access user:rw group:rw all:rw;
        create_full_put_path on;
        client_max_body_size 0M;
        auth_basic "Authorized Users Only";
        auth_basic_user_file /etc/nginx/webdavpasswd;
        satisfy any;
  }
}
```

配置里面8081端口提供了http、8083提供了tls的https。

需要注意`client_body_temp_path`的参数，在webdav上传适合nginx会把临时文件先放到这个路径，然后才转移到实际上传路径。最好放在相同分区的目录下。

`auth_basic_user_file`是普通的http auth格式。 

因为开启了`autoindex on;`，即使不用webdav客户端，直接浏览器访问也是能够下载对应目录和文件的。