# Routing
Keeper routes are accessible via global `RouterInterface`.
You can register them directly in the router or via keeper bootloader and annotations.
The last options allow you to isolate all routes in the given namespace.

## Register routes via bootloader or Router

New route should be declared after the `parent::boot()` call. Controllers must be declared before the route declaration:
```php
<?php

declare(strict_types=1);

use Spiral\Boot\BootloadManager;
use Spiral\Keeper\Bootloader;
use Spiral\Router\RouterInterface;

class AdminBootloader extends Bootloader\KeeperBootloader
{
    protected const NAMESPACE = 'admin';
    protected const PREFIX    = '/admin';

    public function boot(BootloadManager $bootloadManager, RouterInterface $appRouter): void
    {
        parent::boot($bootloadManager, $appRouter);

        // Controllers are used via aliases
        $this->addController('user', 'App\Admin\Controller\User');
        $this->addRoute('/users/new', 'user', 'new', ['GET'], 'createUser');
    }
}
```
The next routes will be created (see `php app.php route:list` output):
```
+-------------------+--------+------------------+---------------------------------- +
| Name:             | Verbs: | Pattern:         | Target:                           |
+-------------------+--------+------------------+---------------------------------- +
| admin[createUser] | POST   | /admin/users/new | App\Admin\Controller\User->create |
| admin[user.new]   | GET    | /admin/users/new | user->new                         |
+-------------------+--------+------------------+---------------------------------- +
```
> Routes will be duplicated by a generated name like `controller.method`.

## Register routes via annotations
Annotations is a more convenient way because annotated routes allows you to use [sitemaps](/keeper/sitemap.md) - another powerful sub-module for building menu navigation and breadcrumbs.
`\Spiral\Keeper\Annotation\Action` and `\Spiral\Keeper\Annotation\Controller` annotations are available. They should be used together.
`Controller` defines:
 - current namespace (optional, `keeper` by default)
 - internal name/alias (required)
 - prefix (optional), used for all actions within the controller
 - default action (optional)
 
`Action` works pretty much the same as a basic `Route` annotation from the framework annotated routes. It defines:
- route pattern (required)
- name (optional, `controller.action` will be used as a fallback)
- methods (optional)
- defaults (optional)
- group (not used for now)
- middleware (optional)

Example:
```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Annotation\Action;
use Spiral\Keeper\Annotation\Controller;
use Spiral\Views\ViewsInterface;

/**
 * @Controller(namespace="admin", name="user", prefix="/users")
 */
class User
{
    /**
     * @Action(route="/create", name="createUser", methods="GET")
     * @param ViewsInterface $views
     * @return string
     */
    public function create(ViewsInterface $views): string
    {
        return $views->render('admin:users/create');
    }
}
``` 
The next routes will be created (see `php app.php route:list` output):
```
+--------------------+--------+---------------------+--------------+
| Name:              | Verbs: | Pattern:            | Target:      |
+--------------------+--------+---------------------+--------------+
| admin[createUser]  | GET    | /admin/users/create | user->create |
| admin[user.create] | GET    | /admin/users/create | user->create |
+--------------------+--------+---------------------+--------------+
```
> Routes will be duplicated by a generated name like `controller.method`.

## Namespace
You can either call a route by its name directly or using a `RouteBuilder`
```php
/**
 * @var \Spiral\Router\RouterInterface     $router 
 * @var \Spiral\Keeper\Helper\RouteBuilder $routeBuilder
 */
$router->uri('admin[user.create]');
$routeBuilder->uri('admin', 'user.create');
```
The output will be the same: `/admin/users/new`
> `RouteBuilder` takes care about namespace isolation pattern.

## Defaults
To enable default controller routing it should be added explicitly to the config.
Either via `KeeperBootloader::DEFAULT_CONTROLLER` value or via config file:
```php
return [
     'routeDefaults' => ['controller' => 'App\Admin\Controller'],
];
```

Default controller action can be also defined either in the config:
```php
return [
     'routeDefaults' => ['controller' => 'App\Admin\Controller', 'action' => 'list'],
];
```

or via `defaultAction` property in `@Controller` annotation.
> For default controller `index` method will be used as a fallback if no `defaultAction` provided in the default controller annotation.
