# Nginx



## LNMP

**1.简介**

工作中常会需要多个容器相互配合来完成某项任务，如实现一个web项目，需要web服务器、数据库服务器、负载均衡等，使用docker逐个构建则任务繁重。Compose是docker官方的开源项目，定义和运行多个docker容器的应用，能实现对docker容器集群的快速编排，定义一组相关联的容器为一个项目。

LNMP（LEMP）即Linux + Nginx + MySQL + PHP 的服务器架构，与LAMP相比，Nginx性能更强，资源占用少，效率更高。

LNMP的实现原理：

 - 浏览器发送http请求到服务器（Nginx），服务器响应并处理web请求，将静态资源保存在服务器
 - PHP脚本通过接口传输协议（网关协议）php-fcgi传输给php-fpm（进程管理程序）,PHP-FPM不做处理，然后PHP-FPM调用PHP解析器进程，PHP解析器解析php脚本信息。PHP解析器进程可以启动多个，进行并发执行。
 - 将解析后的脚本返回到PHP-FPM，PHP-FPM再通过fast-cgi的形式将脚本信息传送给Nginx。
 - 服务器再通过Http response的形式传送给浏览器。浏览器再进行解析与渲染然后进行呈现。

![流程图](https://github.com/bravist/lnmp-docker/raw/master/docker.png)

**2.使用**

- 环境准备
docker
- docker-compose的简单配置
在目标文件夹内创建docker-compose.yml

```
# docker-compose.yml
version: '3'

services:
    # Web server
    Nginx:
        image: nginx:latest
        ports:
            - 23333:80
        depends_on:
            - php
        volumes:
            - ./images/nginx/config:/etc/nginx/conf.d:ro

    # PHP
    php:
        image: php:latest
        build:
            # Dockerfile所在的文件目录和文件名
            context: ./images/php
            dockerfile: Dockerfile
        volumes:
            - ./apps:/mnt/apps

    # database
    MySQL:
        image: mysql:latest
        ports:
		    - 3306:3306
        # 配置环境变量
        environment:
            RACK_ENV: development
            MYSQL_ROOT_PASSWORD: 'root'
            MYSQL_USER: 'root'
            MYSQL_PASSWORD: 'passwd'
        volumes:
            - ./database:/var/lib/mysql

    # 便于命令工具操作项目文件
    console:
		build:
		    context: ~/workspace/lnmp/images/console
		    dockerfile: /Users/eric/workspace/lnmp/images/php/Dockerfile
		tty: true
```

在docker-compose.yml中改变Nginx映射的配置目录，在新目录下增加配置default.conf

```
# default.conf
# 虚拟主机配置
server{
    listen 80;
    server_name localhost;# 域名
    root /mnt/apps;# 站点目录
    index index.php index.html index.htm;
    location / {
        index index.php index.html;
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
	    fastcgi_pass php:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```
添加Dockerfile，代码如下：

```
# Dockerfile
FROM php:7.2-fpm

RUN apt-get update && apt-get install -y apt-transport-https

RUN apt-get update && apt-get install -y pdo pdo_mysql pdo_pgsql \
    && docker-php-ext-install pdo \
    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-install pdo_pgsql

RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng-dev \
    && docker-php-ext-install -j$(nproc) iconv mcrypt \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd

# 安装 composer
RUN curl -o composer.phar https://getcomposer.org/download/1.4.1/composer.phar \
    && chmod +x composer.phar
```

**3.安装及设置**

- 创建容器
  在Dockerfile的目录中运行命令`docker-compose up --build -d`,运行后docker会有四个容器运行。原目录下有三个子目录：apps用于存放项目文件，database是mySQL的数据库映射，images存放Dockerfile等配置文件。

- 测试

 - Nginx与PHP

     可在项目中的apps目录下添加test.php（如下）测试环境，容器运行后打开127.0.0.1:23333/test.php，如成功则显示当前安装的PHP版本和所有配置信息。

 - PHP与mySQL

     在同上位置添加test-mysql.php，内容如下：

```
//test.php
<?php
    phpinfo();
?>

```

```
//test-mysql.php
<?php
$dbms='mysql';   //数据库类型
$host='localhost'; //数据库主机名
$dbName='mysql';    //使用的数据库
$user='sf';      //数据库连接用户名
$pass='passwd';          //对应的密码
$dsn="$dbms:host=$host;dbname=$dbName";

try {
    $dbh = new PDO($dsn, $user, $pass); //初始化一个PDO对象
    echo "连接成功<br/>";
    $dbh = null;
} catch (PDOException $e) {
    die ("Error!: " . $e->getMessage() . "<br/>");
}
// 如需长链接: $db = new PDO($dsn, $user, $pass, array(PDO::ATTR_PERSISTENT => true));
?>
```

- 删除容器
   无需该环境时即可销毁容器，运行`docker-compose down`。

**4.注意（坑）**

- yaml不允许tab替换空格
- dockerfile构建的路径为绝对路径
- php5以上版本已经废除mysql_connect()方法



## 使用Nginx

**1.简介**

Nginx是异步框架的Web服务器，同时可用作反向代理、负载平衡器和HTTP缓存。

Nginx是面向性能设计的HTTP服务器，相较于Apache、lighttpd具有占有内存少，稳定性高等优势。而且在实际工作中，Nginx可以支持二万到四万个并行链接。

**2.docker与Nginx的配置**

 - 安装：通过`docker run -p 127.0.0.1:3423:80 --name mynginx -d nginx`下载并运行Nginx容器，如成功安装运行，打开127.0.0.1:3423即可看见Nginx欢迎页。
 - 停止：`docker container stop mynginx`，在安装时设置了删除参数，容器终止后容器文件会自动删除
 - 修改映射网页目录：新建并如下目录，新建index.html，将子目录映射到容器的目录，代码如下，然后打开127.0.0.1:9384，看到Hello World。（IP写成127.0.0.1:[四位数字]，否则容易报错）

```bash
$ mkdir nginx-docker-demo
$ cd nginx-docker-demo
$ mkdir html
$ emacs index.html
$ <h1>Hello World</h1>
$ docker run -d -p 127.0.0.1:9384:80 --rm --name mynginx --volume $PWD/html:/usr/share/nginx/html nginx
``````

 - 拷贝配置：将容器里的Nginx配置文件复制到本地 `$ docker container cp mynginx:/etc/nginx .`

 - 映射配置目录
 
```bash
$ docker run  --rm --name mynginx --volume $PWD/html:/usr/share/nginx/html --volume $PWD/conf.d:/etc/nginx -p 127.0.0.1:4023:80 -d nginx
```

**3.反向代理**

 - 概念

 反向代理方式是指用代理服务器来接受 internet 上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给 internet 上请求连接的客户端，此时代理服务器对外就表现为一个反向代理服务器。

 例如，用户访问 <http://www.example.com/readme>，但是 <http://www.example.com> 上并不存在 readme 页面，它是偷偷从另外一台服务器上取回来，然后作为自己的内容返回给用户。但是用户并不知情这个过程。对用户来说，就像是直接从 <http://www.example.com> 获取 readme 页面一样。这里的 <http://www.example.com> 这个域名对应的服务器就设置了反向代理功能。

 反向代理服务器，对于客户端而言它就像是原始服务器，并且客户端不需要进行任何特别的设置。客户端向反向代理的命名空间中的内容发送普通请求，接着反向代理将判断向何处(原始服务器)转交请求，并将获得的内容返回给客户端，就像这些内容原本就是它自己的一样。如下图所示：

 ![反向代理](https://moonbingbing.gitbooks.io/openresty-best-practices/images/proxy.png)

 - 应用

 反向代理的典型用途是将防火墙后面的服务器提供给 Internet 用户访问，加强安全防护。反向代理还可以为后端的多台服务器提供负载均衡，或为后端较慢的服务器提供缓 服务。另外，反向代理还可以启用高级 URL 策略和管理技术，从而使处于不同 web 服务器系统的 web 页面同时存在于同一个 URL 空间下。

 本地访问 <http://localhost/deno> 时服务器进行反向代理，从 <https://github.com/ry/deno> 获取页面内容，添加nginx.conf到Nginx配置目录，生效后运行服务器打开 <http://localhost/deno> 会打开 <https://github.com/ry/deno>，代码如下：

```
worker_process 1;

pid logs/nginx.pid;
error_log logs/error.log warn;

events {
    worker_connections 3000;
}

http {
    include mime.types;
    server_tokens off;

    # 反向代理
    server {
        listen 80;

        location / {
            proxy_pass       https://github.com;
            proxy_redirect   off;
            proxy_set_header Host            $host;
            proxy_set_header X-Real-IP       $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

    	location /deno {
        	proxy_set_header X-Real-IP       $remote_addr;
        	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        	proxy_pass       https://github.com/ry/deno;
    	}
    }
}

```

- 正向代理

正向代理好比一个跳板，当用户访问不了某网站时，能够访问一个代理服务器，通过连接代理服务器，告诉他需要的访问内容，代理服务器拉取给用户，翻墙工具、游戏代理都是正向代理。

**4.负载均衡**

负载均衡是一种计算机网络技术，用来在多个计算机（计算机集群）、网络连接、CPU、磁盘驱动器或其他资源中分配负载，以达到最佳化资源使用、最大化吞吐率、最小化响应时间、同时避免过载的目的。

使用带有负载均衡的多个服务器组件，取代单一的组件，可以通过冗余提高可靠性。负载均衡服务通常是由专用软体和硬件来完成。

负载均衡最重要的一个应用是利用多台服务器提供单一服务，这种方案有时也称之为服务器农场。通常，负载均衡主要应用于 Web 网站，大型的 Internet Relay Chat 网络，高流量的文件下载网站，NNTP 服务和 DNS 服务。现在负载均衡器也开始支持数据库服务，称之为数据库负载均衡器。

对于互联网服务，负载均衡器通常是一个软体程序，这个程序侦听一个外部端口，互联网用户可以通过这个端口来访问服务，而作为负载均衡器的软体会将用户的请求转发给后台内网服务器，内网服务器将请求的响应返回给负载均衡器，负载均衡器再将响应发送到用户，这样就向互联网用户隐藏了内网结构，阻止了用户直接访问后台（内网）服务器，使得服务器更加安全，可以阻止对核心网络栈和运行在其它端口服务的攻击。

当所有后台服务器出现故障时，有些负载均衡器会提供一些特殊的功能来处理这种情况。例如转发请求到一个备用的负载均衡器、显示一条关于服务中断的消息等。负载均衡器使得 IT 团队可以显著提高容错能力。它可以自动提供大量的容量以处理任何应用程序流量的增加或减少。

在 Nginx 中，HTTP Upstream 模块负责负载均衡，这个模块通过一个简单的调度算法来实现客户端 IP 到后端服务器的负载均衡。在如下的设定中，通过 upstream 指令指定了一个负载均衡器的名称 test.net。这个名称可以任意指定，在后面需要用到的地方直接调用即可。

```c
upstream test.net{
    ip_hash;
    server 192.168.10.13:80;
    server 192.168.10.14:80  down;
    server 192.168.10.15:8009  max_fails=3  fail_timeout=20s;
    server 192.168.10.16:8080;
}
server {
    location / {
        proxy_pass  http://test.net;
    }
}
```

**Nginx配置负载均衡**

```c
upstream webservers {
    # ip_hash;
    server 192.168.18.201 weight=1 max_fails=2 fail_timeout=2;
    server 192.168.18.202 weight=1 max_fails=2 fail_timeout=2;
    server 127.0.0.1:8080 backup;
}
server {
    listen       80;
    server_name  localhost;
    location / {
        proxy_pass      http://webservers;
        proxy_set_header  X-Real-IP  $remote_addr;
    }
}
```

 - 重新加载配置文件后，访问该地址时发现上述两个IP是交替出现的，达到负载均衡效果
 - 利用max_fails、fail_timeout参数控制异常情况
 - 当所有服务器都停止工作时启动备份服务器

```
server {
    listen 8080;
    server_name localhost;
    root /data/www/errorpage;
    index index.html;
}

# cat index.html
<h1>Sorry......</h1>

```

 - 当算法为ip_hash时，负载均衡调度的状态不能有backup


# Supervisor
