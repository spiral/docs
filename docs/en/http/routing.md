# HTTP — Routing

## Installation

By default, the router component is installed in Spiral, but if you want to use it in your custom build, use composer to
install the router component.

```terminal
composer require spiral/router
```

## Attribute-Based Routing

The simplest way to define routes using attributes directly in your controller methods. This can be a really convenient
way to set up your routes, and it has a few key benefits. For one, it can make your code more concise and easier to
read. It can also help to improve the separation of concerns in your application, and it makes it easier for other
developers to discover and understand the available routes. So if you're looking for a more organized and maintainable
way to set up your routes, attribute-based routing might be worth considering!

> **Note**
> We use Tokenizer component for [static analysis](../advanced/tokenizer.md) identify route attributes. To specify the
> directories where controllers should be searched for, refer to the Tokenizer documentation's section
> on [customizing search directories.](../advanced/tokenizer.md#customizing-search-directories).

> **Warning**
> Be cautious because Tokenizer will ignore any files that contain `include` or `require` statements in your
> controllers.

Just activate the bootloader `Spiral\Router\Bootloader\AnnotatedRoutesBootloader` in your application:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Router\Bootloader\AnnotatedRoutesBootloader::class,
        // ...
    ];
}
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Router\Bootloader\AnnotatedRoutesBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

That's it! Now you can use the component.

### Defining Routes

The `Spiral\Router\Annotation\Route` attribute enables you to establish a route in your controller method by setting
various properties:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Router\Annotation\Route;

class HomeController
{
    #[Route(route: '/', name: 'index', methods: 'GET')] 
    public function index(): string
    {
        return 'hello world';
    }
}
```

Here is a brief description of each of the properties:

| Property   | Type         | Description                                                                                                                                                 |
|------------|--------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| route      | string       | The route pattern, which defines the URL pattern that the route will match. [Router](/http/routing.md). **Required**                                        |
| name       | string       | Route name. **Optional**                                                                                                                                    |
| methods    | array/string | The HTTP methods that the route will match (e.g. `GET`, `POST`, `PUT`, etc.). Defaults to all methods.                                                      |
| defaults   | array        | An array of default values for route parameters.                                                                                                            |
| group      | string       | A route group that the route will belong to. Defaults to `default`.                                                                                         |
| middleware | array        | Route specific middleware class names.                                                                                                                      |
| priority   | int          | Position in a routes list. The lower the number, the more important the route. Helps to solve the cases when one request matches two routes. Defaults to 0. |

Using these properties, you can define the details of your route in a concise and organized way.

### Route name

It's generally a good idea to specify a name for your routes, as it can make it easier to reference them elsewhere in
your application. However, if you don't specify a name, Spiral will generate one for you automatically, which can be
convenient if you don't need to reference the route by name.

The framework will generate a default name for you based on the route pattern and the HTTP method(s) that it matches.

```php
#Route(route: '/api/news', methods: ["POST", "PATCH"]) // => post,patch:/api/news
````

## Routes definition

:::: tabs

::: tab Routing configurator

Spiral offers a convenient and organized way for developers to define routes using the`defineRoutes` method of
the `App\Application\Bootloader\RoutesBootloader` class. This method provides
a `Spiral\Router\Loader\Configurator\RoutingConfigurator` instance , which offers a range of methods for defining and
configuring routes.

> **Warning**
> `App\Application\Bootloader\RoutesBootloader` must be in the `LOAD` section of the bootloaders list.

With the `RoutingConfigurator`, developers can easily apply various settings such as middleware, prefixes, and HTTP
methods to their routes. It allows you to create and automatically register routes.

**Here is an example of how to define routes using the `RoutingConfigurator`:**

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
            ->group('web')
            ->methods(methods: ['GET'])
            ->action(NewsController::class, 'show');
            
        ...
    }
}
```

:::

::: tab Import routes

The `RoutingConfigurator` has a handy feature that lets you import routes from specific files.

**Here's an example of how to do that:**

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

namespace App\Application\Bootloader;

use Spiral\Boot\DirectoriesInterface;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

final class RoutesBootloader extends BaseRoutesBootloader
{
    public function __construct(
        private readonly DirectoriesInterface $dirs
    ) {
    }
    
    // ...

    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->import($this->dirs->get('app') . '/routes/web.php')
            ->group('web');
            
        $routes->import($this->dirs->get('app') . '/routes/api.php')
            ->prefix('/api');
            ->group('api');
    }
}
```

Open the `web.php` file, use the `add` method to add a new route.

```php app/routes/web.php
use App\Controller\HomeController;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

return function (RoutingConfigurator $routes): void {
    $routes->add(name: 'news.show', pattern: '/news/<id:int>')
        ->methods(methods: ['GET'])
        ->action(NewsController::class, 'show');
};
```

:::

::::

## Route configurator

Route configurator provides a variety of methods for defining and configuring routes.

### Route Target

Sets the target controller for the route

:::: tabs

::: tab Namespaced

In certain scenarios, it may be necessary to route to a collection of controllers residing within the same namespace. To
achieve this, employ the `namespaced` method. This target mandates the specification of route parameters `<controller>`
and `<action>` (unless the default value is enforced).

```php
$routes->add(name: 'admin', pattern: '/admin/<controller>/<action>')
    ->namespaced(
        namespace: 'App\Controllers\Admin', // required
    );
```

**Example request**

```bash
GET /admin/users/index
```

In this case, the `<controller>` parameter corresponds to `users` and the `<action>` parameter corresponds to `index`.
As a result, the request will be routed to the `index` action of the `UsersController` class located in
the `App\Controllers\Admin` namespace.

By default, the method assumes that controllers have a `Controller` postfix. However, if you wish to change the
default postfix, you can do so by using the postfix argument.

For example, if your controllers have a `Handler` postfix instead of `Controller`, you can set up the namespaced route
target as follows:

```php
$routes->add(name: 'admin', pattern: '/admin/<controller>/<action>')
    ->namespaced(
        namespace: 'App\Controllers\Admin',
        postfix: 'Handler'
    );
```

:::

::: tab Controller

The controller method enables you to direct your route to all actions of a specific controller at once. This target
necessitates the definition of an `<action>` parameter (unless a default value is imposed).

```php
$routes->add(name: 'user', pattern: '/user/<action>')
    ->controller(controller: UserController::class);
```

**Example request**

```bash
GET /user/list
```

Here, the `<action>` parameter is `list`. The request will be routed to the `list` action of the `UserController` class.

By default, if the route pattern does not contain an `<action>` segment, the `index` action will be used as the default
action for the specified controller.

You can also specify a different default action for the controller route target using the default method. For example,
if you want to set the default action to `list` instead of `index`, you can do so as follows:

```php
$routes->add(name: 'user', pattern: '/user/list')
    ->controller(controller: UserController::class)
    ->defaults(['action' => 'list'])
```

:::

::: tab Action

To designate your route to a particular controller action, use the action method to specify the route handler.

```php
$routes->add(name: 'post.show', pattern: '/post/<id:int>')
    ->action(PostController::class, 'show');
```

**Example request**

```bash
GET /post/42
```

The request will be routed to the show action of the `PostController` class, with the value `42` passed as an argument
to the `show` action.

:::

::: tab Callable

It allows you to define routes that point directly to a callable function, closure, or an array consisting of an
object and a method name. This can be a useful option when you want to handle a specific route with a small amount of
logic without creating a separate controller.

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

$routes->add(name: 'greeting', pattern: '/greeting')
    ->callable(function (ServerRequestInterface $r, ResponseInterface $rsp) {
        return 'Hello, world!';
    });
```

```


:::

::::

### Default route parameters

```php
$routes
    ->add(name: 'html', pattern: '/<action>.html')
    ->defaults(['action' => 'default'])
    ->...;
```

### Custom domain core

```php
$core = new \Spiral\Core\InterceptableCore(...);
$core->addInterceptor(...);

$routes
    ->add(name: 'html', pattern: '/<action>.html')
    ->core($core)
    ->...;
```

### Prefixing routes

```php
$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->prefix('/api')
    ->...;
```

### HTTP methods (verbs)

```php
$routes->add(name: 'html', pattern: '/<action>.html')
    ->methods('GET')
    ->...;

// or

$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->methods(['GET', 'POST'])
    ->...;
```

### Add middleware

```php
$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->middleware(LocaleSelector::class)
    ->...;
```

### Fallback route

In some cases, users may request pages that do not exist, or the application may receive a URL that does not match any
pre-defined route. To handle these scenarios gracefully, it is crucial to have a fallback route in place that can catch
these unmatched requests and return a meaningful response to the user.

The provided code snippet demonstrates an example of a fallback route in Spiral:

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

$routes->default('/<path:.*>')
    ->callable(function (ServerRequestInterface $r, ResponseInterface $rsp) {
        return 'Page not found!';
    });
```

> **Note**
> You can use not only callable, but also any other route targets: controller, action, namespaced, etc.

## Route group configurator

You can easily organize your application's routes into logical groups and apply middleware, prefixes, and other settings
to all routes within a group with just a few lines of code. This makes it easy to maintain and scale your application,
and can save you a lot of time and effort when working with large, complex projects.

You can set up route groups via `App\Application\Bootloader\RoutesBootloader`. This bootloader contains the
`configureRouteGroups` method, which contains the `Spiral\Router\GroupRegistry` in the parameters.

Here is a simple example of how to set up a route group:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Router\GroupRegistry;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function configureRouteGroups(GroupRegistry $groups): void
    {
        $groups->getGroup('api')
            ->setNamePrefix('api.')
            ->setPrefix('/api');
            
        $groups->getGroup('web')
            ->addMiddleware(MyMiddelware::class)
            ->setPrefix('/api');
    }
}
```

And here are examples of how to assign routes to groups:

:::: tabs

::: tab Route

You can assign routes to the group by specifying the group's name.

```php app/src/Application/Bootloader/RoutesBootloader.php
$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->action(NewsController::class, 'show')
    ->group('auth');
    ->methods('GET');
```

:::

::: tab Attributes

You can assign routes to the group by specifying the group's name in the group attribute.

> **Warning**
> Make sure to register a bootloader after `AnnotatedRoutesBootloader`.

```php app/src/Endpoint/Web/HomeController.php
#[Route(route: '/', name: 'index', methods: 'GET', group: 'api')]  
public function index(): ResponseInterface
{
    // ...    
}
```

In this example, the route will be assigned to the `api` group and will be prefixed with `/api/v1` and have the
`SomeMiddleware` middleware applied to it. This allows you to easily apply shared rules and middleware to a group of
routes in a convenient and organized way.

:::

::: tab Globally
You can also set a default group for all routes:

```php app/src/Application/Bootloader/RoutesBootloader.php
protected function configureRouteGroups(GroupRegistry $groups): void
{
    // ...
    
    $groups->getGroup('api')
        ->setNamePrefix('api.')
        ->setPrefix('/api');
    
    $groups->setDefaultGroup('api');
}
```

:::

::::

## Router

You can create new routing using `Spiral\Router\RouterInterface`.

We can start with a simple `/` handler:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

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

### Closure Handler

It is possible to pass the `closure` as a route handler. In this case our function will receive two
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

### Route Pattern and Parameters

You can use a route pattern to specify any number of required and optional parameters. These parameters will later be
passed to the route handler via the `ServerRequestInterface` attribute `matches`.

> **Note**
> Use `attribute:matches.id` in Request Filters to access these values.

Use the `<parameter_name:pattern>` form to define a route parameter, where the pattern is a regexp friendly expression.
You
can omit the pattern and just use `<parameter_name>`, in this case, the parameter will match `[^\/]+`.

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

You can specify any number of parameters and make some of them optional. For example we can match URLs
like `/group/user`, where `user` is optional:

```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

You can specify default parameter value using the third route argument:

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

Use `<parameter:pattern>` to specify a parameter pattern:

```php
$router->setRoute('home', new Route(
    '/user/<id:\d+>',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> **Note**
> This route will only match URLs with numeric `id` but it doesn't mean that the route attribute `id` will contain
> integer
> value. In this case, the attribute will always contain a string value.

#### Route pre-defined options

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

#### Match Host

To match the domain or sub-domain name, prefix your pattern with `//`:

```php
$router->setRoute('home', new Route(
    '//<host>/',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

To match a sub-domain:

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

#### Immutability

All route objects are immutable by design, you can not change their state after creation, but only make a copy
with new values. To set default route parameters outside the constructor:

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

#### Verbs

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

#### Middleware

To associate route-specific middleware, use `withMiddleware`. You can access route parameters via `route` attribute
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

### Multiple Routes

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

### Default Route

Spiral Router enables you to specify the default/fallback route. This route will always be invoked after every
other route and check for matching to its pattern.

E.g., there's no need to define the route for every controller and action if you set up your default routing in the
following way:

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

### Route Targets (Controllers and Actions)

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

#### Route to Action

To point your route to the controller action, specify the route handler as `Spiral\Router\Target\Action`:

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

#### Wildcard Actions

We can point a route to more than one controller action at the same time. To do that we have to define the
parameter `<action>` in our route pattern. Since one of the methods requires `<id>` parameter, we can make it optional:

```php
$router->setRoute(
    'home',
    new Route('/<action>[/<id>]', new Action(HomeController::class, ['index', 'user']))
);
```

> **Note**
> This route will match both `/index` and `/user/1` paths.

Under the hood, the route will be compiled into an expression that is aware of action
constrains `/^(?P<action>index|user)(?:\/(?P<id>[^\/]+))?$/iu`. Such an approach would not only allow you to increase
the performance but also reuse the same pattern with different action sets.

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

You can point your route to all of the controller actions at once using `Spiral\Router\Target\Controller`. This target
requires `<action>` parameter to be defined (unless the default value is forced).

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
> The route matches `/home/index`, `/home/other` and `/home/user/1`.

Combine this target with defaults to make your URLs shorter.

```php
$router->setRoute(
    'home',
    (new Route('/home[/<action>[/<id>]]', new Controller(HomeController::class)))
        ->withDefaults(['action' => 'index'])
);
```

> **Note**
> This route will match `/home` with `action=index`. Note that you must extend optional path segments `[]` till the end
> of
> the route pattern.

#### Route to Namespace

In some cases, you might want to route to the set of controllers located in the same namespace. Use
target `Spiral\Router\Target\Namespaced` for these purposes. This target will require route parameters `<controller>`
and `<action>` (unless the default is forced).

You can specify a target namespace and a controller class postfix:

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
> `/demo` will trigger not-found error as `DemoController` does not define method `index`.

The default web-application bundle sets this
route [as default](https://github.com/spiral/app/blob/2.x/app/src/Bootloader/RoutesBootloader.php#L42).
You don't need to create a route for any of the controllers added to `App\Controller`, simply use `/controller/action`
URLs to access the required method. If no action is specified, the `index` will be used by default. The routing will
point to the public methods only.

> **Note**
> You can turn the default route off once the development is over.

#### Route to Controller Group

The alternative is to specify controller names manually without a common namespace. Use target
`Spiral\Router\Target\Group`. Target requires `<controller>` and `<action>` parameters to be defined (unless the default
is
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

## Named route patterns

If you would like a route parameter to always be constrained by a given regular expression, you may use named patterns.
You should define these patterns via `Spiral\Router\Registry\RoutePatternRegistryInterface` in a bootloader:

```php
use Spiral\Router\Registry\RoutePatternRegistryInterface;

class AppBootloader extends Bootloader
{
   public function boot(RoutePatternRegistryInterface $patternRegistry): void
   {
      $patternRegistry->register(
          'uuid', 
          '[0-9a-fA-F]{8}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{12}'
      );
      $patternRegistry->register(
          'names', 
          new InArrayPattern(['tom', 'jerry'])
      );
   }
}
```

Once the pattern has been defined, it is automatically applied to all routes using that parameter name:

#### Example:

```php
#Route(uri: 'blog/post/<post:uuid>')  // <===== Will match: /blog/post/f403554a-e70f-479a-969b-3edc047912a3
public function show(string $post)
{ 
    \var_dump($post); // f403554a-e70f-479a-969b-3edc047912a3
}
```

```php
#Route(uri: 'user/<name:names>') // <===== Will match: /user/tom || /user/jerry
public function show(string $name)
{ 
    \var_dump($name); // tom
}
```

## RESTful

All of the route targets listed above support the third argument, which specifies the method selection behavior. Set
this parameter as `AbstractTarget::RESTFUL` to automatically prefix all the methods with HTTP verb.

For example, we can use the following controller:

```php app/src/Endpoint/Web/UserController.php
namespace App\Endpoint\Web;

class UserController
{
    public function getUser($id): string
    {
        return "get {$id}";
    }

    public function postUser($id): string
    {
        return "post {$id}";
    }

    public function deleteUser($id): string
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
> Invoking `/user/1` with different HTTP methods will call different controller methods. Note that you still need
> to specify the action name.

### Sharing target across routes

Another way to define RESTful or similar routing to multiple controllers is to share a common target with different
routes. Such an approach will allow you to define your controller style.

For example, we can route different HTTP verbs to the following controller(s):

```php app/src/Endpoint/Web/UserController.php
namespace App\Endpoint\Web;

class UserController
{
    public function load($id): string
    {
        return "get {$id}";
    }

    public function store($id): string
    {
        return "post {$id}";
    }

    public function delete($id): string
    {
        return "delete {$id}";
    }
}
```

Let's create an API that will look like `GET|POST|DELETE /v1/<controller>` and point to the corresponding controller(s)
methods.

Our base route will look like this:

```php
$resource = new Route('/v1/<controller>', new Group([
    'user' => UserController::class,
]));
```

We can register it with different HTTP verbs and action values:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

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

Use method `uri` of `RouterInterface` to generate a URL:

```php app/src/Endpoint/Web/UserController.php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', ['action' => 'index']);

    dump((string)$uri); // /home/index
}
```

Additional parameters will mount as a query string:

```php app/src/Endpoint/Web/UserController.php
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

The `uri` method will return the instance `Psr\Http\Message\UriInterface`:

```php app/src/Endpoint/Web/UserController.php
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

Note that all of the parameters passed into the URL pattern will be slugified:

```php app/src/Endpoint/Web/UserController.php
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

#### Handling Non-Latin Characters in URIs

The Spiral router component, by default, transliterates non-Latin characters into Latin characters when generating URIs.
This can be problematic, especially when maintaining the original character set in the URI is important, such as for SEO
or multilingual applications.

**For example:**

```php
$router->setRoute(
  'page',
  new Route('/page/<path>', ....),
);

$uri = $router->uri('page', ['path' => 'some-path']);
// Generates: /page/some-path

$uri = $router->uri('page', ['path' => 'некоторый-путь']); 
// Default behavior generates: /page/nekotoriy-put
```

To change this behavior, you can replace the default URI handler for a route with a custom encoding function using
the `withPathSegmentEncoder` method. This method allows you to define a custom function for segment encoding.

**Here is an example**

```php
$router->setRoute(
  'page',
  new Route('/page/<path>', ....),
);

// Get the route by name
$route = $router->getRoute('page');

// Replace the default URI handler with a custom encoding function for path segments
$route = $route->withUriHandler(
    $route->getUriHandler()->withPathSegmentEncoder(
      static fn(string $segment): string => \rawurlencode($segment),
    ),
);

// Generate the URI
$uri = $route->uri(['path' => 'некоторый-путь']);

// Generates: /page/%D0%BD%D0%B5%D0%BA%D0%BE%D1%82%D0%BE%D1%80%D1%8B%D0%B9-%D0%BF%D1%83%D1%82%D1%8C
```

To define a custom encoder configure a `Spiral\Router\UriHandler` factory
in the Container via a Bootloader.

**Here is an example of how to do this:**

```php app/src/Application/Bootloader/AppBootloader.php
<?php

declare(strict_types=1);

namespace App\Application\Bootloader;

use Psr\Http\Message\UriFactoryInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\UriHandler;

final class AppBootloader extends Bootloader
{
    public function defineSingletons(): array
    {
        return [
            UriHandler::class => static function (UriFactoryInterface $uriFactory) {
                return (new UriHandler($uriFactory))->withPathSegmentEncoder(
                    static fn(string $segment): string => \rawurlencode($segment)
                );
            },
        ];
    }
}
```

## Events

| Event                             | Description                                                     |
|-----------------------------------|-----------------------------------------------------------------|
| Spiral\Router\Event\Routing       | The Event will be fired `before` matching the route.            |
| Spiral\Router\Event\RouteMatched  | The Event will be fired when the route is successfully matched. |
| Spiral\Router\Event\RouteNotFound | The Event will be fired when the route is not found.            |

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
