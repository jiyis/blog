title: 如何制作自己的docker镜像
categories: Docker
tags: [lnmp,centos7,php7]

## date: 2018-04-16 14:07:00

### 前言

一般来讲Docker镜像有两种构建方式.一种是直接通过`docker commit` 命令来创建镜像,另一种是通过Dockerfile来创建镜像.

### 两种构建方式的区别

* 容器镜像的构建者可以任意修改容器的文件系统后进行发布，这种修改对于镜像使用者来说是不透明的，镜像构建者一般也不会将对容器文件系统的每一步修改，记录进文档中，供镜像使用者参考。
* 容器镜像不能（更准确地说是不建议）通过修改，生成新的容器镜像。


* 容器镜像依赖的父镜像变化时，容器镜像必须进行重新构建。如果没有编写自动化构建脚本，而是手工构建的，那么又要重新修改容器的文件系统，再进行构建，这些重复劳动其实是没有价值的。

* Dockerfile镜像是完全透明的，所有用于构建镜像的指令都可以通过Dockerfile看到。甚至你还可以递归找到本镜像的任何父镜像的构建指令。也就是说，你可以完全了解一个镜像是如何从零开始，通过一条条指令构建出来的。

* Dockerfile镜像需要修改时，可以通过修改Dockerfile中的指令，再重新构建生成，没有任何问题。

* Dockerfile可以在GitHub等源码管理网站上进行托管，DockerHub自动关联源码进行构建。当你的Dockerfile变动，或者依赖的父镜像变动，都会触发镜像的自动构建，非常方便。

  <!-- more -->

​      从镜像运行容器，实际上是在镜像顶部上加了一层可写层，所有对容器文件系统的修改，都在这一层中进行，不影响已经存在的层。比如在容器中删除一个1G的文件，从用户的角度看，容器中该文件已经没有了，但从文件系统的角度看，文件其实还在，只不过在顶层中标记该文件已被删除，当然这个标记为已删除的文件还会占用镜像空间。从容器构建镜像，实际上是把容器的顶层固化到镜像中。
也就是说， 对容器镜像进行修改后，生成新的容器镜像，会多一层，而且镜像的体积只会增大，不会减小。长此以往，镜像将变得越来越臃肿。Docker提供的 `export` 和 `import` 命令可以一定程度上处理该问题，但也并不是没有缺点。

​	不管是官方还是我个人，都推荐使用第二种方式构建镜像。所以这里只讲一下Dockerfile的方式制作docker.

### 基础镜像

首先要找到适合自己的基础镜像,比如我需要用centos的,我可以直接`docker pull centos`来拉取到本地.

### Dockerfile

Dockerfile由一行行命令语句组成，并且支持用“#”开头作为注释，一般的，Dockerfile分为四部分：基础镜像信息，维护者信息，镜像操作指令和容器启动时执行的指令。

* FROM：指定父镜像，可以通过添加多个FROM，在同一个Dockerfile中创建多个镜像

* MAINTAINER：维护者信息，可选

* RUN：用来修改镜像的命令，可以用来安装程序，当一条RUN完成后，会在当前的镜像上创建一个新的镜像层，接下来的指令会在新的镜像层上执行。有2种形式。 

  * RUN ["yum", "update"]，调用exec


  * RUN yum update,调用的/bin/sh

* EXPOSE：用来指明容器内进程对外开放的端口。在docker run的时候可以加-p（可以将EXPOSE中没列出的端口设置为对外开放）和-P（EXPOSE里所指定的端口映射到主机上另外的随机端口？？？）来设置端口。

* ADD：向新容器中添加文件,文件可以是 

  * 主机文件：必须是相对Dockerfile所在目录的相对路径（如果是压缩文件，docker会解压缩）
  * 网络文件：URL文件，在创建容器时会下载下来添加到镜像中。（如果是压缩文件，docker不会解压缩）
  * 目录：必须是相对Dockerfile所在目录的相对路径（如果是压缩文件，docker会解压缩）

* VOLUME：会在镜像里创建一个指定路径的挂载点。这个路径可以来自主机，也可以来自其他容器，多个容器通过同一个挂载点来共享数据，即便有个容器已经停止，其余容器还是可以访问挂载点，只有当挂载点所有的容器引用消失，挂载点才会自动删除。

* WORKDIR：为接下来的指令指定一个新的工作目录。当启动一个容器后，最后一条WORKDIR指令所指向的目录为容器当前运行的工作目录。

* ENV：设置环境变量，在docker run 时可以用-e来设置环境变量`docker run -e WEBAPP_PORT=8000 -e WEBAPP_HOST=www.example.com`

* CMD：设置容器运行时默认运行的命令，CMD参数格式与RUN类似。`CMD ls -l -a` 或`CMD ["ls", "-l", "-a"]`

* ENTRYPOIN：与CMD类似，指定容器运行时默认命令。ENTRYPOINT和CMD的区别，在于运行容器时，镜像后的命令参数，ENTRYPOINT是拼接，CMD是覆盖

* USER：为容器的运行和RUN CMD ENTRYPOINT等指令的运行  指定用户或者UID

* ONBUILD：触发器指令，父镜像中不会执行，只有在子镜像中才会执行。  
  给一个例子

```dockerfile
FROM centos

MAINTAINER gary <dongpengfei@patsnap.com>

RUN \
    yum -y install which sudo gcc autoconf libjpeg libpng freetype libxml2 zlib zlib-devel glibc glib2 bzip2 curl e2fsprogs openssl make cmake gd gd2 libmcrypt readline libxslt libmemcached libmemcached-devel freetds freetds-devel openldap openldap-devel ImageMagick ImageMagick-perl supervisor && \
    yum clean all

RUN \
    rpm -ivh http://192.168.3.60/rpm/7/nginx-1.12.1-2.el7.x86_64.rpm && \
    rpm -ivh http://192.168.3.60/rpm/7/php-7.1.8-6.el7.x86_64.rpm && \
    rpm -ivh http://192.168.3.60/rpm/7/jre-8u111-linux-x64.rpm && \
    rpm -ivh http://192.168.3.60/rpm/7/mysql57-community-release-el7-7.noarch.rpm

RUN yum -y install mysql-community-client

COPY conf/supervisor/*.ini /etc/supervisord.d/

COPY conf/php/ /usr/local/php/etc/

COPY conf/php/extensions/* /usr/local/php/lib/php/extensions/no-debug-non-zts-20160303/

COPY conf/nginx/ /usr/local/nginx/conf/conf.d/


COPY conf/init/ /config/init/

EXPOSE 80

```

### 制作自己的RPM包

我不太喜欢直接用yum的方式来安装lnmp环境,我喜欢自己编译,所以通过rpm包的方式来制作docker是最佳实践.

首选你要基于你选中的base镜像来制作,否则会出现不明的bug,这边我在dockerfile中是基于官方的centos的,所以我就直接在此镜像中制作.

以前打包rpm是一个非常复杂的一件事情，自从有了fpm，打包rpm就和tar打包文件一样简单.

#### 支持的源类型包

-  dir: 将目录打包成所需要的类型，可以用于源码编译安装的软件包
-  rpm: 对rpm进行转换
-  gem: 对rubygem包进行转换
-  python: 将Python模块打包成相应的类型

#### 支持的目标类型包

- rpm: 转换为rpm包
- deb: 转换为deb包
- solaris: 转换为solaris包
- puppet: 转换为puppet包

#### FPM常用参数

- -s:指定源类型
- -t:指定目标类型，即想要制作为什么包
- -n:指定包的名字
- -v:指定包的版本号
- -C:指定打包的相对路径
- -d:指定依赖于哪些包
- -f:第二次包时目录下如果有同名安装包存在，则覆盖它；
- -p:制作的rpm安装包存放路径，不想放在当前目录下就需要指定；
- --post-install:软件包安装完成之后所要运行的脚本；同--offer-install
- --pre-install:软件包安装完成之前所要运行的脚本；同--before-install
- --post-uninstall:软件包卸载完成之后所要运行的脚本；同--offer-remove
- --pre-uninstall:软件包卸载完成之前所要运行的脚本；同—before-remove
  --prefix:制作好的rpm包默认安装路径；

#### 安装FPM 

```shell
   yum -y install ruby rubygems ruby-devel
　　# 添加淘宝Ruby仓库
　　gem sources -a http://ruby.taobao.org/
　　# 移除原生的Ruby仓库
　　gem sources --remove http://rubygems.org/
　　# 安装fpm
　　gem install fpm
```

#### 几个参照demo

```shell
 #  php 
 fpm -f -s dir -t rpm -n php --epoch 0 -v 7.1.8 --iteration 3.el7 -C /data/scripts/php -d 'libjpeg libpng freetype libxml2 zlib glibc glib2 bzip2 curl e2fsprogs openssl make  cmake libmcrypt readline libxslt' --before-install /data/scripts/php/before_install.sh --after-install /data/scripts/php/after_install.sh --after-remove /data/scripts/php/after_remove.sh --workdir /data/scripts/php etc usr

#  nginx
 fpm -f -s dir -t rpm -n nginx --epoch 0 -v 1.12.1 --iteration 2.el7 -C /data/scripts/nginx -d 'openssl' --before-install /data/scripts/nginx/before_install.sh --after-install /data/scripts/nginx/after_install.sh --after-remove /data/scripts/nginx/after_remove.sh --workdir /data/scripts/nginx etc usr

#  supervisor
 fpm -f -s dir -t rpm -n supervisor --epoch 0 -v 3.1.4 --iteration 1.el7 -C /data/scripts/supervisor -d 'openssl' --before-install /data/scripts/supervisor/before_install.sh --after-install /data/scripts/supervisor/after_install.sh --after-remove /data/scripts/supervisor/after_remove.sh --workdir /data/scripts/supervisor etc

```

#### 制作镜像常用的docker命令

```shell
# build镜像
sudo docker build -t registry.cn-hangzhou.aliyuncs.com/jiyis/ms_basic:2.0.3 .

# 推送到指定的dcoker仓库
sudo docker push registry.cn-hangzhou.aliyuncs.com/jiyis/rpmpackage:2.1.1

# 打tag
sudo docker tag [ImageId] registry.cn-hangzhou.aliyuncs.com/jiyis/rpmpackage:2.1.1

# 进入容器内部  16c759fffa3a 是docker容器id或者容器名称,可通过docker ps查看
sudo docker exec -it 16c759fffa3a /bin/bash

# -m表示一个commit message，-a 表示author的信息，指定创建的new iamge来自于container id 0b2616b0e5a8，此外还为新建的image定义一个组合名称ouruser/sinatra:v2。使用docker images可以看到新建的images
sudo docker commit -m "Added json gem" -a "Kate Smith" 0b2616b0e5a8 ouruser/sinatra:v2

# 从docker 复制到物理机
docker cp <containerId>:/file/path/within/container /host/path/target

# 一次性进入docker容器并销毁
sudo docker run -p 8333:80 -v /jiyi/www/php/project-pms:/data/inno/miscroservice/ --rm -it  dtr.patsnap.com/inno/ms_basic:2.1.1 bash

```

