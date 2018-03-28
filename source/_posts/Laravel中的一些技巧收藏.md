title: Laravel中的一些技巧收藏
categories: Laravel
tags: [Laravel,依赖注入]
date: 2016-08-31 15:23:00
---
### 禁用Controller中某个方法的CSRF
有时候需要在某个控制器的某个方法中禁用csrf_token，这时候可以通过befireFilter来实现。同时在也可借用befireFilter实现预处理功能。
```php
$this->beforeFilter('auth', ['except' => 'login']);
$this->beforeFilter('csrf', ['on' => 'post']);
```
<!-- more -->
### 依赖注入的时候传递参数
熟悉Laravel人都知道Laravel的Service Provider，但是如果要注入的类需要初始化参数呢？这个时候可以通过ServiceProvider中的register来绑定实现。
```php
public function register()
{
    $this->app->bind('Bloom\Security\ChannelAuthInterface', function()
    {
        $request = $this->app->make(Request::class);
        $guard   = $this->app->make(Guard::class);

        return new ChannelAuth($request->input('channel_name'), $guard->user());
    });
}
```