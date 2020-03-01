# 速查手册 - 领域内核

除了通过路由来调用控制器方法（action），Spiral 还允许从服务中甚至是从其它的控制器中调用控制器方法（HMVC: Hierarchical Model View Controller）。但是对控制器方法的方法的调用必须通过 `Spiral\Core\CoreInterface` 来进行。`CoreInterface` 以及*领域内核*使开发者能够更改调用流程，以及为控制器实现特定于域的功能。

> 领域内核依赖 `spiral/hmvc` 组件。官方 Web 应用模板中已经默认包含了这个组件。

## 调用控制器方法

Spiral 中的控制器都是普通的类，供调度程序调用。框架本身并不提供路由和控制器之间的直接耦合。这样的实现方式下，就可以在必要的时候在代码中直接调用控制器中定义的方法。

```php
namespace App\Controller;

use Spiral\Core\CoreInterface;

class HomeController
{
    public function index(CoreInterface $core)
    {
        return "Index: " . $core->callAction(HomeController::class, "other", ["name" => "Antony"]);
        // 这行代码最终返回 "Index: Hello, Antony"
    }

    public function other(string $name)
    {
        return sprintf("Hello, %s", $name);
    }
}
```

默认状态下，`CoreInterface` 接口由 `Spiral\Core\Core` 这个类实现，它只提供方法注入。

## 内核拦截器

为了实现自定义的调用逻辑，需要借助 `Spiral\Core\CoreInterceptorInterface` 接口 `Spiral\Core\InterceptableCore` 类。

首先创建一个实现 `Spiral\Core\CoreInterceptorInterface` 的自定义拦截器：

```php
namespace App\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;

class CustomInterceptor implements CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core)
    {
        return 'intercepted: ' . $core->callAction($controller, $action, $parameters);
    }
}
```

接下来通过创建一个 `InterceptableCore` 类的实例来使用我们的自定义拦截器：

```php
namespace App\Controller;

use App\Interceptor\CustomInterceptor;
use Spiral\Core\CoreInterface;
use Spiral\Core\InterceptableCore;

class HomeController
{
    public function index(CoreInterface $core)
    {
        $customCore = new InterceptableCore($core);
        $customCore->addInterceptor(new CustomInterceptor());

        // 截获执行结果：Hello, Antony
        return $customCore->callAction(HomeController::class, "other", ["name" => "Antony"]);
    }

    public function other(string $name)
    {
        return sprintf("Hello, %s", $name);
    }
}
```

通过拦截器可以针对控制器、控制器方法和参数进行改动。根据需要也可使用多个拦截器。

## 全局领域内核

默认情况下，`CoreInterface` 只作用于 Spiral 框架路由的目标。通过把 `Spiral\Core\CoreInterface` 绑定到自定义的内核类，可以改变目标：

```php
namespace App\Bootloader;

use App\Interceptor\CustomInterceptor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Core;
use Spiral\Core\CoreInterface;
use Spiral\Core\InterceptableCore;

class CoreBootloader extends Bootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'core']
    ];

    private function core(Core $core): CoreInterface
    {
        $customCore = new InterceptableCore($core);
        $customCore->addInterceptor(new CustomInterceptor());

        return $customCore;
    }
}
```

别忘了激活上面的引导程序，激活之后即可拦截所有的路由目标。

### 路由级领域内核

除了在引导程序中激活的全局领域内核以外，也可以只针对特定路由激活领域内核：

```php
$customCore = new InterceptableCore($core);
$customCore->addInterceptor(new CustomInterceptor());

$router->setRoute(
    'home',
    new Route(
        '/home/<action>',
        (new Controller(HomeController::class))->withCore($customCore)
    )
);
```

## 领域内核构建器

Spiral 提供了 `Spiral\Bootloader\DomainBootloader` 这个引导程序，可以很方便地自动配置核心拦截器，可以创建一个继承 `DomainBootloader` 的引导程序：

```php
namespace App\Bootloader;

use App\Interceptor\CustomInterceptor;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CustomInterceptor::class
    ];
}
```

把要配置的拦截器类添加到该引导程序的 `INTEREPTORS` 集合中，引导程序通过这一组默认拦截器即可全局性地配置应用程序行为。

### Cycle 实体解析

使用 `Spiral\Domain\CycleInterceptor` 拦截器，可以根据参数自动解析和注入 Cycle 实体：

```php
$router->setRoute(
    'home',
      new Route(
          '/home/<action>/<id>',
          new Controller(HomeController::class)
      )
);
```

按照前文介绍的方法，把 `CycleInterceptor` 加入到默认拦截器集合中：

```php
namespace App\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Domain\CycleInterceptor;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class
    ];
}
```

然后即可在控制器方法中直接依赖 Cycle 实体，系统会把路由中的 `<id>` 参数作为实体的主键，根据声明的类型自动解析出对应的实体对象并注入到方法。如果指定的主键没有查询到实体，会抛出 404 错误。

```php
namespace App\Controller;

use App\Database\User;

class HomeController
{
    public function index(User $user)
    {
        dump($user);
    }
}
```

要使用的实体超过一个时，路由参数就不能再使用 `<id>`，而必须使用命名参数：

```php
$router->setRoute(
    'home',
      new Route(
          '/home/<action>/<user>/<author>',
          new Controller(HomeController::class)
      )
);
```

控制器方法中的参数名要和路由参数名对应：

```php
namespace App\Controller;

use App\Database\User;
use App\Database\Post;

class HomeController
{
    public function index(User $user, User $author)
    {
        dump($user);
    }
}
```

### 过滤器验证

Spiral 提供了 `Spiral\Filter\FilterInterface` 接口和 `Spiral\Domain\FilterInterceptor` 拦截器，开发者使用继承 `Spiral\Filter\Filter` 创建请求类，就可以实现请求参数的自动验证和过滤。如果参数检查失败，错误会以 JSON 形式返回（如果要改变这一行为，可以继承 `FilterInterceptor` 来创建自己的拦截器）。使用方法如下：

```php
namespace App\Request;

use Spiral\Filters\Filter;

class LoginRequest extends Filter
{
    public const SCHEMA = [
        // 绑定参数与请求的关系
        'username' => 'query:username',
        'password' => 'query:password'
    ];

    public const VALIDATES = [
        // 定义验证规则
        'username' => ['notEmpty'],
        'password' => ['notEmpty']
    ];
}
```

在控制器方法中，依赖注入上面定义的 `LoginRequest`, 系统会自动验证请求并过滤出需要的参数，然后通过 `LoginRequest` 对象传入：

```php
namespace App\Controller;

use App\Request\LoginRequest;

class HomeController
{
    public function index(LoginRequest $request)
    {
        dump($request);
    }
}
```

> Use `/home/index?username=n&password=p` 即可通过上面的验证。

如果发生参数验证错误，客户端会收到类似下面实例的 `application/json` 类型的响应结果：

```json
{
    "status": 400,
    "errors": {
        "username": "This value is required.",
        "password": "This value is required."
    }
}
```

### 访问权限拦截器

使用 `Spiral\Domain\GuardInterceptor` 可以实现 RBAC（Role-Based Access Control: 基于角色的访问控制）预鉴权逻辑（需要安装和激活 `spiral/security` 组件）。

```php
namespace App\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Domain\GuardInterceptor;
use Spiral\Security\Actor\Guest;
use Spiral\Security\PermissionsInterface;
use Spiral\Security\Rule;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        GuardInterceptor::class
    ];

    public function boot(PermissionsInterface $rbac)
    {
        $rbac->addRole(Guest::ROLE);
        $rbac->associate(Guest::ROLE, 'home.*', Rule\AllowRule::class);
        $rbac->associate(Guest::ROLE, 'home.other', Rule\ForbidRule::class);
    }
}
```

可以在控制器方法上用注释（annotation）声明访问权限：

```php
namespace App\Controller;

use Spiral\Domain\Annotation\Guarded;

class HomeController
{
    /**
     * @Guarded("home.index")
     */
    public function index()
    {
        return 'OK';
    }

    /**
     * @Guarded("home.other")
     */
    public function other()
    {
        return 'OK';
    }
}
```

用注释声明访问权限时，也可以用 `Guarded` 注释的 `else` 属性来指定权限验证失败时应该调用的控制器方法：

```php
/**
 * @Guarded("home.other", else="notFound")
 */
public function other()
{
    return 'OK';
}
```

> 注意：`else` 属性可以使用的值包括：`notFound` 返回 404, `forbidden` 返回 401, `error` 返回 500, `badAction` 返回 400.

另外，使用 `Spiral\Domain\Annotation\GuardNamespace` 注释还可以给控制器指定 RBAC 命名空间，这样在各个方法的注释中就可以省略掉控制器前缀。在控制器指定了 RBAC 命名空间的情况下，方法上的 `Guarded` 注释可以省略参数，安全组件会自动使用 `namespace.methodName` 作为权限名称。
 
```php
namespace App\Controller;

use Spiral\Domain\Annotation\Guarded;
use Spiral\Domain\Annotation\GuardNamespace;

/**
 * @GuardNamespace("home")
 */
class HomeController
{
    /**
     * @Guarded()
     */
    public function index()
    {
        return 'OK';
    }

    /**
     * @Guarded(else="notFound")
     */
    public function other()
    {
        return 'OK';
    }
}
```

#### 规则上下文
所有的路由参数（控制器方法参数）可以可以作为规则上下文调用。比如，创建一个规则类：

```php
namespace App\Security;

use Spiral\Security\ActorInterface;
use Spiral\Security\RuleInterface;

class SampleRule implements RuleInterface
{
    public function allows(ActorInterface $actor, string $permission, array $context): bool
    {
        return $context['user']->getID() !== 1;
    }
}
```

在引导程序中激活这个规则类：

```php
namespace App\Bootloader;

use App\Security\SampleRule;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Domain\CycleInterceptor;
use Spiral\Domain\GuardInterceptor;
use Spiral\Security\Actor\Guest;
use Spiral\Security\PermissionsInterface;
use Spiral\Security\Rule;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        GuardInterceptor::class
    ];

    public function boot(PermissionsInterface $rbac)
    {
        $rbac->addRole(Guest::ROLE);
        $rbac->associate(Guest::ROLE, 'home.*', SampleRule::class);
        $rbac->associate(Guest::ROLE, 'home.other', Rule\ForbidRule::class);
    }
}

```

> 注意：路由器中需要包括被调用的 `<id>` 或者 `<user>` 参数。

然后控制器方法可以改为：

```php
/**
 * @Guarded()
 */
public function index(User $user)
{
    return 'OK';
}
```

上述示例中，如果访问该方法（路由）的用户 id 不等于 `1`，就会鉴权失败，被阻止访问。

> 由于这里的 `User` 依赖 `CycleInterceptor` 进行解析，所以在领域核心中一定要在 `GuardInterceptor` 之前启用 `CycleInterceptor`。

## 综合应用

把上面介绍到的所有拦截器综合到一起，就能实现丰富的领域逻辑和安全的控制器方法：

```php
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Domain;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        Domain\CycleInterceptor::class,
        Domain\GuardInterceptor::class,
        Domain\FilterInterceptor::class
    ];
}
```
