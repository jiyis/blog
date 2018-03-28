title: 分享一个 Laravel5.2 写的简单的报名系统主页
categories: Laravel
tags: [Laravel]
date: 2016-08-30 09:59:00
---


### 介绍说明
 接触Laravel快半年了,前段时间开始用laravel做点小项目练手.正好母校老师说要成立一个敬文书院,需要做个简单的报名系统.索性就分享出来,顺便聊聊自己的一些见解.希望各位大虾多多支持.
 
### 主要功能和扩展
*  采用laravel自带auth认证，实现前后台的用户登录
*  采用Laravel-Excel实现后台学生信息的批量导入和导出
*  采用BootstrapTable实现报名学生和用户的增删改查和排序
*  采用Entrust实现后台用户的RBAC的权限管理
*  采用laravel generator实现后台功能的快速生成
*  采用l5-repository 实现一些模式和思想
*  一些其他的前端插件，Toastr,dateimepicker,sweetalert,fancybox.......
<!-- more -->
### 开发过程中用到的一些技巧

说真的感觉自己作为一名RD有点懒,说的懒是只不想去写前端的东西.其实我们经常会遇到这样的情况,项目经理说今天我们要上一个项目, 然后我们不得不从后台登录，权限验证，注册，字段验证，界面效果等等开始做起。虽然这中间会有很多工资自己开发一套适合自己的通用后台，但是后续的扩展性和便捷性问题依旧解决不掉。我曾经尝试过用TP，CI和YII去开发一个通用的后台，集成一些基本的功能。但是最后都失败了，与其说失败，倒不如说是败给了后续的可扩展性。因为最后你上项目还是无法摆脱前端和UI的困扰，扩展上也不方便。

后来接触到了laravel和phphub社区，感觉思路一下子清晰出来。这期间前端方便借鉴了summer的phphub源码的不少经验，RBAC方便借鉴了Ryan兄的不少经验。于是乎有了现在的一个粗糙的模型。数据库可以利用json文件格式把字段都设计好，然后利用generator几行命令就生成一个后台的CURD，Form的UI上是定制过的，用了select2，dropzone，datetimepicker，toastr等等插件.同时后台有基本的RBAC权限和menu的定制。前后台的登录也已经集成。可以大大节省后台模块上花费的时间，把中心和精力转移到前端业务上去。
![](http://register.yearn.cc/demopic/index.png)
![](http://register.yearn.cc/demopic/admin.png)
![](http://register.yearn.cc/demopic/excel.png)
![](http://register.yearn.cc/demopic/import.png)
![](http://register.yearn.cc/demopic/register.png)

## 注意事项

所有的前端资源和插件都是通过bower安装的，目录在composer安装包的vendor下面。所以需要环境安装node 、npm、gulp等基本插件。

## 安装步骤
* sudo npm install -g gulp      #全局安装npm
* sudo npm install -g bower     #全局安装bower
* cd register
* sudo npm install gulp   #在项目本地安装gulp
* sudo npm install bower  #在项目本地安装bower
* sudp npm install        #安装项目Node依赖、Laravel Elixir

###### 在项目根目录新增文件.bowerrc

```json
{
    "directory": "vendor/bower_dl"
}
```
* 执行命令 bower init创建文件bower.json


###### 如果是linux 后面加上 --allow-root  windows不需要

*  sudo bower install admin-lte --allow-root
*  sudo bower install fontawesome --allow-root
*  sudo bower install ionicons --allow-root
*  sudo bower install https://github.com/smalot/bootstrap-datetimepicker.git --allow-root
*  sudo bower install DataTables --allow-root
*  sudo bower install sweetalert --allow-root
*  sudo bower install dropzone --allow-root
*  sudo bower install toastr --allow-root
*  sudo bower install fileupload --allow-root
*  sudo bower install fancybox --allow-root
*  npm install --save-dev gulp-import-css
*  npm install gulp-cssimport
*  npm install gulp-rename
*  gulp copyfiles
*  gulp
*  php artisan migrate
*  php artisan key:generate
*  entrust缓存不支持file和database,所以需要在.env 中把CACHE_DRIVER=file 改为 CACHE_DRIVER=array
* 火狐中datetmepick的插件会报错，但是其他的插件没有时间选项，解决方法把js文件中this.defaultTimeZone=(new Date()).toString().split("(")[1].slice(0,-1);改为this.defaultTimeZone='GMT '+(new Date()).getTimezoneOffset()/60


## 项目地址
oschina http://git.oschina.net/GarySource/register   
github  https://github.com/jiyis/register
generator   http://labs.infyom.com/laravelgenerator/
