# HTTP - 定制 PSR-15 处理器

Spiral 框架遵循 `PSR-7`, `PSR-15` 和 `PSR-17` 社区标准；它允许你把 HTTP 层的实现换成任何符合标准的替代方案。

> 默认情况下，`Psr\Http\Server\RequestHandlerInterface` 由 `Spiral\Router\Router` 实现，并默认绑定到该实现。如果你需要使用其它实现方案，必须在应用程序中禁用 `Spiral\Bootloader\Http\RouterBootloader` 引导程序。

## 示例：Fast Route

作为示例，我们可以用一个基于 [FastRoute](https://github.com/nikic/FastRoute) 实现的路由组件来代替默认的 Spiral 路由组件。该实现由 [https://github.com/middlewares/fast-route](https://github.com/middlewares/fast-route) 提供。

首先，安装需要的两个依赖：

```bash
$ composer require middlewares/fast-route middlewares/request-handler
```

创建引导程序来把 Fast Route 绑定到我们的 http 服务器。要做到这一点，可以通过 `HttpBootloader` 来绑定，也可以简单地在 `SINGLETONS` 中声明 `Psr\Http\Server\RequestHandlerInterface`:

```php
use FastRoute;
use Middlewares;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Boot\Bootloader\Bootloader;

class FastRouteBootloader extends Bootloader
{
    const SINGLETONS = [
        RequestHandlerInterface::class => [self::class, 'psr15Handler']
    ];

    private function createRoutes(FastRoute\RouteCollector $r)
    {
        $r->addRoute('GET', '/hello/{name}', function ($request) {
            $name = $request->getAttribute('name');

            return sprintf('Hello %s', $name);
        });
    }

    private function psr15Handler(ResponseFactoryInterface $responseFactory)
    {
        $dispatcher = FastRoute\simpleDispatcher(function (FastRoute\RouteCollector $r) {
            $this->createRoutes($r);
        });

        return new Middlewares\Utils\Dispatcher([
            new Middlewares\FastRoute($dispatcher, $responseFactory),
            new Middlewares\RequestHandler(),
        ]);
    }
}
```

> 请确保 `Spiral\Bootloader\Http\HttpBootloader` 已经启用。

把上面创建的引导程序添加到你的应用，路由 `/name/{name}` 立即生效了。

## 自定义处理器

当然你也可以实现自己的处理器，为了节省时间，我们直接在引导程序中实现 PSR-15 处理器：

```php
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Boot\Bootloader\Bootloader;

class HttpHandlerBootloader extends Bootloader implements RequestHandlerInterface
{
    const SINGLETONS = [
        RequestHandlerInterface::class => self::class
    ];

    private $responseFactory;

    public function __construct(ResponseFactoryInterface $responseFactory)
    {
        $this->responseFactory = $responseFactory;
    }

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        $response = $this->responseFactory->createResponse(200);
        $response->getBody()->write("hello world");

        return $response;
    }
}
```

> 网上有大量实现了此接口的开源路由方案可用。
