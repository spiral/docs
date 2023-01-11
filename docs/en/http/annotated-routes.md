# HTTP — Annotated Routing

If you're using the Spiral Framework to build a PHP web application, you will be able to define HTTP routes using 
attributes. This means that you can define your routes directly in your controller methods, rather than in a separate 
routing file or a bootloader.

This can be a really convenient way to set up your routes, and it has a few key benefits. For one, it can make your code
more concise and easier to read. It can also help to improve the separation of concerns in your application, and it
makes it easier for other developers to discover and understand the available routes. So if you're looking for a more
organized and maintainable way to set up your routes, attribute-based routing might be worth considering!

Activate the bootloader `Spiral\Router\Bootloader\AnnotatedRoutesBootloader` in your application (you can disable the
default `RoutesBootloader`):

```php
protected const LOAD = [
    // ...
    \Spiral\Router\Bootloader\AnnotatedRoutesBootloader::class,
    // ...
];
```

That's it! Now you can use the component.

## Define the route

The `Spiral\Router\Annotation\Route` attribute allows you to define a route in your controller method by specifying
various properties:

```php
namespace App\Controller;

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

| Property   | Type         | Description                                                                                                                                                  |
|------------|--------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| route      | string       | The route pattern, which defines the URL pattern that the route will match. [Router](/http/routing.md). **Required**                                         |
| name       | string       | Route name. **Optional**                                                                                                                                     |
| methods    | array/string | The HTTP methods that the route will match (e.g. `GET`, `POST`, `PUT`, etc.). Defaults to all methods.                                                       |
| defaults   | array        | An array of default values for route parameters.                                                                                                             |
| group      | string       | A route group that the route will belong to. Defaults to `default`.                                                                                          |
| middleware | array        | Route specific middleware class names.                                                                                                                       |
| priority   | int          | Position in a routes list. Higher priority routes are sorted before lower ones. Helps to solve the cases when one request matches two routes. Defaults to 0. |

Using these properties, you can define the details of your route in a concise and organized way.

### Route name

It's generally a good idea to specify a name for your routes, as it can make it easier to reference them elsewhere in
your application. However, if you don't specify a name, the Spiral Framework will generate one for you automatically,
which can be convenient if you don't need to reference the route by name.

If you don't specify a name for your route, the framework will generate a default name for you based on the route
pattern and the HTTP method(s) that it matches. For
example, `#Route(route: '/api/news', methods: ["POST", "PATCH"])` will be named  `post,patch:/api/news`

## Route Groups

The Spiral Framework allows you to define route groups using the `Spiral\Router\GroupRegistry` class, which you can then
use to apply shared rules, middleware, [domain](/cookbook/domain-core.md), or prefix to a group of routes.

Here is an example of how you can define a route group:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\GroupRegistry;

class RoutesBootloader extends Bootloader
{
    // ...

    protected function configureRouteGroups(GroupRegistry $groups): void
    {
        $groups->getGroup('api')
               ->setPrefix('/api/v1')
               ->addMiddleware(SomeMiddleware::class);
    }
}
```

Once you have defined a route group, you can assign routes to the group by specifying the group's name in the group
attribute.

> **Note**
> Make sure to register a bootloader after `AnnotatedRoutesBootloader`.

Now you can assign the route to the group using the `group` attribute.

```php
#[Route(route: '/', name: 'index', methods: 'GET', group: 'api')]  
public function index(): ResponseInterface
{
    // ...    
}
```

In this example, the route will be assigned to the 'api' group and will be prefixed with `/api/v1` and have the
`SomeMiddleware` middleware applied to it. This allows you to easily apply shared rules and middleware to a group of
routes in a convenient and organized way.