# HTTP - Cookies

官方的项目框架默认已经启用了 cookie 集成.

如果你需要在其它版本的项目中启用 cookie, 需要通过 composer 安装 `spiral/cookies` 包并把 `Spiral\Bootloader\Http\CookiesBootloader` 引导程序添加到你的应用程序。

## Cookie 管理器

在 Spiral 中管理 cookies 最简单的方法是通过 `Spiral\Cookies\CookieManager` 的实例。该实例可以存储在单例模式的服务和控制器中，并提供对当前活动请求范围内的 cookies 访问。

```php
public function index(CookieManager $cookies)
{
    dump($cookies->getAll());
    $cookies->set('name', 'value'); // 阅读下面的更多选项
}
```

如果你使用了原型开发辅助扩展 `spiral/prototype`, 也可以直接通过原型开发辅助提供的 `cookies` 属性来访问 `CookieManager`:

```php
use PrototypeTrait;

public function index()
{
    dump($this->cookies->getAll());
    $this->cookies->set('name', 'value'); // 阅读下面的更多选项
}
```

阅读下面的文档了解更多更底层的 cookie 管理的内容。

## 读取 Cookie

默认情况下，Spiral 框架使用存储在 `ENCRYPTER_KEY` 环境变量中的密钥来加密和解密所有的 cookie 值。如果这个变量被改变，那么之前对所有用户设置过的 cookie 值都会自动失效。

> 你也可以为了提高性能而禁用 cookie 加密，或者改用其它可选的 HMAC 签名方法（参见下文）。

Cookie 组件会解密所有的 cookie 值并更新请求对象，所以你可以通过默认的 PSR-7 `ServerRequestInterface` 来读取所有的 Cookie 值，不用再考虑解密的问题。

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function index(ServerRequestInterface $request)
{
    dump($request->getCookieParams());
}
```

或者，你也可以通过 `Spiral\Http\Request\InputManager` 来读取 Cookie. 它可以自动解析请求范围，并且可以作为单例引用。

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

> 提示：你还可以在请求过滤器（Request Filter）中读取 cookie 值。

请注意，如果 Cookie 值无效，或者无法解密，那么对应的值会被设置为 NULL，并且对应用不可见。

## 写 Cookie

由于所有的 Cookie 值都必须经过加密或者签名，所以你必须用恰当的方式来进行写 Cookie 的操作。最好是用特定上下文对象 `Spiral\Cookies\CookieQuery`.

```php
public function index(CookieQuery $cookies)
{
    $cookies->set('name', 'value');
}
```

这里的 `set` 方法按照顺序接受如下参数：

| 参数     | 数据类型 | 描述                                                     |
| -------- | -------- |  ------------------------------------------------------ |
| Name     | string   | Cookie 名称                                             |
| Value    | string   | Cookie 值。Cookie 值存储在用户电脑上，请勿存入敏感信息     |
| Lifetime | int      | Cookie 有效期。单位为秒，它声明了该 Cookie 相对于当前时间（time()）的过期时间 |
| Path     | string   | Cookie 的可用路径。如果设置为 "/", 则 Cookie 在整个域名内可用；如果设置为 "/foo/"，则只有同域名下 "/foo/" 及其子目录的 URL 可用。默认值是写 Cookie 时的当前路径。|
| Domain   | string   | Cookie 的作用域。要使 Cookie 在 example.com 的所有子域都可用，可以设置为 ".example.com"，实际上 "." 并不是必须的，但可以增加浏览器兼容性。如果设置为 "www.example.com"，那么 Cookie 只能在 www 子域中可用。详情请参考 Cookie 规范中的尾部匹配。|
| Secure   | bool     | 仅限安全传输。如果设为 true, 表示 Cookie 只会在 https 安全连接中才传输。在服务端，开发者只能在使用安全连接（比如，\$\_SERVER['HTTPS']）的时候才能发送这种 Cookie. |
| HttpOnly | bool     | 仅限 HTTP 协议。如果设为 true, 表示该 Cookie 只能通过 HTTP 协议访问。意味着脚本语言（Javascript）将不能访问该 Cookie. 这个设置一定程度上能有效减少 XSS 攻击（但不是所有浏览器都支持这个选项）。 |

> `Spiral\Cookies\CookieManager` 的 `set` 方法参数同上。

## 单例化使用

`CookieQuery` 不能作为构造函数 `__construct` 的参数。`CookieQueue` 只在 CookieMiddleware 的 IoC 上下文中可用，并且必须直接从容器中请求（如上面的示例那样通过方法注入）。


```php
$container->get(CookieQuery::class)->set($name, $value);
```

> 使用 `CookieQuery` 的最佳场景就是在控制器方法中使用。

如果你已经获得了 `ServerRequestInterface` 的实例，那么也可以使用它的 `cookieQueue` 属性来读取 Cookie:

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function index(ServerRequestInterface $request)
{
    $request->getAttribute('cookieQueue')->set('name', 'value');
}
```

## 手动设置 Cookie

通过 `Psr\Http\Message\ResponseInterface` 的 `withAddedHeader` 方法，你可以手工写入 Cookie 头信息：

```php
return $response->withAddedHeader('Set-Cookie', 'name=value');
```

> 请确保将 Cookie 添加到白名单中，否则 CookieMiddleware 不会让它通过。

## 配置

要配置 `CookieQueue` 的行为，可以使用 `Spiral\Bootloader\Http\CookiesBootloader`.

在其它引导程序中可以通过它来把 Cookie 加入白名单（禁用保护）：

```php
public function boot(CookiesBootloader $cookies)
{
    $cookies->whitelistCookie('CustomCookie');
}
```

还可以在 `app/config` 目录下创建 `cookies.php` 配置文件，来对 Cookie 组件进行更深入的配置：

```php
<?php
declare(strict_types=1);

use Spiral\Cookies\Config\CookiesConfig;

return [
    // 所有的 cookie 默认会设置在 .domain.com 域下可用
    'domain'   => '.%s',

    // 保护方法
    'method'   => CookiesConfig::COOKIE_ENCRYPT,

    // cookie 白名单（不需要加密、解密）
    'excluded' => ['PHPSESSID', 'csrf-token']
];
```
