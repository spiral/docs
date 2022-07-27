# HTTP - Routing

The framework [Web Bundle](https://github.com/spiral/app) includes pre-configured router component. To install the
router in alternative builds:

```bash
composer require spiral/router
```

> **Note**
> Please note that the spiral/framework >= 2.6 already includes this component.

And activate its Bootloader:

```php
[
    //...
    Spiral\Bootloader\Http\RouterBootloader::class,
]
```

> **Note**
> Read how to define routing using attributes or annotations [here](/http/annotated-routes.md).

## Creating new routes via RoutesBootloader

The default web application bundle allows you to configure application routes via `App\Bootloader\RoutesBootloader`.
This bootloader contains the `defineRoutes` method, which contains in the parameters the `Spiral\Router\Loader\Configurator\RoutingConfigurator`.
The `RoutingConfigurator` allows you to add routes or import routes from specific files.

### Import from files

By default, the import of routes from the `app/routes/web.php` file is already configured.

Open the `web.php` file, to add new route use the `add` method.

```php
use App\Controller\HomeController;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

return function (RoutingConfigurator $routes): void {
    // route to the controller action
    $routes->add('index', '/')->action(HomeController::class, 'index');
    
    // route to all of the controller actions at once
    $routes->add('html', '/<action>.html')->controller(HomeController::class);
    
    // route to all actions in multiple controllers
    $routes->add('group', '/<controller>/<action>/<id>')->groupControllers([
        'home' => HomeController::class,
        'demo' => DemoController::class
    ]);
    
    // route to the set of controllers located in the same namespace
    $routes->add('namespaced', '/<controller>/<action>/<id>')->namespaced('\App\Controllers');
    
    // route with callable handler
    $routes->add('hello', '/hello')->callable(static fn () => 'hello');

    // route with custom handler
    $routes->add('custom-handler', '/custom-handler')->handler(\App\Handlers\Handler::class);
};
```

To add HTTP verbs, default parameters, custom core, prefix, group:

```php
use App\Controller\HomeController;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

return function (RoutingConfigurator $routes): void {
    // HTTP verbs
    $routes
        ->add('html', '/<action>.html')
        ->controller(HomeController::class)
        ->methods('GET'); // or ->methods(['GET', 'POST'])
    
    // default parameters
    $routes
        ->add('html', '/<action>.html')
        ->controller(HomeController::class)
        ->defaults(['action' => 'default']);
    
    // custom core
    $routes
        ->add('html', '/<action>.html')
        ->controller(HomeController::class)
        ->core($core);
    
    // prefix
    $routes
        ->add('html', '/<action>.html')
        ->controller(HomeController::class)
        ->prefix('profile');

    // group
    $routes
        ->add('html', '/<action>.html')
        ->controller(HomeController::class)
        ->group('admin');    
};
```

Routes with the same `prefix` and `group` can be grouped in a separate file. Need to create a new file, for example, 
`app/routes/api.php` and add api routes:

```php
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

return function (RoutingConfigurator $routes): void {
    // routes
};
```

Import this file in the `defineRoutes` method:

```php
// file app/src/Bootloader/RoutesBootloader.php
protected function defineRoutes(RoutingConfigurator $routes): void
{
    // all routes from this file will have a prefix and a group
    $routes
        ->import($this->dirs->get('app') . '/routes/api.php')
        ->group('api')
        ->prefix('api');
}
```

### Adding routes in defineRoutes method

Routes can be added in the `defineRoutes` method using the `RoutingConfigurator`. 
The process of adding routes is exactly the same as in separate files:

```php
namespace App\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...
    
    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->add('index', '/')->action(HomeController::class, 'index');
    }
}
```

### Middlewares and groups

Middlewares are configured globally, which will be applied for all routes and for each group of routes separately.
Each group will have the global middlewares and the middlewares configured for that group applied.

You can configure middlewares in the `RoutesBootloader`:

```php
namespace App\Bootloader;

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
    protected function globalMiddleware(): array
    {
        return [
            LocaleSelector::class,
            ErrorHandlerMiddleware::class,
            JsonPayloadMiddleware::class,
            HttpCollector::class,
        ];
    }

    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                CookiesMiddleware::class,
                SessionMiddleware::class,
                CsrfMiddleware::class,
                // new Autowire(AuthTransportMiddleware::class, ['transportName' => 'cookie'])
            ],
            'api' => [
                // new Autowire(AuthTransportMiddleware::class, ['transportName' => 'header'])
            ],
        ];
    }
}
```

## Creating new routes via RouterInterface

### Default Configuration

The default web application bundle allows you to call any controller action located in `App\Controller`namespace using
`/<controller>/<action>` pattern. See below how to alter this behavior.

> **Note**
> Your controllers must have a `Controller` suffix.

## Configuration

The component does not require any external configuration. You can create new routing
via `Spiral\Router\RouterInterface` in your Bootloader. We can start with a simple `/` handler:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'home',                    // route name 
            new Route(
                '/',                   // pattern
                fn () => 'hello world' // handler
            )
        );
    }
}
```

> **Note**
> The Route class can accept a handler of type `Psr\Http\Server\RequestHandlerInterface`, closure, invokable class, 
> or `Spiral\Router\TargetInterface`. Simply pass a class or a binding name instead of a real object if you want it
> to be constructed on demand.

## Closure Handler

It is possible to pass the `closure` as route handler, in this case our function will receive two
arguments: `Psr\Http\Message\ServerRequestInterface` and `Psr\Http\Message\ResponseInterface`.

```php
$router->setRoute('home', new Route(
    '/<name>',
    function (ServerRequestInterface $request, ResponseInterface $response): ResponseInterface {
        $response->getBody()->write('hello world');

        return $response;
    }
));
```

## Route Pattern and Parameters

You can use a route pattern to specify any number of required and optional parameters. These parameters will later pass
to the route handler via the `ServerRequestInterface` attribute `matches`.

> **Note**
> Use `attribute:matches.id` in Request Filters to access these values.

Use the `<parameter_name:pattern>` form to define a route parameter, where pattern is a regexp friendly expression. You
can omit pattern and just use `<parameter_name>`, in this case, the parameter will match `[^\/]+`.

We can add a simple parameter `name`:

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute('home', new Route(
            '/<name>',
            function (ServerRequestInterface $request, ResponseInterface $response): array {
                return $request->getAttribute('route')->getMatches(); // returns JSON ['name' => '']
            }
        ));
    }
}
```

Use `[]` to make a part of route (including the parameters) optional, for example:

```php
$router->setRoute('home', new Route(
    '/[<name>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> **Note**
> This route will match `/`, the name parameter will be `null`.

You can specify any number of parameters and make some of them optional, for example we can match URLs
like `/group/user`, where `user` is optional:

```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

You can specify default parameter value using third route argument:

```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    },
    [
        'user' => 'default'
    ]
));
```

Use `<parameter:pattern>` to specify parameter pattern:

```php
$router->setRoute('home', new Route(
    '/user/<id:\d+>',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> **Note**
> This route will only match URLs with numeric `id`.

You can also specify multiple pre-defined options:

```php
$router->setRoute('home', new Route(
    '/do/<action:login|logout>',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
``` 

> **Note**
> This route will only match `/do/login` and `/do/logout`.

### Match Host

To match the domain or sub-domain name, prefix your pattern with `//`:

```php
$router->setRoute('home', new Route(
    '//<host>/',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

To match sub-domain:

```php
$router->setRoute('home', new Route(
    '//<sub>.domain.com/',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

You can combine host and path matching:

```php
$router->setRoute('home', new Route(
    '//<sub>.domain.com/[<action>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

### Immutability

All route objects are immutable by design, you can not change their state after creation, but only make a copy
with new values. To set default route parameters outside constructor:

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/[<action>]', function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('home', $route->withDefaults([
            'action' => 'default'
        ]));
    }
}
```

### Verbs

Use `withVerbs` method to match routes with only certain HTTP verbs:

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/[<action>]', function (ServerRequestInterface $request, ResponseInterface $response): array {
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

### Middleware

To associate route-specific middleware use `withMiddleware`, you can access route parameters via `route` attribute
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
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/<param>', function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('home', $route->withMiddleware(
            ParamWatcher::class
        ));
    }
}
```

where `ParamWatcher` is:

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

This route will trigger an Unauthorized exception on `/forbidden`.

> **Note**
> You can add as many middlewares as you want.

## Multiple Routes

The router will match all routes in the order they were registered. Make sure to avoid situations where the previous
route matches the conditions of the following routes.

```php
$router->setRoute(
    'home',
    new Route('/<param>',
        function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        }
    )
);

// this route will never trigger
$router->setRoute(
    'hello',
    new Route('/hello',
        function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        }
    )
);
```

## Default Route

Spiral Router provides the ability to specify the default/fallback route. This route will always be invoked after every
other route and check for matching to its pattern.

E.g., there's no need to define the route for every controller and action if you set up default routing like so:

```php
$router->setRoute(
    'home',
    new Route('/<param>',
        function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        }
    )
);

$router->setDefault(new Route('/', fn (): string => 'default'));
``` 

See below how to use the default route to scaffold application paths quickly.

## Route Targets (Controllers and Actions)

The most effective way to use the router is to target routes to the controllers and their actions. To demonstrate all
the capabilities, we will need multiple controllers in `App\Controller` namespace:

```php
namespace App\Controller;

class HomeController
{
    public function index(): string
    {
        return 'index';
    }

    public function other(): string
    {
        return 'other';
    }

    public function user(int $id): string
    {
        return "hello {$id}";
    }
}
```

Create a second controller using scaffolding `php ./app.php create:controller demo -a test`:

```php
namespace App\Controller;

class DemoController
{
    public function test(): string
    {
        return 'demo test';
    }
}
```

### Route to Action

To point your route to the controller action specify route handler as `Spiral\Router\Target\Action`:

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'index',
            new Route('/index', new Action(HomeController::class, 'index'))
        );
    }
}
```

You can combine this target with the required or optional parameter. The parameter will be available as method injection
to the desired target:

```php
$router->setRoute(
    'user',
    new Route('/user/<id:\d+>', new Action(HomeController::class, 'user'))
);
```

### Wildcard Actions

We can point a route to more than one controller action at the same time, to do that we have to define the
parameter `<action>` in our route pattern. Since one of the methods require `<id>` parameter, we can make it optional:

```php
$router->setRoute(
    'home',
    new Route('/<action>[/<id>]', new Action(HomeController::class, ['index', 'user']))
);
```

> **Note**
> This route will match both `/index` and `/user/1` paths.

Behind the hood, the route will compile into expression which is aware of action
constrains `/^(?P<action>index|user)(?:\/(?P<id>[^\/]+))?$/iu`. Such approach would not only allow you to increase the 
performance but also reuse the same pattern with different action sets.

```php
// match "/index"
$router->setRoute(
    'home',
    new Route('/<action>', new Action(HomeController::class, 'index'))
);

// match "/other"
$router->setRoute(
    'home',
    new Route('/<action>', new Action(HomeController::class, 'other'))
);

// match "/test"
$router->setRoute(
    'demo',
    new Route('/<action>', new Action(DemoController::class, 'test'))
);
```

### Route to Controller

You point your route to all of the controller actions at once using `Spiral\Router\Target\Controller`. This target
requires `<action>` parameter to be defined (unless the default value forced).

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'home',
            new Route('/home/<action>[/<id>]', new Controller(HomeController::class))
        );
    }
}
```

> **Note**
> Route matches `/home/index`, `/home/other` and `/home/user/1`.

Combine this target with defaults to make your URLs shorter.

```php
$router->setRoute(
    'home',
    (new Route('/home[/<action>[/<id>]]', new Controller(HomeController::class)))
        ->withDefaults(['action' => 'index'])
);
```

> **Note**
> This route will match `/home` with `action=index`. Note, you must extend optional path segments `[]` till the end of the
> route pattern.

### Route to Namespace

In some cases, you might want to route to the set of controllers located in the same namespace. Use
target `Spiral\Router\Target\Namespaced` for these purposes. This target will require route parameters `<controller>` 
and `<action>` (unless default is forced).

You can specify target namespace and controller class postfix:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Namespaced;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute('app', new Route(
            '/<controller>/<action>',
            new Namespaced('App\Controller', 'Controller')
        ));
    }
}
```

> **Note**
> This route will match `/home/index`, `/home/other` and `/demo/test`.

You can make all the parameters optional and set the default values:

```php
$router->setRoute('app',
    (new Route(
        '[/<controller>[/<action>]]',
        new Namespaced('App\Controller', 'Controller')
    ))->withDefaults([
        'controller' => 'home',
        'action'     => 'index'
    ])
);
```

> **Note**
> This route will match `/` (home->index), `/home` (home->index), `/home/index`, `/home/other` and `/demo/test`. The 
> `/demo` will trigger not-found error as `DemoController` does not defines method `index`.

The default web-application bundle sets this route [as default](https://github.com/spiral/app/blob/2.x/app/src/Bootloader/RoutesBootloader.php#L42).
You don't need to create a route for any of the controllers added to `App\Controller`, simply use `/controller/action` URLs
to access the required method. If no action is specified, the `index` will be used by the default. The routing will point
to the public methods only.

> **Note**
> You can turn the default route off once the development is over.

### Route to Controller Group

The alternative is to specify controller names manually without common namespace. Use target
`Spiral\Router\Target\Group`. Target requires `<controller>` and `<action>` parameters to be defined (unless default is
forced).

```php
namespace App\Bootloader;

use App\Controller\DemoController;
use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Group;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute('app', new Route('/<controller>/<action>', new Group([
            'home' => HomeController::class,
            'demo' => DemoController::class
        ])));
    }
}
```

> **Note**
> Such an approach is useful when you want to assemble multiple modules under one path (i.e., admin panels).

## RESTful

All of the route targets listed above support the third argument, which specifies the method selection behavior. Set
this parameter as `AbstractTarget::RESTFUL` to automatically prefix all the methods with HTTP verb.

For example, we can use the following controller:

```php
namespace App\Controller;

class UserController
{
    public function getUser(int $id): string
    {
        return "get {$id}";
    }

    public function postUser(int $id): string
    {
        return "post {$id}";
    }

    public function deleteUser(int $id): string
    {
        return "delete {$id}";
    }
}
```

And route to it:

```php
$router->setRoute('user', new Route(
    '/user/<id:\d+>',
    new Controller(UserController::class, Controller::RESTFUL),
    ['action' => 'user']
));
```

> **Note**
> Invoking `/user/1` with different HTTP methods will call different controller methods. Note, you still need
> to specify the action name.

### Sharing target across routes

Another way to define RESTful or similar routing to multiple controllers is to share a common target with different
routes. Such an approach will allow you to define your controller style.

For example, we can route different HTTP verbs to the following controller(s):

```php
namespace App\Controller;

class UserController
{
    public function load(int $id): string
    {
        return "get {$id}";
    }

    public function store(int $id): string
    {
        return "post {$id}";
    }

    public function delete(int $id): string
    {
        return "delete {$id}";
    }
}
```

Let's create an API that will look like `GET|POST|DELETE /v1/<controller>` and point to the corresponding controller(s)
methods.

Our base route will look like:

```php
$resource = new Route('/v1/<controller>', new Group([
    'user' => UserController::class,
]));
```

We can register it with different HTTP verbs and action values:

```php
namespace App\Bootloader;

use App\Controller\UserController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Group;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $resource = new Route('/v1/<controller>/<id>', new Group([
            'user' => UserController::class,
        ]));

        $router->setRoute(
            'resource.get',
            $resource->withVerbs('GET')->withDefaults(['action' => 'load'])
        );

        $router->setRoute(
            'resource.store',
            $resource->withVerbs('POST')->withDefaults(['action' => 'store'])
        );

        $router->setRoute(
            'resource.delete',
            $resource->withVerbs('DELETE')->withDefaults(['action' => 'delete'])
        );
    }
}
```

Such an approach allows you to use the same route-set for multiple resource controllers.

## Url Generation

The router can generate Uri based on any given route and its parameters.

```php
$router->setRoute(
    'home',
    new Route('/home/<action>', new Controller(HomeController::class))
);
```

Use method `uri` of `RouterInterface` to generate URL:

```php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', ['action' => 'index']);

    dump((string)$uri); // /home/index
}
```

Additional parameters will mount as a query string:

```php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', [
        'action' => 'index',
        'page'   => 123
    ]);

    dump((string)$uri); // /home/index?page=123
}
```

The `uri` method will return an instance of `Psr\Http\Message\UriInterface`:

```php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', [
        'action' => 'index',
        'page'   => 123
    ]);

    dump((string)$uri->withFragment('hello')); // /home/index?page=123#hello
}
```

Note, all of the parameters passed into the URL pattern will be slugified:

```php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', [
        'action' => 'hello World',
    ]);

    dump((string)$uri); // /home/hello-world
}
```

> **Note**
> You can use `@route(name, params)` directive in Stempler views.
