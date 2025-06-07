# 容器 — IoC 作用域

构建长期运行的应用程序需要适当管理上下文。
在后台运行的应用程序中，您不能再将用户请求作为存储在各服务中的全局单例对象。

这意味着您需要在处理用户输入时显式请求上下文。
Spiral 通过 IoC（控制反转）容器作用域提供了一种优雅的方式来管理这一点。

作用域允许您创建隔离的上下文，在其中重新定义服务并管理它们的生命周期。

## 创建隔离作用域

要创建隔离上下文，请使用 `$container->runScope()` 方法。
第一个参数是包含作用域选项的 `Scope` 对象，第二个参数是将在此作用域内运行的函数。
此函数的结果由 `runScope()` 返回。

```php
$result = $container->runScope(
    new Scope(bindings: [
        LoggerInterface::class => FileLogger::class,
    ]),
    function () {
        // 在这里编写您的代码
    },
);
```

在这个例子中，`LoggerInterface` 将在作用域内被解析为 `FileLogger`。

## 工作原理

当您调用 `$container->runScope(new Scope(...), fn() => ...)`，会创建一个具有自己绑定的新容器。现有容器成为这个新容器的父容器。

新容器将在提供的函数内部使用，并在函数完成后销毁。

重要点：
- **可见性**：父容器不知道其子容器。但是，父容器中的服务在子容器中是可访问的。
- **作用域命名**：
  - 主全局作用域始终为 `root`。
  - 层级结构中的命名作用域必须具有唯一的名称，以避免冲突。  
    ![scopes-conflict](https://gist.github.com/user-attachments/assets/32f1ae89-9e35-4e7a-9e53-b3db15fee0ea)
  - 具有相同名称的并行作用域（如在协程中）可以存在，并将拥有各自的层级结构。
- 退出作用域时，相关联的容器会被销毁。

### 依赖解析顺序

在隔离作用域内解析依赖项时：
1. 容器首先尝试在当前作用域中查找绑定。
2. 如果未找到绑定，容器尝试在父作用域中查找，以此类推直到根容器。
3. 实例在找到绑定的作用域中创建。这意味着该实例的依赖项在同一作用域内解析。

## 预定义作用域

Spiral 提供了几个预定义的作用域：

![spiral-scopes](https://gist.github.com/user-attachments/assets/aa12be0a-bea1-439c-a676-ef8d6158bda9)

1. `root` — 主全局作用域。所有其他作用域都是它的子作用域。
2. **调度器作用域** — 当相应的[调度器](../framework/dispatcher.md)启动时打开的作用域：
   `http`、`console`、`grpc`、`centrifugo`、`tcp`、`queue` 或 `temporal`。
3. **请求作用域** — 在执行控制器之前打开的作用域，当请求对象完全形成并准备好处理时。
   对于 HTTP 调度器，中间件将在 `http` 作用域中执行，而拦截器在 `http-request` 作用域中执行。

如果您确信服务只在特定调度器内工作，使用相应的作用域是有意义的。
例如，HTTP 中间件应该绑定在 `http` 作用域级别。

您可以创建自己的作用域来隔离上下文并仅使特定服务可用。

![http-scopes](https://gist.github.com/user-attachments/assets/a402f166-4396-40ec-a376-d2136fb25824)

## 配置命名作用域的绑定

您可以使用 `BinderInterface::getBinder()` 方法预先配置特定于命名作用域的绑定。
这允许您为作用域设置默认绑定。

```php
$container->bindSingleton(Interface::class, Implementation::class);

// 配置 'request' 作用域的默认绑定
$binder = $container->getBinder('request');
$binder->bindSingleton(Interface::class, Implementation::class);
$binder->bind(Interface::class, factory(...));
```

> **注意**
> 作用域中的绑定不会影响该作用域的现有容器（除了 `root`）。

## 覆盖默认绑定

使用 `Container::runScope()` 时，您可以传递绑定以覆盖特定作用域的默认值。

```php
$container->bindSingleton(SomeInterface::class, SomeImplementation::class);

$container->runScope(
    new Scope(
        name: 'request',
        bindings: [SomeInterface::class => AnotherImplementation::class],
    ),
    function () {
        // 在这里编写您的代码
    }
);
```

在这个例子中，即使 `request` 作用域有 `SomeInterface` 的默认绑定，这个特定的运行也使用 `AnotherImplementation`。

## 作用域限制

您可以使用 `#[Scope('name')]` 属性限制依赖项可以在哪里解析。

```php
use Spiral\Boot\Environment\DebugMode;
use Spiral\Core\Attribute\Scope;
use Spiral\Core\Attribute\Singleton;

#[Singleton]
#[Scope('http')]
final readonly class DebugMiddleware implements \Psr\Http\Server\MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // ...
    }
}
```

在这个例子中，`DebugMiddleware` 只能在作用域层级结构中存在 `http` 作用域时实例化。
否则，将抛出异常。

## 销毁作用域和终结

退出作用域时，相关联的容器会被销毁。
这意味着在作用域内创建的单例应该被垃圾回收，所以避免循环引用。

如果您需要在作用域内解析依赖项时执行清理操作，使用 `#[Finalize('methodName')]` 属性指定一个在作用域销毁时将被调用的方法。

```php
#[Finalize('destroy')]
class MyService
{
    /**
     * 如果服务在此作用域中被解析，则在作用域销毁前将调用此方法。
     * 参数将使用容器解析。
     */
    public function destroy(LoggerInterface $logger): void
    {
        // 执行清理...
    }
}
```

## 代理对象

作用域像嵌套容器，但它们不仅仅是简单的委托。

如果您想在父作用域（`root` 或 `http`）中创建一个无状态服务，
该服务将处理 `http-request` 作用域中的 `ServerRequestInterface` 对象，该怎么办？
使用嵌套容器，这是不可能的，因为 `ServerRequestInterface` 仅在 `http-request` 作用域内可用。
而且，每个请求的 `ServerRequestInterface` 都不相同。

Spiral 提供代理对象，这些对象会延迟依赖项解析，直到实际需要。

使用 `#[Proxy]` 属性为接口创建代理：

```php
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Core\Attribute\Proxy;
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final readonly class DebugService
{
    public function __construct(
        #[Proxy] private ServerRequestInterface $request,
    ) {}

    public function hasDebugInfo(): bool
    {
        return $this->request->hasHeader('X-Debug');
    }
}
```

重要点：
- 代理仅为接口配置。
- 对代理的每个方法调用都将从容器中解析真实对象。
- 调用接口中未定义的方法是不允许的。

您可以使用 `Binder` 类为必须仅在特定作用域中可用的服务配置代理。
例如，如果 `AuthInterface` 服务必须仅在 `http` 作用域中可用，您可以为 `root` 作用域使用代理对象：

```php
// 在 `root` 作用域中为 `AuthInterface` 配置代理
$rootBinder = $container->getBinder('root');
$rootBinder->bindSingleton(new \Spiral\Core\Config\Proxy(
    AuthInterface::class,
    singleton: true,
    fallbackFactory: static fn() => throw new \LogicException(
        '无法在 `http` 作用域外接收 AuthInterface 实例。'
    ),
));

// 在 `http` 作用域中绑定 `AuthInterface`
$container->getBinder('http')
    ->bindSingleton(AuthInterface::class, Auth::class);
```

如果在 `http` 作用域外使用代理，则将调用 `fallbackFactory` 来解析依赖项。
如果未提供 `fallbackFactory`，则将抛出 `RecursiveProxyException`。