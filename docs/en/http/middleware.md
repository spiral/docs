# HTTP — Middleware

Spiral uses [PSR-15 compatible](https://www.php-fig.org/psr/psr-15/) HTTP middleware.

Middleware is responsible for handling functionality that is related to the request and response, such as
authentication, caching, or logging. It can modify the request and response before they are passed on to the router, but
it cannot make decisions about which routes should be handled by the application. This is the responsibility of the
router, and middleware should not attempt to bypass or override the router's decisions.

[Interceptors](./interceptors.md) are well suited to handle functionality that is related to the application router.
They are executed after the request has been passed on to the application and have more access to the application's
internal state, including the router.

## Create Middleware

The `Psr\Http\Server\MiddlewareInterface` is a standard interface provided by PSR-15 for creating middleware in PHP. To
create your own middleware, you need to implement this interface and define the methods it requires.

```php
use Psr\Http\Server\MiddlewareInterface;

class MyMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        return $handler->handle($request)->withAddedHeader('My-Header', 'my-value');
    }
}
```

> **Note**
> Check https://github.com/middlewares/psr15-middlewares to find many publicly maintained middlewares.


Spiral provides several ways to set middleware, allowing developers to choose the approach that best fits their needs.

### Global Middleware

These middlewares are applied to all routes and requests. They are typically used for functionality that should be
applied to all requests, such as authentication or logging.

You can activate a global middleware in the `RoutesBootloader`:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use App\Endpoint\Web\Middleware\LocaleSelector;
use Spiral\Auth\Middleware\AuthTransportMiddleware;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Core\Container\Autowire;
use Spiral\Csrf\Middleware\CsrfMiddleware;
use Spiral\Debug\StateCollector\HttpCollector;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Http\Middleware\JsonPayloadMiddleware;
use Spiral\Session\Middleware\SessionMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            LocaleSelector::class,
            ErrorHandlerMiddleware::class,
            JsonPayloadMiddleware::class,
            HttpCollector::class,
            MyMiddleware::class,
        ];
    }
    
    // ...
}
```

Or you can activate a global middleware for every user request, use `Spiral\Bootloader\Http\HttpBootloader`. You can
only set this value in application bootloaders.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\HttpBootloader;
use Spiral\Core\Container\Autowire;
use Psr\Container\ContainerInterface;

class AppBootloader extends Bootloader
{
    public function boot(HttpBootloader $http, ContainerInterface $container): void
    {
        // automatically resolved by Container
        $http->addMiddleware(MyMiddleware::class);
        
        // automatically resolved by Container
        $container->bind('my:middleware', fn() => new MyMiddleware);
        $http->addMiddleware('my:middleware');
        
        // Autowire allows creating an object with dependency resolving from the container
        // and passing some parameters manually
        $http->addMiddleware(new Autowire(MyMiddleware::class, ['someParameter' => 'value']));
    }
}
```

Middleware object will be instantiated on demand.

Or you can configure middleware in the config file `app/config/http.php`:

```php app/config/http.php
return [
    // ...
    'middleware' => [
        // via fully qualified class name
        MyMiddleware::class,
        
        'my:middleware',
        
        // via Autowire 
        new Autowire(MyMiddleware::class, ['someParameter' => 'value']),
        
        // or manual instantiating object
        new MyMiddleware(),
    ],
];
```

## Middleware groups

Middleware that's grouped will only be applied to routes within the corresponding group. These groups are registered in
the app's container as pipelines with the name `middleware:{group}`, so you can use them on any routes.

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use App\Middleware\LocaleSelector;
use Spiral\Auth\Middleware\AuthTransportMiddleware;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Core\Container\Autowire;
use Spiral\Csrf\Middleware\CsrfMiddleware;
use Spiral\Debug\StateCollector\HttpCollector;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Http\Middleware\JsonPayloadMiddleware;
use Spiral\Session\Middleware\SessionMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                CookiesMiddleware::class,
                SessionMiddleware::class,
                CsrfMiddleware::class,
                MyMiddleware::class,
                // new Autowire(AuthTransportMiddleware::class, ['transportName' => 'cookie'])
            ],
            'api' => [
                // new Autowire(AuthTransportMiddleware::class, ['transportName' => 'header'])
            ],
        ];
    }
    
    // ...
}
```

## Route Specific Middleware

These middlewares are applied to a specific route. This allows developers to apply middleware to a single route, such as
a specific API endpoint.

:::: tabs

::: tab Routing configurator

To add a middleware to the route object, use `middleware` method:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...
 
    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->add(name: 'news.show', pattern: '/news/<id:int>')
            ->middleware(['middleware:web', MyMiddleware::class]);
        ...
    }
}
```

:::

::: tab Router

To add a middleware to the route object, use `withMiddleware` method. Make sure to use a newly created route instance,
here's an example:

```php app/src/Application/Bootloader/AppBootloader.php
use App\Controller\HomeController;
use App\MyMiddleware;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;

// ...

public function boot(RouterInterface $router): void
{
    $route = new Route('/index', new Action(HomeController::class, 'index'));
    $route = $route->withMiddleware(MyMiddleware::class);

    $router->addRoute('index', $route);
}
```

:::

::::

## Combine with IoC scopes

Spiral Framework allows developers to combine middleware with the [IoC scope](../framework/scopes.md) to create a 
request-specific application context.

This allows developers to set up a context for the current request, which can be accessed by other parts of the
application. This can be useful for tasks such as logging or data access, where the context of the request is important.

```php
class UserContext
{
    public int $id;
    public string $name;

    public function __construct(int $id, string $name)
    {
        $this->id = $id;
        $this->name = $name;
    }
}
```

By using the `Spiral\Core\ScopeInterface` in your middleware, you can set an application scope that is specific
to the current request.

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Core\ScopeInterface;

class MyMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ScopeInterface $scope
    ) {
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

Once the request-specific context has been set up in your middleware, you can then request it from the container or via
method injection in your controllers.

```php
public function index(UserContext $ctx): void
{
    dump($ctx);
}
```

> **Note**
> It's also important to note that the scope set up in the middleware is only valid for the duration of the request, 
> and it will not affect other requests. This allows you to maintain the isolation and integrity of the context for 
> each request.

## Non-Direct Scope Configuration

You can use already existing requests scope to carry user values. Create a bootloader providing access method for the
context specific value:

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

To gain access to this value from container:

```php
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Exception\ScopeException;

class UserContextBootloader extends Bootloader 
{
    protected const BINDINGS = [
        UserContext::class => [self::class, 'userContext']
    ];
    
    private function userContext(ServerRequestInterface $request): UserContext
    {
        $userContext = $request->getAttribute('userContext', null);
        if ($userContext === null) {
            throw new ScopeException('Unable to resolve UserContext, invalid request scope');
        }
        
        return $userContext;
    }
}
```

## Available Middlewares

HTTP extension includes multiple middlewares you might want to activate in your project:

| Bootloader                                    | Middleware                                                            |
|-----------------------------------------------|-----------------------------------------------------------------------|
| Spiral\Bootloader\Http\ErrorHandlerBootloader | Hide exceptions in non debug mode and render HTTP error pages.        |
| Spiral\Bootloader\Http\JsonPayloadsBootloader | Parse body of `application/json` requests.                            |
| Spiral\Bootloader\Http\PaginationBootloader   | Use request query parameters to automatically configure paginator(s). |
| Spiral\Bootloader\Http\DiactorosBootloader    | Use Zend/Diactoros as PSR-7 implementation (legacy).                  |

## Events

| Event                                  | Description                                              |
|----------------------------------------|----------------------------------------------------------|
| Spiral\Http\Event\MiddlewareProcessing | The Event will be fired `before` calling the middleware. |

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
