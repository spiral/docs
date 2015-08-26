# Error Handling and Isolation
HttpDispatcher provides ability to handle exception and convert them into valid response automatically inside middleware pipeline, this provides ability to perform some
request without being worrying about exception happen inside perform method, i'm calling this ability "error isolation" and it can be very useful in many scenarious.

## Turn isolation on/off
You can turn isolation for primary request by editing http configuration file (option "isolate") or by setting parameter "isolated" for request you going to send to perform method.

```php
$this->http->perform($request->withAttribute('isolate', false));
```

When exception happens in some code executed by middleware or pipeline target and isolation is disabled exception will be pushed up into perform method and eventually handled by general `handleException` method defined in Core.

```php
try {
    $this->http->perform($request->withAttribute('isolate', false));
} catch (Exception $e) {
    //Failed to execute
}

```

Side effect of turning isolation off is that no middleware will be able to complete it's work due chain will be broken by exception.

## Client Exceptions
HttpDispatcher defines set of "soft" client exception you can use in your code to force some HTTP error to happen. For example:

```php
public function index()
{
     throw new ClientException(ClientException::NOT_FOUND);
}
```

Exceptions like that will be handled by HttpDisaptcher and converted into response with appropriate status code. If you want to define custom view for your error,
edit http configuration file to link error code to view name:

```php
    'httpErrors'   => [
        400 => 'spiral:http/badRequest',
        403 => 'spiral:http/forbidden',
        404 => 'spiral:http/notFound',
        500 => 'spiral:http/serverError',
    ]
```

There is few exceptions predefined by generic scenarious:

| Code | Exception class       |
| ---  | ---                   |
| 400  | BadRequestException   |
| 401  | UnauthorizedException |
| 403  | ForbiddenException    |
| 404  | NotFoundException     |
| 500  | ServerErrorException  |

## Exposing errors to client
Exceptions which are not instances of `Spiral\Http\Exception\ClientException` will be treated as server errors, handled via `SnapshotInterface` and sent to client as exception backtrace. You can disable sending trace to client and replace it with generic 500 error via editing "exposeErrors" flag in your http configuration.