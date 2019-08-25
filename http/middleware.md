# Web and HTTP - Middleware
Framework allows you to set [PSR-15 compatible](https://www.php-fig.org/psr/psr-15/) HTTP middleware globally or to a 
specific route. 

> Check https://github.com/middlewares/psr15-middlewares to find many publicly maintained middleware.

## Create Middleware
Implement `Psr\Http\Server\MiddlewareInterface` to create your middleware:

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

## Global Middleware
To activate middleware for every user request use `Spiral\Bootloader\Http\HttpBootloader`. You can only set this value in 
application bootloaders.

```php
use Spiral\Bootloader\Http\HttpBootloader;

class MyMiddlewareBootloader extends Bootloader
{
    public function boot(HttpBootloader $http)
    {
        // automatically resolved by Container
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

Middleware object will be instantiated on demand. 

> Make sure to add this bootloader before `RoutesBootloader` in your app.

## Route Specific Middleware
To add middleware to the route object use `withMiddleware` method, make sure to use newly created route instance (example 
is given for bootloader):

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

## Combine with IoC scopes
You can combine middleware with IoC scope to create user-specific application context.

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

Use `Spiral\Core\ScopeInterface` to set application scope in your middleware:

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

You can request this context from container or via method injection of your controller:

```php
public function index(UserContext $ctx)
{
    dump($ctx);
}
```