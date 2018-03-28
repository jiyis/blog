title: Laravel5.3中的多字段登录
categories: Laravel
tags: [Laravel多字段登录]
date: 2016-10-14 15:24:00
---
### 前言
现在许多网站都是兼容邮箱、手机号或者用户名来登录的，而Larave中默认是只支持email登录的，我们知道可以通过设置```protected $username = 'name';```来更改为用户名登录。但是如果想兼容多个字段该怎么设置呢？百度一下的话你会发现许多方式，其实有部分是修改源码的方式，有部分是重构的方式。个人还是比较喜欢重构这种方式的，毕竟能保留框架自带的许多功能，但是这些都是针对于Laravel5.和5.2版本的。在Laravel5.3中并不适用，看了下Laravel5.3的改动，最终通过重构trait的方法实现了。
<!-- more -->
> ### Laravel5.3重构方式实现多字段登录
修改`/app/Http/Controllers/Auth/LoginController.php`
```php
//use AuthenticatesUsers;
use AuthenticatesUsers {
	login as primaryLogin;
}
/**
     * Where to redirect users after login.
     *
     * @var string
     */
    protected $redirectTo = '/admin/home';
    protected $username = 'email';

    /**
     * Create a new controller instance.
     *
     * @return void
     */
    public function __construct()
    {
        $this->middleware('guest:admin', ['except' => 'logout']);
    }
/**
     * 重写登录方法，兼容email和username登录方式
     * @param Request $request
     * @return \Illuminate\Http\Response
     */
    public function login(Request $request)
    {
        $field = filter_var($request->input('username'), FILTER_VALIDATE_EMAIL) ? 'email' : 'username';
        $request->merge([$field => $request->input('username')]);

        $this->username = $field;

        return $this->primaryLogin($request);
    }

    /**
     * 判断登录字段问题,重构trait的username方法
     * @return string
     */
    public function username()
    {
        return $this->username;
    }
	protected function guard()
    {
        return Auth::guard('admin');
    }
```
主要实录就是重构username方法，在henticatesUsers中，username方法被写死了，而更改系统文件是不建议的。所以为了尽可能保留laravel自带的特性，最终验证还是交给laravel，这边做一层过滤和校验。

> ### Laravel5.3 之前的方法
1. **方法一** 
修改 `/app/Http/Controllers/Auth/AuthController.php` 文件，重写方法.
```php
namespace App\Http\Controllers\Auth;
......
use Illuminate\Http\Request; // 增加该行
 
class AuthController extends Controller
{
    protected $username = 'login';
 
    ....
 
    protected function getCredentials(Request $request)
    {
        $login = $request->get('login');
        $field = filter_var($login, FILTER_VALIDATE_EMAIL) ? 'email' : 'name';
 
        return [
            $field => $login,
            'password' => $request->get('password'),
        ];
    }
}
```
2.  ** 方法二**
修改 `/app/Http/Controllers/Auth/AuthController.php` 文件，这也是使用 Laravel 自带认证系统的一种方法。
```php
namespace App\Http\Controllers\Auth;
......
use Illuminate\Http\Request; // 增加该行
 
class AuthController extends Controller
{
 
    // 修改这里
    use AuthenticatesAndRegistersUsers, ThrottlesLogins {
        AuthenticatesAndRegistersUsers::postLogin as laravelPostLogin;
    }
 
    ......
 
    // 增加方法
    public function postLogin(Request $request)
    {
        $field = filter_var($request->input('login'), FILTER_VALIDATE_EMAIL) ? 'email' : 'name';
        $request->merge([$field => $request->input('login')]);
        $this->username = $field;
 
        return self::laravelPostLogin($request);
    }
}
```
