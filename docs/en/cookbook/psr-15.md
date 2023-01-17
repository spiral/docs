# Cookbook — Custom HTTP request handler

The Spiral Framework is a modern, open-source PHP web application framework designed to help developers build scalable
and flexible web applications. It is compliant with several community standards,
including [PSR-7](https://www.php-fig.org/psr/psr-7/) (HTTP message
interfaces), [PSR-15](https://www.php-fig.org/psr/psr-15/) (HTTP server request handlers),
and [PSR-17](https://www.php-fig.org/psr/psr-17/) (HTTP factories).

This means that you can use any request handler implementation you want with PSR-15, which means you can choose the
solution that works best for your application.

By default, the Spiral Framework includes the `Spiral\Router\Router` class which implements the
`Psr\Http\Server\RequestHandlerInterface` interface and handles HTTP requests. However, if desired, developers can
disable the `Spiral\Bootloader\Http\RouterBootloader` and use an alternative request routing solution.

## Fast Route

In this guide, we will show you how to use the [FastRoute](https://github.com/nikic/FastRoute) as an alternative to the
default Spiral Framework router. FastRoute is a fast routing library that allows developers to easily route HTTP
requests to callback functions.

As an example, we will replace the default spiral router with one based
on [FastRoute](https://github.com/nikic/FastRoute). The implementation is provided
by [https://github.com/middlewares/fast-route](https://github.com/middlewares/fast-route).

### Prerequisites

Before you can use FastRoute, you need to install the required packages:

```bash
composer require middlewares/fast-route middlewares/request-handler
```

### Implementing FastRoute

To use FastRoute with the Spiral Framework, you will need to create a custom bootloader to bind the FastRoute
implementation to the HTTP server. The bootloader should bind this implementation to our http server. Simply
declare `Psr\Http\Server\RequestHandlerInterface` in `SINGLETONS` constant.

Here is an example of a FastRoute bootloader:

```php
namespace App\Bootloader;

use FastRoute;
use Middlewares;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Psr\Http\Message\ServerRequestInterface;

final class FastRouteBootloader extends Bootloader
{
    protected const SINGLETONS = [
        RequestHandlerInterface::class => [self::class, 'psr15Handler'],
    ];

    private function defineRoutes(FastRoute\RouteCollector $router): void
    {
        $router->addRoute('GET', '/hello/{name}', static function (ServerRequestInterface $request): string {
            $name = $request->getAttribute('name');

            return \sprintf('Hello %s', $name);
        });
    }

    private function psr15Handler(ResponseFactoryInterface $responseFactory): RequestHandlerInterface
    {
        $dispatcher = FastRoute\simpleDispatcher(function (FastRoute\RouteCollector $r) {
            $this->defineRoutes($r);
        });

        return new Middlewares\Utils\Dispatcher([
            new Middlewares\FastRoute($dispatcher, $responseFactory),
            new Middlewares\RequestHandler(),
        ]);
    }
}
```

In the example above, the `defineRoutes` function is used to define the routes that will be handled by FastRoute. In
this case, we have defined a single route that matches requests to the `/hello/{name}` URL and passes the request to a
callback function that returns response.

Add this Bootloader to your application:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Http\HttpBootloader::class,
    \App\Bootloader\FastRouteBootloader::class,
    // ...
];
```

Once you have implemented the FastRoute bootloader and added it to your Spiral Framework application, you will be able
to use FastRoute to handle HTTP requests.

Congratulations, now you know how to use a custom implementation of the `Psr\Http\Server\RequestHandlerInterface`
interface in the Spiral Framework! By using the FastRoute library or any
other [PSR-15 compliant library]((https://packagist.org/?query=psr-15%20router), you can easily swap out the default
Spiral Framework router and use an alternative solution to handle HTTP requests. Whether you need to use a different 
routing library, implement custom request handling logic, or use components from multiple sources, the Spiral 
Framework's compliance with PSR-15 makes it easy to use a wide variety of request handler implementations in
your applications. **So go ahead and start building the best application you can!**