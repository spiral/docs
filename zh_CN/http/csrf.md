# HTTP - CSRF 防护

The default Web bundle includes CSRF protection middleware. To install it in alternative bundles:
默认的 Web 应用模板包含 CSRF 防护中间件。但在非默认 Web 应用中可能需要先安装它：

```bash
$ composer require spiral/csrf
```

然后把它的引导程序添加到 App.php 中来启用：

```php
[
    //...
    Spiral\Bootloader\Http\CsrfBootloader::class,
]
```

CSRF 扩展会启用 `Spiral\Csrf\Middleware\CsrfMiddleware` 中间件，以确保每个用户有唯一的 token.

## 启用防火墙

CSRF 扩展提供了两个中间件，这两个中间件为引用提供全局或路由级别的防护。如果要对除了 `GET`, `HEAD`, `OPTIONS` 以外的所有请求提供防护，需要使用 `Spiral\Csrf\Middleware\CsrfFirewall`:

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

> 提示，如果要对所有的 HTTP 动词提供防护，需要使用 `Spiral\Csrf\Middleware\StrictCsrfFirewall`.

## 使用

一旦启用防护，每个请求都必须通过 PSR-7 属性 `csrfToken` 来提供合法的 token.

在控制器或者视图中，可以读取请求中的 token:

```php
public function index(ServerRequestInterface $request)
{
    $csrfToken = $request->getAttribute('csrfToken');
}
```

用户发出的每个 `POST`, `PUT` 或者 `DELETE` 请求必须在 POST 参数 `csrf-token` 或者 `X-CSRF-Token` 头信息中携带合法的 token.

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

## 全局启用

如果需要全局启用 CSRF 防护，可以通过 `HttpBootloader` 引导程序注册 `Spiral\Csrf\Middleware\CsrfFirewall` 或者 `Spiral\Csrf\Middleware\StrictCsrfFirewall`:

```php
use Spiral\Csrf\Middleware\CsrfFirewall;

// ...

public function boot(HttpBootloader $http)
{
    $http->addMiddleware(CsrfFirewall::class);
}
```
