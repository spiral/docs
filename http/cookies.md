# Web and HTTP - Cookies
Default application skeleton enables cookie integration by default.
 
If you need to enable cookies it in alternative bundle require composer package `spiral/cookies` and add 
bootloader `Spiral\Bootloader\Bootloader\Http\CookiesBootloader` into your app.

## Cookie Manager
The easiest way to manage cookies in Spiral is to obtain instance of `Spiral\Cookies\CookieManager`. This instance can be stored inside
singleton services and controllers and provide access to active request scope.

```php
public function index(CookieManager $cookies)
{
    dump($cookies->getAll());
    $cookies->set('name', 'value'); // read about more options down below
}
```

If you use `spiral/prototype` extension you can also access cookie managed using `cookies` prototype property:

```php
use PrototypeTrait;

public function index()
{
    dump($this->cookies->getAll());
    $this->cookies->set('name', 'value'); // read about more options down below
}
```

Read more about low-level cookie managent down below.

## Read Cookie
By default, the framework will encrypt and decrypt all cookies values using ENV key `ENCRYPTER_KEY`. Changing this value will
automatically invalidate all cookie values set for all users.

> You can also disable encryption for performance reasons or use alternative HMAC signature method (see below). 

Cookie component will decrypt all values and update the request object, so you can get all cookies values using default PSR-7 `ServerRequestInterface`:

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function index(ServerRequestInterface $request)
{
    dump($request->getCookieParams());
}
```

Alternatively, you can read cookie values using `Spiral\Http\Request\InputManager` which automatically resolves the request
scope and can be stored as a singleton:

```php
class HomeController
{
    private $input;

    public function __construct(InputManager $input)
    {
        $this->input = $input;
    }

    public function index()
    {
        dump($this->input->cookies->all());
    }
}
```

> You can also request cookie value in request filters.

Note, if the cookie value is invalid and can't be decrypted it will be set to NULL and not available to the application. 

## Write Cookie
Since all of the cookie values must be encrypted or signed you must use the proper way to write them. 
Use context-specific object `Spiral\Cookies\CookieQuery`.

```php
public function index(CookieQuery $cookies)
{
    $cookies->set('name', 'value');
}
```

The method accepts the following arguments in the same order: 

Parameter | Type | Description
--- | --- | ---
name | string | The name of the cookie.
value |  string | The value of the cookie. This value is stored on the client's computer; do not store sensitive information.
lifetime | int | Cookie lifetime. This value specified in seconds and declares a period of time in which cookie will expire relatively to current time() value.
path | string | The path on the server in which the cookie will be available on. If set to '/', the cookie will be available within the entire domain. If set to '/foo/', the cookie will only be available within the /foo/ directory and all sub-directories such as /foo/bar/ of the domain. The default value is the current directory that the cookie is being set in.
domain | string | The domain that the cookie is available. To make the cookie available on all subdomains of example.com then you'd set it to '.example.com'. The . is not required but makes it compatible with more browsers. Setting it to www.example.com will make the cookie only available in the www subdomain. Refer to tail matching in the spec for details.
secure | bool | Indicates that the cookie should only be transmitted over a secure HTTPS connection from the client. When set to true, the cookie will only be set if a secure connection exists. On the server-side, it's on the programmer to send this kind of cookie only on a secure connection (e.g. with respect to $_SERVER["HTTPS"]).
httpOnly | bool | When true the cookie will be made accessible only through the HTTP protocol. This means that the cookie won't be accessible by scripting languages, such as JavaScript. This setting can effectively help to reduce identity theft through XSS attacks (although it is not supported by all browsers).

## Usage with Singletons
You can not use `CookieQuery` as `__construct` argument. Queue is only available withing IoC context of CookieMiddleware
and must be requested from container directly (use use method injection as showed above):

```php
$container->get(CookieQuery::class)->set($name, $value);
```

> The best place to use `CookieQuery` is controller methods.

If you already have access to `ServerRequestInterface` use can also use attribute `cookieQueue`:

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function index(ServerRequestInterface $request)
{
    $request->getAttribute('cookieQueue')->set('name', 'value');
}
```

## Set Cookie Manually
You can always write cookie header manually by invoking `withAddedHeader` of `Psr\Http\Message\ResponseInterface`:

```php
return $response->withAddedHeader('Set-Cookie', 'name=value');
```

> Make sure to add cookie to whitelist, otherwise CookieMiddleware won't let it pass.

## Configuration
You can configure `CookieQueue` behaviour using `Spiral\Bootloader\Http\CookiesBootloader`.

To whitelist cookie (disable protection) in your bootloader:

```php
public function boot(CookiesBootloader $cookies)
{
    $cookies->whitelistCookie('CustomCookie');
}
```

To perform deeper configuration on cookie component create config file `cookies.php` in `app/config` directory:

```php
<?php
declare(strict_types=1);

use Spiral\Cookies\Config\CookiesConfig;

return [
    // by default all cookies will be set as .domain.com
    'domain'   => '.%s',
    
    // protection method
    'method'   => CookiesConfig::COOKIE_ENCRYPT,
    
    // whitelisted cookies (no ecnrypt/descrypt)
    'excluded' => ['PHPSESSID', 'csrf-token'] 
];
```
