title: Laravel5.3中的一个坑
categories: Laravel
tags: [Laravel5.3]
date: 2016-10-14 09:36:00
---
### 前言
最近准备用Laravel5.3写一个微信的第三方后台系统，然而在做到后台用户登录，从基类控制前传递登录用户信息到视图的时候发现传不过去，研究了下发现是Laravel5.3中把construct和middleware的顺序对调了。在Laravel5.3中会先执行constructor然后在执行middleware。
<!-- more -->
### 在laravel5.2中的例子
`route.php`
```php
Route::group([ 'middleware' => 'auth'], function () {
        Route::get('/', 'HomeController@index')->name('admin.home');
});
```
 `App\Http\Controllers\Controller.php`
 ```php
public function __construct()
{
    var_dump(222);
}
 ```
 `App\Http\Controllers\HomeController.php`
 ```php
public function __construct()
{
    parent::__construct();
}
public function index()
{
    return view('welcome');
}
 ```
`auth middleware`
```php
public function handle($request, Closure $next)
{
    var_dump(111);
    return $next($request);
}
```
以上代码会先输出111  然后输出222。
但是在Laravel5.3中恰恰相反，先输出222  后输出333

### 造成的影响
对于这个改动，以前你在constructor中获取登录用户，然后view share到全局的方法就行不通了。同样session的变量也会获取不到了。

### 官方的文档说明
Session In The Constructor

In previous versions of Laravel, you could access session variables or the authenticated user in your controller's constructor. This was never intended to be an explicit feature of the framework. In Laravel 5.3, you can't access the session or authenticated user in your controller's constructor because the middleware has not run yet.

As an alternative, you may define a Closure based middleware directly in your controller's constructor. Before using this feature, make sure that your application is running Laravel 5.3.4 or above:

```php
<?php

namespace App\Http\Controllers;

use App\User;
use Illuminate\Support\Facades\Auth;
use App\Http\Controllers\Controller;

class ProjectController extends Controller
{
    /**
     * All of the current user's projects.
     */
    protected $projects;

    /**
     * Create a new controller instance.
     *
     * @return void
     */
    public function __construct()
    {
        $this->middleware(function ($request, $next) {
            $this->projects = Auth::user()->projects;

            return $next($request);
        });
    }
}
```
> 其实就是在构造函数中，加入下面代码即可获取:
```php
$this->middleware(function ($request, $next) {
    $this->projects = Auth::user()->projects;

    return $next($request);
});
```
