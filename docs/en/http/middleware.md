# HTTP - Middleware

The framework allows you to set [PSR-15 compatible](https://www.php-fig.org/psr/psr-15/) HTTP middleware globally or to
a specific route.

> **Note**
> Check https://github.com/middlewares/psr15-middlewares to find many publicly maintained middlewares.

## Create Middleware

Implement `Psr\Http\Server\MiddlewareInterface` to create your middleware:

```php
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

## Global Middleware

To activate a middleware for every user request, use `Spiral\Bootloader\Http\HttpBootloader`. You can only set this value
in application bootloaders.

```php
use Spiral\Bootloader\Http\HttpBootloader;
use Spiral\Core\Container\Autowire;
use Psr\Container\ContainerInterface;

class MyMiddlewareBootloader extends Bootloader
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

Or you can configure middlewares in the config file `app/config/http.php`:

```php
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

> **Note**
> Make sure to add this Bootloader before `RoutesBootloader` in your app.

## Route Specific Middleware

To add a middleware to the route object, use `withMiddleware` method. Make sure to use a newly created route instance, here's an example for Bootloader:

```php
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

## Combine with IoC scopes

You can combine middleware with the IoC scope to create a request-specific application context.

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

Use `Spiral\Core\ScopeInterface` to set application scope in your middleware:

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Core\ScopeInterface;

class MyMiddleware implements MiddlewareInterface
{
    private ScopeInterface $scope;

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

You can request this context from the container or via method injection of your controller:

```php
public function index(UserContext $ctx): void
{
    dump($ctx);
}
```

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

## Events

| Event                                  | Description                                              |
|----------------------------------------|----------------------------------------------------------|
| Spiral\Http\Event\MiddlewareProcessing | The Event will be fired `before` calling the middleware. |
