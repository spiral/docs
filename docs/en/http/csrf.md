# HTTP - CSRF protection

The default Web bundle includes CSRF protection middleware. To install it in alternative bundles:

```bash
composer require spiral/csrf
```

> Please note that the spiral/framework >= 2.6 already includes this component.

To activate the extension:

```php
[
    //...
    Spiral\Bootloader\Http\CsrfBootloader::class,
]
```

The extension will activate `Spiral\Csrf\Middleware\CsrfMiddleware` to issue a unique token for every user.

## Enable Firewall

The extension provides two middleware, which activates the protection on your routes or globally. To protect all the
requests except for `GET`, `HEAD`, `OPTIONS `use `Spiral\Csrf\Middleware\CsrfFirewall`:

```php
use Spiral\Csrf\Middleware\CsrfFirewall;

// ...

public function boot(RouterInterface $router)
{
    $route = new Route('/', new Target\Action(HomeController::class, 'index'));

    $router->setRoute(
        'index',
        $route->withMiddleware(CsrfFirewall::class)
    );
}
```

> To protect against all the HTTP verbs use `Spiral\Csrf\Middleware\StrictCsrfFirewall`.

## Usage

Once the protection is activated, you must sign every request with the token available via PSR-7 attribute `csrfToken`.

To receive this token in the controller or view:

```php
public function index(ServerRequestInterface $request)
{
    $csrfToken = $request->getAttribute('csrfToken');
}
``` 

Every `POST`/`PUT`/`DELETE` request from the user must include this token as POST parameter `csrf-token` or
header `X-CSRF-Token`. Users will receive `412 Bad CSRF Token` if a token is missing or not set.

```php
public function index(ServerRequestInterface $request)
{
    $form = '
        <form method="post">
          <input type="hidden" name="csrf-token" value="{csrfToken}"/>
          <input type="text" name="value"/>
          <input type="submit"/>
        </form>
    ';

    $form = str_replace(
        '{csrfToken}',
        $request->getAttribute('csrfToken'),
        $form
    );

    return $form;
}
```

## Activate Globally

To activate CSRF protection globally register `Spiral\Csrf\Middleware\CsrfFirewall`
or `Spiral\Csrf\Middleware\StrictCsrfFirewall` via `HttpBootloader`:

```php
use Spiral\Csrf\Middleware\CsrfFirewall;

// ...

public function boot(HttpBootloader $http)
{
    $http->addMiddleware(CsrfFirewall::class);
}
```
