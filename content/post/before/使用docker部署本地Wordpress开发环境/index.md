---
title: "使用Docker部署本地Wordpress开发环境"
date: 2020-09-23T16:47:28+08:00
tags: ["Wordpress", "Docker", "Config"]
categories: ["Technology"]
image: CleanShot.png
draft: false
---

# 使用Docker部署本地Wordpress开发环境

为了便于维护开发线上的wordpress站点，在本地构建一个wordpress开发环境。使用Docker来实现轻量化开发与部署环境的构建。

## 背景

由于最近接手了一个网站的运维工作，使用的还是Wordpress，因为有涉及到一些布局的调整，因此决定在本地构建一个开发调试环境用于进行上线前的测试，但又不想离开本机中Vscode等开发环境（如代码提示等）。在考虑了包括虚拟机LANMP，本机直接构建环境等方案后，最后选择了Docker来部署开发环境，同时也可以进一步熟悉Docker的使用方法。有了Docker，免去了虚拟机中繁杂的环境配置过程，同时对环境进行模块化解耦后可以十分便捷的启动或更新开发环境组件，也使得开发环境尽可能的贴近生产环境，之后的部署上线将变得十分容易。

## 本机环境

MacOS 10.15.6（理论上适用于所有支持Docker的环境）

Docker Desktop 19.03.12

### 镜像准备

```shell
$ docker images
nginx                     latest              7e4d58f0e5f3        12 days ago         133MB
mysql                     5.7.31              ef08065b0a30        13 days ago         448MB
daocloud.io/library/php   7.2.24-fpm          48432d192e1a        11 months ago       398MB
```

由于使用了Wordpress，所以环境的搭建涉及到了三个部分，ngin、mysql以及php，为了与生产环境一致，特别选择了对应的mysql与php的版本。理论上也不建议选择latest版本的镜像，选择相对稳定的版本会有更好的兼容性。

可以从[docker hub](hub.docker.com)上搜索对应版本的镜像

#### 拉取镜像

```shell
$ docker pull nginx:latest
$ docker pull mysql:5.7.31
$ docker pull daocloud.io/library/php:7.2.24-fpm
```

## 构建Nginx+PHP环境

### Nginx配置

Nginx的镜像是不能解析php文件的，因此我们需要为nginx配置php-fpm来负责解析处理php代码并返回处理结果。

为此，我们需要做一下几件事情：

1. 修改nginx的配置，指定php处理方式
2. 通过设置使nginx能够访问到php容器
3. 启动nginx与php的容器

首先，为了持久化配置，我在`/Users/kagaya/Docker/nginx/conf.d/`下新建了`default.conf`配置文件，挂载到目标文件`/etc/nginx/conf.d:ro`并指定只读属性

最后对nginx的配置文件做对应修改即可：

```conf
# default.conf
server {
    listen       80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.php index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    location ~ \.php$ {
        fastcgi_pass   php:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /var/www/html/$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

同样，为了方便开发，可以把nginx中的网页目录映射到主机中，这里使用了`Users/kagaya/docker/www`作为主机中的网站根目录，使用`-v`将其挂载到nginx中的`/usr/share/nginx/html`

使用Docker 挂载主机配置目录到容器，并使用`--link`参数来连接php容器，docker link采用了修改Hosts文件的方式使得可以通过自定义的名称访问到对应的ip地址，使得容器之间能通过网络互相访问

最后使用`-p 80:80`将nginx的80端口开放到主机的80端口，使得主机可以通过localhost访问到nginx

### PHP配置

php配置相对简单，只需要挂载网站根目录到容器即可

```shell
$ docker run -d --name php -v /Users/kagaya/docker/www:/var/www/html daocloud.io/library/php:7.2.24-fpm
```



### 使用Docker-compose一键部署多个服务

由于涉及到了多个容器服务，可以采用Docker-compose工具来方便的同时配置部署多个容器服务

新建目录，如`webdev`，目录下新建docker-compose.ym文件，配置如下：

```yaml
version: '3'
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    depends_on:
      - php
      - mysql
    volumes:
      - "/Users/kagaya/docker/nginx/conf.d:/etc/nginx/conf.d:ro"
      - "/Users/kagaya/docker/www:/usr/share/nginx/html"
    links:
        - "php:php"
  php:
    image: daocloud.io/library/php:7.2.24-fpm
    container_name: php
    volumes:
        - "/Users/kagaya/docker/www:/var/www/html"
```

在www目录下放一个index.php文件输出phpinfo用于测试，最后在浏览器里访问localhost

![php](https://raw.githubusercontent.com/kagaya85/ImageHost/master/uPic/2020/09/CleanShot%202020-09-24%20at%2015.01.24@2x.png)



## 在Docker内连接Mysql

对于Wordpress来说，需要配置对应的mysql数据库，为此，我们需要让nginx和php能够访问到mysql容器，于是我们同样可以使用Link方式来实现

同时，我们还可以将mysql的3306端口映射到主机，方便我们用数据库管理软件进行操作

如果相对数据库中的数据进行持久化，也可以把mysql容器中的`/var/lib/mysql`目录映射出来

```shell
$ docker run -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7.31
```

在配置好以上服务后，我们可以对wordpress目录下的`wp-config.php`文件进行配置，如数据库连接，注意：数据库的Host此时为link后的名称，如在运行nginx和php时使用了如`--link myslq:db`的参数，那么在容器中对于的数据库主机则为`db`

打开Wordpress调试模式，这样可以及时在浏览器中看到错误信息：

```php
/**
 * 开发者专用：WordPress调试模式。
 *
 * 将这个值改为true，WordPress将显示所有用于开发的提示。
 * 强烈建议插件开发者在开发环境中启用WP_DEBUG。
 *
 * 要获取其他能用于调试的信息，请访问Codex。
 *
 * @link https://codex.wordpress.org/Debugging_in_WordPress
 */
define('WP_DEBUG', true);
```

本次配置中使用的PHP版本为`7.2.24`，在PHP7以后，`mysql_connect`函数被完全移除了，而wordpress为了兼容会先检测php环境中有没有`mysqli`模块，否则会使用原来的`mysql_connect`函数。

![wordpress mysql_connect](https://raw.githubusercontent.com/kagaya85/ImageHost/master/uPic/2020/09/8qhyNM.jpg)

需要注意的是PHP的镜像中为了最小化镜像体积，是不包含`mysqli`模块的，当然，官方也很方便的提供了`docker-php-ext-install`工具来方便我们安装模块，因此需要我们使用Dockerfile文件安装`mysqli`模块后再次封装新的镜像。

在`webdev`下新建Dockerfile文件：

```
FROM daocloud.io/library/php:7.2.24-fpm
RUN docker-php-ext-install mysqli
```

最终的docke-compose.yaml文件如下：

```yaml
version: '3'
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    depends_on:
      - php
      - mysql
    volumes:
      - "/Users/kagaya/docker/nginx/conf.d:/etc/nginx/conf.d:ro"
      - "/Users/kagaya/docker/www:/usr/share/nginx/html"
    links:
        - "php:php"
        - "mysql:db"

  php:
    # image: daocloud.io/library/php:7.2.24-fpm
    build: .
    container_name: php
    volumes:
      - "/Users/kagaya/docker/www:/var/www/html"
    links:
      - "mysql:db"

  mysql:
    image: mysql:5.7.31
    container_name: mysql
    ports:
      - "3306:3306"
    environment: 
      MYSQL_ROOT_PASSWORD: "123456"
```

最后，只需要在docker-compose.yaml所在目录运行以下命令

```shell
$ docker-compose up -d
```

初次运行会自动编译镜像，之后每次运行配置不发生改变就会启用原来的容器，若配置发生改变，也会自动重建对应容器

![docker-compose](https://raw.githubusercontent.com/kagaya85/ImageHost/master/uPic/2020/09/qXQEW7.jpg)

这样，我们就可以直接在本地调试wordpress中的php代码了，并且在vscode中也可以获得代码提示

![CleanShot2020-09-24at15.00.35@2x](https://raw.githubusercontent.com/kagaya85/ImageHost/master/uPic/2020/09/CleanShot%202020-09-24%20at%2015.00.35@2x.png)