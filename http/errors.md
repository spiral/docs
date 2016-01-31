# Error Handling and Isolation
HttDispather does not provide direct functionality for error conversion, however it supplies spiral application with `Spiral\Http\Middlewares\ExceptionWrapper` middleware used to handle and represent client exception (not found, forbidden and etc) using set of defined view files. 

> Please read about [application level error handling](/application/errors.md) to understand how error exceptions a handled.

## Client Exceptions
HttpDispatcher defines a set of "soft" client exceptions you can use in your code to force some HTTP error to happen. For example:

```php
protected function indexAction()
{
     throw new ClientException(ClientException::NOT_FOUND);
}
```

Exceptions like that will be handled by `ExceptionWrapper`, rendered using view component and written into response with an appropriate status code. If you want to define a custom view for your error, edit the http configuration file to link error code to view name:

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
