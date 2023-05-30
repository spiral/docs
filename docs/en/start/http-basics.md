# Getting started — First HTTP controller

If you're using the Spiral for a PHP project and ready to dive into setting up your first controller, here are 
the basic steps to follow:

## Create a controller

First things first, you'll need to create a controller. In Spiral, a controller is a class that defines
the behavior of your application for a specific set of routes. It's responsible for handling
incoming [PSR-7](https://www.php-fig.org/psr/psr-7/) compatible requests, processing data, and then returning a response
to the client.

To create your first controller effortlessly, use the scaffolding command:

```terminal
php app.php create:controller CurrentDate
```

After executing this command, the following output will confirm the successful creation:

```output
Declaration of '[32mCurrentDateController[39m' has been successfully written into '[33mapp/src/Endpoint/Web/CurrentDateController.php[39m'.
```

Now, let's inject some logic into our freshly created controller.

Here's an example of a controller that returns the current date and time:

```php app/src/Endpoint/Web/CurrentDateController.php
namespace App\Endpoint\Web;

final class CurrentDateController 
{
    public function show(): string
    {
        return \date('Y-m-d H:i:s');
    }
}
```

The next step involves associating a route with your controller.

## Create a route

:::: tabs

::: tab Using Attributes

Spiral simplifies route definition in your application by utilizing PHP attributes. You just need to add 
the `#[Route]` attribute to the controller's method, as shown below:

```php app/src/Endpoint/Web/CurrentDateController.php
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

For developers seeking a convenient and organized approach to defining application routes, Spiral offers 
the `defineRoutes` method in the `App\Application\Bootloader\RoutesBootloader` class.

Here's an example of how to define a route that will be handled by our controller:

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

To view the list of routes, use the following command:

```terminal
php app.php route:list
```

You should observe your `current-date` route within the displayed list:

```output
+--------------+--------+----------+------------------------------------------------+--------+
|[32m Name:        [39m|[32m Verbs: [39m|[32m Pattern: [39m|[32m Target:                                        [39m|[32m Group: [39m|
+--------------+--------+----------+------------------------------------------------+--------+
| current-date | [32mGET[39m    | /date    | App\Endpoint\Web\CurrentDateController->show | web    |
+--------------+--------+----------+------------------------------------------------+--------+
```

## Test your controller

Once your controller is all set, it's time to test it out by running the RoadRunner server using the command:

```terminal
./rr serve
```

Now you can test your controller by visiting the route in your browser. Just open the following URL in your
browser: http://127.0.0.1/date

<br><br>

**That's it! You've successfully set up your first controller in Spiral.**

<hr>

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Routing](../http/routing.md)
* [Annotated Routing](../http/annotated-routes.md)
* [Middleware](../http/middleware.md)
* [Error Pages](../http/errors.md)
* [Custom HTTP handler](../cookbook/psr-15.md)
* [Scaffolding](../basics/scaffolding.md)