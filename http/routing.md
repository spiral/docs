# HTTP - Routing
The framework [Web bundle](https://github.com/spiral/app) include pre-configured router components. To install router in
alternative builds:

```bash
$ composer require spiral/router
``` 

And active its bootloader:

```php
[
    //...
    Spiral\Bootloader\Http\RouterBootloader::class,
]
```

To enable MVC/HMVC routing (to controllers and actions) install `spiral/hmvc` package.

## Configuration
The component does not require any external configuration. You can create new routing via `Spiral\Router\RouterInterface` 
in your bootloader. We can start with simple `/` handler:

```
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute('home', new Route(
            '/',
            function () {
                return 'hello world';
            }
        ));
    }
}
```

> The Route class can accept handler of type `Psr\Http\Server\RequestHandlerInterface`, closure, invokable class 
> or `Spiral\Router\TargetInterface`. You can pass class or binding name instead of real object so it can be constructed
> on demand.

## Closure Handler
It is possible to pass the `closure` as route handler, in this case our function will receive two
arguments: `Psr\Http\Message\ServerRequestInterface` and `Psr\Http\Message\ResponseInterface`.

```php
router->setRoute('home', new Route(
    '/<name>',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        $response->getBody()->write("hello world");

        return $response;
    }
));
```

## Route Pattern and Parameters
You can use route pattern to specify any number of required and optional parameters, this parameters will be later passed 
to our route handler via `ServerRequestInterface` attribute `route`.

To define route parameter use the form `<parameter_name:pattern>`, where pattern is regexp friendly expression. You can 
omit patter and use just `<parameter_name>`, in this case the parameter will match to `[^\/]+`. 

We can add simple parameter `name`:

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute('home', new Route(
            '/<name>',
            function (ServerRequestInterface $request, ResponseInterface $response) {
                return $request->getAttribute('route')->getMatches(); // returns JSON ['name' => '']
            }
        ));
    }
}
```

To specify the part of route (including the parameters) as optional use `[]`, for example:

```php
$router->setRoute('home', new Route(
    '/[<name>]',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> This route will match `/`, the name parameter will be equal to `null`.

You can specify any number of parameters and some of them optional, for example we can match the URLs like `/group/user`, 
where `user` parameter is optional:

 
```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

You can specify the default parameter value using third route argument:

```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    },
    [
        'user' => 'default'
    ]
));
```

To specify parameter pattern use `<parameter:pattern>` construction:

```php
$router->setRoute('home', new Route(
    '/user/<id:\d+>',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> This route will only match the urls where `id` is numeric.

You can also specify multiple pre-defined options:

```php
$router->setRoute('home', new Route(
    '/do/<action:login|logout>',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
``` 

> This route will only match the `/do/login` and `/do/logout` urls.

#### Match Host
To match the domain or sub-domain name prefix your pattern with `//`:

```php
$router->setRoute('home', new Route(
    '//<host>/',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

To match sub-domain:

```php
$router->setRoute('home', new Route(
    '//<sub>.domain.com/',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

You can combine host and path matcing:

```php
$router->setRoute('home', new Route(
    '//<sub>.domain.com/[<action>]',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
``` 

#### Immutability
All the route objects are immutable by design, you can not change their state after the creation, but make a copy 
with new values. To set default route parameters outside of the constructor:

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $route = new Route('/[<action>]', function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('home', $route->withDefaults([
            'action' => 'default'
        ]));
    }
}
```

#### Verbs
To specify route that matches only specific HTTP verb use method `withVerbs`:

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $route = new Route('/[<action>]', function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('get.route',
            $route->withVerbs('GET')->withDefaults(['action' => 'GET'])
        );

        $router->setRoute(
            'post.route',
            $route->withVerbs('POST', 'PUT')->withDefaults(['action' => 'POST'])
        );
    }
}
```

#### Middleware
To associate route specific middleware use `withMiddleware`, you can access the route parameters via `route` attribute
of the request object:

```php
namespace App\Bootloader;

use App\Middleware\ParamWatcher;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $route = new Route('/<param>', function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('home', $route->withMiddleware(
            ParamWatcher::class
        ));
    }
}
```

Where `ParamWatcher` is: 

```php
namespace App\Middleware;

use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Http\Exception\ClientException\UnauthorizedException;
use Spiral\Router\RouteInterface;

class ParamWatcher implements MiddlewareInterface
{
    public function process(Request $request, RequestHandlerInterface $handler): Response
    {
        /** @var RouteInterface $route */
        $route = $request->getAttribute('route');

        if ($route->getMatches()['param'] === 'forbidden') {
           throw new UnauthorizedException();
        }

        return $handler->handle($request);
    }
}
```

This route will trigger Unauthorized exception on `/forbidden`.

> You can add as many middleware to the route as you want.

## Multiple Routes
Router will match all the routes in order they were registered. Make sure to avoid situations were previous route
matches the conditions of the following routes.

```php
$router->setRoute(
    'home',
    new Route('/<param>',
        function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        }
    )
);

// this route will never trigger
$router->setRoute(
    'hello',
    new Route('/hello',
        function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        }
    )
);
```

## Default Route
Spiral Router provides the ability to specify default/fallback route. This route will always be invoked after every
other route and check for matching to it's pattern.

Use this route to define default routing behaviour of your application without need to write the route
for every controller and action.


```php
$router->setRoute(
    'home',
    new Route('/<param>',
        function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        }
    )
);

$router->setDefault(new Route('/', function () {
    return 'default';
}));
``` 

See below, how to effectively use default route to quickly scaffold applications. 

## Route Targets (Controllers and Actions)

phew
