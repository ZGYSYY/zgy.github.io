---
title: Docker搭建JavaWeb环境
date: 2020-01-03 11:38:02
tags:
- Java
- Docker
categories:
- 后端
- Docker
---

[本教程用到的资源，点击下载](/download/Docker搭建JavaWeb环境/source.rar)

# 准备

- CentOS7.5
- 必须有root权限

<!-- more -->

# 安装docker

[docker CentOS安装文档](https://docs.docker.com/install/linux/docker-ce/centos/)

1. 安装一些工具和驱动 `yum install -y yum-utils device-mapper-persistent-data lvm2`

2. 添加yum源 `yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`

3. 安装docker `yum install docker-ce`

4. 验证docker是否安装成功 `docker -v`

5. 如果步骤4成功，启动docker `sytemctl start docker`

6. 配置镜像加速器

   ```bash
   sudo mkdir -p /etc/docker
   sudo tee /etc/docker/daemon.json <<-'EOF'
   {
     "registry-mirrors": ["https://jtpbxytq.mirror.aliyuncs.com"]
   }
   EOF
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```

7. 运行一个docker测试 `docker run hello-world`

8. 可以自行百度，安装一个docker命令自动补全的工具（非必须）

# 安装数据库-mariadb

1. 搜索镜像，`docker search mariadb`

2. 拉取镜像，`docker pull mariadb`

3. 创建容器，`docker run -tid --name mariadb -p 7788:3306 -e MYSQL_ROOT_PASSWORD=root -v /usr/mariadb/:/var/lib/mysql mariadb`

   **注意：**/var/lib/mysql 这个是不变的，这是容器中的数据库文件位置。

4. 连接数据库

# 制作java运行环境进行

1. 到jdk官网下载server-jre，并上传到Linux服务器中

   [server-jre下载](https://www.oracle.com/technetwork/java/javase/downloads/server-jre8-downloads-2133154.html)

2. 编写Dockerfile文件

   ```
   # 基于frolvlad/alpine-glibc镜像
   FROM frolvlad/alpine-glibc
   
   # 作者
   MAINTAINER ZGY <3030392760@qq.com>
   
   # 在容器中更新软件包并安装tzdata（一款管理时间的软件），然后设置运行环境时间与宿主机相同
   RUN apk update && apk add tzdata && cp -rf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
   
   # 在容器中创建jar运行目录
   RUN mkdir -p /apps
   
   # 将宿主机中的server-jar包复制到容器中，并解压
   ADD ./server-jre-8u191-linux-x64.tar.gz /usr/local/
   
   # 设置环境变量
   EVN JAVA_HOME /usr/local/jdk1.8.0_191
   EVN JAVA_OPTS "-server -Xms512M -Xmx1024M"
   EVN CLASSPATH $JAVA_HOME/bin
   EVN PATH $PATH:$JAVA_HOME/bin
   ```

3. 创建镜像

   **注意：**需要进入Dockerfile文件和jar包对应的目录中，运行下面的命令

   `docker build -t zgy/demo .`

4. 运行容器

   `docker run -tid --name zgy.demo zgy/demo`

参考链接：

1. [Alpine Linux 中的 apk 命令讲解](https://blog.csdn.net/liupeifeng3514/article/details/80418887)
2. [构建自定义镜像](https://blog.csdn.net/xie19900123/article/details/81410006)

# 安装DNS服务器-dnsmasq

> 什么是dnsmasq？
>
> DNSmasq是一个小巧且方便地用于配置[DNS](https://baike.baidu.com/item/DNS/427444)和[DHCP](https://baike.baidu.com/item/DHCP)的工具，适用于小型[网络](https://baike.baidu.com/item/%E7%BD%91%E7%BB%9C/143243)，它提供了DNS功能和可选择的DHCP功能。它服务那些只在本地适用的域名，这些域名是不会在全球的DNS服务器中出现的。DHCP服务器和DNS服务器结合，并且允许DHCP分配的地址能在DNS中正常解析，而这些DHCP分配的地址和相关命令可以配置到每台[主机](https://baike.baidu.com/item/%E4%B8%BB%E6%9C%BA/455151)中，也可以配置到一台核心设备中（比如路由器），DNSmasq支持静态和动态两种DHCP配置方式。 

1. 搜索镜像

   `docker search dnsmasq`

2. 拉取镜像

   `docker pull dnsmasq`

3. 运行容器

   `docker run -d -p 53:53/udp -p 53:53/tcp --cap-add=NET_ADMIN -v /usr/dnsmasq:/etc/dnsmasq.d --name dns-server andyshinn/dnsmasq`

4. 修改配置文件，在宿主机的配置文件中添加要解析的域名

   1. `vim /user/dnsmasq/dns-zgy.conf`，添加如下配置

   ```
   address=/noobgg.test/192.168.31.100
   ```

5. 重启容器

   `docker restart dns-server`

6. 测试

   在另一台机器上将首选dns改为安装有dnsmasq服务器的ip地址，然后ping noobgg.test，看是否能够ping通，如果能，则安装成功。

参考链接：

1. [解决Docker容器时区及时间不同步问题](https://www.cnblogs.com/javacspring/p/6172327.html) 

# 安装Nginx

1. 搜索镜像

   `docker search nginx`

2. 拉取镜像

   `docker pull nginx`

3. 运行容器

   `docker run -d --name zgy.nginx -p 80:80 -v /usr/nginx/conf/conf.d:/etc/nginx/conf.d -----link zgy.demo nginx`

4. 修改配置文件default.conf，配置如下

   ```conf
   server {
       listen       80;
       server_name  noobgg.test;
   
       location / {
           proxy_pass  http://zgy.demo:9090;
           root   /usr/share/nginx/html;
           index  index.html index.htm;
       }
   
       #error_page  404              /404.html;
   
       # redirect server error pages to the static page /50x.html
       #
       error_page   500 502 503 504  /50x.html;
       location = /50x.html {
           root   /usr/share/nginx/html;
       }
   
   }
   ```

5. 重启容器

   `docker restart zgy.nginx`

6. 测试

   打开浏览器访问：http://noobgg.test

参考链接：

1. [docker容器相互访问](https://blog.csdn.net/subfate/article/details/81396532)
