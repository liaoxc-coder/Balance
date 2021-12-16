
1、项目结构总览，新建 nginx 目录用于保存本项目所需文件，项目中文件及目录结构。
其中，docker-compose.yml 是 Docker compose 配置文件，定义了服务、
容器及容器行为，docker-compose的版本是1.27.4；
nginx.conf 是 Nginx 的配置文件，其中配置了负载均衡策略；
NginxDockerfile 的功能是基于 ubuntu 镜像构建 Nginx 镜像，容器集群中三个容器均基于该镜像；
sources.list 是 ubuntu 的软件源配置，用于加速软件安装；
web1 和web2 目录是 Nginx_http1 和 Nginx_http2 容器中 Nginx 服务的网页文件目录，其中保存了网页的首页文件。

2、docker-compose.yml内容说明：
```YAML
version: '3.8' 
services: 
  Nginx_proxy: 
    image: nginx 
    build: 
      context: . 
      dockerfile: NginxDockerfile 
    ports: 
      - 8000:8000 
    volumes: 
      - ./nginx.conf:/etc/nginx/nginx.conf 
    networks: 
      - web_network 
  Nginx_http1: 
    image: nginx 
    volumes: 
      - ./web1:/var/www/html 
    networks: 
      - web_network 
    command: nginx -c /etc/nginx/nginx.conf -g "daemon off;" 
  Nginx_http2: 
    image: nginx 
    volumes: 
      - ./web2:/var/www/html 
    networks: 
      - web_network 
    command: nginx -c /etc/nginx/nginx.conf -g "daemon off;" 
networks: 
  web_network: 
  driver: bridge
```YAML

集群中共三个服务，Nginx_proxy、Nginx_http1和Nginx_http2，其中Nginx_proxy提供负载均衡服务，Nginx_http1和Nginx_http2提供网页服务。这三个服务对应的容器的来源镜像都基于 NginxDockerfile，容器的网络都是所定义的桥接网络 web_network。指定 Nginx_proxy 服务的 8000 端口映射到宿主机的 8000端口，将项目资源文件中的 nginx.conf 绑定挂载到容器的/etc/nginx/nginx.conf。将网页目录web1 和 web2 分别绑定挂载到对应服务的/var/www/html，指定 Nginx_http1 和 Nginx_http2服务启动后运行的指令是 nginx -c /etc/nginx/nginx.conf -g "daemon off;"，该指令的功能是启动 Nginx 服务。



3、nginx.conf内容如下：

user www-data; 
worker_processes auto; 
pid /run/nginx.pid; 
include /etc/nginx/modules-enabled/*.conf; 
daemon off;
events { 
    worker_connections 768; 
    # multi_accept on; 
}
http {
    upstream http_pool {
        server Nginx_http1:80 weight=1 max_fails=2 fail_timeout=30;
    server Nginx_http2:80 weight=1 max_fails=2 fail_timeout=30;
    }
    server {
        listen 8000;
    location / {
        proxy_pass http://http_pool;
    }
    }
    sendfile on; 
    tcp_nopush on; 
    tcp_nodelay on; 
    keepalive_timeout 65; 
    types_hash_max_size 2048; 
    include /etc/nginx/mime.types; 
    default_type application/octet-stream; 
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE 
    ssl_prefer_server_ciphers on; 
    access_log /var/log/nginx/access.log; 
    error_log /var/log/nginx/error.log; 
    gzip on; 
    include /etc/nginx/conf.d/*.conf; 
    include /etc/nginx/sites-enabled/*;
}

该配置文件中配置了负载均衡策略，所配置的内容是 http 中的 upstream 和 server。首先通过 upstream 定义一个负载均衡池 http_pool，在池中通过 server 定义了两个服务器，即Nginx_http1 和 Nginx_http2，并通过 weight 指定了它们的权重，权重越大，则该服务器被用户访问的可能性越大；接着在与 upstream 同级的 server 中定义负载均衡服务器监听 8000 端口，当用户访问负载均衡服务器的根目录时（即通过 http://负载均衡服务器 IP:8000 访问），服务器从负载均衡池 http_pool 中选择主机。由于 Nginx_http1 和 Nginx_http2 的权重相同，因此用户访问负载均衡服务器时，基本可视为交替访问 Nginx_http1 和 Nginx_http2。



4、NginxDockerfile内容如下：

FROM ubuntu 
MAINTAINER lxc
ADD sources.list /etc/apt
ENV DEBIAN_FRONTEND=noninteractive
RUN apt update && apt dist-upgrade -y \
             && apt install nginx -y
CMD nginx -c /etc/nginx/nginx.conf 

该 Dockerfile 中定义了基于 ubuntu 镜像部署 Nginx 服务的操作，首先将资源文件sources.list 通 过 ADD 添 加 到 待 构 建镜像的 /etc/apt 下 ； 然 后 定 义 环 境 变 量DEBIAN_FRONTEND，其值为 noninteractive，该环境变量的功能是去掉 apt 的交互操作，如果不定义该环境变量，安装 Nginx 时可能需要用户输入时区配置；再通过 RUN 指定安装Nginx 的指令，通过 apt update 和 apt dist-upgrade 更新系统，再通过 apt install nginx -y 安装Nginx ； 最 后 通 过 CMD 指 定 由 该 镜 像 启 动 的容器 默 认 执 行 的 指 令 是 nginx -c /etc/nginx/nginx.conf，该指令的功能是启动 Nginx 服务。



5、web1 和 web2 中保存了 Nginx_http1 和 Nginx_http2 的网页文件，其中 Nginx_http1 的首页信息为This is Server 1，Nginx_http2 的首页信息为This is  Server 2。



6、构建镜像，使用指令 docker-compose build 根据配置文件构建镜像。

root@localhost: ~/nginx#  docker-compose build



7、启动服务，使用指令 docker-compose up -d 启动服务并使服务在后台运行。

root@localhost: ~/nginx#  docker-compose up -d



8、浏览器多次访问 http://宿主机 IP:8000 。或者在宿主机通过 curl 命令多次访问宿主机 IP:8000 访问负载均衡容器集群（在 ubuntu 中，可以通过 sudo apt install curl -y 安装 curl）。

root@localhost: ~/nginx#  curl 127.0.0.1:8000

curl 可以获取指定 URL 对应的内容，在宿主机上通过 curl 127.0.0.1:8000 可以请求负载均衡集群所提供的 Nginx 服务，从输出结果上看，用户的请求被分发给了所设置的两个Nginx 容器中，当用户多次请求负载均衡服务器，用户请求被等概率地分配给两个 Nginx 容器。



9、停止服务，使用指令 docker-compose down 停止服务并清理所创建的容器，停止容器后再次请求，请求失败。


