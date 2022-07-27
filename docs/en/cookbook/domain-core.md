# Cookbook - Domain Cores

You can invoke controller actions not only via routes but also from your services and other controllers (HMVC). Every
controller action invocation made via `Spiral\Core\CoreInterface`. The `CoreInterface` or *Domain Core* provides the
developer with the ability to alter the invocation flow and implement domain-specific functionality for controllers.

> **Note**
> The package `spiral/hmvc` required for the domain cores. The web bundle includes this package by default.

## Invoke Controller Action

Spiral controllers are clean classes built to be invoked from any dispatcher. The framework does not provide the direct
coupling between controller and route. Such an approach makes it possible to invoke methods manually:

```php
namespace App\Controller;

use Spiral\Core\CoreInterface;

class HomeController
{
    public function index(CoreInterface $core): string
    {
        return 'Index: ' . $core->callAction(HomeController::class, 'other', ['name' => 'Antony']);
    }

    public function other(string $name): string
    {
        return sprintf('Hello, %s', $name);
    }
}
```

By default, `CoreInterface` implemented by `Spiral\Core\Core` class and only provides support for the method injection.

## Core Interceptors

Use `Spiral\Core\InterceptableCore` and `Spiral\Core\CoreInterceptorInterface` to implement custom invoke logic:

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

To use the interceptor create an instance of `InterceptableCore`:

```php
namespace App\Controller;

use App\Interceptor\CustomInterceptor;
use Spiral\Core\CoreInterface;
use Spiral\Core\InterceptableCore;

class HomeController
{
    public function index(CoreInterface $core): string
    {
        $customCore = new InterceptableCore($core);
        $customCore->addInterceptor(new CustomInterceptor());

        // intercepted: Hello, Antony
        return $customCore->callAction(HomeController::class, 'other', ['name' => 'Antony']);
    }

    public function other(string $name): string
    {
        return sprintf('Hello, %s', $name);
    }
}
```

You can use interceptors to alter the target controller, action, or parameters. Multiple interceptors are possible
as well.

## Global Domain Core

By default, the `CoreInterface` only used to drive targets for framework routing. You can change the default
target via `Spiral\Core\CoreInterface` binding:

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

Activate the Bootloader to make all route targets to be intercepted.

### Route Specific Core

To activate the core for the specific route:

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

## Domain Core Builder

The framework provides convenient Bootloader to configure core interceptors `Spiral\Bootloader\DomainBootloader`
automatically:

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

Use this Bootloader to configure the application behavior globally via the set of default interceptors.

### Cycle Entity Resolution

Use `Spiral\Domain\CycleInterceptor` to automatically resolve entity injections based on parameter values:

```php
$router->setRoute(
    'home',
      new Route(
          '/home/<action>/<id>',
          new Controller(HomeController::class)
      )
);
```

To activate interceptor:

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

You can use any cycle entity injection in your HomeController methods, the `<id>` parameter will be used as the primary
key. If an entity can't be found the 404 exception will be thrown.

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

You must use named parameters if more than one entity expected:

```php
$router->setRoute(
    'home',
      new Route(
          '/home/<action>/<user>/<author>',
          new Controller(HomeController::class)
      )
);
```

The method arguments must be named as route parameters.

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

### Filter Validation

You can automatically pre-validate `Spiral\Filter\FilterInterface` using `Spiral\Domain\FilterInterceptor`, the error
will be returned in JSON form (extend `FilterInterceptor` to customize it).

```php
namespace App\Request;

use Spiral\Filters\Filter;

class LoginRequest extends Filter
{
    public const SCHEMA = [
        'username' => 'query:username',
        'password' => 'query:password'
    ];

    public const VALIDATES = [
        'username' => ['notEmpty'],
        'password' => ['notEmpty']
    ];
}
```

Now, the `LoginRequest` object passed to controller method will always be valid:

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

> **Note**
> Use `/home/index?username=n&password=p` to pass the validation.

In case of the error, the following `application/json` payload will be sent to the client:

```json
{
  "status": 400,
  "errors": {
    "username": "This value is required.",
    "password": "This value is required."
  }
}
```

### Guard Interceptor

Use `Spiral\Domain\GuardInterceptor` to implement RBAC pre-authorization logic (make sure to install and
activate `spiral/security`).

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

    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole(Guest::ROLE);
        $rbac->associate(Guest::ROLE, 'home.*', Rule\AllowRule::class);
        $rbac->associate(Guest::ROLE, 'home.other', Rule\ForbidRule::class);
    }
}
```

You can use attributes to configure which permissions to apply for the controller action:

```php
namespace App\Controller;

use Spiral\Domain\Annotation\Guarded;

class HomeController
{
    #[Guarded(permission: 'home.index')]
    public function index(): string
    {
        return 'OK';
    }

    #[Guarded(permission: 'home.other')]
    public function other(): string
    {
        return 'OK';
    }
}
```

To specify the fallback action when permission is not checked use `else` attribute of `Guarded`:

```php
#[Guarded(permission: 'home.other', else: 'notFound')]
public function other(): string
{
    return 'OK';
}
```

> **Note**
> Allowed values: `notFound` (404), `forbidden` (401), `error` (500), `badAction` (400).


Use the attribute `Spiral\Domain\Annotation\GuardNamespace` to specify controller RBAC namespace and remove the prefix
from every action. You can also skip the permission definition in `Guarded` when a namespace is specified (security component 
will use `namespace.methodName` as permission name).

```php
namespace App\Controller;

use Spiral\Domain\Annotation\Guarded;
use Spiral\Domain\Annotation\GuardNamespace;

#[GuardNamespace(namespace: 'home')]
class HomeController
{
    #[Guarded]
    public function index(): string
    {
        return 'OK';
    }

    #[Guarded(else: 'notFound')]
    public function other(): string
    {
        return 'OK';
    }
}
```

#### Rule Context

You can use all method parameters as rule context, for example, we can create a rule:

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

To activate the rule:

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

    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole(Guest::ROLE);
        $rbac->associate(Guest::ROLE, 'home.*', SampleRule::class);
        $rbac->associate(Guest::ROLE, 'home.other', Rule\ForbidRule::class);
    }
}

```

> **Note**
> Make sure that route includes `<id>` or `<user>` parameter.

And modify the method:

```php
#[Guarded] 
public function index(User $user): string
{
    return 'OK';
}
```

The method would not allow invoking the method with user id `1`.

> **Note**
> Make sure to enable `CycleInterceptor` before `GuardInterceptor` in domain core.

### DataGrid Interceptor

You can automatically apply datagrid specifications to an iterable output using `DataGrid` attribute and `GridInterceptor`.
This interceptor is called after the endpoint invocation because it uses the output.

```php
namespace App\Controller;

use App\Repository\UserRepository;
use App\View\Keeper\UserGrid;
use Spiral\DataGrid\Annotation\DataGrid;
use Spiral\Router\Annotation\Route;

class UsersController
{
    #[Route(route: '/users', name: 'users')]
    #[DataGrid(grid: UserGrid::class)]
    public function list(UserRepository $userRepository): iterable
    {
        return $userRepository->select();
    }
}   
```

> **Note**
> `grid` property should refer to a `GridSchema` class with specifications declared in the constructor.

```php
namespace App\View;

use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter;
use Spiral\DataGrid\Specification\Value;

class UserGrid extends GridSchema
{
    public function __construct()
    {
        $this->addSorter('email', new Sorter\Sorter('email'));
        $this->addSorter('name', new Sorter\Sorter('name'));
        $this->addFilter('status', new Filter\Equals('status', new Value\EnumValue(new Value\StringValue(), 'active', 'disabled')));
        $this->setPaginator(new PagePaginator(20, [10, 20, 50, 100]));
    }
}
```

Optionally, you can specify `view` property to point to a callable presenter for every record.
Without specifying it `GridInterceptor` will call `__invoke` in the declared grid.

```php
namespace App\View;

use Spiral\DataGrid\GridSchema;
use App\Database\User;

class UserGrid extends GridSchema
{
    //...
    public function __invoke(User $user): array
    {
        return [
            'id'     => $user->id,
            'name'   => $user->name,
            'email'  => $user->email,
            'status' => $user->status
        ];
    }
}
```

You can specify grid defaults (such as default sorting, filtering, pagination) via `defaults` property or
using `getDefaults()` method in your grid:

```php
#[DataGrid(
    grid: UserGrid::class,
    defaults: [
        'sort' => ['name' => 'desc'],
        'filter' => ['status' => 'active'],
        'paginate' => ['limit' => 50, 'page' => 10]
    ]
)]
```

By default, grid output will look like this:

```json
{
  "status": 200,
  "data": [
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ]
}
```

You can rename `data` property or pass the exact `status` code `options` or `getOptions()` method in the grid:

```php
#[DataGrid(grid: UserGrid::class, options: ['status' => 201, 'property' => 'users'])]
```

```json
{
  "status": 201,
  "users": [
    ...
  ]
}
```

`GridInterceptor` will create a `GridFactoryInterface` instance to wrap given iterable source with the declared grid
schema.
`GridFactory` is used by default, but if you need more complicated logic, such as using a custom counter or
specifications utilization, you can declare your own factory in the annotation:

```php
#[DataGrid(grid: UserGrid::class, factory: InheritedFactory::class)]
```

### Pipeline Interceptor

This interceptor allows customising endpoint interceptors using `@Pipeline` annotation.
When declared in the domain core interceptors list, this interceptor injects the specified annotated interceptors on the
position where the `PipelineInterceptor` is declared.

```php
namespace App\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\DataGrid\Interceptor\GridInterceptor;
use Spiral\Domain;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        Domain\CycleInterceptor::class,
        Domain\PipelineInterceptor::class, //all annotated interceptors go here
        Domain\GuardInterceptor::class,
        Domain\FilterInterceptor::class,
        GridInterceptor::class,
    ];
}
```

`Pipeline` attribute allows skipping the subsequent interceptors:

```php
    #[Pipeline(pipeline: [OtherInterceptor::class], skipNext: true)]
    public function action(): string
    {
        //
    }
 ```

Using the prev bootloader we will get the next interceptors list:

- Domain\CycleInterceptor
- OtherInterceptor

> **Note**
> All interceptors after `PipelineInterceptor` will be omitted.

### Use cases

For example, it can be helpful when an endpoint should not apply any interceptor or not all of them are required
currently:

```php
    #[Route(route: '/show/<user:int>/email/<email:int>', name: 'emails')]
    #[Pipeline(pipeline: [CycleInterceptor::class, GuardInterceptor::class], skipNext: true)]
    public function email(User $user, Email $email, EmailFilter $filter): string
    {
        $filter->setContext(compact('user', 'email'));
        if (!$filter->isValid()) {
            throw new ForbiddenException('Email doesn\'t belong to a user.');
        }
        //...
    }
 ```

> **Note**
> `FilterInterceptor` should not be applied here because of a complicated context, so we set it manually and call a
> custom `isValid()` check. Also, `GridInterceptor` is redundant here.

To have the full control over the interceptors list you need to specify `PipelineInterceptor` as the first one.

## All Together

Use all interceptors together to implement rich domain logic and secure controller actions:

```php
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\DataGrid\Interceptor\GridInterceptor;
use Spiral\Domain;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        Domain\CycleInterceptor::class,
        Domain\GuardInterceptor::class,
        Domain\FilterInterceptor::class,
        GridInterceptor::class,
    ];
}
```
