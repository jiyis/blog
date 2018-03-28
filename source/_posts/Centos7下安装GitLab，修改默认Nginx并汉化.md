title: Centos7下安装GitLab，修改默认Nginx并汉化
categories: Linux
tags: [Linux,GitLab,Git]
date: 2016-12-01 10:32:00
---


### 前言 
尝试够了自己搭建git服务器，突然想给自己公司搭建个带web的gitlab玩玩，然后这一玩就是折腾了一天。踩了各种坑后终于安装好了，顺手记录一下。

### GitLab的安装方法
- 编译安装
	优点：可定制性强。数据库既可以选择MySQL,也可以选择PostgreSQL;服务器既可以选择Apache，也可以选择Nginx。
	缺点：国外的源不稳定，被墙时，依赖软件包难以下载(Mysql，Redis，Postfix，Ruby，Nginx……)。配置流程繁琐、复杂，容易出现各种各样的问题。卸载GitLab相对麻烦。
- 通过rpm包安装
	优点：安装过程简单，安装速度快。采用rpm包安装方式，安装的软件包便于管理。
	缺点：数据库默认采用PostgreSQL，服务器默认采用Nginx，不容易定制，但是可以更改默认Nginx。
<!-- more -->
### 安装依赖库
GitLab的中文社区地址是https://www.gitlab.cc/downloads/#centos7 。
```sh
    # 安装依赖包
    sudo yum install curl openssh-server openssh-clients postfix cronie
    # 启动 postfix 邮件服务
    sudo service postfix start
    # 检查 postfix
    sudo chkconfig postfix on
```
### 下载RPM包安装
从清华镜像下载rpm包，由于中文版本目前最新式8.8，所以下载8.8版本
```sh
curl -LJO https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-8.8.3-ce.0.el7.x86_64.rpm
rpm -i gitlab-ce-8.8.3-ce.0.el7.x86_64.rpm
sudo gitlab-ctl reconfigure  ##初始化配置
```
### 修改host和默认nginx
添加nginx主机配置
```sh
# gitlab socket 文件地址
upstream gitlab {
  # 7.x 版本在此位置
  # server unix:/var/opt/gitlab/gitlab-rails/tmp/sockets/gitlab.socket;
  # 8.0 位置
  server unix://var/opt/gitlab/gitlab-rails/sockets/gitlab.socket;
}

server {
  listen *:80;

  server_name gitlab.liaohuqiu.com;   # 请修改为你的域名

  server_tokens off;     # don't show the version number, a security best practice
  root /opt/gitlab/embedded/service/gitlab-rails/public;

  # Increase this if you want to upload large attachments
  # Or if you want to accept large git objects over http
  client_max_body_size 250m;

  # individual nginx logs for this gitlab vhost
  access_log  /var/log/gitlab/nginx/gitlab_access.log;
  error_log   /var/log/gitlab/nginx/gitlab_error.log;

  location / {
    # serve static files from defined root folder;.
    # @gitlab is a named location for the upstream fallback, see below
    try_files $uri $uri/index.html $uri.html @gitlab;
  }

  # if a file, which is not found in the root folder is requested,
  # then the proxy pass the request to the upsteam (gitlab unicorn)
  location @gitlab {
    # If you use https make sure you disable gzip compression 
    # to be safe against BREACH attack

    proxy_read_timeout 300; # Some requests take more than 30 seconds.
    proxy_connect_timeout 300; # Some requests take more than 30 seconds.
    proxy_redirect     off;

    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   Host              $http_host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Frame-Options   SAMEORIGIN;

    proxy_pass http://gitlab;
  }

  # Enable gzip compression as per rails guide: http://guides.rubyonrails.org/asset_pipeline.html#gzip-compression
  # WARNING: If you are using relative urls do remove the block below
  # See config/application.rb under "Relative url support" for the list of
  # other files that need to be changed for relative url support
  location ~ ^/(assets)/  {
    root /opt/gitlab/embedded/service/gitlab-rails/public;
    # gzip_static on; # to serve pre-gzipped version
    expires max;
    add_header Cache-Control public;
  }

  error_page 502 /502.html;
}
```
添加访问的 host，修改/etc/gitlab/gitlab.rb的external_url
```sh
sudo vi /etc/gitlab/gitlab.rb
...
#这里填写你的域名或者ip
external_url 'http://git.home.com'
#设置nginx为false
nginx['enable'] = false
...
```
重启 nginx, 重启gitlab
```sh
sudo gitlab-ctl reconfigure
systemctl restart nginx
```
这个时候访问会报502。原本是 nginx 用户无法访问gitlab用户的 socket 文件，用户权限配置，因人而异
```sh
sudo chmod -R o+x /var/opt/gitlab/gitlab-rails
```
这时候打开你的域名，可以看到gitlab登录页面，代表安装成功。查看下版本
```sh
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```


### 开始汉化
这里一定要看清楚GitLab最新的中文语言包版本。可以到中文社区看https://gitlab.com/larryli/gitlab/wikis/home
```sh
git clone https://git.coding.net/larryli/gitlab.git
#这里比较分支网上的大部分都不能执行成功，这里给出正确的方式
cd gitlab  ##一定要进入git目录你才能用git diff
git diff origin/8-8-stable origin/8-8-zh > /tmp/8.8.diff ## 注意这边 不要写.. ，如果报错可以用git branch -a查看下远程最新的分支版本号
#停止gitlab服务
sudo gitlab-ctl stop
#把中文语言包覆盖，如果没有patch可以yum install patch
sudo patch -d /opt/gitlab/embedded/service/gitlab-rails -p1 < /tmp/8.8.diff
#重新启动gitlab，并且给予nginx权限
sudo gitlab-ctl restart
sudo chmod -R o+x /var/opt/gitlab/gitlab-rails
```
至此汉化完毕，打开http://git.home.com ，便会看到中文版的GitLab。如下：
![S85DO[16YV793@64{[P{5B6.png](http://gary.yearn.cc/usr/uploads/2016/12/361759954.png)

### 卸载方法
这一步我被坑了不少，一开始我安装了最新的gitlab8.14，发现中文包没有更新过来，但是找不到卸载方法，于是各种覆盖安装和删除文件，最后导致了500错误，redis启动不了。
```git
# Stop gitlab and remove its supervision process
sudo gitlab-ctl uninstall
# Debian/Ubuntu
sudo dpkg -r gitlab-ce
# Redhat/Centos
sudo rpm -e gitlab-ce
```
如果需要使用Webhooks，可以参考[GitLab Web Hook For PHP](https://github.com/bravist/gitlab-webhook-php "GitLab Web Hook For PHP")