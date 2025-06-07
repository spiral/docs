# 框架 — 拦截器

Spiral 的关键特性之一是其对拦截器的支持，拦截器可用于为应用程序添加功能，而无需修改应用程序的核心代码。这有助于保持你的代码库更加模块化和可维护。

**使用拦截器的好处：**

- **关注点分离：** 使用拦截器可以让你将应用程序的不同部分分离并组织起来。例如，你可以使用拦截器来处理身份验证，而无需将该代码添加到需要身份验证的应用程序的每个单独部分。
- **可重用性：** 通过拦截器，你可以编写一次代码并在应用程序的多个部分中使用它，减少代码重复。
- **模块化：** 在不影响应用程序其余部分的情况下添加、删除或替换拦截器的能力使其更灵活且易于更新。
- **性能：** 拦截器可用于通过缓存响应、减少数据库查询次数等方式优化应用程序的性能。
- **易用性：** 将拦截器添加到你的应用程序相对容易和直接。

你可以将拦截器与各种组件一起使用，例如：

- [HTTP](../http/interceptors.md)
- [事件](../advanced/events.md#interceptors)
- [gRPC](../grpc/interceptors.md)
- [Websocket](../websockets/interceptors.md)
- [队列](../queue/interceptors.md)
- [Temporal](../temporal/interceptors.md)

## 拦截器接口

拦截器实现 `Spiral\Interceptors\InterceptorInterface` 接口：

```php
namespace Spiral\Interceptors;

use Spiral\Interceptors\Context\CallContextInterface;

interface InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed;
}
```

该接口提供了一种灵活的方式来拦截对任何目标的调用，无论是方法、函数还是自定义处理程序。

## 调用上下文

`CallContextInterface` 包含有关被拦截调用的所有信息：

- **Target（目标）** — 调用目标的定义（方法、函数、闭包等）
- **Arguments（参数）** — 调用的参数列表
- **Attributes（属性）** — 可用于在拦截器之间传递数据的附加上下文

```php
namespace Spiral\Interceptors\Context;

interface CallContextInterface extends AttributedInterface
{
    public function getTarget(): TargetInterface;
    public function getArguments(): array;
    public function withTarget(TargetInterface $target): static;
    public function withArguments(array $arguments): static;
    
    // AttributedInterface 中的方法：
    public function getAttributes(): array;
    public function getAttribute(string $name, mixed $default = null): mixed;
    public function withAttribute(string $name, mixed $value): static;
    public function withoutAttribute(string $name): static;
}
```

> **注意**
> `CallContextInterface` 是不可变的，所以调用 `withTarget()` 和 `withArguments()` 会返回具有更新值的新实例。

## 目标接口

`TargetInterface` 定义了要拦截调用的目标。它可以表示方法、函数、闭包，甚至是 RPC 或消息队列端点的路径字符串。

```php
namespace Spiral\Interceptors\Context;

interface TargetInterface extends \Stringable
{
    public function getPath(): array;
    public function withPath(array $path, ?string $delimiter = null): static;
    public function getReflection(): ?\ReflectionFunctionAbstract;
    public function getObject(): ?object;
    public function getCallable(): callable|array|null;
}
```

### 创建目标

`Target` 类中的静态工厂方法使创建不同类型的目标变得容易：

```php
use Spiral\Interceptors\Context\Target;

// 从方法反射创建
$target = Target::fromReflectionMethod(new \ReflectionMethod(UserController::class, 'show'), UserController::class);

// 从函数反射创建
$target = Target::fromReflectionFunction(new \ReflectionFunction('array_map'));

// 从闭包创建
$target = Target::fromClosure(fn() => 'Hello, World!');

// 从路径字符串创建（用于 RPC 端点或消息队列处理程序）
$target = Target::fromPathString('user.show');

// 从控制器-操作对创建
$target = Target::fromPair(UserController::class, 'show');
```

## 创建拦截器

让我们创建一个简单的拦截器，用于记录调用的执行时间：

```php
namespace App\Interceptor;

use Psr\Log\LoggerInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

class ExecutionTimeInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {
    }

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $target = $context->getTarget();
        $startTime = \microtime(true);

        try {
            return $handler->handle($context);
        } finally {
            $executionTime = \microtime(true) - $startTime;

            $this->logger->debug(
                'Target executed',
                [
                    'target' => (string)$target,
                    'execution_time' => $executionTime,
                ]
            );
        }
    }
}
```

这个拦截器：
1. 记录开始时间
2. 将调用传递给链中的下一个处理程序
3. 一旦处理程序完成，计算并记录执行时间

## 构建拦截器管道

要使用拦截器，你需要使用 `PipelineBuilderInterface` 构建一个拦截器管道：

```php
use Spiral\Interceptors\PipelineBuilder;
use Spiral\Interceptors\Context\CallContext;
use Spiral\Interceptors\Context\Target;
use Spiral\Interceptors\Handler\CallableHandler;
use App\Interceptor\ExecutionTimeInterceptor;
use App\Interceptor\AuthorizationInterceptor;

// 创建拦截器
$interceptors = [
    new ExecutionTimeInterceptor($logger),
    new AuthorizationInterceptor($auth),
];

// 构建管道
$pipeline = (new PipelineBuilder())
    ->withInterceptors(...$interceptors)
    ->build(new CallableHandler());

// 创建调用上下文
$context = new CallContext(
    target: Target::fromPair(UserController::class, 'show'),
    arguments: [42],
    attributes: ['request' => $request]
);

// 执行管道
$result = $pipeline->handle($context);
```

## 处理程序

管道应该以执行目标的处理程序结束。Spiral 提供了几个内置的处理程序：

### CallableHandler

`CallableHandler` 简单地调用目标，不进行任何额外处理：

```php
use Spiral\Interceptors\Handler\CallableHandler;

$handler = new CallableHandler();
```

### AutowireHandler

`AutowireHandler` 使用容器解析缺失的参数：

```php
use Spiral\Interceptors\Handler\AutowireHandler;

$handler = new AutowireHandler($container);
```

当处理控制器时，这个处理程序很有用，你可以自动注入服务依赖。

## 高级示例

以下是使用拦截器的更全面示例：

```php
namespace App\Controller;

use App\Interceptor\AuthorizationInterceptor;
use App\Interceptor\CacheInterceptor;
use App\Interceptor\ExecutionTimeInterceptor;
use App\User\UserService;
use Psr\Container\ContainerInterface;
use Spiral\Core\Attribute\Proxy;
use Spiral\Core\CompatiblePipelineBuilder;
use Spiral\Interceptors\Context\CallContext;
use Spiral\Interceptors\Context\Target;
use Spiral\Interceptors\Handler\AutowireHandler;

class UserController
{
    private $pipeline;

    public function __construct(
        private readonly UserService $userService,
        #[Proxy] ContainerInterface $container
    ) {
        // 使用拦截器构建管道
        $this->pipeline = (new CompatiblePipelineBuilder())
            ->withInterceptors(
                new ExecutionTimeInterceptor($container->get(LoggerInterface::class)),
                new AuthorizationInterceptor($container->get(AuthInterface::class)),
                new CacheInterceptor($container->get(CacheInterface::class))
            )
            ->build(new AutowireHandler($container));
    }

    public function show(int $id)
    {
        // 为目标方法创建上下文
        $context = new CallContext(
            target: Target::fromReflectionMethod(
                new \ReflectionMethod($this->userService, 'findUser'),
                $this->userService
            ),
            arguments: ['id' => $id]
        );

        // 执行管道
        return $this->pipeline->handle($context);
    }
}
```

在这个例子中：
1. 我们构建了一个管道，包括执行时间记录、授权检查和结果缓存
2. 我们使用 `AutowireHandler` 从容器中解析缺失的参数
3. 控制器方法创建一个上下文，目标是 `UserService` 的 `findUser` 方法
4. 管道执行所有拦截器，然后调用目标方法

> **注意**
> 要了解容器作用域和代理对象，请参阅文档中的 [IoC 作用域](../container/scopes.md) 部分。

## 与旧版拦截器的比较

> **注意**
> 不再推荐基于 `spiral/hmvc` 的旧实现拦截器。
> 你可以在 [https://spiral.dev/docs/framework-interceptors/3.13](https://spiral.dev/docs/framework-interceptors/3.13) 找到旧文档。

在 Spiral 3.14.0 中，`spiral/interceptors` 包引入了新的拦截器实现。
以下是新实现与旧实现的区别：

:::: tabs

::: tab 旧版拦截器
```php
namespace App\Interceptor;

use Psr\SimpleCache\CacheInterface;
use Spiral\Core\CoreInterface;
use Spiral\Core\CoreInterceptorInterface;

class CacheInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
        private readonly int $ttl = 3600,
    ) {}

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        // 第1步：根据控制器、操作和参数生成缓存键
        $cacheKey = $this->generateCacheKey($controller, $action, $parameters);

        // 第2步：检查结果是否已缓存
        if ($this->cache->has($cacheKey)) {
            // 如果可用，返回缓存的结果
            return $this->cache->get($cacheKey);
        }

        // 第3步：如果没有缓存的结果，执行控制器操作
        $result = $core->callAction($controller, $action, $parameters);

        // 第4步：为未来的请求缓存结果
        if ($this->isCacheable($result)) {
            $this->cache->set($cacheKey, $result, $this->ttl);
        }

        return $result;
    }

    private function generateCacheKey(string $controller, string $action, array $parameters): string
    {
        // 从控制器、操作和参数创建确定性的缓存键
        return \md5($controller . '::' . $action . '::' . \serialize($parameters));
    }

    private function isCacheable(mixed $result): bool
    {
        // 只缓存可序列化的结果
        return !\is_resource($result) && (
                \is_scalar($result) ||
                \is_array($result) ||
                $result instanceof \Serializable ||
                $result instanceof \stdClass
            );
    }
}
```
:::

::: tab 新版拦截器
```php
namespace App\Interceptor;

use Psr\SimpleCache\CacheInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\Context\TargetInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final class CacheInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
        private readonly int $ttl = 3600,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        // 第1步：使用目标路径和参数生成缓存键
        $cacheKey = $this->generateCacheKey($context->getTarget(), $context->getArguments());

        // 第2步：检查结果是否已缓存
        if ($this->cache->has($cacheKey)) {
            // 如果可用，返回缓存的结果
            return $this->cache->get($cacheKey);
        }

        // 第3步：如果没有缓存的结果，执行目标
        $result = $handler->handle($context);

        // 第4步：为未来的请求缓存结果
        if ($this->isCacheable($result)) {
            $this->cache->set($cacheKey, $result, $this->ttl);
        }

        return $result;
    }

    private function generateCacheKey(TargetInterface $target, array $args): string
    {
        // 从目标字符串和参数创建确定性的缓存键
        return \md5((string) $target . '::' . serialize($args));
    }

    private function isCacheable(mixed $result): bool
    {
        // 只缓存可序列化的结果
        return !\is_resource($result) && (
                \is_scalar($result) ||
                \is_array($result) ||
                $result instanceof \Serializable ||
                $result instanceof \stdClass
            );
    }
}
```
:::

::::

### 接口变化

**旧接口：**
```php
interface CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed;
}
```

**新接口：**
```php
interface InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed;
}
```

主要区别：
- `$controller` 和 `$action` 参数已被更灵活的 `Target` 概念替代
- `$parameters` 数组现在是 `CallContext` 的一部分
- 取代 `CoreInterface`，现在有一个提供更多灵活性的 `HandlerInterface`

### 兼容性

在 Spiral 3.x 中，同时支持旧拦截器（`CoreInterceptorInterface`）和新拦截器（`InterceptorInterface`）。
但是，旧实现已被弃用，将在 Spiral 4.x 中排除。

如果你需要同时使用两种实现，请使用 `CompatiblePipelineBuilder`：

```php
use Spiral\Core\CompatiblePipelineBuilder;
use Spiral\Core\CoreInterceptorInterface; // 旧版拦截器
use Spiral\Interceptors\InterceptorInterface; // 新版拦截器

$pipeline = (new CompatiblePipelineBuilder())
    ->withInterceptors(
        new LegacyStyleInterceptor(), // 实现 CoreInterceptorInterface
        new NewStyleInterceptor()     // 实现 InterceptorInterface
    )
    ->build(new CallableHandler());
```

### 迁移指南

从旧实现迁移时：
- 将 `CoreInterceptorInterface` 替换为 `InterceptorInterface`
- 使用 `Target` 替代 `$controller` 和 `$action` 参数
- 将 `$parameters` 移至 `CallContext`
- 将 `$core->callAction()` 替换为 `$handler->handle()`

## 事件

| 事件                                        | 描述                                     |
|----------------------------------------------|-----------------------------------------|
| Spiral\Interceptors\Event\InterceptorCalling | 在调用拦截器之前触发。                    |

> **警告**
> `Spiral\Core\Event\InterceptorCalling` 事件仅由旧实现中的已弃用 `\Spiral\Core\InterceptorPipeline` 分发。
> 框架的新实现不使用此事件。

> **注意**
> 要了解有关分发事件的更多信息，请参阅文档中的 [事件](../advanced/events.md) 部分。
