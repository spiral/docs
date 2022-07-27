# HTTP - Request Lifecycle

Unlike most of the PHP frameworks, the HTTP requests begin outside of the application in the application server

- [RoadRunner](https://roadrunner.dev).

![Screenshot_31](https://user-images.githubusercontent.com/796136/67088146-1bd39c80-f1ad-11e9-9d5e-6b2499654395.png)

> The Response comes the way in a backward direction.

## PSR

The Spiral Framework based on a set of standards that make it compatible with other frameworks, routers, middleware,
etc. You can read more about PSR standards used here:

- [PSR-7: HTTP message interfaces](https://www.php-fig.org/psr/psr-7/)
- [PSR-15: HTTP Server Request Handlers](https://www.php-fig.org/psr/psr-15/)
- [PSR-17: HTTP Factories](https://www.php-fig.org/psr/psr-17/)

## Flow Description

The user request comes to the RoadRunner application server. The server will pass it through the number of middleware
layers, some of which used to enable web-socket broadcasting, serve static files
or [implement domain-specific logic](/http/golang.md).

Once all of the middleware processing is complete, the `net/http` request will be converted into `PSR-7` format and
passed to the first available PHP worker.

The worker will handle this request using the `spiral/http` extension and `Spiral\Http\Http` core. The core will pass
the PSR-7 request object (`ServerRequestInterface`) through a set of PSR-15 compatible middleware.

Once all of the middleware processing is complete, the framework will create an [IoC scope](/framework/scopes.md) for
the request object. Such an approach allows you to use PSR-7 request as a classic global object, while technically, it
only exists during the user request.

The request is passed into the PSR-15 handler of your choice (by default `spiral/router`). The handler must generate the
response which will be sent back to the user through all middleware layers.

> Spiral Router provides the ability to associate custom middleware set with each route.

## Invoke HTTP Manually

It is possible to invoke http core within the application. It can be useful for testing purposes or if you want to run
Spiral from inside other frameworks. Obtain the instance of `Spiral\Http\Http` to do that:

```php
namespace App\Controller;

use Nyholm\Psr7\Uri;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Http\Http;

class HomeController implements SingletonInterface
{
    private $http;

    public function __construct(Http $http)
    {
        $this->http = $http;
    }

    public function index(ServerRequestInterface $request): string
    {
        $response = $this->http->handle(
            $request->withUri(new Uri('/home/other')) // modify Uri of current request
        );

        return (string)$response->getBody(); // "other"
    }

    public function other(): string
    {
        return 'other';
    }
}
```

> The IoC scopes can be nested, so all the functionality will work properly. However, be aware that not all extensions
> will allow nesting (you are not allowed to create nested sessions yet).
