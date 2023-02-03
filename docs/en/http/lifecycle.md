# HTTP — Request Lifecycle

Unlike most of the PHP frameworks, the HTTP requests begin outside of the application in the application server

- [RoadRunner](https://roadrunner.dev).

![Request Lifecycle](https://user-images.githubusercontent.com/773481/181182150-8cc2b6c4-2b50-4e85-afd7-e5c2d1c98b2c.png)

> **Note**
> The Response makes its way in a backward direction.

## PSR

Spiral is based on a set of standards that make it compatible with other frameworks, routers, middleware, etc. You can 
read more about PSR standards used here:

- [PSR-7: HTTP message interfaces](https://www.php-fig.org/psr/psr-7/)
- [PSR-15: HTTP Server Request Handlers](https://www.php-fig.org/psr/psr-15/)
- [PSR-17: HTTP Factories](https://www.php-fig.org/psr/psr-17/)

## Flow Description

A user request comes to the RoadRunner application server. The server will pass it through a number of middleware
layers, some of which are used to enable serve static files or implement domain-specific logic.

Once all of the middleware processing is complete, the `net/http` request will be converted into `PSR-7` format and
passed to the first available PHP worker.

The worker will handle this request using the `spiral/http` component and `Spiral\Http\Http` core. The core will pass
the PSR-7 request object (`Psr\Http\Message\ServerRequestInterface`) through a set of PSR-15 compatible middleware.

Once all of the middleware processing is complete, the framework will create an [IoC scope](../framework/scopes.md) for
the request object. Such an approach allows you to use PSR-7 request as a classic global object, while technically, it
only exists during the user request.

The request is passed into the PSR-15 handler of your choice (by default `spiral/router`). The handler must generate a
response which will be sent back to the user through all the middleware layers.

> **Note**
> Spiral Router provides an ability to associate custom middleware set with each route.

## Invoke HTTP Manually

It is possible to invoke http core within the application. It can be useful for testing purposes or if you want to run
Spiral from inside other frameworks. Obtain the instance of `Spiral\Http\Http` to do that:

```php app/src/Interface/Controller/HomeController.php
namespace App\Interface\Controller;

use Nyholm\Psr7\Uri;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Http\Http;

class HomeController implements SingletonInterface
{
    public function __construct(
        private readonly Http $http
    ) {
    }

    public function index(ServerRequestInterface $request): string
    {
        $response = $this->http->handle(
            $request->withUri(new Uri('/home/other')) // modify Uri of current request
        );

        return (string) $response->getBody(); // "other"
    }

    public function other(): string
    {
        return 'other';
    }
}
```

> **Note**
> The IoC scopes can be nested, so all the functionality will work properly. However, be aware that not all extensions
> will allow nesting (you are not allowed to create nested sessions yet).

## Events

| Event                             | Description                                                       |
|-----------------------------------|-------------------------------------------------------------------|
| Spiral\Http\Event\RequestReceived | The Event will be fired when the request is received.             |
| Spiral\Http\Event\RequestHandled  | The Event will be fired when the request is successfully handled. |

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
