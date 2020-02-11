#  Nginx学习笔记

[TOC]

## 一、nginx简介

### 1、什么是nginx

* Nginx 是一个很强大的高性能Web和反向代理服务；
* 其特点是占有内存少，并发能力强；
* 能够支持高达 50,000 个并发连接数的响应。

### 2、相关概念

#### 2.1、正、反向代理

* Nginx 不仅可以做**反向代理**，实现负载均衡；还能用作**正向代理**来进行上网等功能。 

  * 正向代理：

    * 指的是一个位于客户端和原始服务器(origin server)之间的服务器，为了从**原始服务器**取得内容，客户端向代理发送一个请求并指定目标(原始服务器)，然后代理向原始服务器转交请求并将获得的内容返回给客户端

    * 和反向代理不同之处在于，典型的正向代理是一种**最终用户知道并主动使用**的代理方式

    * 比如需要访问某些国外网站，我们可能需要购买vpn。并且vpn是在我们的用户浏览器端设置的(并不是在远端的服务器设置)。浏览器先访问vpn地址，vpn地址转发请求，并最后将请求结果原路返回来。

      ![](E:\git-files\Nginx\picture\正向代理.png)
  
  * 反向代理：
  
    * 反向代理服务器位于用户与目标服务器之间，但是对于用户而言，反向代理服务器就相当于目标服务器，即用户直接访问反向代理服务器就可以获得目标服务器的资源。同时，用户不需要知道目标服务器的地址，也无须在用户端作任何设定。
  
    * 反向代理服务器通常可用来作为Web加速，即使用反向代理作为Web服务器的前置机来降低网络和服务器的负载，提高访问效率。
  
      ![](E:\git-files\Nginx\picture\反向代理.png)

#### 2.2、负载均衡

* 传统做法：客户端发送多个请求到服务器，服务器处理请求，有一些可能要与数据库进行交互，服 务器处理完毕后，再将结果返回给客户端；这种架构模式对于早期的系统相对单一，并发请求相对较少的情况下是比较适合的，成 本也低。但是随着信息数量的不断增长，访问量和数据量的飞速增长，以及系统业务的复杂 度增加，这种架构会造成服务器相应客户端的请求日益缓慢，并发量特别大的时候，还容易 造成服务器直接崩溃。

  ![](E:\git-files\Nginx\picture\未使用负载均衡的架构模式.png)

* 负载均衡是高可用架构的一个关键组件，主要用来提高性能和可用性，通过负载均衡将流量**分发到多个服务器**，同时多服务器能够消除这部分的单点故障。

  ![](E:\git-files\Nginx\picture\使用负载均衡的架构模式.png)

#### 2.3、动静分离

* 动静分离是将网站静态资源（HTML，JavaScript，CSS，img等文件）与后台应用分开部署，提高用户访问静态代码的速度，降低对后台应用访问。

* 传统动静不分离的产品架构(随着访问量在增长，性能会成为瓶颈) 

  ![](E:\git-files\Nginx\picture\传统动静不分离的产品架构.png)

* 实现动分离的产品架构（灵活的架构支持海量的用户访问）

  ![](E:\git-files\Nginx\picture\实现动分离的产品架构.png)

## 二、nginx安装

### 1、进入官网下载

[Nginx官网](http://nginx.org/ )

### 2、安装

#### 2.1、准备工作

* 安装Nginx必要的组件（pcre、openssl、zlib）

  * 执行命令

    ```shell
    yum -y install make zlib zlib-devel gcc-c++ libtool  openssl openssl-devel 
    ```

#### 2.2、安装Nginx

*  解压缩 nginx-xx.tar.gz 包；
*  进入解压缩目录，执行./configure；
*  make && make install 
* 完成

#### 2.3、注意事项

* 查看防火墙开放的端口

  ```shell
  firewall-cmd --list-all
  ```

* 设置开发的端口号

  ```shell
  firewall-cmd --add-service=http –permanent 
  sudo firewall-cmd --add-port=80/tcp --permanent 
  ```

* 重启防火墙

  ```shell
  firewall-cmd –reload 
  ```

  

## 三、nginx 常用的命令和配置文件

### 1、nginx 常用的命令

* 启动命令

  * nginx安装目录下的sbin目录下执行 

    ```shell
    ./nginx
    ```

* 关闭命令

  * nginx安装目录下的sbin目录下执行

    ```shell
    ./nginx -s  stop
    ```

* 重新加载命令

  * nginx安装目录下的sbin目录下执行

    ```shell
    ./nginx -s  reload 
    ```

    

### 2、nginx.conf 配置文件

#### 2.1、配置文件位置

* nginx安装目录下的conf文件夹中

#### 2.2、配置文件示例

```xml
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
```

#### 2.3、配置文件详解

根据上述文件，我们可以很明显的将 nginx.conf 配置文件分为三部分：

* 第一部分： 

  * 从配置文件开始到 events 块之间的内容，主要会设置一些影响nginx 服务器整体运行的配置指令，主要包括配 置运行 Nginx 服务器的用户（组）、允许生成的 worker process 数，进程 PID 存放路径、日志存放路径和类型以 及配置文件的引入等。

  * 例如：

    ```xml
    worker_processes  1;
    
    //这是 Nginx 服务器并发处理服务的关键配置，worker_processes 值越大，可以支持的并发处理量也越多，但是 会受到硬件、软件等设备的制约 
    ```

* 第二部分

  * events 块： 这块涉及的指令主要影响 Nginx 服务器与用户的网络连接，常用的设置包括是否开启对多 work process 下的网络连接进行序列化，是否允许同时接收多个网络连接，选取哪种事件驱动模型来处理连接请求，每个 word process 可以同时支持的最大连接数等。

  * 例如

    ```xml
    events {
        worker_connections  1024;//表示每个 work process 支持的最大连接数为 1024
    }
    ```

  *  这部分的配置对 Nginx 的性能影响较大，在实际中应该灵活配置。

* 第三部分

  * http块：Nginx 服务器配置中最频繁的部分，代理、缓存和日志定义等绝大多数功能和第三方模块的配置都在这里。http 块也包括 http全局块、server 块。
    * http 全局块 ： http全局块配置的指令包括文件引入、MIME-TYPE 定义、日志自定义、连接超时时间、单链接请求数上限等。 
    * server 块： 这块和虚拟主机有密切关系，虚拟主机从用户角度看，和一台独立的硬件主机是完全一样的，该技术的产生是为了 节省互联网服务器硬件成本。每个server块分为全局 server 块和 locaton 块。
      * 全局 server 块： 最常见的配置是本虚拟机主机的监听配置和本虚拟主机的名称或IP配置
      * location 块 ： 这块的主要作用是基于 Nginx  服务器接收到的请求字符串（例如 server_name/uri-string），对虚拟主机名称 （也可以是IP 别名）之外的字符串（例如 前面的 /uri-string）进行匹配，对特定的请求进行处理。地址定向、数据缓 存和应答控制等功能，还有许多第三方模块的配置也在这里进行
      *  一个 server 块可以配置多个 location 块。 

  *  每个 http 块可以包括多个 server 块，而每个 server 块就相当于一个虚拟主机。 而每个 server 块也分为全局 server 块，以及可以同时包含多个 locaton 块。 