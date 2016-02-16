# Routing - OUTDATED
Like other frameworks, Spiral provides a pre-built mechanism to manage your application's url structure. This operations is performed by the Http `Router` and `Route` classes.

## Router
To manage set of routes inside application spiral provides higher level abstraction `RouterInterface` which is responsible for processing
request to detect valid route. Router by default has been implemented as `Spiral\Http\Routing\Router` and can be treated as PSR-7 endpoint. 

```php
interface RouterInterface
{
    /**
     * Valid endpoint for MiddlewarePipeline.
     *
     * @param ServerRequestInterface $request
     * @param ResponseInterface      $response
     * @return ResponseInterface
     * @throws ClientException
     */
    public function __invoke(ServerRequestInterface $request, ResponseInterface $response);
    
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
     * @return Route
     * @throws RouterException
     * @throws UndefinedRouteException
     */
    public function getRoute($name);
    
    /**
     * Get all registered routes.
     *
     * @return RouteInterface[]
     */
    public function getRoutes();
    
    /**
     * Generate valid route URL using route name and set of parameters. Should support controller
     * and action name separated by ":" - in this case router should find appropriate route and
     * create url using it.
     *
     * @param string             $route Route name.
     * @param array|\Traversable $parameters
     * @return UriInterface
     * @throws RouterException
     * @throws RouteException
     * @throws UndefinedRouteException
     */
    public function uri($route, $parameters = []);
}
```

Per declaration you are able to add as many route instances into router as you want, in addition you can add "default route" which is going to be automatically invoked when no other routes matched request (see examples below).

## Custom Routers
Spiral router provides deep integration with application container, if you wish to use custom router you can do it by setting endpoint in your HttpDispatcher. Let's review an example of how we can use [Aura.Router](https://github.com/auraphp/Aura.Router) in spiral application. Instead of implementing RouterInterface we are going to change endpoint using routing bootloader:

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

> Attention, Aura.Router will not create container scope for request and response so your functionality are limited, try using MiddlewarePipeline inside your endpoint to define such scope.

## Routes
Spiral routes work little bit different way compared to Aura.Routes (see example above).
