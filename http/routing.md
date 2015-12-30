# Routing
Like other frameworks, Spiral provides a pre-built mechanism to manage your application's url structure. This operations is performed by the Http `Router` and `Route` classes.

## What is Router?
Router is the default http endpoint (see Http Request Flow) which accepts `ServerRequestInterface` applies it to a set of created routes to find match (based on Uri, method or any other request property). When the route is matched, Router will execute it by providing instances of `ServerRequestInterface`, `ResponseInterface` and `ContainerInterface`. The rest of the logic (what controller is called and what parameters are passed) is located in the `Route` class itself.

Let's check out the `RouterInterface` to better understand how it works:

```php
interface RouterInterface
{
    /**
     * @param ContainerInterface $container
     * @param RouteInterface[]   $routes Pre-defined array of routes (if were collected externally).
     */
    public function __construct(ContainerInterface $container, array $routes = []);

    /**
     * Valid endpoint for MiddlewarePipeline.
     *
     * @param ServerRequestInterface $request
     * @param ResponseInterface $response
     * @return ResponseInterface
     * @throws ClientException
     */
    public function __invoke(ServerRequestInterface $request, ResponseInterface $response);

    /**
     * @param RouteInterface $route
     */
    public function addRoute(RouteInterface $route);

    /**
     * Fetch route by it's name.
     *
     * @param string $route
     * @return RouteInterface
     * @throws RouterException
     */
    public function getRoute($route);

    /**
     * @return RouteInterface[]
     */
    public function getRoutes();

    /**
     * Route which matches with the incoming request will be marked as active and can be fetched using
     * this method.
     *
     * @return RouteInterface
     */
    public function activeRoute();

    /**
     * Generate valid route URL using route name and set of parameters. This should support controller
     * and action name separated by ":" - In this case, the router should find the appropriate route and
     * create a url using it.
     *
     * @param string           $route Route name.
     * @param array            $parameters
     * @param SlugifyInterface $slugify
     * @return UriInterface
     * @throws RouterException
     * @throws RouteException
     */
    public function createUri($route, array $parameters = [], SlugifyInterface $slugify = null);
}
```

As you can see, Router defines a few methods which allow you to add routes in runtime. In addition to that, you can pre-create some routes before the router is created. For example, you can use `RouterTrait`:

```php
//This is Application bootstrap method
public function bootstrap()
{
    //This route will be sent to Router when created
    $this->http->addRoute(new MyRoute());
}
```

In addition to that, declaring the `RouterInterface` gives you the ability to retrieve route which matches the request (Router has to be invoked first), get a list of every declared route
or even create some user via the route name and set of parameters.

## Default Spiral Router
The default Spiral implementation of `RouterInterface` includes one additional thing which provides the ability to use default (fallback) route. Such a route can be provided as a pre-created instance, changed using `setDefaultRoute` or even consist of options that can be used to construct `DirectRoute` (see below). The main purpose of the default route is to handle Uris which weren't matched to any other route. Usually the default route is specified as an instance of `DirectRoute` which can fetch the controller action name from Uri and the controller class itself (using aliases, namespaces and postfixes).

## What is Route?
Every declared route must use `RouteInterface` which declares the basic route abilities:

```php
interface RouteInterface
{
    /**
     * Controller and action in route targets and createURL route name must be separated like
     * below.
     */
    const SEPARATOR = '::';

    /**
     * @return string
     */
    public function getName();

    /**
     * Check if route is matched with to the provided request.
     *
     * @param ServerRequestInterface $request
     * @param string                 $basePath
     * @return bool
     * @throws RouteException
     */
    public function match(ServerRequestInterface $request, $basePath = '/');

    /**
     * Execute route on a given request. Has to be named after the match method.
     *
     * @param ServerRequestInterface $request
     * @param ResponseInterface $response
     * @param ContainerInterface     $container
     * @return ResponseInterface
     */
    public function perform(ServerRequestInterface $request, ResponseInterface $response, ContainerInterface $container);

    /**
     * Generate valid route URL using the route name and a set of parameters.
     *
     * @param array            $parameters
     * @param string           $basePath
     * @param SlugifyInterface $slugify
     * @return UriInterface
     * @throws RouteException
     */
    public function createUri(
        array $parameters = [],
        $basePath = '/',
        SlugifyInterface $slugify = null
    );
}
```

You can create your own routes with custom logic inside, however its best to stick to the default implementation seen in this guide section.

## Route definition
In most of cases, you are going to work with the default spiral routes, represented by class `Spiral\Http\Routing\Route`. Such route lets you specify the
url pattern to be matched, the set of url chunks to be converted into parameters (which can later be sent to controller),  define a route target as controller::action, closure or even some callable class (for example inner Router or Module).

```php
//This is Application bootstrap method
public function bootstrap()
{
    $this->http->addRoute(new Route('name', 'myurl', 'Controllers\MyController::myAction'));
    $this->http->addRoute(new Route('another', 'another', MyModule::class));
    
    $this->http->addRoute(new Route('another', 'another', function(ServerRequestInterface $request) {
        return 'something';
    }));
}
```

> Controller actions are executed using `CoreInterface->callAction`. Every catched route parameter is passed to action. Technically, you can even assign your own router instance as a Route endpoint to create nested routing.

If your component or module utilizes `RouterTrait` (as `HttpDispatcher` which we are going to use as an example) we can define routes in this shorter form:
```php
//This is Application bootstrap method
public function bootstrap()
{
    //Default route name will be based on route target. In our case, with 'Controllers\MyController::myAction' you can use this name later to generate urls
    $this->http->route('myurl', 'Controllers\MyController::myAction')->setName('name');
}
```

Let's look at how we can define Uris to be handled, parameters are definded and any additional route conditions. Let's start with a simple definition which will point "url" to our home controller action index.

```php
$this->http->route('url', 'Controllers\HomeController::index');
```

> Route will match url respecting value of `basePath` in your `HttpDispatcher` configuration.

If you want to fetch a value from our Uri and convert it into action parameter, you can use `< >` braces:

```php
$this->http->route('profile-<id>', 'Controllers\UserController::showProfile');
```

This route will match every url, which will look like "profile-*" (profile-1, profile-abc) and convert the second part of url into a parameter named "id". We can now modify our controller action to accept such parameters:

```php
public function showProfile($id)
{
    //Doing something
}
```

Since every route url pattern will be converted into regular expression, we can clarify some url segments with a specific pattern. For example let's make showProfile only react to numeric ids:

```php
$this->http->route('profile-<id:\d+>', 'Controllers\UserController::showProfile');
```

You are able to use any amount of route parameter and include / into your pattern:

```php
$this->http->route('pages/<name>', 'Controllers\PageController::show')->setName('page');
```

In addtion, you may want to state that some of your url segments are optional. For that, we can use `[ ]`:

```php
$this->http->route('profile[-<id:\d+>]', 'Controllers\UserController::showProfile');
```

This route now matches urls like "profile-1" and "profile". however you have to modify your action to state the default value for id. For example null:

```php
public function showProfile($id = null)
{
    //Doing something
}
```

You can also use a default argument for the route definition to give an optional parameter a value:

```php
$this->http->route('profile[-<id:\d+>]', 'Controllers\UserController::showProfile', ['id' => 0]);
```

There are a few scenarios where you might want to point your route to every controller action and include the action name into your route. We can do it using special parameter
`<action>` and include this action into our target:

```php
$this->http->route('accounts/<action>', 'Controllers\UserController::<action>');
```

As is the case with other parameters, your action might be stated as optional. In this case, your controller will execute it's default action (usually index):

```php
$this->http->route('accounts[/<action>]', 'Controllers\UserController::<action>');
$this->http->route('users[/<action:edit|save|open>]', 'Controllers\UserController::<action>');
```

Let's try to create a route which is going to execute a specific controller action and provide an id parameter to these actions:

```php
$this->http->route('accounts[/<action>[/<id:\d+>]]', 'Controllers\UserController::<action>');
```

You can also ask route to match the full uri (including hostname) instead of just the path only (`basePath` value will be ingnored in this case):

```php
$this->http->route(
      '<username>.domain.com[/<action>[/<id>]]',
      'Controllers\UserController::<action>'
)->useHost();
```

## Routes and [Middlewares](middlewares.md)
Middlewares is assigned to Route and will be executed before controller/endpoint action is met. It can be used to perform route specific request/response filtering.
For example we can create a cached middleware which will store the controller response in memory and set a valid response cookies:

```php
//Cache content for 1 day
$this->http->route('showSomething', 'Controllers\SomeController::show')->with(new CacheMiddleware(86400));
```

> Route middlewares are a perfect spot for caching, csrf and access limiting middlewares.

## Direct and Default routes
If you haven't defined any of your routes or your request can't be matched, Router will attempt to match by performing your request via the default route. The default routes defintion is located in http config and looks like:

```php
 'router'       => [
        'class'   => Http\Routing\Router::class,
        'default' => [
            'pattern'     => '[<controller>[/<action>[/<id>]]]',
            'namespace'   => 'Controllers',
            'postfix'     => 'Controller',
            'defaults'    => [
                'controller' => 'home'
            ],
            'controllers' => [
                'index' => Controllers\HomeController::class
            ]
        ]
    ],
```

This defintition will be used to construct an instance of `DirectRoute`. Direct route is different from the usual routes because it grants the ability to include controller name into url. The controller name will be combined with default namespace and postfix to create the controller class. In addtion, you can define the default controller name
and set of controller aliases.

Let's check a url like "/users". This url will be matched and will create set of parameters `[controller => users, action => null, id => null]`. Since there isn't a controller alias "users" (there is only "index", alias is pointing to `Controllers\HomeController`) controller class is generated using a defined namespace and postfix:
`Controllers\UsersController`.

## Generate URLs using routes
In many cases, you may want to generate a URL specific to some route without locking yourself to a url pattern. You can do this by using Router method, `createUri` (attention, method will return instance of `UriInterface`).

```php
//routeName = 'profile[/<id>]'
dump($this->router->createUri('routeName', ['id' => $user->id])); // profile/ID
```

You should only provide a route name and set of parameters to be interpolated into URL. If route does not have all of the provided parameters in it's pattern, such parameters will be added into query string:

```php
//routeName = 'profile[/<id>]'
dump($this->router->createUri('routeName', ['id' => $user->id, 'success' => 'yes'])); // /profile/ID?success=yes
```

Additionally, you can specify the route name in a format "controller::action", so that the name will be interpolated into a pattern defined in the default route:
```php
dump($this->router->createUri('users::edit', ['id' => $user->id, 'success' => 'yes'])); // /users/edit/ID?success=yes
```
