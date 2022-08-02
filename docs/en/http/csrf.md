# HTTP - CSRF protection

The default Web bundle includes CSRF protection middleware. To install it in alternative bundles:

To activate the extension add `Spiral\Bootloader\Http\CsrfBootloader` to list of App bootloaders:

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

public function boot(RouterInterface $router): void
{
    $route = new Route('/', new Target\Action(HomeController::class, 'index'));

    $router->setRoute(
        'index',
        $route->withMiddleware(CsrfFirewall::class)
    );
}
```

> **Note**
> To protect against all the HTTP verbs use `Spiral\Csrf\Middleware\StrictCsrfFirewall`.

## Usage

Once the protection is activated, you must sign every request with the token available via PSR-7 attribute `csrfToken`.

To receive this token in the controller or view:

```php
public function index(ServerRequestInterface $request): void
{
    $csrfToken = $request->getAttribute('csrfToken');
}
``` 

Every `POST`/`PUT`/`DELETE` request from the user must include this token as POST parameter `csrf-token` or
header `X-CSRF-Token`. Users will receive `412 Bad CSRF Token` if a token is missing or not set.

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function index(ServerRequestInterface $request): string
{
    $form = '
        <form method="post">
          <input type="hidden" name="csrf-token" value="{csrfToken}"/>
          <input type="text" name="value"/>
          <input type="submit"/>
        </form>
    ';

    $form = \str_replace(
        '{csrfToken}',
        $request->getAttribute('csrfToken'),
        $form
    );

    return $form;
}
```

## Activate Globally

To activate CSRF protection globally register `Spiral\Csrf\Middleware\CsrfFirewall`
or `Spiral\Csrf\Middleware\StrictCsrfFirewall` via `Spiral\Bootloader\Http\HttpBootloader`:

```php
use Spiral\Csrf\Middleware\CsrfFirewall;
use Spiral\Bootloader\Http\HttpBootloader;

// ...

public function boot(HttpBootloader $http): void
{
    $http->addMiddleware(CsrfFirewall::class);
}
```
