# Web and HTTP - Middleware
Framework allows you to set [PSR-15 compatible](https://www.php-fig.org/psr/psr-15/) HTTP middleware globally or to a 
specific route. 

> Check https://github.com/middlewares/psr15-middlewares to find many publicly maintained middleware.

## Create Middleware
Implement `Psr\Http\Server\MiddlewareInterface` to create your middleware:

```php
class MyMiddleware implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        return $handler->handle($request)->withAddedHeader('My-Header', 'my-value');
    }
}
```

## Global Middleware
To activate middleware for every user request use `Spiral\Bootloader\Http\HttpBootloader`. You can also set this value 
during application bootstrap process and not during application runtime.

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