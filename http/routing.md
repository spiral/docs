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

## Default Configuration
The default web application bundle allows you to call any controller action located in `App\Controller` using
`/<controller>/<action>` pattern. See below how to alter this behaviour.

> You controllers must have `Controller` suffer.

## Configuration
The component does not require any external configuration. You can create new routing via `Spiral\Router\RouterInterface` 
in your bootloader. We can start with simple `/` handler:

```php
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
The most effective way to use router is to target routes to the controllers and their actions. To demonstrate all the 
capabilities we will need multiple controllers in `App\Controller` namespace:

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

Create second controller using scaffolding `php .\app.php create:controller demo -a test`:

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

#### Route to Action
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
    public function boot(RouterInterface $router)
    {
        $router->setRoute(
            'index',
            new Route('/index', new Action(HomeController::class, 'index'))
        );
    }
}
```

You can combine this target with require or optional parameter, the parameter will be amiable as method injection to 
the desired target:

```php
$router->setRoute(
    'user',
    new Route('/user/<id:\d+>', new Action(HomeController::class, 'user'))
);
```

#### Wildcard Actions
We can point route to more than one controller action at the same time, in order to do that we have to define the
parameter `<action>` in our route pattern. Since one of the method require `<id>` parameter we can define it as 
optional:

```php
$router->setRoute(
    'home',
    new Route('/<action>[/<id>]', new Action(HomeController::class, ['index', 'user']))
);
```

> This route will match both `/index` and `/user/1` paths.

Behind the hood the route will compile into expression which is aware of action constrains `/^(?P<action>index|user)(?:\/(?P<id>[^\/]+))?$/iu`.
Such approach would not only allow you to increase the performance but also reuse same pattern with different action set.

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

#### Route to Controller
You point your route to all of the controller actions as once using `Spiral\Router\Target\Controller`. This target
requires `<action>` (unless default is forced) parameter to be defined. 

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute(
            'home',
            new Route('/home/<action>[/<id>]', new Controller(HomeController::class))
        );
    }
}
```

> Route matches `/home/index`, `/home/other` and `/home/user/1`.

Combine this target with the route defaults to make your urls shorter.

```php
$router->setRoute(
    'home',
    (new Route('/home[/<action>[/<id>]]', new Controller(HomeController::class)))
        ->withDefaults(['action' => 'index'])
);
```

> This route will match `/home` with `action=index`. Note, you must extend optional path segments `[]` till the end of the
> route pattern.

#### Route to Namespace
In some cases you might want to route to the set of controllers located in a same namespace. Use target `Spiral\Router\Target\Namespaced`
for this purposes. This target will require route parameters `<controller>` and `<action>` (unless default is forced). 

You can specify target namespace and controller class postfix:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Namespaced;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute('app', new Route(
            '/<controller>/<action>',
            new Namespaced('App\Controller', 'Controller')
        ));
    }
}
```

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

> This route will match `/` (home->index), `/home` (home->index), `/home/index`, `/home/other` and `/demo/test`. The 
> `/demo` will trigger not-found error as `DemoController` does not defines method `index`.

The default web-application bundle sets this route [as default](https://github.com/spiral/app/blob/master/app/src/Bootloader/RoutesBootloader.php#L42).
You don't need to create route for any of the controller added to `App\Contoller`, simply use `/controller/action` urls
to access needed method. If no action is specified the `index` will be used by default. The routing will be made to 
the public methods only.

> You can turn default route off, once the development is over.

#### Route to Controller Group
Alternative approach is to specify controller names manually without the need of common namespace. Use target
`Spiral\Router\Target\Group`. Target requires `<controller>` and `<action>` (unless default is forced) parameters to be defined.

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
    public function boot(RouterInterface $router)
    {
        $router->setRoute('app', new Route('/<controller>/<action>', new Group([
            'home' => HomeController::class,
            'demo' => DemoController::class
        ])));
    }
}
```

## RESTFul
All of the route targets listed above support third argument which specifies the method selection behaviour. Set this
parameter as `TargetInterface::RESTFUL` to automatically prefix all the methods with HTTP verb.

For example we can use the following controller:

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

> Invoking `/user/1` with different HTTP methods will call different controller methods. Note, you are still required
> to specify action.

#### Sharing target across routes
Another way to define RESTFul or similar routing to multiple controllers is to share common target with different routes.
Such approach will allow you to define your own controller style. 

For example we can route different HTTP verbs to the following controller(s):

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

Let's create an API which will look like `GET|POST|DELETE /v1/<controller>` and point to according controller(s)
methods.

Our base route will look like:

```php
$resource = new Route('/v1/<controller>', new Group([
    'user' => UserController::class,
]));
```

We can register it the router with different HTTP verbs and action values:

```php
namespace App\Bootloader;

use App\Controller\UserController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Group;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
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

Such approach allows you to use same route-set for multiple resource controllers.

## Url Generation 
The router provides the ability to generate Uri to based on any given route and it's parameters. 

```php
$router->setRoute(
    'home',
    new Route('/home/<action>', new Controller(HomeController::class))
);
```

Use method `uri` of `RouterInterface` to generate the URL:

```php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', ['action' => 'index']);

    dump((string)$uri); // /home/index
}
```

Additional parameters will be mounted as query string:

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

Note, all of the parameters passed into URL pattern will be slugified:

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

> You can use `@route(name, params)` directive in stempler views.
