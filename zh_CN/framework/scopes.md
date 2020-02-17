# 框架 - IoC 作用域

开发长期存在的应用程序的一个重要方面是恰当的上下文管理。在守护程序里，不允许将用户请求视为全局单例对象，或者对用户请求实例的引用存储在服务中。

实际上这意味着处理用户输入时必须显示请求上下文。Spiral 框架用全局 IoC 容器作为上下文载体，简化了此类操作，可以通过限制作用域的上下文，把特定的实例作为全局对象访问。

## 原理解释

处理用户请求时，上下文相关的数据位于 IoC 作用域下，并且只在有限时间内对容器可见。这样的操作通过 `Spiral\Core\ScopeInterface` 接口定义的 `runScope` 方法实现。Spiral 的默认容器 `Spiral\Core\Container` 实现了这个接口。

```php
$container->runScope(
    [
        UserContext::class => $user
    ],
    function () use($container) {
        dump($container->get(UserContext::class);
    }
);
```

> 即使执行过程中发生了任何异常，Spiral 也能保证执行后的作用域是干净的。

你可以直接从容器中获取在作用域内设置的对象，也可以IoC作用域下的服务或控制器代码中借助方法注入来调用。

```php
public function doSomething(UserContext $user)
{
    dump($user);
}
```

简而言之，你能像平常对任何常规依赖项所做的那样，从 IoC 作用域内的容器或注入对象中获取当前请求的上下文，但不要在不同作用域之间（对于 WEB 应用而言，也即不同请求之间）存储它们。

## 上下文管理器

如上所述，你不能保存作用域实例的任何引用。因此下面的代码是无效的并且会导致控制器锁定第一个作用域（请求）的值：

```php
class HomeController implements SingletonInterface
{
    private $userContext; // 不允许！
    
    public function __construct(UserContext $userContext)
    {
        $this->userContext = $userContext;
    }
}
``` 

建议使用专门设计的单例对象来访问 IoC 作用域内的上下文，即上下文管理器（context manager）。下面的代码展示了一个简单的上下文管理器：

```php
class UserScope 
{
    private $container;
    
    public function __construct(ContainerInterface $container)
    {
        $this->container = $container;
    }
    
    public function getUserContext(): ?UserContext 
    {
        // 错误检查将被忽略
        return $this->container->get(UserContext::class);
    }

    public function getName(): string
    {
        return $this->getUserContext()->getName();
    }
}
```

这样的上下文管理器即可在服务/控制器中自由使用，包括单例模式的对象：

```php
class HomeController implements SingletonInterface
{
    private $userScope; // allowed
    
    public function __construct(UserScope $userManager)
    {
        $this->userScope = $userManager;
    }
}
```

> `Spiral\Http\Request\InputManager` 是一个很好的示例。这个上下文管理器用来访问只在 HTTP 请求作用域下可用的 `Psr\Http\Message\ServerRequestInterface`.
