---
title: "搭建了自己的 RSS 服务"
description: "在服务器上部署自己的 RSS 服务"
date: 2023-10-26
tags:
  - 工具资源
  - 折腾
categories:
  - 网站环境搭建教程合集
cover: https://img.cczywyc.com/post-cover/run_ttrss.jpeg
---

先来简单说一下什么是 RSS 吧，RSS 全称叫做 Really Simple Syndication，中文叫做简易信息聚合，也叫聚合内容，它是一种消息来源格式规范，用以聚合多个网站更新的内容并自动通知网站订阅者。使用 RSS 后，网站订阅者便无需手动查看网站是否有新的内容，同时 RSS 可将多个网站更新的内容进行整合，以摘要的形式呈现，有助于订阅者快速获取重要信息，并选择性地点阅查看。说的简单点就是它可以把各种信息源整合起来，我只需要在一个 RSS 服务里面就可以获取到来自各种不同渠道的订阅更新，现在我们看到个很多技术博客和网站都提供了 RSS 订阅链接，细心的话你应该在很多地方就见过这个东西。

说来奇怪，RSS 本应是上个时代的产物，近些年缺被越来越多的人追捧，在目前这个信息爆炸的时代，各种信息流充满着我们的屏幕，不同的信息平台也有着不同的推送方式，导致信息获取破碎，无法筛选自己真正感兴趣的信息。如果你跟我有差不多同样的想法，那么，搭建一个自己的 RSS 服务就显的很有必要了。

我的信息获取来源，一方面是一些微信公众号，还有一些技术博客和刊栏，导致我的信息获取非常的分散不集中，于是便萌生了搭建一个自己的 RSS 服务的想法，在进行了一番调研之后，我的其中一个很重要的要求就是要能独立部署，所以在抛弃了众多优秀的 RSS 阅读器之后，[tt-rss](https://tt-rss.org) 似乎成为了我最佳的选择，那么说干就干。

# 环境准备

要搭建一个自己的 RSS 服务，首先需要一个主机，可以是物理机，当然也可以是云主机，这里我选择一个 Ubuntu 服务器来搭建，使用云服务器也是现在更容易选择的便捷方式。具体需要准备的如下：

* 云服务器：Ubuntu 22.04 LTS
* 域名：xxx.xxx
* SSL 证书

需要说明的是，域名和 SSL 证书是必要的，一方面我们尽量不要使用 ip 访问我们的网站，另外为了安全起见，应当使用 HTTPS 访问我们的 RSS 服务。基本上每一个云服务厂商都提供了域名和 SSL 证书的申请，域名首单购买超级便宜，SSL 证书则基本都有免费的测试证书，对于搭建个人的 RSS 服务来说应该是够用了。域名和 SSL 的配置各大云服务厂商都有很详细的配置说明，这里就不作展开。

# 服务搭建

这里我选择的是 tt-rss，由于它依赖于 postgres 数据库和其他的服务，可以按照官网的方式搭建，社区也有人提供了 docker-compose 的方式，可以全部把需要的服务都以 docker 容器的方式启动。这里我强烈推荐 [Awesome TTRSS](https://ttrss.henry.wang/#about) 来快速搭建 RSS 服务。

#### 安装 docker 和 docker-compose

docker 的安装有很多种方式，首选的当然是官网的教程，[docker 官网](https://docs.docker.com/engine/install/ubuntu/)针对于不同的操作系统都提供了很详细的安装方式。比如以 Ubuntu 为例，官网就写出了很详细的教程：

设置 docker 源

```bash
# set up docker's apt repository

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

安装 docker

```bash
# install docker packages
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

安装 docker-compose

```bash
# install docker-compose
sudo apt-get install docker-compose
```

检查是否安装成功

```bash
# check docker
docker -v
Docker version 24.0.7, build afdd53b

# check docker-compose
docker-compose -v
docker-compose version 1.29.2, build unknown
```

当出现上面的回显，说明 docker 和 docker-compose 安装成功

#### 安装 tt-rss 相关服务

去 AweSome-TTRSS [github 仓库](https://github.com/HenryQW/Awesome-TTRSS) 下载 [docker-compose.yml](https://github.com/HenryQW/Awesome-TTRSS/blob/main/docker-compose.yml)，示例如下：

```yaml
version: "3"
services:
  service.rss:
    image: wangqiru/ttrss:latest
    container_name: ttrss
    ports:
      - 181:80
    environment:
      - SELF_URL_PATH=https://your_domain/ # please change to your own domain
      - DB_PASS=ttrss # use the same password defined in `database.postgres`
      - PUID=1000
      - PGID=1000
    volumes:
      - feed-icons:/var/www/feed-icons/
    networks:
      - public_access
      - service_only
      - database_only
    stdin_open: true
    tty: true
    restart: always

  service.mercury: # set Mercury Parser API endpoint to `service.mercury:3000` on TTRSS plugin setting page
    image: wangqiru/mercury-parser-api:latest
    container_name: mercury
    networks:
      - public_access
      - service_only
    restart: always

  service.opencc: # set OpenCC API endpoint to `service.opencc:3000` on TTRSS plugin setting page
    image: wangqiru/opencc-api-server:latest
    container_name: opencc
    environment:
      - NODE_ENV=production
    networks:
      - service_only
    restart: always

  database.postgres:
    image: postgres:13-alpine
    container_name: postgres
    environment:
      - POSTGRES_PASSWORD=ttrss # feel free to change the password
    volumes:
      - ~/postgres/data/:/var/lib/postgresql/data # persist postgres data to ~/postgres/data/ on the host
    networks:
      - database_only
    restart: always

  # utility.watchtower:
  #   container_name: watchtower
  #   image: containrrr/watchtower:latest
  #   volumes:
  #     - /var/run/docker.sock:/var/run/docker.sock
  #   environment:
  #     - WATCHTOWER_CLEANUP=true
  #     - WATCHTOWER_POLL_INTERVAL=86400
  #   restart: always

volumes:
  feed-icons:

networks:
  public_access: # Provide the access for ttrss UI
  service_only: # Provide the communication network between services only
    internal: true
  database_only: # Provide the communication between ttrss and database only
    internal: true
```

其中第 7 行，是 ttrss 服务端口，这里默认是 181 端口映射到容器里面的 80 端口，映射出来的端口可以自行修改；第 9 行就是 ttrss 的访问地址，这里设置成你的域名；最后是第 10 行和第 44 行，设置 pg 的密码，两个地方成一样的就行。

利用 docker-compose 启动容器：

```bash
# at your docker-compose.yml dir
docker-compose up -d
```

执行完成后，检查容器是否启动成功：

```bash
# chekc docker containers
docker ps

CONTAINER ID   IMAGE                                COMMAND                  CREATED        STATUS        PORTS                                   NAMES
9d7a30db8f54   postgres:13-alpine                   "docker-entrypoint.s…"   44 hours ago   Up 44 hours                                           postgres
30f0ca736138   wangqiru/mercury-parser-api:latest   "dumb-init -- npm ru…"   44 hours ago   Up 44 hours   3000/tcp                                mercury
918e35b76206   wangqiru/opencc-api-server:latest    "docker-entrypoint.s…"   44 hours ago   Up 44 hours                                           opencc
aff0d1127aac   wangqiru/ttrss:latest                "sh /docker-entrypoi…"   44 hours ago   Up 44 hours   0.0.0.0:181->80/tcp, :::181->80/tcp   ttrss
```

当看到有 4 个容器时，则启动成功

#### 设置 HTTPS 访问

在服务器上安装 nginx，以 Ubuntu 为例：

```bash
# install nginx
sudo apt install nginx
```

编辑 nginx.conf：

```bash
# /etc/nginx/nginx.conf

user root;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
	worker_connections 768;
	# multi_accept on;
}

http {

	##
	# Basic Settings
	##

	sendfile on;
	tcp_nopush on;
	types_hash_max_size 2048;
	# server_tokens off;

	# server_names_hash_bucket_size 64;
	# server_name_in_redirect off;

	include /etc/nginx/mime.types;
	default_type application/octet-stream;

	##
	# SSL Settings
	##

	ssl_session_timeout 5m;
	ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;

	##
	# Logging Settings
	##

	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;

	##
	# Gzip Settings
	##

	gzip on;

	# gzip_vary on;
	# gzip_proxied any;
	# gzip_comp_level 6;
	# gzip_buffers 16 8k;
	# gzip_http_version 1.1;
	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

	##
	# Virtual Host Configs
	##

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;

	upstream rss {
		server 127.0.0.1:181;
	}

	server {
		listen 443 ssl;
		server_name your_domain.com;

		ssl_certificate crt_file_path;
		ssl_certificate_key key_file_path;

		location / {
      proxy_redirect off;
			proxy_pass http://rss;

      proxy_set_header  Host                $http_host;
      proxy_set_header  X-Real-IP           $remote_addr;
      proxy_set_header  X-Forwarded-Ssl     on;
      proxy_set_header  X-Forwarded-For     $proxy_add_x_forwarded_for;
      proxy_set_header  X-Forwarded-Proto   $scheme;
      proxy_set_header  X-Frame-Options     SAMEORIGIN;

      client_max_body_size        100m;
      client_body_buffer_size     128k;

      proxy_buffer_size           4k;
      proxy_buffers               4 32k;
      proxy_busy_buffers_size     64k;
      proxy_temp_file_write_size  64k;
    }		
	}

	server {
		listen 80;
    server_name  your_domain.com;
    return 301 https://your_domain.com$request_uri;
	}

}
```

上面 nginx 的配置大家应该都能看懂，第 72 行配置成你的域名，同理在第 100 行也配置成你的域名，http 的访问重定向到 https；74、75 行 配置 ssl 证书路径；66-68 行配置 nginx 代理的地址，因为这里 tt-rss 服务和 nginx 安装在同一台机器，所以 配置成 127.0.0.1:181，端口就是在上面 docker-compose.yml 文件里面配置的服务映射端口。

最后就是重启 nginx 服务

```bash
# restart nginx service
systemctl restart nginx	
```

最后在浏览器访问 https://your_domain 正常情况下就能看到如下页面，默认用户名和密码是 admin/passoword，登陆成功后在偏好设置里面可以更改管理员密码

![](https://img.cczywyc.com/tt-rss.png)

(本文完)
