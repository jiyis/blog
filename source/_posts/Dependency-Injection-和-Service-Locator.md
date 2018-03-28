title: Dependency Injection 和 Service Locator
categories: 设计模式
tags: [IoC控制反转,依赖注入,服务定位器,设计模式]
date: 2016-08-31 15:34:00
---
说起 IoC，其实是 Inversion of Control 的缩写，翻译成中文叫控制反转，不得不说这个名字起得让人丈二和尚摸不着头脑，实际上简而言之它的意思是说对象之间难免会有各种各样的依赖关系，如果我们的代码直接依赖于具体的实现，那么就是一种强耦合，从而降低了系统的灵活性，为了解耦，我们的代码应该依赖接口，至于具体的实现，则通过第三方注入进去，这里的第三方通常就是我们常说的容器。因为在这个过程中，具体实现的控制权从我们的代码转移到了容器，所以称之为控制反转。

Ioc有两种不同的实现方式，分别是：Dependency Injection 和 Service Locator。现在很多 PHP 框架都实现了容器，比如 Phalcon，Yii，Laravel 等。

至于 Dependency Injection 和 Service Locator 的区别，与其说一套云山雾绕的概念，不能给出几个鲜活的例子来得自然。

如果没有容器，那么 Dependency Injection 看起来就像：
```php
class Foo
{
    protected $_bar;
    protected $_baz;
    public function __construct(Bar $bar, Baz $baz) {
        $this->_bar = $bar;
        $this->_baz = $baz;
    }
}
// In our test, using PHPUnit's built-in mock support
$bar = $this->getMock('Bar');
$baz = $this->getMock('Baz');
$testFoo = new Foo($bar, $baz);
```
如果有容器，那么 Dependency Injection 看起来就像：
```php
// In our test, using PHPUnit's built-in mock support
$container = $this->getMock('Container');
$container['bar'] = $this->getMock('Bar');
$container['baz'] = $this->getMock('Baz');
$testFoo = new Foo($container['bar'], $container['baz']);
```
通过引入容器，我们可以把所有的依赖都集中管理，这样有很多好处，比如说我们可以很方便的替换某种依赖的实现方式，从而提升系统的灵活性。
看看下面这个实现怎么样？是不是 Dependency Injection？
```php
class Foo
{
    protected $_bar;
    protected $_baz;
    public function __construct(Container $container) {
        $this->_bar = $container['bar'];
        $this->_baz = $container['baz'];
    }
}
// In our test, using PHPUnit's built-in mock support
$container = $this->getMock('Container');
$container['bar'] = $this->getMock('Bar');
$container['baz'] = $this->getMock('Bar');
$testFoo = new Foo($container);
```
虽然从表面上看它也使用了容器，并不依赖具体的实现，但你如果仔细看就会发现，它依赖了容器本身，实际上这不是 Dependency Injection，而是 Service Locator。

于是乎判断 Dependency Injection 和 Service Locator 区别的关键是在哪使用容器：

1. **如果在非工厂对象的外面使用容器，那么就属于 Dependency Injection**。
1. **如果在非工厂对象的内部使用容器，那么就属于 Service Locator**。