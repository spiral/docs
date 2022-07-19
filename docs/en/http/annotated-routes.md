# HTTP - Annotated Routing
Use `spiral/annotated-routes` extension to declare application routes using attributes or annotations. To install the extension:

```bash
composer require spiral/annotated-routes
```

> **Note**
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
<?php

declare(strict_types=1);

namespace App\Controller;

use Spiral\Router\Annotation\Route;

class HomeController
{
    #[Route(route: '/', name: 'index', methods: 'GET', group: 'default')] 
    public function index(): string
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

## Route Cache
By default, all the annotated routes cached when `DEBUG` is off. To turn on/off route-cache separately from `DEBUG` env variable
set the `ROUTE_CACHE` env variable:

```dotenv
DEBUG=true
ROUTE_CACHE=true
```

Run the `route:reset` to reset the route cache:

```bash
php app.php route:reset
```
