# Http Error Handling
HttpDispatcher does not provide direct functionality for a error conversion, though it supplies spiral application with `Spiral\Http\Middlewares\ExceptionWrapper` middleware used to handle and represent client exception (not found, forbidden and etc) by rendering appropriate view. 

> Please read about [application level error handling](/v1.0.0/debug/errors.md) to understand how error exceptions are handled.

## Exposing errors
HttpDispatcher provides ability to expose error exceptions to the client in a form of snapshots (see error handling), by default exception rendering will be dedicated to `SnaphotInterface` which will show stack stace and environment details.

Once application moved to production consider disabling debug mode which will force HttpDispatcher replace snapshot response with regular 500 error. Check `.env` file to locate option to enable and disable debug mode.

> By default HttpDispatcher is capable to expose exception in html and json formats, if you need more specific error responses use using external middlewares such as Whoops.

## Client Exceptions
HttpDispatcher defines a set of "soft" client exceptions you can use in your code to force some HTTP error to happen. For example:

```php
protected function indexAction()
{
     throw new ClientException(ClientException::NOT_FOUND);
}
```

Exceptions like that will be handled by `ExceptionWrapper`, and rendered using view component with associated status code. To define a custom view for your error, edit the http configuration file to link error code to view name:

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

```php
protected function indexAction()
{
     throw new ForbiddenException();
}
```

> You can also define your own exception handler middleware or custom exception classes.

## Custom Errors
You can also automatically convert your domains into http errors by creating a middleware:

```php
public function __invoke(Request $request, Response $response, callable $next)
{
    try {
        return $next($request, $response);
    } catch (\DomainException $e) {
        throw new ServerErrorException($e->getMessage());
    }
}
```