title: Centos7 下编译安装nginx+php7+mariadb
categories: Linux
tags: [lnmp,centos7,php7]
date: 2016-10-13 14:07:00
---
> ### 前言
由于自己采用vagrant作为本地开发环境，所以注定会经常的编译和升级环境了。升级到Centos 7后不得不重新搭建一个本地的box，所以特此记录下方法，供自己以后参考。废话不多说，接上步骤！
<!-- more -->
> ### 更新系统软件
备份下默认的yum源
```sh
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```
更换为阿里云的yum源
```sh
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```
生成缓存
```sh
yum makecache
```
更新系统软件
```sh
yum update
```

> ### 准备工作

安装基本的依赖库
```sh
mkdir -p /web/source
cd /web/source
yum install wget
yum install pcre
yum install openssl*
yum -y install gcc gcc-c++ autoconf libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libxml2 libxml2-devel zlib zlib-devel glibc glibc-devel glib2 glib2-devel bzip2 bzip2-devel ncurses ncurses-devel curl curl-devel e2fsprogs e2fsprogs-devel krb5 krb5-devel libidn libidn-devel openssl openssl-devel openldap openldap-devel nss_ldap openldap-clients openldap-servers make jemalloc-devel  cmake boost-devel
yum -y install gd gd2 gd-devel gd2-devel
/usr/sbin/groupadd www
/usr/sbin/useradd -g www www
ulimit -SHn 65535
wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.39.tar.gz
tar zxvf pcre-8.39.tar.gz
cd pcre-8.39
./configure --prefix=/web/server/pcre
make && make install
cd ../
```
> ### 编译安装nginx

```sh
wget http://nginx.org/download/nginx-1.10.1.tar.gz
tar zxvf nginx-1.10.1.tar.gz  
cd nginx-1.10.1  
./configure --user=www --group=www --prefix=/web/server/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre=/web/source/pcre-8.39 --with-http_realip_module --with-http_image_filter_module  
make  
make install  
cd ../  
```
编写nginx启动脚本
```sh
vi /usr/lib/systemd/system/nginx.service
```
然后讲下列脚本复制进去，保存。
```sh
[Unit]
Description=The nginx HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/web/server/nginx/nginx.pid
ExecStartPre=/web/server/nginx/sbin/nginx -t
ExecStart=/web/server/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP /web/server/nginx/nginx.pid
ExecStop=/bin/kill -s QUIT /web/server/nginx/nginx.pid
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
给予执行权限并设置开机启动
```sh
chmod +x /usr/lib/systemd/system/nginx.service
systemctl enable nginx.service
```

> ### 编译安装mariadb
添加mysql用户组和用户
```sh
/usr/sbin/groupadd mysql  
/usr/sbin/useradd -g mysql mysql
mkdir -p /web/data/mysql 
chown -R mysql:mysql /web/data/mysql
wget https://mirrors.tuna.tsinghua.edu.cn/mariadb//mariadb-5.5.52/source/mariadb-5.5.52.tar.gz
```
编译安装，由于tokudb一直报gcc错误，先不编译。
```sh
tar -zxvf mariadb-5.5.52.tar.gz &&
cd mariadb-5.5.52
cmake . -DCMAKE_INSTALL_PREFIX=/web/server/mysql -DMYSQL_DATADIR=/web/data/mysql -DSYSCONFDIR=/etc -DWITHOUT_TOKUDB=1
make
make install
```
初始化数据库,拷贝启动脚本
```sh
rm -rf  /etc/my.cnf 
cd /web/server/mysql
./scripts/mysql_install_db --user=mysql --basedir=/web/server/mysql --datadir=/web/data/mysql
ln -s /web/server/mysql/my.cnf /etc/my.cnf 
cp ./support-files/mysql.server /etc/rc.d/init.d/mysqld 
chmod 755 /etc/init.d/mysqld 
chkconfig --add mysqld
chkconfig mysqld on
service mysqld start
```

> ### 编译安装php
安装php所需要的基本依赖库，centos源不能安装libmcrypt-devel，由于版权的原因没有自带mcrypt的包
```sh
wget http://www.atomicorp.com/installers/atomic
sh ./atomic
yum install libxml2 libxml2-devel openssl openssl-devel bzip2 bzip2-devel libcurl libcurl-devel libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel gmp gmp-devel php-mcrypt  libmcrypt libmcrypt-devel readline readline-devel libxslt libxslt-devel
```
编译配置
```sh
wget http://cn2.php.net/distributions/php-7.0.11.tar.gz
tar zxvf php-7.0.11.tar.gz
cd php-7.0.11.tar.gz
./configure --prefix=/web/server/php --with-config-file-path=/web/server/php/etc --with-fpm-user=www --with-fpm-group=www --with-iconv-dir --with-freetype-dir --with-jpeg-dir --with-png-dir --with-zlib --with-libxml-dir --enable-xml --disable-rpath --enable-bcmath --enable-shmop --enable-sysvsem --enable-inline-optimization --with-curl --enable-mbregex --enable-fpm --enable-mbstring --with-mcrypt --with-gd --enable-gd-jis-conv --enable-gd-native-ttf --with-openssl --with-mhash --enable-pcntl --enable-sockets --with-xmlrpc --enable-zip --enable-soap --enable-opcache --with-libmbfl --with-onig --enable-pdo --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-pdo-mysql --enable-mysqlnd-compression-support --with-pear --enable-maintainer-zts   
make  
make install  
cp php.ini-development /web/server/php/etc/php.ini  
cp php.ini-development /web/server/php/etc/php.ini
cp /web/server/php/etc/php-fpm.conf.default /web/server/php/etc/php-fpm.conf
cp /web/server/php/etc/php-fpm.d/www.conf.default /web/server/php/etc/php-fpm.d/www.conf
```
编写php-fpm启动脚本
```sh
vi /usr/lib/systemd/system/php-fpm.service
```
然后讲下列脚本复制进去，保存。
```sh
[Unit]
Description=The PHP FastCGI Process Manager
After=syslog.target network.target

[Service]
Type=simple
PIDFile=/web/server/php/var/run/php-fpm.pid
ExecStart=/web/server/php/sbin/php-fpm --nodaemonize --fpm-config /web/server/php/etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID

[Install]
WantedBy=multi-user.target

```
给予执行权限并设置开机启动
```sh
chmod +x /usr/lib/systemd/system/php-fpm.service
systemctl enable php-fpm.service
```

> ### 设置nginx、mysql和php的环境变量
```sh
vi /etc/profile
```
然后按下shift+g到文档最底部，假如下列代码
```sh
PATH=$PATH:/web/server/nginx/sbin:/web/server/php/bin:/web/server/mysql/bin
export PATH
```
刷新使其立即生效
```sh
source /etc/profile
```

> ### 更换防火墙
关闭firewall
```sh
systemctl stop firewalld.service 
systemctl disable firewalld.service 
```
安装iptables
```sh
yum install iptables-services 
vi /etc/sysconfig/iptables 
```
在中间加入下列两行
```sh
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT
```
重启生效
```sh
systemctl restart iptables.service 
systemctl enable iptables.service 
```

> ### 关闭SELINUX
```sh
vi /etc/selinux/config
#SELINUX=enforcing #注释掉
#SELINUXTYPE=targeted #注释掉
SELINUX=disabled #增加
```
:wq! #保存退出
使配置立即生效
```sh
setenforce 0 
```