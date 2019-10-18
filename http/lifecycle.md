# HTTP - Request Lifecycle
Unlike most of PHP frameworks the HTTP requests begins outside of the application in the application server - [RoadRunner](https://roadrunner.dev).

![Screenshot_31](https://user-images.githubusercontent.com/796136/67088146-1bd39c80-f1ad-11e9-9d5e-6b2499654395.png)

> The Response comes the way in backward direction.

## PSR
The Spiral Framework is based on a set of standards which make it compatible with other frameworks, routers, middleware 
and etc. You can read more about PSR standards used here:

- [PSR-7: HTTP message interfaces](https://www.php-fig.org/psr/psr-7/)
- [PSR-15: HTTP Server Request Handlers](https://www.php-fig.org/psr/psr-15/)
- [PSR-17: HTTP Factories](https://www.php-fig.org/psr/psr-17/)

## Flow Description
The user request comes to the RoadRunner application server. The server will pass it thought the number of middleware
layers, some of which used to enable web-socket broadcasting, serve static files or [implement domain specific logic](/http/golang.md).

Once all the middleware processing complete, the `net/http` request will be converted into `PSR-7` format and passed
to the first available PHP worker. 

The worker will handle this request using the `spiral/http` extension and `Spiral\Http\Http` core. The core will pass 
PSR-7 request object (`ServerRequestInterface`) though the set of PSR-15 compatible middleware.
 
Once all the middleware processing is complete, the framework with create an [IoC scope](/framework/scopes.md) for the request object.
Such approach allows you to use PSR-7 request as classic global object, while technically, it only exists during the user request.

The request will be passed into PSR-15 handler on your chaise (by default `spiral/router`). The handler must generate the response
which will be send back to user should all middleware layers.

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

    public function index(ServerRequestInterface $request)
    {
        $response = $this->http->handle(
            $request->withUri(new Uri('/home/other'))
        );

        assert((string)$response->getBody() == 'other');
    }

    public function other()
    {
        echo 'other';
    }
}
```

> The IoC scopes can be nested, so all the functionality will work properly. However, aware that not all extensions will
> allow nesting (you are not allowed create nested sessions yet).
