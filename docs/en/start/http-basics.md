# Getting started — First HTTP controller

If you're using the Spiral Framework for a PHP project and you're ready to set up your first controller, here are the
basic steps you'll need to follow:

## Create a controller

First things first, you'll need to create a controller. In the Spiral Framework, a controller is a class that defines
the behavior of your application for a specific set of routes. It's responsible for handling
incoming [PSR-7](https://www.php-fig.org/psr/psr-7/) compatible requests, processing data, and then returning a response
to the client.

Here's an example of a simple controller that returns the current date and time:

```php app/src/Interface/Http/CurrentDateController.php
declare(strict_types=1);

namespace App\Interface\Http;

final class CurrentDateController 
{
    public function show(): string
    {
        return \date('Y-m-d H:i:s');
    }
}
```

The next step is to associate a route with your controller.

## Create a route

:::: tabs

::: tab Using Attributes

Spiral Framework makes it easy to define your application's
routes by using PHP attributes. All you have to do is add the `#[Route]` attribute to the controller's method like so:

```php app/src/Interface/Http/CurrentDateController.php
use Spiral\Router\Annotation\Route;

// ...

#[Route(route: '/date', name: 'current-date', methods: 'GET')]
public function show(): string
{
    return \date('Y-m-d H:i:s');
}
```

:::

::: tab Using RoutingConfigurator

Spiral Framework offers a convenient and organized way for developers to define their application's routes using the
`defineRoutes` method of the `App\Application\Bootloader\RoutesBootloader` class.

Here is an example of how to define a route that will handle our controller:

```php app/src/Application/Bootloader/RoutesBootloader.php
final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->add(name: 'current-date', pattern: '/date')
            ->action(controller: CurrentDateController::class, action: 'show');
    }
}
```

:::

::::

To see the list of routes, use the `php app.php route:list` command in your terminal.

```bash
php app.php route:list
```

You should see your `current-date` route in the list:

```output
+--------------+--------+----------+------------------------------------------------+--------+
|[32m Name:        [39m|[32m Verbs: [39m|[32m Pattern: [39m|[32m Target:                                        [39m|[32m Group: [39m|
+--------------+--------+----------+------------------------------------------------+--------+
| current-date | [32mGET[39m    | /date    | App\Interface\Http\CurrentDateController->show | web    |
| data-time    | [32mGET[39m    | date     | App\Interface\Http\CurrentDateController->show | web    |
+--------------+--------+----------+------------------------------------------------+--------+
```

## Test your controller

Once you have your controller set up, you'll need to test it by running the RoadRunner server with the command

```bash
./rr serve
```

Now you can test your controller by visiting the route in your browser. Just open the following URL in your
browser: http://127.0.0.1/date

<br><br>

**That's it! You've successfully set up your first controller in the Spiral Framework.**

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Routing](../http/routing.md)
* [Annotated Routing](../http/annotated-routes.md)
* [Middleware](../http/middleware.md)
* [Error Pages](../http/errors.md)
* [Custom HTTP handler](../cookbook/psr-15.md)