# HTTP - Annotated Routing
Use `spiral/annotated-routes` extension to declare application routes using annotations. To install the extension:

```bash
$ composer require spiral/annotated-routes
```

Activate the bootloader `` in your application (you can disable the default `RoutesBootloader`):

```php
protected const LOAD = [
    Spiral\Router\Bootloader\AnnotatedRoutesBootloader::class
];
```

You can use the extension.

## Define the route
To define the route make sure to use `Spiral\Router\Annotation\Route`: 

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

Following route attributes are avialable:

Attribute | Type | Description
--- | --- | ---
asd | asd | asa
route | string | Route patterns, follows the same rules as the default [Router](/http/routing.md). Required.
name | string | Route name. Required.
methods | array|string | HTTP methods. Defaults to all methods. 
defaults | array | Default values for route pattern.
group | string | Route group, defaults to `default`.
middleware | array | Route specific middleware class names.

## Route Groups
