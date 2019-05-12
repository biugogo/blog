---
title: Nginx简单学习
date: 2018-4-5 18:02:10
tags:
 -web server
categories: web server
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# Nginx简单配置
------------
### Overview
当个备忘录
参考：
*  https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching
* http://seanlook.com/2015/05/17/nginx-install-and-config/
* https://blog.csdn.net/kisscatforever/article/details/73129270
---

### What is Nginx
>  Nginx是一个高性能的HTTP和反向代理服务器，也是一个IMAP/POP3/SMTP代理服务器。Nginx是一款轻量级的Web服务器/反向代理服务器以及电子邮件代理服务器，并在一个BSD-like协议下发行。由俄罗斯的程序设计师lgor Sysoev所开发，供俄国大型的入口网站及搜索引擎Rambler使用。其特点是占有内存少，并发能力强，事实上nginx的并发能力确实在同类型的网页服务器中表现较好。Nginx相较于Apache\lighttpd具有占有内存少，稳定性高等优势，并且依靠并发能力强，丰富的模块库以及友好灵活的配置而闻名。在Linux操作系统下，nginx使用epoll事件模型,得益于此，nginx在Linux操作系统下效率相当高。同时Nginx在OpenBSD或FreeBSD操作系统上采用类似于Epoll的高效事件模型kqueue.

----------
### Why we need Nginx
最开始对于Nginx也是不是很了解，只知道这个东西可以接受连接的数量很大，但是一直以为java本身自带有Tomcat这个东西，跟nginx一样是web服务器，而且Tomcat是咱java特有的，肯定选Tomcat。
为什么要用nginx服务器代理，不直接用tomcat 7.0，还做多了一次接请求？然后我就入nginx坑了。

##### 最简单的明白为什么要nginx
在传统的Web项目中，并发量小，用户使用的少。所以在低并发的情况下，用户可以直接访问tomcat服务器，然后tomcat服务器返回消息给用户。
![nginx-tomcat1](https://github.com/apollochen123/image/blob/master/nginx-Tomcat1.png?raw=true)

当然我们知道，为了解决并发，可以使用负载均衡：也就是我们多增加几个tomcat服务器。当用户访问的时候，请求可以提交到空闲的tomcat服务器上。
![nginx-Tomcat2](https://github.com/apollochen123/image/blob/master/nginx-Tomcat2.png?raw=true)
但是这种情况下可能会有一种这样的问题：上传图片操作。我们把图片上传到了tomcat1上了，当我们要访问这个图片的时候，tomcat1正好在工作，所以访问的请求就交给其他的tomcat操作，而tomcat之间的数据没有进行同步，所以就发生了我们要请求的图片找不到。为了解决这种情况，我们就想出了分布式。我们专门建立一个图片服务器，用来存储图片。这样当我们都把图片上传的时候，不管是哪个服务器接收到图片，都把图片上传到图片服务器。图片服务器上需要安装一个http服务器，可以使用tomcat、apache、nginx。
![nginx-tomcat3](https://github.com/apollochen123/image/blob/master/nginx-Tomcat3.png?raw=true)

##### nginx的一些强大功能
这里十分推荐这篇[文章](https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching)，本文会截取很大部分原文，翻译对照起来，更有韵味

**General Proxying Information**

>General Proxying Information
If you have only used web servers in the past for simple, single server configurations, you may be wondering why you would need to proxy requests.

如果你只使用一个简单的单个的webserver，你可能想知道为什么我们需要代理request。
>One reason to proxy to other servers from Nginx is the ability to scale out your infrastructure. Nginx is built to handle many concurrent connections at the same time. This makes it ideal for being the point-of-contact for clients. The server can pass requests to any number of backend servers to handle the bulk of the work, which spreads the load across your infrastructure. This design also provides you with flexibility in easily adding backend servers or taking them down as needed for maintenance.

一个理由由nginx代理其他server是为了扩展你的架构。nginx是专为handle高连接时而诞生的。这使得它成为客户的接触点的理想选择。Nginx可以传递request给任意数量的后台，这就实现了负载均衡。这种设计还可以让您灵活地轻松添加后端服务器或根据维护需要减少服务器数量。

>Another instance where an http proxy might be useful is when using an application servers that might not be built to handle requests directly from clients in production environments. Many frameworks include web servers, but most of them are not as robust as servers designed for high performance like Nginx. Putting Nginx in front of these servers can lead to a better experience for users and increased security.
http

另一个实例是，有可能后端应用程序server并没有被设计来直接处理来自生产环境中的客户端的请求。许多框架包括网络服务器，但其中大多数并不像为nginx这样的高性能而设计的服务器。将nginx放在这些服务器之前可以为用户带来更好的体验，并提高安全性。
>Proxying in Nginx is accomplished by manipulating a request aimed at the Nginx server and passing it to other servers for the actual processing. The result of the request is passed back to Nginx, which then relays the information to the client. The other servers in this instance can be remote machines, local servers, or even other virtual servers defined within Nginx. The servers that Nginx proxies requests to are known as upstream servers.

nginx代理是通过转发请求到其他服务器进行实际处理来完成的。请求的结果被传递回nginx，然后nginx将这些信息转发给客户端。
这个实例中的其他服务器可以是远程机器，本地服务器，甚至是在nginx中定义的其他虚拟服务器。nginx代理请求的服务器称为上游服务器。
>Nginx can proxy requests to servers that communicate using the http(s), FastCGI, SCGI, and uwsgi, or memcached protocols through separate sets of directives for each type of proxy. In this guide, we will be focusing on the http protocol. The Nginx instance is responsible for passing on the request and massaging any message components into a format that the upstream server can understand.

nginx可以将请求代理到使用http（s），fastcgi，scgi和uwsgi或memcached协议进行通信的服务器，这些服务器通过对每种代理类型的单独指令集进行通信。在本指南中，我们将重点介绍http协议。nginx实例负责传递请求并将任何消息组件转换为上游服务器可以理解的格式。

1. **让我们建立一个基础的代理**
  ```
  # server context
  location /match/here {
      proxy_pass http://example.com;
  }
  ```
  >For example, when a request for /match/here/please is handled by this block, the request URI will be sent to the example.com server as http://example.com/match/here/please.

  假如一个请求是/match/here/please，nginx将会把它转换为http://example.com/match/here/please.
假如是如下配置
  ```
  location /match/here {
    proxy_pass http://example.com/new/prefix;
  }
  ```
  >For example, a request for /match/here/please on the Nginx server will be passed to the upstream server as http://example.com/new/prefix/please. The /match/here is replaced by /new/prefix. This is an important point to keep in mind.

  假如一个请求是/match/here/please，在本例子中会被转发到http://example.com/new/prefix/please

2. **配置转发头**
   nginx可以将request的header转发给后台服务器。如下配置即可，如果想了解更多细节，请参考原文
   ```
   location /match/here {
       proxy_set_header HOST $host;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

       proxy_pass http://example.com/new/prefix;
   }
   ```
   当然我们可以使转发头成为全局参数
   ```
   proxy_set_header HOST $host;
   proxy_set_header X-Forwarded-Proto $scheme;
   proxy_set_header X-Real-IP $remote_addr;
   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

   location /match/here {
       proxy_pass http://example.com/new/prefix;
   }

   location /different/match {
       proxy_pass http://example.com;
   }
   ```

3. **关于配置负载均衡**
  看一个简单例子
  ```
  upstream backend_hosts {
      server host1.example.com;
      server host2.example.com;
      server host3.example.com;
  }

  server {
      listen 80;
      server_name example.com;

      location /proxy-me {
          proxy_pass http://backend_hosts;
      }
  }
  ```
  当然负载均衡策略是可以配置的，nginx有如下几种配置
  * round robin: The default load balancing algorithm that is used if no other balancing directives are present. Each server defined in the upstream context is passed requests sequentially in turn.
  循环分发，默认的负载均衡方式。在上游上下文中定义的每个服务器依次轮流传递请求。
  * least_conn: Specifies that new connections should always be given to the backend that has the least number of active connections. This can be especially useful in situations where connections to the backend may persist for some time.
    ```
    upstream backend_hosts {

      least_conn;

      server host1.example.com;
      server host2.example.com;
      server host3.example.com;
    }
    ```
  指定应始终将新连接提供给活动连接数最少的后端。这在与后端的连接可能持续一段时间的情况下尤其有用。
  * ip_hash: This balancing algorithm distributes requests to different servers based on the client's IP address. The first three octets are used as a key to decide on the server to handle the request. The result is that clients tend to be served by the same server each time, which can assist in session consistency.
  这种平衡算法根据客户端的IP地址将请求分发给不同的服务器。前三个八位字节被用作决定服务器处理请求的关键字。结果是客户端每次都倾向于由同一台服务器提供服务，这可以帮助实现会话一致性。
    ```
    upstream backend_hosts {

      ip_hash;

      server host1.example.com;
      server host2.example.com;
      server host3.example.com;
    }
    ```
  * hash: This balancing algorithm is mainly used with memcached proxying. The servers are divided based on the value of an arbitrarily provided hash key. This can be text, variables, or a combination. This is the only balancing method that requires the user to provide data, which is the key that should be used for the hash.
  自定义hash算法进行负载。这是唯一需要用户提供数据的平衡方法，这是散列应该使用的关键。
    ```
    upstream backend_hosts {

      hash $remote_addr$remote_port consistent;

      server host1.example.com;
      server host2.example.com;
      server host3.example.com;
    }
    ```

4. **Using Buffers to Free Up Backend Servers**

  代理服务有两个请求会影响客户的体验
  1. The connection from the client to the Nginx proxy.
  2. The connection from the Nginx proxy to the backend server.
  nginx能够根据您想要的来优化的这些连接中的任何一个来调整其行为。

  use buffer的作用：
  >Without buffers, data is sent from the proxied server and immediately begins to be transmitted to the client. If the clients are assumed to be fast, buffering can be turned off in order to get the data to the client as soon as possible. With buffers, the Nginx proxy will temporarily store the backend's response and then feed this data to the client. If the client is slow, this allows the Nginx server to close the connection to the backend sooner. It can then handle distributing the data to the client at whatever pace is possible.

  没有缓冲区，数据从代理服务器发送并立即开始传输到客户端。如果客户端速度很快，那么可以关闭缓冲以提高响应速度。
  使用缓冲区，nginx代理将临时存储后端的响应，然后将这些数据提供给客户端。但是如果客户的网速很慢，那nginx可以缓存客户的数据，等数据完成后再交给后端服务器处理。这样可以大大减少后端服务器被占用连接的时间。

  ```
  proxy_buffering on;
  proxy_buffer_size 1k;
  proxy_buffers 24 4k;
  proxy_busy_buffers_size 8k;
  proxy_max_temp_file_size 2048m;
  proxy_temp_file_write_size 32k;

  location / {
    proxy_pass http://example.com;
  }
  ```

  1. proxy_buffering: This directive controls whether buffering for this context and child contexts is enabled. By default, this is "on".**是否开启buffer，默认是on**
  2. proxy_buffers: This directive controls the number (first argument) and size (second argument) of buffers for proxied responses. The default is to configure 8 buffers of a size equal to one memory page (either 4k or 8k). Increasing the number of buffers can allow you to buffer more information.**此指令控制代理响应的缓冲区的数量（第一个参数）和大小（第二个参数）。缺省情况下，配置8个大小等于一个内存页（4k或8k）的缓冲区。增加缓冲区的数量可以让你缓冲更多的信息。**
  3. proxy_buffer_size: The initial portion of the response from a backend server, which contains headers, is buffered separately from the rest of the response. This directive sets the size of the buffer for this portion of the response. By default, this will be the same size as proxy_buffers, but since this is used for header information, this can usually be set to a lower value.**设置代理服务器（nginx）从后端realserver读取并保存用户头信息的缓冲区大小，默认与proxy_buffers大小相同，其实可以将这个指令值设的小一点**
  4. proxy_busy_buffers_size: This directive sets the maximum size of buffers that can be marked "client-ready" and thus busy. While a client can only read the data from one buffer at a time, buffers are placed in a queue to send to the client in bunches. This directive controls the size of the buffer space allowed to be in this state.**高负荷下缓冲大小（proxy_buffers*2）**
  5. proxy_max_temp_file_size: This is the maximum size, per request, for a temporary file on disk. These are created when the upstream response is too large to fit into a buffer.**当 proxy_buffers 放不下后端服务器的响应内容时，会将一部分保存到硬盘的临时文件中，这个值用来设置最大临时文件大小，默认1024M，它与 proxy_cache 没有关系。大于这个值，将从upstream服务器传回。设置为0禁用。**
  6. proxy_temp_file_write_size: This is the amount of data Nginx will write to the temporary file at one time when the proxied server's response is too large for the configured buffers.**当缓存被代理的服务器响应到临时文件时，这个选项限制每次写临时文件的大小。**
  7. proxy_temp_path: This is the path to the area on disk where Nginx should store any temporary files when the response from the upstream server cannot fit into the configured buffers.proxy_temp_path **（可以在编译的时候）指定写到哪那个目录。**


--------------

### Nginx实现反向代理Tomcat+动静资源分离
这里只上一个配置，实现本地双Tomcat负载均衡，nginx代理并实现动静分离
```shell
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

	#日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

	#设置是否保存访问日志，设置为off可以降低磁盘IO而提升速度。
    access_log  logs/access.log  main;


	#sendfile 指向sendfile()函数。sendfile()在磁盘和TCP端口（或者任意两个文件描述符）之间复制数据。sendfile()直接从磁盘上读取数据到操作系统缓冲，因此会更有效率。
    sendfile        on;
    #配置nginx在一个包中发送全部的头文件，而不是一个一个发送。
	tcp_nopush     on;
	#tcp_nodelay 配置nginx不要缓存数据，快速发送小数据。
    #tcp_nodelay    on;

	#keepalive_timeout 指定了与客户端的keep-alive链接的超时时间。服务器会在这个时间后关闭链接。
    #keepalive_timeout  0;
    #keepalive_timeout  65;

	#gzip 打开压缩功能可以减少需要发送的数据的数量。
    gzip  on;
	#gzip_proxied 允许或禁止基于请求、响应的压缩。设置为any，就可以gzip所有的请求.
	gzip_proxied any;
	#gzip_comp_level 设置了数据压缩的等级。等级可以是 1-9 的任意一个值，9 表示最慢但是最高比例的压缩.
	gzip_comp_level 3;
	#gzip_types 设置进行 gzip 的类型。
	gzip_types text/plain text/css application/json application/xml text/javascript;

    #动态服务器组,负载均衡的后端服务器，weight为权重值
    upstream tomcats_proxy {
       server localhost:8080 weight=5;
       server localhost:8081 weight=5;
    }

    server {
	    #listen 表示当前的代理服务器监听的端口，默认的是监听80端口。
        listen       80;

		#server_name 表示监听到之后需要转到哪里去，localhost表示转到本地，也就是直接到nginx文件夹内。
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        #转发jsp服务
		location ~ \.jsp$ {
            proxy_pass http://tomcats_proxy;
			proxy_redirect default;
			#proxy_set_header Host $host; 后端的Web服务器可以通过X-Forwarded-For获取用户真实IP。
            #client_max_body_size 10m; 允许客户端请求的最大单文件字节数。
            #client_body_buffer_size 128k; 缓冲区代理缓冲用户端请求的最大字节数。
            #proxy_connect_timeout 90; Nginx跟后端服务器连接超时时间。
            #proxy_read_timeout 90; 连接成功后，后端服务器响应时间。
            #proxy_buffer_size 4k; 设置代理服务器保存用户头信息的缓冲区大小。
            #proxy_buffers 6 32k; proxy_buffers缓冲区。
            #proxy_busy_buffers_size 64k; 高负荷下缓冲大小。
            #proxy_temp_file_write_size 64k; 设定缓存文件夹大小。
        }
		#设置本地静态资源，路径为root D:\apache-tomcat-8.5.28\webapps\ROOT;
		location ~ \.(html|js|css|png|gif)$ {
		    root D:\apache-tomcat-8.5.28\webapps\ROOT;

			#禁止/同意这个ip访问
			#deny 192.168.1.1;
			#allow 192.168.1.1;

			#设置不限速传输的响应大小。当传输量大于此值时，超出部分将限速传送。
			#limit_rate_after 3m;
			#限制向客户端传送响应的速率限制,参数的单位是字节/秒，设置为0将关闭限速。比如此处配置表示不限速部分为3m，超过了3m后限速为20k/s。
			#limit_rate 20k;

		}
}

```
