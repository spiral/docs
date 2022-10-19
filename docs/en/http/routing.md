# Routing
Spiral framework includes routing functionality into it's Http component. Like other frameworks you are able to define URL routes to a specific controller, controller actions and group of controllers in order to minimize amount of routes.

Each route can be associated to a specific HMVC core in order to get additional flexibility.

## Router
To manage set of routes inside application spiral provides higher level abstraction `RouterInterface` which is responsible for aggregating routes and processing request to detect valid route.

Router by default has been implemented as `Spiral\Http\Routing\Router` and can be treated as PSR-7 endpoint. 

```php
interface RouterInterface
{
    /**
     * Valid endpoint for MiddlewarePipeline.
     *
     * @param ServerRequestInterface $request
     * @param ResponseInterface      $response
     *
     * @return ResponseInterface
     * @throws ClientException
     */
    public function __invoke(
        ServerRequestInterface $request,
        ResponseInterface $response
    ): ResponseInterface;

    /**
     * @param RouteInterface $route
     */
    public function addRoute(RouteInterface $route);

    /**
     * Default route is needed as fallback if no other route matched the request.
     *
     * @param RouteInterface $route
     */
    public function defaultRoute(RouteInterface $route);

    /**
     * Get route by it's name.
     *
     * @param string $name
     *
     * @return RouteInterface
     *
     * @throws RouterException
     * @throws UndefinedRouteException
     */
    public function getRoute(string $name): RouteInterface;

    /**
     * Get all registered routes.
     *
     * @return RouteInterface[]
     */
    public function getRoutes(): array;

    /**
     * Generate valid route URL using route name and set of parameters. Should support controller
     * and action name separated by ":" - in this case router should find appropriate route and
     * create url using it.
     *
     * @param string             $route Route name.
     * @param array|\Traversable $parameters
     *
     * @return UriInterface
     * @throws RouterException
     * @throws RouteException
     * @throws UndefinedRouteException
     */
    public function uri(string $route, $parameters = []): UriInterface;
}
```

## Routes
Each route in spiral is represent by immutable, container specific object:

```php
interface RouteInterface
{
    /**
     * Isolate route endpoint in a given container.
     *
     * @param ContainerInterface $container
     *
     * @return self
     */
    public function withContainer(ContainerInterface $container): RouteInterface;

    /**
     * Returns new route instance.
     *
     * @param string $name
     *
     * @return RouteInterface
     */
    public function withName(string $name): RouteInterface;

    /**
     * @return string
     */
    public function getName(): string;

    /**
     * Prefix must always include back slash at the end of prefix!
     *
     * @param string $prefix
     *
     * @return self
     */
    public function withPrefix(string $prefix): RouteInterface;

    /**
     * @return string
     */
    public function getPrefix(): string;

    /**
     * Returns new route instance.
     *
     * @param array $matches
     *
     * @return self
     */
    public function withDefaults(array $matches): RouteInterface;

    /**
     * Get default route values.
     *
     * @return array
     */
    public function getDefaults(): array;

    /**
     * Check if route matched with provided request. Must return new route in case of success.
     *
     * @param ServerRequestInterface $request
     *
     * @return self|null
     *
     * @throws RouteException
     */
    public function match(ServerRequestInterface $request);

    /**
     * Execute route on given request. Has to be called after match method.
     *
     * @param ServerRequestInterface $request
     * @param ResponseInterface      $response
     *
     * @return ResponseInterface
     */
    public function __invoke(
        ServerRequestInterface $request,
        ResponseInterface $response
    ): ResponseInterface;

    /**
     * Generate valid route URL using route name and set of parameters.
     *
     * @param array|\Traversable $parameters
     *
     * @return UriInterface
     *
     * @throws RouteException
     */
    public function uri($parameters = []): UriInterface;
}
```

> Please note that routes will utilize associated container in order to resolve it's target and target dependencies, in classic example container will deliver HMVC code (i.e. your App class).

To better understand how routing works let's try to do a quick "math".

##### Find route
First of all, once router receives incoming request and response it will try to find appropriate route by calling method `match` of every registered route, if route matches request (valid URL pattern, method and etc) it MUST return it's own copy where internal route parameters (matches) are set accordingly to request.

> This technique used to properly generate route specific URIs.

If no route can be found, Router will use fallback route method `match` which expected to work for every request.

> If fallback route is missing or request can not be matched (for example due prefix/base-path), 404 error will be raised.

##### Perform route
Once appropriate route found it is going to be performed using method `__invoke` and incoming request and response. Right before request will be passed into route, attribute `route` will be set in it, which provides you ability to access active route using IoC scoping:

```php
public function indexAction(RouteInterface $route)
{
    dump($route);
    //Alternative
    dump($this->route);
}
```

> Every route is performed under same IoC container as associated with Router!

## Setting up Routing
Once you know the basics, it's time to set routing inside your application, if you already installed [skeleton application](https://github.com/spiral/application) you can find application routing set in app/classes/Bootloaders/HttpBootloader.php file:

```php
public function boot(HttpDispatcher $http)
{
    //Register route in a default http router (you can change router using setRouter() method)
    $http->addRoute($this->sampleRole());

    //Default route used as "fallback" when no other route work
    $http->defaultRoute($this->defaultRoute());
}
```

You can easily add new route by calling method `addRoute` of HttpDispatcher, prefix will be mounted automatically by Router:

```php
public function boot(HttpDispatcher $http)
{
    //Register route in a default http router (you can change router using setRouter() method)
    $http->addRoute($this->sampleRole());

    //Default route used as "fallback" when no other route work
    $http->defaultRoute($this->defaultRoute());
}
```

## Route Patterns and options
Route suppose to match only specific set of urls, such URLs are specified in a second parameter of default route implementation `Spiral\Http\Routing\Route`, in a previous example we used pattern "test", which is going to react ONLY for urls like "test", "TEST" and etc.

```php
$route = new Route('test', 'test', function () {
    return 'test';
})
```

As you might notice there is no start slash in a beginning of pattern as it will be added automatically as basePath prefix (see http config).

You are able to define more complex patterns to march longer URLsl:

```php
$route = new Route('test', 'test/abc', function () {
    return 'test';
});

$http->addRoute($route);
```

#### Parameters and Variable parts
No route can be useful without ability to define it's parameters and default values, let's try to define part of out route to be variable:

```php
$route = new Route('test', 'test/<abc>', function () {
    return 'test';
});
```

Wrapping route segment inside `<` and `>` braces will make it variable but not optional. Such segment, by default, will react to almost every character except `/`, meaning we can access our closure code using urls like "test/hello-world" or "test/abc". 

If you wish to access matched value in your closure access route via request attribute:

```php
$route = new Route('test', 'test/<abc>', function ($req, $res) {
    dump($req->getAttribute('route')->getMatch('abc'));

    return 'test';
});
```

To define specific pattern for your segment simply specify it in regex form like that: 

```php
$route = new Route('test', 'test/<abc:\d+>', function ($req, $res) {
    dump($req->getAttribute('route')->getMatch('abc'));

    return 'test';
});
```

Now, `abc` segment going to react only for numeric values.

Also, there is an ability to define vaiable parts which are not part of any match using `(` and `)`, for example:

```php
$route = new Route('test', '(test|hello)/<abc:\d+>', function ($req, $res) {
    dump($req->getAttribute('route')->getMatch('abc'));

    return 'test';
});
```

We just allowed route to react not only to "test" keyword but to "hello" as well.

Spiral will automatically pass route parameters to your controllers based on their name:

```php
$route = new Route('test', '(test|hello)/<abc:\d+>', 'MyController::index');
```

```php
public function indexAction(strign $abc)
{
    //...
}
```

#### Optional parts
To define some segments as optional, wrap them inside square parenthesis.

```php
$route = new Route('test', '(test|hello)[/<abc:\d+>]', function ($req, $res) {
    dump($req->getAttribute('route')->getMatch('abc'));

    return 'test';
});
```

Such route will react to urls like "test", "test/123", "hello" and "hello/786". You can nest multiple optional parts inside each other.

#### Routing using domain name
Routing using domain name does not differs from regular routing, however you must enable such flag before registering route and corrent your pattern in a needed form:

```php
$route = new Route('user', '<username>.domain.com(/<action>)', '...');

//Routes are immutable
$route = $route->withHost();
```

> At this moment you are not able to route based on http method out of the box, simply extend Route class and overload match method to implement such ability (see method `withMatches`).

## Route Endpoints
Every route must point somewhere, this somewhere defined as route endpoint. In default class `Spiral\Http\Routing\Route` you can specific endpoint as third argument of class constructor. Such endpoint can be set in a multiple forms, see below.

#### Routing to Closures
The easiest way to set endpoint route is to use signature `__invoke(request, reponse)` and provide some class or closure, let's write such example here:

```php
$route = new Route('test', 'test', function ($req, $res) {
    $res->getBody()->write("Hello World!");

    return $res;
});

$http->addRoute($route);
```

#### Routing to Endpoints and Middlewares
Since Route can accept any valid instance of http endpoint (`__invoke(request, reponse)` signature) you are allowed to use any compatible class for such purposes including router itself (nested routers):

```php
//With base path /test
$router = new Router($container, '/test/');

$router->addRoute(new Route('abc', 'abc', function () {
    return 'abc';
}));

//We need <end> match to allow every url ending
$http->addRoute(new Route('test', 'test<ending:.*>', $router));
```

Routes can construct endpoints on demand via ServiceLocator:

```
$http->addRoute(new Route('adr', 'adr', CoolAction::class));
```

## Routing to Controller Action
Existed routes has pre-built integration with HMVC core(s) of spiral framework, this provides us ability to route to a specific controller action using :: separator in our target:

```php
new Route('test', 'test', 'Controllers\HomeController::index');
```

Using such method you can also pass additional parameters inside your controller action:

```php
new Route('test', 'test/<abc:\d+>', 'Controllers\HomeController::index');
```

And inside controller:

```php
public function indexAction(int $abc)
{
    dump($abc);
}
```

You can make your action name to be variable as well, let's try to demonstrate that:

```php
new Route('home', 'home/<action>', 'Controllers\HomeController::<action>');
```

Now our route can cover every method of a given controllers.

## Routing to Controllers
Since we are able to specify routes which can define variable controller action, let's also define route which can point to variable controller, to do that we have to use different route implementation `Spiral\Http\Routing\ControllersRoute`.

```php
$route = new ControllersRoute(
    'default',                          //Route name
    '[<controller>[/<action>[/<id>]]]', //Pattern [] braces define optional segment
    'Controllers'                       //Default namespace
);
```

Controller name will be composed automatically based on specified namespace and postfix (by default Controller), such route will not allow used to access controller located in any other namespaces (including child one). To bypass such limitation, you can define specific controller alias:

```php
$route = new ControllersRoute(
    'default',                          //Route name
    '[<controller>[/<action>[/<id>]]]', //Pattern [] braces define optional segment
    'Controllers'                       //Default namespace
);

$route = $route->withControllers([
    'index' => HomeController::class,
    'auth'  => \Vendor\Module\Controllers\AuthController::class
])
```

## Fallback Route
When no routes matched request, fallback route will be called, such technique
provides you ability to avoid route registration for every created controller and only specify routes when they needed, since fallback route can point to multiple controllers, it must be an instance of `ControllersRoute`:

```php
//Default route points to controllers located in namespace "Controllers" but not deeper
$route = new ControllersRoute(
    'default',                          //Route name
    '[<controller>[/<action>[/<id>]]]', //Pattern [] braces define optional segment
    'Controllers'                       //Default namespace
);

//Here we can define controller aliases and default controller
$route = $route->withControllers([
    //Aliases (you can register controllers with non default namespace here)
    'index' => HomeController::class
])->withDefaults([
    'controller' => 'index',
])->withMiddleware([
    CsrfFirewall::class
]);
```

Default route will react for every url which looks like "/", "/controller", "/controller/action" or "/controller/action/ID".

> The only important part of ControllersRoute to be used as fallback route is valid default value for controller (default website controller).

## Middlewares
To set middleware to be executed only for specific route simple call method `withMiddleware` which can accept middleware object, class name, array of class names or container bindings:

```php
$route = $route->withMiddleware([
    CsrfFilter::class
]);

$route = $route->withMiddleware('module.middleware');
$route = $route->withMiddleware(new Some Middleware());
```

## Accessing active route thought Request scope
You are able to access active route using IoC request scope, route can be accessed multiple ways:

```php
public function indexAction(RouteInterface $route)
{
    //Via DI
    dump($route);

    //Via shortcut
    dump($this->route);

    //Via request
    dump($this->request->getAttribute('route'));

    //Via Input Manager
    dump($this->input->attribute('route'));
}
```

Each of given calls will return parent/active route.

## Custom Routers
Spiral router provides deep integration with application container, if you wish to use custom router you can do it by setting endpoint in your HttpDispatcher. Let's review an example of how we can use [Aura.Router](https://github.com/auraphp/Aura.Router) in spiral application:

```php
class AuraBootloader extends Bootloader
{
    const BOOT = true;

    /**
     * @param HttpDispatcher $http
     * @param \App           $app
     */
    public function boot(HttpDispatcher $http, \App $app)
    {
        $router = new RouterContainer();
        $this->defineRoutes($router->getMap(), $app);
        
        $http->setEndpoint($this->createEndpoint($router, $app));
    }

    /**
     * @param Map  $map
     * @param \App $app
     */
    public function defineRoutes(Map $map, \App $app)
    {
        //Simple route
        $map->get('hello', '/hello/{name}', function ($request, $response) use ($app) {
            dump($request);

            return 'Hello world';
        });

        //Route which bypasses to controller
        $map->get('home', '/home/{action}', function ($request, $response) use ($app) {
            return $app->callAction(
                HomeController::class,
                $request->getAttribute('route')->attributes['action'],
                compact('request', 'response')
            );
        });
    }

    public function createEndpoint(RouterContainer $router, \App $app)
    {
        return function ($request, $response) use ($router, $app) {
            $route = $router->getMatcher()->match($request);

            if (empty($route)) {
                throw new NotFoundException();
            }

            $handler = new \ReflectionFunction($route->handler);

            return $handler->invokeArgs(
                $app->container->resolveArguments(
                    $handler,
                    [
                        'request'  => $request->withAttribute('route', $route),
                        'response' => $response
                    ]
                )
            );
        };
    }
}
```

> Attention, Aura.Router will not create container scope for request and response so your functionality are limited, try using MiddlewarePipeline inside your endpoint to define such scope
