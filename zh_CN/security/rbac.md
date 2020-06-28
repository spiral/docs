# 安全 - 基于角色的权限控制

Spiral 框架包含 `spiral/security` 安全组件，提供了根据权限列表授权用户/行动者访问特定资源或行为的能力。安全组件实现了 [NIST RBAC 研究](https://csrc.nist.gov/projects/role-based-access-control) 中阐述的“扁平 RBAC”设计模式。

该安全组件包括多项附加功能，比如：

- 用于控制权限/许可上下文的额外的 _rules_ 层
- 通过通配符模式为一个角色分配多个权限
- 用高优先级规则覆盖已分配给角色的权限

> 借助这些附加功能，可以使用该安全组件作为 ACL, DAC 和 ABAC 安全模型的框架。

在项目中使用该安全组件，需要通过激活 `Spiral\Bootloader\Security\GuardBootloader` 引导程序来启用组件，但不需要对组件进行配置。

> 在 Web 和 GRPC 应用程序模板中已经默认包含和启用了该安全组件。

## 行为人（Actor）

应用中的所有权限会根据与当前 `Spiral\Security\ActorInterface` 关联的角色来进行授予。

```php
interface ActorInterface
{
    public function getRoles(): array;
}
```

> 请在用户请求时使用 IoC 作用域来设置行为人.

要了解如何将已认证用户作为行为人，请阅读[用户认证文档](/zh_CN/security/authentication.md).

默认情况下，应用程序使用 `Spiral\Security\Actor\Guest` 作为默认的行为人，你可以在全局范围或者 IoC 作用域内用容器绑定进行行为人设置。

```php
namespace App\Controller;

use Spiral\Core\ScopeInterface;
use Spiral\Security\Actor\Actor;
use Spiral\Security\ActorInterface;

class HomeController
{
    public function index(ScopeInterface $scope)
    {
        return $scope->runScope([ActorInterface::class => new Actor(['admin'])], function () {

            // 当前请求作用域内行为人拥有 `admin` 角色
            return 'ok';
        });
    }
}
```

> 你可以使用领域核心拦截器、GRPC 拦截器、HTTP 中间件、自定义 IoC 作用域等手段来设置 Actor。

为了简化文档，下面的代码通过自定义的引导程序设置全局范围内可见的默认行为人：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container;
use Spiral\Security\Actor\Actor;
use Spiral\Security\ActorInterface;

class SecurityBootloader extends Bootloader
{
    public function boot(Container $container)
    {
        $container->bindSingleton(ActorInterface::class, new Actor(['user']));
    }
}
```

## 守卫接口（GuardInterface）

要使用 RBAC 组件，必须注册可用的角色，并在角色和权限之间创建关联，可以在设置行为人的同一个引导程序中，使用 `Spiral\Security\PermissionsInterface` 接口来完成这项工作：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container;
use Spiral\Security\Actor\Actor;
use Spiral\Security\ActorInterface;
use Spiral\Security\PermissionsInterface;

class ActorBootloader extends Bootloader
{
    public function boot(Container $container, PermissionsInterface $rbac)
    {
        $container->bindSingleton(ActorInterface::class, new Actor(['user']));

        $rbac->addRole('user');
        $rbac->associate('user', 'home.read');
    }
}
```

> 下面的文档会详细解释“角色-规则-权限（role-rule-permission）的关联。

一旦引导程序被激活，就可以使用 `Spiral\Core\GuardInterface` 来检查用户是否具有特定权限，守卫（Guard）会自动通过当前活动的作用域，使用 `Spiral\Security\GuardScope` 解析当前活动的行为人。该接口提供了名为 `allows` 的方法，用于检查行为人是否具有指定的权限：

```php
namespace App\Controller;

use Spiral\Security\GuardInterface;

class HomeController
{
    public function index(GuardInterface $guard)
    {
        if (!$guard->allows('home.read')) {
            return '不可读';
        }

        return '可读';
    }
}
```

> 你可以改变默认行为人的角色来看看它对结果有什么影响。

当然为了开发更便利，可以使用原型开发辅助提供的 `guard` 属性。

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        if (!$this->guard->allows('home.read')) {
            return '不可读';
        }

        return '可读';
    }
}
```

> `GuardInterface` 可以用于控制器、服务和视图。

### 权限上下文

守卫对象的 `allows` 方法支持第二个参数，该参数代表权限上下文，通常它必须包含当前行为人试图访问或改动的目标实体对象。

```php
namespace App\Controller;

use Spiral\Security\GuardInterface;

class HomeController
{
    public function index(GuardInterface $guard)
    {
        if (!$guard->allows('home.read', ['key' => 'value'])) {
            return '不可读';
        }

        return '可读';
    }
}
```

接下来，我们一起了解一下如何使用上下文来创建更为复杂的角色-权限关联。

## 权限管理

RBAC 组件的核心部分是 `Spiral\Security\PermissionInterface` 接口。尽管你可以在你的实现中用它进行动态角色和权限的配置，但默认情况下，它的设计目的是用于在引导程序中配置角色和权限映射。

### 创建角色

每个应用程序都必须使用 `addRole` 方法在 RBAC 组件中注册可用的用户角色：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Security\PermissionsInterface;

class SecurityBootloader extends Bootloader
{
    public function boot(PermissionsInterface $rbac)
    {
        $rbac->addRole('guest');
    }
}
```

### 权限

一旦角色创建好，就可以用 `associate` 方法将它们和具体权限进行关联：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Security\PermissionsInterface;

class SecurityBootloader extends Bootloader
{
    public function boot(PermissionsInterface $rbac)
    {
        $rbac->addRole('guest');

        $rbac->associate('guest', 'home.read');
    }
}
```

### 通配符关联

你也可以一次性把单个权限与多个权限进行关联：

```php
$rbac->associate('guest', 'home.(read|write)');
```

你还可以使用通配符 `*` 来创建角色与某个命名空间下的所有权限的关联：

```php
$rbac->associate('guest', 'home.*');
```

注意，如果你使用了带 `.` 分隔符的多级命名空间，只有第一级的命名空间会被关联：

```php
$rbac->associate('guest', 'home.*');
$rbac->associate('guest', 'home.*.*');
// ...
```

### 排除权限

有时候，你需要授予某个命名空间下的所有权限，排除特定的少数权限。这种情况可以用第三个参数，第三个参数可以指定覆盖规则。比如要禁止某项已经授予的权限，就可以用 `Spiral\Security\Rule\ForbidRule`:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Security\PermissionsInterface;
use Spiral\Security\Rule\ForbidRule;

class SecurityBootloader extends Bootloader
{
    public function boot(PermissionsInterface $rbac)
    {
        $rbac->addRole('guest');

        $rbac->associate('guest', 'home.*');
        $rbac->associate('guest', 'home.read', ForbidRule::class);
    }
}
```

第三个参数的默认值是 `AllowRule`。上面的例子与以下表达形式相同：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Security\PermissionsInterface;
use Spiral\Security\Rule;

class SecurityBootloader extends Bootloader
{
    public function boot(PermissionsInterface $rbac)
    {
        $rbac->addRole('guest');

        $rbac->associate('guest', 'home.*', Rule\AllowRule::class);
        $rbac->associate('guest', 'home.read', Rule\ForbidRule::class);
    }
}
```

> 守卫（Guard）会检查行为人的所有角色，其中至少要有一个角色被授予了相关权限，行为人才会拥有该项权限。

## 规则

如上所述，所有的角色到权限的关联都由一套规则来控制。每个规则都必须实现 `Spiral\Security\RuleInterface` 接口。

```php
interface RuleInterface
{
    public function allows(ActorInterface $actor, string $permission, array $context): bool;
}
```

在检查某个角色的某项权限时，对应的规则接收到由 `GuardInterface`->`allows` 方法提供的当前上下文：

```php
if (!$guard->allows('home.read', ['key' => 'value'])) {
    return '不可读';
}
```

这样的实现允许你在角色和权限集之间建立复杂的规则关联。

> 默认规则 `AllowRule` 和 `ForbidRule` 总是相应地返回 `true` 和 `false`.

### 自定义规则

如果需要创建自定义规则，只需要实现 `Spiral\Security\RuleInterface` 接口。例如，我们创建只在上下文中 `key` 的值为 `value` 时才授予权限的一个规则：

```php
namespace App\Security;

use Spiral\Security\ActorInterface;
use Spiral\Security\RuleInterface;

class SampleRule implements RuleInterface
{
    public function allows(ActorInterface $actor, string $permission, array $context): bool
    {
        return $context['key'] === 'value';
    }
}
```

定义好之后，你可以通过规则的类名将其应用于角色权限关联：

```php
namespace App\Bootloader;

use App\Security\SampleRule;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Security\PermissionsInterface;

class SecurityBootloader extends Bootloader
{
    public function boot(PermissionsInterface $rbac)
    {
        $rbac->addRole('guest');

        $rbac->associate('guest', 'home.*', SampleRule::class);
    }
}
```

现在，改变 `allows` 方法中的上下文可以让权限检查通过或拒绝：

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        if ($this->guard->allows('home.read', ['key' => 'value'])) {
            echo '通过';
        }

        if ($this->guard->allows('home.read', ['key' => 'else'])) {
            echo '拒绝';
        }
    }
}
```

> 可以使用行为人接口创建更负责的规则。

传入业务实体作为上下文来检查权限：

```php
if ($this->guard->allows('post.edit', ['post' => $post])) {
    echo '通过';
}
```

还可以定义规则来检查当前用户（行为人）是否实体的作者：

```php
class SampleRule implements RuleInterface
{
    public function allows(ActorInterface $actor, string $permission, array $context): bool
    {
        return $context['post']->user === $actor;
    }
}
```

### 静态规则

为了简化规则的创建，可以使用 `Spiral\Security\Rule`, 它通过 `check` 方法来启用了方法注入：

```php
namespace App\Security;

use Spiral\Security\Rule;

class SampleRule extends Rule
{
    public function check(string $key): bool
    {
        return $key === 'value';
    }
}
```

> 通过这里的方法注入，你可以使用应用程序的任何服务。

建议将规则声明为单例模式，以提升应用程序的性能：

```php
namespace App\Security;

use Spiral\Core\Container\SingletonInterface;
use Spiral\Security\Rule;

class SampleRule extends Rule implements SingletonInterface
{
    public function check(string $key): bool
    {
        return $key === 'value';
    }
}
```

## @Guarded 注解

在控制器方法上，可以使用 `Guarded` 注解来自动检查对[领域核心](/zh_CN/cookbook/domain-core.md)的访问权限。

```php
namespace App\Controller;

use Spiral\Domain\Annotation\Guarded;

class HomeController
{
    /**
     * @Guarded("home.index", else="notFound")
     */
    public function index()
    {
        return 'OK';
    }
}
```
