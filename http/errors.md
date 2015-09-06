# Error Handling and Isolation
HttpDispatcher provides the ability to handle exceptions and convert them into valid responses automatically inside the middleware pipeline, this provides the ability to perform some
requests without worrying about exceptions happening inside the perform method. I'm calling this ability "error isolation" and it can be very useful in many scenarious.

## Turn isolation on/off
You can turn isolation for primary request by editing http configuration file (option "isolate") or by setting parameter "isolated" for request you going to send to perform method.

```php
$this->http->perform($request->withAttribute('isolate', false));
```

When an exception happens in code executed by middleware or pipeline target and isolation is disabled, the exception will be pushed up into perform method and eventually handled by general `handleException` method defined in Core.

```php
try {
    $this->http->perform($request->withAttribute('isolate', false));
} catch (Exception $e) {
    //Failed to execute
}

```

A side effect of turning isolation off is that no middleware will be able to complete its work since the chain will be broken by the exception.

## Client Exceptions
HttpDispatcher defines a set of "soft" client exceptions you can use in your code to force some HTTP error to happen. For example:

```php
public function index()
{
     throw new ClientException(ClientException::NOT_FOUND);
}
```

Exceptions like that will be handled by HttpDisaptcher and converted into a `ExceptionResponse` with an appropriate status code. If you want to define a custom view for your error, edit the http configuration file to link error code to view name:

```php
    'httpErrors'   => [
        400 => 'spiral:http/badRequest',
        403 => 'spiral:http/forbidden',
        404 => 'spiral:http/notFound',
        500 => 'spiral:http/serverError',
    ]
```

There are a few exceptions predefined for generic scenarios:

| Code | Exception class       |
| ---  | ---                   |
| 400  | BadRequestException   |
| 401  | UnauthorizedException |
| 403  | ForbiddenException    |
| 404  | NotFoundException     |
| 500  | ServerErrorException  |

## Handling Client Exception by Middleware
As metnion in previous section every instance of ClientException will be converted into ExceptionResponse, this makes possible to handle such response "exception like" way and redine output. For example we can create CMS middleware which it going to check for page associated with current Uri when endpoint returned NotFound Error.

```php
class CmsMiddleware extends Service implements MiddlewareInterface
{
    /**
     * @param ServerRequestInterface $request
     * @param \Closure               $next Next middleware/target. Always returns ResponseInterface.
     * @return ResponseInterface
     */
    public function __invoke(ServerRequestInterface $request, \Closure $next)
    {
        /**
         * @var ResponseInterface $response
         */
        $response = $next($request);

        if (
            $response instanceof ExceptionResponse
            && $response->getStatusCode() == ClientException::NOT_FOUND
        ) {
            return $this->cmsResponse($request);
        }

        return $response;
    }

    /**
     * @param ServerRequestInterface $request
     * @return HtmlResponse
     */
    private function cmsResponse(ServerRequestInterface $request)
    {
        //Some custom logic
        return new HtmlResponse('This is CMS page.');
    }
}
```

## Exposing errors to client
Exceptions which are not instances of `Spiral\Http\Exception\ClientException` will be treated as server errors, handled via `SnapshotInterface` and sent to client as exception backtrace. You can disable sending trace to client and replace it with generic 500 error by editing the "exposeErrors" flag in your http configuration.
