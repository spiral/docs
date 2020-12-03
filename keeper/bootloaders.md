# Bootloaders
Keeper contains the next bootloaders:
- KeeperBootloader is the entrypoint for all other bootloaders
- GuestBootloader grants full access to guest users (use it only for tests)
- AnnotatedBootloader reads `Controller` and `Action` annotations (see [Routing](/keeper/routing.md))
- SitemapBootloader reads the sitemap annotations, also can be used for `Sitemap` definition via code (see [Sitemap](/keeper/sitemap.md))
- UIBootloader registers keeper views - layout, sidebar, breadcrumbs, grids, etc. (see [Sitemap](/keeper/sitemap.md))

## Usage
`KeeperBootloader` is abstract. First of all create your own inherited bootloader.
All the rest keeper bootloaders if needed should be registered in the `LOAD` const.
```php
<?php

declare(strict_types=1);

use App\Bootloader\Keeper;
use Spiral\DataGrid\Interceptor\GridInterceptor;
use Spiral\Domain\CycleInterceptor;
use Spiral\Domain\FilterInterceptor;
use Spiral\Domain\GuardInterceptor;
use Spiral\Keeper\Bootloader;
use Spiral\Keeper\Middleware;

class KeeperBootloader extends Bootloader\KeeperBootloader
{
    protected const NAMESPACE          = 'keeper';
    protected const PREFIX             = 'keeper/';
    protected const DEFAULT_CONTROLLER = 'dashboard';
    protected const CONFIG_NAME        = '';

    protected const LOAD = [
        Bootloader\SitemapBootloader::class,
        Bootloader\AnnotatedBootloader::class,
    ];

    protected const MIDDLEWARE = [
        Middleware\LoginMiddleware::class
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        GuardInterceptor::class,
        FilterInterceptor::class,
        GridInterceptor::class,
    ];
}
```

### Namespace
You can define the current keeper namespace using `NAMESPACE` const. `keeper` is by default.

### Prefix
You can define the current route prefix using `PREFIX` const. `keeper/` is by default.

### Default Controller
You can define the default route controller using `DEFAULT_CONTROLLER` const.
Default action can be declared only via config file (or, if set, `defaultAction` annotation property or `index` method).

### Config name
You can define the config name for the current keeper namespace using `CONFIG_NAME` const.
If omitted, `NAMESPACE` const value will be used.

### Middlewares
You can list custom middlewares in the `MIDDLEWARE` const.
Additional middlewares can be applied in the routes separately for each route, both sets will be applied.

### Interceptors
You can list custom interceptors in the `INTERCEPTORS` const.

## Config
Keeper config can be fully declared in the config file.
> Config file should be stored in the app `config` directory.
```php
<?php

return [
   'routePrefix'   => '',
   'routeDefaults' => ['controller' => '', 'action' => ''],
   'loginView'     => 'keeper:login',
   'middleware'    => [],
   'modules'       => [],
   'interceptors'  => [],
];
```
Prefix, default controller, list of middlewares, modules' bootloaders and interceptors from the constants used as config defaults.

