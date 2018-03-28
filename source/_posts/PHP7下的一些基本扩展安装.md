title: PHP7下的一些基本扩展安装
categories: Linux
tags: [PHP7扩展编译]
date: 2016-10-14 14:50:00
---


### mysql扩展的安装
由于php7已经默认把mysql去除，所以需要自己去pecl上编译mysql扩展。具体链接如下：http://git.php.net/?p=pecl/database/mysql.git;a=summary

```sh
tar zxvf mysql-version.tar.gz
cd mysql-version
/web/server/php/bin/phpize
./configure --with-php-config=/web/server/php/bin/php-config
make && make install
```
修改配置文件php.ini,添加 `extension=mysql.so`
重启php-fpm使其生效
<!-- more -->
### xdebug扩展的安装
```sh
git clone git://github.com/xdebug/xdebug.git
cd xdebug
/web/server/php/bin/phpize
./configure --with-php-config=/web/server/php/bin/php-config
make && make install
```
修改配置文件php.ini,添加以下代码，然后重启php-fpm使其生效
```sh
;[xdebug]
zend_extension=xdebug.so
xdebug.remote_enable = On
xdebug.remote_handler = dbgp   
xdebug.remote_host= localhost
xdebug.remote_port = 9001
xdebug.remote_connect_back = on
;xdebug.var_display_max_depth =  6  
xdebug.idekey = PHPSTORM
xdebug.remote_autostart = 1

;xdebug.auto_trace=On
xdebug.collect_params=On
xdebug.collect_return=On
xdebug.trace_output_dir="/vagrant/xdebug"
;xdebug.profiler_enable=On
xdebug.profiler_output_dir="/vagrant/xdebug"
;xdebug.show_exception_trace  =  On    ;是否开启远程调试自动启动
xdebug.max_nesting_level  =  200  ;最大循环或调试次数，防止死循环    
```

### memcached 安装（php-memcache的扩展）
首先下载memcached的服务端，然后编译安装
```sh
wget http://www.memcached.org/files/memcached-1.4.32.tar.gz
tar zxvf memcached-1.4.32.tar.gz
cd memcached-1.4.32
./configure --prefix=/web/server/memcached
make && make install
```
编译php7-memcache的扩展,首先从http://www.qingxinzui.com/wp-content/uploads/2016/02/pecl-memcache-php7.tar.gz下载好源码文件
```sh
tar zxvf pecl-memcache-php7.tar.gz
cd pecl-memcache-php7
/web/server/php/bin/phpize
./configure --with-php-config=/web/server/php/bin/php-config
make && make install
```
修改配置文件php.ini,添加 extension=memcache.so
重启php-fpm使其生效

编写memcached的启动脚本
```sh
vi /etc/init.d/memcached
```
写入下面脚本
```sh
#!/bin/sh
#
# memcached    Startup script for memcached processes
#
# chkconfig: - 90 10
# description: Memcache provides fast memory based storage.
# processname: memcached

[ -f memcached ] || exit 0

memcached="/web/server/memcached/bin/memcached"
prog=$(basename  $memcached)
port=11211
user=www
# memory use
mem=64

start() {
    echo -n $"Starting $prog "
    # Starting memcached with 64MB memory on port 11211 as deamon and user nobody
    $memcached -m $mem -p $port -d -u $user

    RETVAL=$?
    echo
    return $RETVAL
}

stop() {
    if test "x`pidof memcached`" != x; then
        echo -n $"Stopping $prog "
        killall memcached
        echo
    fi
    RETVAL=$?
    return $RETVAL
}

case "$1" in
        start)
            start
            ;;

        stop)
            stop
            ;;

        restart)
            stop
            start
            ;;
        condrestart)
            if test "x`pidof memcached`" != x; then
                stop
                start
            fi
            ;;

        *)
            echo $"Usage: $0 {start|stop|restart|condrestart}"
            exit 1

esac

exit $RETVAL
```
给予执行权限，然后设置开机启动
```sh
chkconfig --add memcached  
chkconfig memcached on
service memcached start
```

### redis的编译安装
```sh
wget http://download.redis.io/releases/redis-3.2.3.tar.gz
tar zxvf redis-3.2.3.tar.gz
cd redis-3.2.3
make
make PREFIX=/web/server/redis install
mkdir -p /web/server/redis/etc
cp redis.conf /web/server/redis/etc
cd /web/server/redis
cp redis-benchmark redis-cli redis-server /usr/bin/
```
调整下内存分配使用方式并使其生效
```sh
#此参数可用的值为0,1,2 
#0表示当用户空间请求更多的内存时，内核尝试估算出可用的内存 
#1表示内核允许超量使用内存直到内存用完为止 
#2表示整个内存地址空间不能超过swap+(vm.overcommit_ratio)%的RAM值 
echo "vm.overcommit_memory=1">>/etc/sysctl.conf
sysctl -p
```
修改redis的配置
```sh
vim /usr/local/redis/etc/redis.conf

# 修改以下配置
# redis以守护进程的方式运行
# no表示不以守护进程的方式运行(会占用一个终端)  
daemonize yes

# 客户端闲置多长时间后断开连接，默认为0关闭此功能                                      
timeout 300

# 设置redis日志级别，默认级别：notice                    
loglevel verbose

# 设置日志文件的输出方式,如果以守护进程的方式运行redis 默认:"" 
# 并且日志输出设置为stdout,那么日志信息就输出到/dev/null里面去了 
logfile stdout
```
redis环境变量配置
```sh
vim /etc/profile

PATH=$PATH:/web/server/nginx/sbin:/web/server/php/bin:/web/server/mysql/bin:/web/server/redis/bin
export PATH

# 保存退出

# 让环境变量立即生效
source /etc/profile
```
设置redis启动脚本
```sh
#!/bin/bash
#chkconfig: 2345 80 90
# Simple Redis init.d script conceived to work on Linux systems
# as it does use of the /proc filesystem.

PATH=/usr/local/bin:/sbin:/usr/bin:/bin
REDISPORT=6379
EXEC=/web/server/redis/bin/redis-server
REDIS_CLI=/web/server/redis/bin/redis-cli
   
PIDFILE=/var/run/redis.pid
CONF="/web/server/redis/etc/redis.conf"
   
case "$1" in
    start)
        if [ -f $PIDFILE ]
        then
                echo "$PIDFILE exists, process is already running or crashed"
        else
                echo "Starting Redis server..."
                $EXEC $CONF
        fi
        if [ "$?"="0" ] 
        then
              echo "Redis is running..."
        fi
        ;;
    stop)
        if [ ! -f $PIDFILE ]
        then
                echo "$PIDFILE does not exist, process is not running"
        else
                PID=$(cat $PIDFILE)
                echo "Stopping ..."
                $REDIS_CLI -p $REDISPORT SHUTDOWN
                while [ -x ${PIDFILE} ]
               do
                    echo "Waiting for Redis to shutdown ..."
                    sleep 1
                done
                echo "Redis stopped"
        fi
        ;;
   restart|force-reload)
        ${0} stop
        ${0} start
        ;;
  *)
    echo "Usage: /etc/init.d/redis {start|stop|restart|force-reload}" >&2
        exit 1
esac
```
设置redis开机启动
```sh
# 复制脚本文件到init.d目录下
cp redis /etc/init.d/

# 给脚本增加运行权限
chmod +x /etc/init.d/redis

# 添加服务
chkconfig --add redis

# 配置启动级别
chkconfig --level 2345 redis on
service redis start
```
利用pecl直接安装phpredis扩展
```sh
pecl install redis
```
修改配置文件php.ini,添加 `extension=redis.so`
重启php-fpm使其生效
