# HTTP - 中间件

Spiral 框架允许用户在全局范围或路由级别使用[兼容 PSR-15 规范](https://www.php-fig.org/psr/psr-15/) 的 HTTP 中间件。

> 在 https://github.com/middlewares/psr15-middlewares 可以找到很多别人维护的开源的 PSR-15 中间件。

## 创建中间件

要创建中间件，需要实现 `Psr\Http\Server\MiddlewareInterface` 接口：

```php
class MyMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface
    {
        return $handler->handle($request)->withAddedHeader('My-Header', 'my-value');
    }
}
```

## 全局中间件

如果要为所有的请求启用上面的中间件，可以使用 `Spiral\Bootloader\Http\HttpBootloader` 引导程序。请注意，只能在应用的引导程序使用它。

```php
use Spiral\Bootloader\Http\HttpBootloader;

class MyMiddlewareBootloader extends Bootloader
{
    public function boot(HttpBootloader $http)
    {
        // 容器会自动解析中间件
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

中间件对象会按需实例化。

> 特别提醒：在顺序上，启用全局中间件的引导程序必须放在 `RoutesBootloader` 之前。

## 路由级别中间件

如果只想给特定的路由对象添加中间件，可以在定义路由时使用 `withMiddleware` 方法。之前介绍过，路由是不可变对象，因此务必用新创建的路由实例来添加中间件（通常是在路由引导程序中）：

```php
use App\Controller\HomeController;
use App\MyMiddleware;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;

// ...

public function boot(RouterInterface $router)
{
    $route = new Route('/index', new Action(HomeController::class, 'index'));
    $route = $route->withMiddleware(MyMiddleware::class);

    $router->addRoute('index', $route);
}
```

## 与 IoC 作用域结合

```php
class UserContext
{
    public $id;
    public $name;

    public function __construct(int $id, string $name)
    {
        $this->id = $id;
        $this->name = $name;
    }
}
```

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Core\ScopeInterface;

class MyMiddleware implements MiddlewareInterface
{
    private $scope;

    public function __construct(ScopeInterface $scope)
    {
        $this->scope = $scope;
    }

    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        return $this->scope->runScope([
            UserContext::class => new UserContext(123, 'test')
        ], function () use ($handler, $request) {
            return $handler->handle($request);
        });
    }
}
```

上面的实例代码中，增加了一个用户上下文（UserContext），你可以在容器中请求这个上下文，也可以在控制器中借助方法注入来请求它：

```php
public function index(UserContext $ctx)
{
    dump($ctx);
}
```

## 非直接范围配置

你可以使用已有的请求上下文来携带上面实例中展示的用户上下文。只要创建一个引导程序，为访问该上下文提供访问方法：

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class MyMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        return $handler->handle($request->withAttribute('userContext', new UserContext(123, 'test')));
    }
}
```

下面的代码展示了如何在容器中访问这个值：

```php
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Exception\ScopeException;

class UserContextBootloader extends Bootloader 
{
    protected const BINDINGS = [
        UserContext::class => [self::class, 'userContext']
    ];
    
    /**
     * @param ServerRequestInterface $request
     * @return UserContext
     */
    private function userContext(ServerRequestInterface $request): UserContext
    {
        $userContext = $request->getAttribute('userContext', null);
        if ($userContext === null) {
            throw new ScopeException('无法解析 UserContext，无效的请求范围');
        }
        
        return $userContext;
    }
}
```
