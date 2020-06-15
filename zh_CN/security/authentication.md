# 安全 - 用户认证

Spiral 框架包含了一系列组件，以通过不同来源的临时或永久令牌认证用户身份，并安全地管理用户上下文。

> 用户认证组件不强制执行任何特定的用户实体接口，也不会将应用程序限制在 HTTP 应用类型（GRPC 应用也可以进行认证）。

## 工作原理

认证组件会为 `Spiral\Auth\AuthContextInterface` 创建一个 IoC 作用域，指向当前授权的行为人（用户、API 客户端）。行为人由 `Spiral\Auth\TokenInterface` 从 `Spiral\Auth\ActorProviderInterface` 进行获取。

Token 由 `Spiral\Auth\TokenStorageInterface` 管理，并且始终包含有效数据（例如 `['UserId' => $id]`、LDAP 证书等）。Token 数据会被 `Spiral\Auth\ActorProviderInterface` 用于查询当前应用程序用户。

Token 存储既可以把 token 存储于外部源（例如数据库、Redis 或文件），也可以即时解码。为了便于使用，Spiral 框架默认包含了多套 token 实现。

> 你可以在同一个应用程序中同时使用多套 token 和 actor 提供程序。

## 安装和配置

要在 Web 应用程序模板创建的应用中安装用户认证组件，可以执行：

```bash
$ composer require spiral/auth spiral/auth-http
```

`spiral/auth` 包提供了不与任何调度方式（HTTP、GRPC）关联的标准接口，而 `spiral/auth-http` 则包含了用于 Web 应用的 HTTP 中间件、令牌传输（Cookie、Header）以及防火墙组件。

要激活组件，需要添加 `Spiral\Bootloader\Auth\HttpAuthBootloader` 引导程序：

```php
[
    // ...
    Framework\Auth\HttpAuthBootloader::class,
    // ...
]
```

安装完成后，必须要选择用什么方式来存储用户认证令牌。

### Session 令牌存储

如果想用 PHP session 来存储令牌，首先确保 `spiral/session` 组件已安装，并用 `Spiral\Bootloader\Auth\TokenStorage\SessionTokensBootloader` 来启用 session 存储：

```php
[
    // ...
    Framework\Auth\HttpAuthBootloader::class,
    Framework\Auth\TokenStorage\SessionTokensBootloader::class,
    // ...
]
```

### 数据库令牌存储
Spiral 也可以把通过 Cycle ORM 把令牌存储在数据库中，要使用这种方式，需要启用 `Spiral\Bootloader\Auth\TokenStorage\CycleTokensBootloader` 引导程序：

```php
[
    // ...
    Framework\Auth\HttpAuthBootloader::class,
    Framework\Auth\TokenStorage\CycleTokensBootloader::class,
    // ...
]
```

此外还必须运行控制台命令 `cycle:sync` 来生成和执行数据库迁移，以便在数据库中创建所需的表：

```php
$ php app.php migrate:init
$ php app.php cycle:migrate -v -r
```

## 行为人提供器和令牌数据

下一步是配置从令牌数据中获取行为人/用户的方式，为了实现这个功能，需要实现和注册 `Spiral\Auth\ActorProviderInterface` 接口。

```php
interface ActorProviderInterface
{
    public function getActor(TokenInterface $token): ?object;
}
```

在本文中，我们以 Cycle 实体和数据仓库为例：

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity(repository="Repository/UserRepository")
 * @Cycle\Table(indexes={
 *     @Cycle\Table\Index(columns={"username"}, unique=true)
 * })
 */
class User
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /** @Cycle\Column(type="string") */
    public $name;

    /** @Cycle\Column(type="string") */
    public $username;

    /** @Cycle\Column(type="string") */
    public $password;
}
```

在 `UserRepository` 上实现 `ActorProviderInterface` 接口：

```php
namespace App\Database\Repository;

use Cycle\ORM\Select\Repository;
use Spiral\Auth\ActorProviderInterface;
use Spiral\Auth\TokenInterface;

class UserRepository extends Repository implements ActorProviderInterface
{
    public function getActor(TokenInterface $token): ?object
    {
        if (!isset($token->getPayload()['userID'])) {
            return null;
        }

        return $this->findByPK($token->getPayload()['userID']);
    }
}
```

在准备工作完成后，先来创建一个用户：

```php
public function index(Transaction $t)
{
    $u = new User();
    $u->name = 'Antony';
    $u->username = 'username';
    $u->password = password_hash('password', PASSWORD_DEFAULT);

    $t->persist($u)->run();
}
```

在你的应用程序中创建和启用一个引导程序，并在引导程序中注册行为人提供器来启用它：

```php
namespace App\Bootloader;

use App\Database\Repository\UserRepository;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Auth\AuthBootloader;

class UserBootloader extends Bootloader
{
    public function boot(AuthBootloader $auth)
    {
        $auth->addActorProvider(UserRepository::class);
    }
}
```

## 认证用户

用户认证过程通过 `Spiral\Auth\AuthContextInterface` 进行。你可以通过方法注入获得实现该接口的当前认证上下文对象实例。

```php
public function index(AuthContextInterface $auth)
{
    // 使用认证上下文
}
```

> 注意：不要在单例服务中存储 `AuthContextInterface`，你可以参考上面我们是如何传递这个对象的。

除此之外，你也可以使用 `Spiral\Auth\AuthScope`, 这是可以存储在单例服务中的，或者使用原型开发辅助里的 `auth` 属性。

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        dump($this->auth);
    }
}
```

### 用户登录

实现用户登录需要我们创建一个登录表单以及相应的请求过滤器。

```php
namespace App\Request;

use Spiral\Filters\Filter;

class LoginRequest extends Filter
{
    public const SCHEMA = [
        'username' => 'data:username',
        'password' => 'data:password'
    ];

    public const VALIDATES = [
        'username' => ['notEmpty'],
        'password' => ['notEmpty']
    ];
}
```

Create login method in the controller dedicated to authentication:
在控制器中创建一个专门用于用户认证的登录方法，

```php
public function login(LoginRequest $login)
{
    if (!$login->isValid()) {
        return [
            'status' => 400,
            'errors' => $login->getErrors()
        ];
    }

    // application specific login logic
    $user = $this->users->findOne(['username' => $login->getField('username')]);
    if (
        $user === null
        || !password_verify($login->getField('password'), $user->password)
    ) {
        return [
            'status' => 404,
            'error'  => '用户不存在'
        ];
    }

    // 创建令牌
}
```

要为上面的请求认证用户，必须创建和你选择的 `ActorProviderInterface` 兼容的数据对象（`['UserId' => $id]`），要实现这个，我们会用到 `AuthContextInterface` 接口的实例以及 `TokenStorageInterface` 的实例。这两个对象，分别可以通过原型开发辅助的 `auth` 和 `authTokens` 属性来访问到：

```php
public function login(LoginRequest $login)
{
    // ... 代码略，见上文

    // 创建令牌
    $this->auth->start(
        $this->authTokens->create(['userID' => $user->id])
    );

    return [
        'status'  => 200,
        'message' => '认证成功！'
    ];
}
```

这样，就完成了用户登录认证。

### 检查登录

要检查当前用户的登录状态，只要检查认证上下文（auth context）拥有一个不为空的行为人（actor）即可：

```php
public function index()
{
    if ($this->auth->getActor() === null) {
        throw new ForbiddenException();
    }
    
    dump($this->auth->getActor());
}
```

> 你可以使用 RBAC 安全组件来同时进行用户认证和用户鉴权。

### 用户注销

要注销当前用户的登录，可以调用认证上下文或者认证作用域（AuthScope）的 `close` 方法：

```php
public function logout()
{
    $this->auth->close();
}
```

## RBAC 安全组件

你可以使用已经认证的用户作为 RBAC 安全组件的行为人，如果需要这样使用，需确保你的用户实体（例如 `App\Database\User`）实现 `Spiral\Security\ActorInterface` 接口：

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;
use Spiral\Security\ActorInterface;

/**
 * @Cycle\Entity(repository="Repository/UserRepository")
 * @Cycle\Table(indexes={
 *     @Cycle\Table\Index(columns={"username"}, unique=true)
 * })
 */
class User implements ActorInterface
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /** @Cycle\Column(type="string") */
    public $name;

    /** @Cycle\Column(type="string") */
    public $username;

    /** @Cycle\Column(type="string") */
    public $password;

    public function getRoles(): array
    {
        return ['user'];
    }
}
```

并且启用 `Spiral\Bootloader\Auth\SecurityActorBootloader` 引导程序把两个组件（用户认证组件、RBAC 安全组件）连接起来：


```php
[
    // ...
    Framework\Auth\SecurityActorBootloader::class,
    // ...
]
```

## 防火墙中间件

你可以把防火墙中间件绑定到特定路由目标，以避免未授权的访问。

默认情况下，Spiral 只提供了一个防火墙，它会将未认证的用户重定向到其它 url:

```php
use Spiral\Auth\Middleware\Firewall\OverwriteFirewall;

// ...

(new Route('/account/<controller>/<action>', $accountTarget))
        ->withMiddleware(new OverwriteFirewall(new Uri('/account/login')));
```

### 自定义防火墙

要实现自己的防火墙中间件，只要扩展 `Spiral\Auth\Middleware\Firewall\AbstractFirewall` 即可：

```php
final class OverwriteFirewall extends AbstractFirewall
{
    protected function denyAccess(Request $request, RequestHandlerInterface $handler): Response
    {
        // 用户未认证
        return $handler->handle($request);
    }
}
```
