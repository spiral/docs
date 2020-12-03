# Sitemap
Sitemap is a sub-module for building menu navigation and breadcrumbs. Consists of 4 types of nodes:
- link and view represent a page
- segment and group are containers for nested elements.
 
Sitemaps can be declared via annotations or directly using `Sitemap` module.

## Direct sitemap declaration
Create a custom bootloader with `Sitemap` injection and add it to `KeeperBootloader` dependency:
```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader;

class KeeperBootloader extends Bootloader\KeeperBootloader
{
    protected const LOAD = [
        Bootloader\SitemapBootloader::class,
        NavigationBootloader::class,
    ];
}
```
Now you're ready to declare sitemap nodes:
```php
<?php

declare(strict_types=1);

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends Bootloader
{
    public function boot(Sitemap $sitemap): void
    {
        $sitemap->link('dashboard.index', 'Dashboard', ['icon' => 'home']);
        // ...
    }
}
```
> In that case you will have difficulties with sitemap annotations. This approach fits when you're not going to use annotations.

We recommend extending the `SitemapBootloader` and use the `declareSitemap()` method to declare the structure:
```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        $sitemap->link('dashboard.index', 'Dashboard', ['icon' => 'home']);

        $group = $sitemap->group('users', 'Users and Groups', ['icon' => '...']);
        if ($group !== null) {
            $users = $group->link('users.index', 'Users', ['icon' => '...']);
            if ($users !== null) {
                $users->view('users.create', 'Create User');
                $users->view('users.edit', 'Edit User');
            }
            $group->link('groups.index', 'Groups', ['icon' => '...']);
        }
        // ...
    }
}
```
> In that case sitemaps module will be able to sync with sitemap annotations.

## Annotations
`Group` and `Segment` annotations can be applied for classes, while `Link` and `View` for class methods.
`Link` and `View` will work only if a method also has `Action` annotation.
```php
<?php

declare(strict_types=1);

namespace App\Controller\Keeper;

use Spiral\Keeper\Annotation as Keeper;
use Spiral\Views\ViewsInterface;

/**
 * @Keeper\Controller(name="users", prefix="/users")
 * @Keeper\Sitemap\Group(name="users", title="Users and Groups")
 */
class UsersController
{
    /**
     * @Keeper\Action(route="", methods="GET")
     * @Keeper\Sitemap\Link(title="Users", options={"icon": "user-friends"})
     * @param ViewsInterface $views 
     * @return string
     */
    public function index(ViewsInterface $views): string
    {
        return $views->render('keeper:users/list');
    }

    /**
     * @Keeper\Action(route="/create", methods="GET")
     * @Keeper\Sitemap\View(title="Add new user")
     * @param ViewsInterface $views 
     * @return string
     */
    public function new(ViewsInterface $views): string
    {
        return $views->render('keeper:users/create');
    }
    // ...
}
```
It is convenient to declare groups and segments in the bootloader with full details and use only the names in annotations (in case of multiple controllers under a singe group). See example below:
```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        $sitemap->group('users', 'Users and Groups', ['icon' => '...']);
        // ...
    }
}
```
```php
<?php

declare(strict_types=1);

namespace App\Controller\Keeper;

use Spiral\Keeper\Annotation as Keeper;
use Spiral\Views\ViewsInterface;

/**
 * @Keeper\Controller(name="users", prefix="/users")
 * @Keeper\Sitemap\Group(name="users")
 */
class UsersController
{
    // ...
}
```

## Visibility
By default, all nodes are available for the user, `withVisibleNodes()` allows hiding forbidden nodes.
If any node is forbidden, it will be removed from the tree with all its children.
Also, passing a `$targeNode` will mark all active nodes if match found, so it will allow you to use breadcrumbs.

Permissions are taken from `@GuardNamespace` and `@Guarded` annotation: `<guard Namespace (or controller name)>.<guarded permission (or method name)>`.
Note that keeper namespace isn't used here automatically because these annotations come from external module.

Example with `@GuardNamespace` annotation:
```php
/**
 * @Controller(name="with", prefix="/with", namespace="with")
 * @GuardNamespace(namespace="withNamespace")
 */
class WithNamespaceController
{
    /**
     * @Link(title="A")
     * ...
     */
    public function a(): void
    {
        // permission is "withNamespace.a"
    }

    /**
     * @Link(title="B")
     * @Guarded(permission="permission")
     * ...
     */
    public function b(): void
    {
        // permission is "withNamespace.permission"
    }
}
```
Example without `@GuardNamespace` annotation:
```php
/**
 * @Controller(name="without", prefix="/without", namespace="without")
 */
class WithoutNamespaceController
{
    /**
     * @Link(title="A")
     * ...
     */
    public function a(): void
    {
        // permission is "without.a"
    }

    /**
     * @Link(title="B")
     * @Guarded(permission="permission")
     * ...
     */
    public function b(): void
    {
        // permission is "without.permission"
    }
}
```

## Sorting
By default, sitemap sorts nodes in the way they are found in the classes or declared in the `Sitemap` directly.
You can use `position` (float) attribute in the annotations or `position` option in the direct declaration options.
The nodes will be sorted in ascending order (or in the appearance order if the position matches).
```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        $sitemap->group('users', 'Users and Groups', ['position' => -0.8]);
    }
}
```
```php
<?php

declare(strict_types=1);

namespace App\Controller\Keeper;

use Spiral\Keeper\Annotation as Keeper;
use Spiral\Views\ViewsInterface;

/**
 * @Keeper\Controller(name="users", prefix="/users")
 * @Keeper\Sitemap\Group(name="users")
 */
class UsersController
{
    /**
     * @Keeper\Action(route="", methods="GET")
     * @Keeper\Sitemap\Link(title="Users", position=2.7)
     * @param ViewsInterface $views 
     * @return string
     */
    public function index(ViewsInterface $views): string
    {
        return $views->render('keeper:users/list');
    }
}
```

## Nesting
Any node may be a child of another parent node. Use `parent` attribute in annotations.
If you are referring to the method within the same controller you can use only the name, otherwise use `controller.method` notation:
```php
<?php

declare(strict_types=1);

namespace App\Controller\Keeper;

use Spiral\Keeper\Annotation as Keeper;
use Spiral\Views\ViewsInterface;

/**
 * @Keeper\Controller(name="users", prefix="/users")
 */
class UsersController
{
    /**
     * @Keeper\Action(route="", methods="GET")
     * @Keeper\Sitemap\Link(title="Users")
     * @param ViewsInterface $views 
     * @return string
     */
    public function index(ViewsInterface $views): string
    {
        return $views->render('keeper:users/list');
    }

    /**
     * @Keeper\Action(route="/create", methods="GET")
     * @Keeper\Sitemap\View(title="Add new user", parent="index")
     * @param ViewsInterface $views 
     * @return string
     */
    public function new(ViewsInterface $views): string
    {
        return $views->render('keeper:users/create');
    }

    /**
     * @Keeper\Action(route="/create", methods="GET")
     * @Keeper\Sitemap\View(title="Add new user", parent="groups.index")
     * @param ViewsInterface $views 
     * @return string
     */
    public function create(ViewsInterface $views): string
    {
        return $views->render('keeper:users/create');
    }
    // ...
}
```
For the direct sitemap declaration you receive a new node after declaring, so you can use chaining:
```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        // [root]->[link]
        $sitemap->link('dashboard.index', 'Dashboard', ['icon' => 'home']);

        $group = $sitemap->group('users', 'Users and Groups', ['icon' => '...']);
        if ($group !== null) {
            // [root]->[users group]
            $users = $group->link('users.index', 'Users', ['icon' => '...']);
            if ($users !== null) {
                // [root]->[users group]->[view]
                $users->view('users.create', 'Create User');
                $users->view('users.edit', 'Edit User');
            }
            // [root]->[users group]
            $group->link('groups.index', 'Groups', ['icon' => '...']);
        }
        // ...
    }
}
```

## Sidebar
Sidebar doesn't render `View` nodes. Possible nesting hierarchy can be:
- segment
  - group
    - link
  - link
- group
  - link
- link
 
> Any other combinations will be ignored even though sitemap supports any kind of nesting and depth.

You can use `icon` option to add icons to the sidebar.
 

## Breadcrumbs
Opposite to the sidebar, breadcrumbs are able to render all the node types within the various nesting and depth.
```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        $sitemap
            ->link('dashboard.index', 'Level 1')
            ->link('users.index', 'Level 2')
            ->link('groups.index', 'Level 3')
            ->link('logs.index', 'Level 4')
            ->link('system.index', 'Level 5');
    }
}
```

On the `system.index` endpoint breadcrumbs will be (the `[root]` and the current node will be ignored:
```
[dashboard.index]->[users.index]->[groups.index]->[logs.index]
```
