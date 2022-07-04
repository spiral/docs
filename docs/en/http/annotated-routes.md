# HTTP - Annotated Routing
Use `spiral/annotated-routes` extension to declare application routes using annotations. To install the extension:

```bash
$ composer require spiral/annotated-routes
```

> Please note that the spiral/framework >= 2.6 already includes this component.

Activate the bootloader `Spiral\Router\Bootloader\AnnotatedRoutesBootloader` in your application (you can disable the default `RoutesBootloader`):

```php
protected const LOAD = [
    Spiral\Router\Bootloader\AnnotatedRoutesBootloader::class
];
```

You can use the extension.

## Define the route
To define the route, make sure to use `Spiral\Router\Annotation\Route`: 

```php
namespace App\Controller;

use Spiral\Router\Annotation\Route;

class HomeController
{
    /**
     * @Route(route="/", name="index", methods="GET")
     */
    public function index()
    {
        return 'hello world';
    }
}
```

Following route attributes are available:

Attribute | Type | Description
--- | --- | ---
route | string | Route patterns, follow the same rules as the default [Router](/http/routing.md). Required.
name | string | Route name. Required.
methods | array/string | HTTP methods. Defaults to all methods. 
defaults | array | Default values for route pattern.
group | string | Route group, defaults to `default`.
middleware | array | Route specific middleware class names.
priority | int | (Available since v2.9). Position in a routes list. Higher priority routes are sorted before lower ones. Helps to solve situations when one request matches two routes. Defaults to 0.

## Route Groups
It is possible to apply shared rules, middleware, [domain core](/cookbook/domain-core.md), or prefix to the group of routes. Create and register bootloader to achieve that:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\GroupRegistry;

class APIRoutes extends Bootloader
{
    public function boot(GroupRegistry $groups)
    {
        $groups->getGroup('api')
               ->setPrefix('/api/v1')
               ->addMiddleware(SomeMiddleware::class);
    }
}
```

> Make sure to register bootloader after `AnnotatedRoutesBootloader`. Use the `default` group to configure all the routes.

You can now assign the route to the group using the `group` attribute. 

```php
/**
 * @Route(route="/", name="index", methods="GET", group="api")
 */
public function index(){
    // ...    
}
```

> Route middleware, prefix, and domain core will be added automatically.

## Route Cache
By default, all the annotated routes cached when `DEBUG` is off. To turn on/off route-cache separately from `DEBUG` env variable
set the `ROUTE_CACHE` env variable:

```dotenv
DEBUG = true
ROUTE_CACHE=true
```

Run the `route:reset` to reset the route cache:

```bash
$ php app.php route:reset
```
