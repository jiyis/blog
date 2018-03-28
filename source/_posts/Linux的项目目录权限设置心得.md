title: Linux的项目目录权限设置心得
categories: Linux
tags: [Linux,Web安全,目录权限]
date: 2016-09-07 13:50:00
---


### 前言
从接触Linux开始就一直对项目（网站）目录的权限设置有疑问。网上的大部分做法是设置为www用户，但是对于其他的设置并未有很好的说明。直到前段时间研究git自动部署的时候，彻底被权限这东西搞晕了。钩子自动更新的代码经常会使项目的文件所属用户组改变，导致程序不能正常运行。下面整理下比较合理的做法：
<!-- more -->
假设网站根目录为`/data/webroot`，`/data/webroot`要确保是属于root用户的，所有的项目都放到`/data/webroot/（项目名）`下。

- 新建www用户组，然后在www用户组下新建www用户和web用户。
- 把php-fpm和nginx（apache）设置为www用户来运行。
- 把/data/webroot/laravel目录所属设置为www组下的web用户并设置为750权限。（普通文件夹是750，文件是640）  
```sh
chown web:www /data/webroot/laravel -R   
chmod -R 750 /data/webroot/laravel
```
- git钩子使用www组下web用户来clone和pull。    
```sh
sudo -Hu web git pull
```
- 如果项目需要对某些文件有写入权限，则将文件或目录权限修改为770。比如Laravel的storage目录以及public/upload目录。
```sh
chmod 770 /data/webroot/laravel/public/upload -R
chmod 770 /data/webroot/laravel/storage -R
```
- 把所有设置770的目录，禁止解析PHP。
```
location ~* ^/(upload|images)/.*.(php|php5)$
{
	deny all;
}
```
### 总结
遵循以上规则就可以打造一个相对安全的web环境。切记php和nginx的运行用户一定不能是网站目录文件的所有者。 凡是违背这个原则，则不符合最小权限原则。遇到必须给与写入权限的文件夹，则设法防止代码在其中解析。