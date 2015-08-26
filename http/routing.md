# Routing
Like other frameworks, Spiral provides a pre-built mechanism to manage your application's url structure. This operation is performed by the Http `Router` and `Route` classes.

## What is Router?
Router is the default http endpoint (see Http Request Flow) which accepts `ServerRequestInterface` and applies it to a set of created routes to find match (based on Uri, method or any other request property). When the route is matched, Router will execute it by providing instances of `ServerRequestInterface` and `ContainerInterface`. The rest of the logic (what controller is called and what parameters are passed) is located in the `Route` class itself.

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
     * @return ResponseInterface
     * @throws ClientException
     */
    public function __invoke(ServerRequestInterface $request);

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
     * @param ContainerInterface     $container
     * @return ResponseInterface
     */
    public function perform(ServerRequestInterface $request, ContainerInterface $container);

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

> Controller actions will be executed using `CoreInterface->callAction` every catched route parameter will be passed to action. Technically you can even assign your own router instance as Route endpoint to create nested routing.

If your component or module utilizes `RouterTrait` (as `HttpDispatcher` which we going to use as example) we can defined routes in shorter form:
```php
//This is Application bootstrap method
public function bootstrap()
{
    //Default route name will be based on route target in our case 'Controllers\MyController::myAction' you can use this name later to generate urls
    $this->http->route('myurl', 'Controllers\MyController::myAction')->setName('name');
}
```

Let's try to understand how we can defined Uris to be handled, parameters and additional route conditions. Let's start with simple definition which will point "url" to our home controller action index.

```php
$this->http->route('url', 'Controllers\HomeController::index');
```

> Route will match url repsecting value of `basePath` in your `HttpDispatcher` configuration.

If we wish to fetch some value from our Uri and convert it into action parameter we can use `< >` braces:

```php
$this->http->route('profile-<id>', 'Controllers\UserController::showProfile');
```

This route will match every url which will look like "profile-*" (profile-1, profile-abc) and convert second part of url into parameter named "id", we can now modify our controller action to accept such parameter:

```php
public function showProfile($id)
{
    //Doing something
}
```

Due every route url pattern will be converted into regular expressing we can clarify some url segments with specific pattern, for example let's make showProfile react only for numeric ids:

```php
$this->http->route('profile-<id:\d+>', 'Controllers\UserController::showProfile');
```

You are able to use any amount of route parameter and include / into your pattern:

```php
$this->http->route('pages/<name>', 'Controllers\PageController::show')->setName('page');
```

In addtion to that, you might want to state that some of your url segments are optional, we can use `[ ]` for that:

```php
$this->http->route('profile[-<id:\d+>]', 'Controllers\UserController::showProfile');
```

This route now match urls like "profile-1" and "profile", hovewer you have to modify your action to state default value for id, for example null:

```php
public function showProfile($id = null)
{
    //Doing something
}
```

You also can use default argument of route defintition to give value for optional parameter:

```php
$this->http->route('profile[-<id:\d+>]', 'Controllers\UserController::showProfile', ['id' => 0]);
```

There is few scenarious when you might want to point your route to every controller action and include action name into route, we can do it using special parameter
`<action>` for that and including such action into our target:

```php
$this->http->route('accounts/<action>', 'Controllers\UserController::<action>');
```

As in case with other paramerts your action might be stated as optional, in this case controller will execute it's default action (usually index):

```php
$this->http->route('accounts[/<action>]', 'Controllers\UserController::<action>');
$this->http->route('users[/<action:edit|save|open>]', 'Controllers\UserController::<action>');
```

Let's try to create route which is going to execute specific controller action and provide id parameter to such actions:

```php
$this->http->route('accounts[/<action>[/<id:\d+>]]', 'Controllers\UserController::<action>');
```

You can also ask rounte to match full uri (including hostname) rather than path only (`basePath` value ingnored in this case):

```php
$this->http->route(
      '<username>.domain.com[/<action>[/<id>]]',
      'Controllers\UserController::<action>'
)->useHost();
```

## Routes and [Middlewares](middlewares.md)
Middlewares assigned to Route will be executed before controller/endpoint action met and can be used to perform route specific request/response filteting.
For example we can create cache middleware which is going to store controller response in memory and set valid response cookies:

```php
//Cache content for 1 day
$this->http->route('showSomething', 'Controllers\SomeController::show')->with(new CacheMiddleware(86400));
```

> Route middlewares are perfect spot for caching, csrf and access limiting middlewares.

## Direct and Default routes
As mention before, if you didn't define any of your route, or request can not be matched Router will try to match and perform request using default route, default route
defintion located in http config and looks like:

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

This defintition will be used to construct instance of `DirectRoute`, direct route has differs from usual routes by granting ability to include controller name
into url, controller name will be combined with default namespace and postfix to create controller class. In addtional to that you can defined default controler name
and set of controller alises.

Let's check url like "/users", this url will be matched and will create set of parameters `[controller => users, action => null, id => null]`, due there is no such controller alias "users" (there is only "index" alias pointing to `Controllers\HomeController`) controller class will be generated using defined namespace and postfix:
`Controllers\UsersController`.

## Generate URLs using routes
In many cases you may want to generate URL specific to some route without locking yourself to url pattern, you can do that by using Router method `createUri` which will return `UriInterface`.

```php
//routeName = 'profile[/<id>]'
dump($this->router->createUri('routeName', 'id' => $user->id)); // profile/ID
```

You should only provide route name and set of parameters to be interpolated into URL. If route does not have some of provided parameters in it's pattern, such parameters will be added into query string:

```php
//routeName = 'profile[/<id>]'
dump($this->router->createUri('routeName', 'id' => $user->id, 'success' => 'yes')); // /profile/ID?success=yes
```

In addition to that you can specify route name in a format "controller::action", such name will be exploded and interpolated into pattern defined in default route:
```php
dump($this->router->createUri('users::edit', 'id' => $user->id, 'success' => 'yes')); // /users/edit/ID?success=yes
```

> Do not forget to convert `UriInterface` to string before sending to client over json.

## Using Router separatelly from HttpDispatcher
You can eslity use Router class outside of HttpDispatcher, for example by writing 
