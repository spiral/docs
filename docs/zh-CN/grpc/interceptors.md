# GRPC — 拦截器

Spiral 为 gRPC 服务提供了拦截器，允许您在请求生命周期的各个阶段拦截和修改请求和响应。

> **查看更多**
> 在 [Framework — 拦截器](../framework/interceptors.md) 部分阅读更多关于拦截器的信息。

拦截器分为两种类型：

1. 服务器拦截器
2. 客户端拦截器

## 服务器拦截器

服务器拦截器用于拦截和修改服务器接收的请求和响应。它们通常用于为服务器添加横切功能，如日志记录、身份验证或监控。

### 日志拦截器

以下是一个简单拦截器的示例，它在处理前后记录请求：

```php
namespace App\Endpoint\GRPC\Interceptor;

use Psr\Log\LoggerInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final class LogInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $target = $context->getTarget();

        $this->logger->info('Request received...', [
            'target' => (string) $target,
            'path' => $target->getPath(),
        ]);

        $response = $handler->handle($context);

        $this->logger->info('Request processed', [
            'target' => (string)$target,
            'path' => $target->getPath(),
        ]);

        return $response;
    }
}
```

### 异常处理拦截器

以下是一个简单拦截器的示例，它处理服务器抛出的异常。它会捕获所有异常并将其转换为 gRPC 异常。

```php
namespace App\Endpoint\GRPC\Interceptor;

use Spiral\Exceptions\ExceptionReporterInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;
use Spiral\RoadRunner\GRPC\Exception\GRPCException;
use Spiral\RoadRunner\GRPC\Exception\GRPCExceptionInterface;

final class ExceptionHandlerInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly ExceptionReporterInterface $reporter,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        try {
            return $handler->handle($context);
        } catch (\Throwable $e) {
            $this->reporter->report($e);

            if ($e instanceof GRPCExceptionInterface) {
                throw $e;
            }

            throw new GRPCException(
                message: $e->getMessage(),
                previous: $e,
            );
        }
    }
}
```

### 跟踪

跟踪传入调用不需要编写拦截器：从 `spiral/roadrunner-bridge` 5.0 开始，每个调用都在[遥测](../advanced/telemetry.md)跟踪器的
`SERVER` span 中运行，您的拦截器也在该 span 内执行。跟踪会从请求的 `traceparent` 元数据继续，响应头会把 span 上下文带回调用方。

span 以方法命名，例如 `ping.dto.PingService/Ping`，并带有 OpenTelemetry gRPC 属性 `rpc.system.name`、`rpc.method` 和
`rpc.response.status_code`。以 `OK` 以外的任何状态结束的调用都会被标记为错误，状态写入 `error.type`；如果异常不是
`GRPCExceptionInterface`，则写入异常的类名。

### 守卫拦截器

以下是一个简单拦截器的示例，它检查用户是否已认证。它将使用 PHP 属性确定哪些方法需要认证。认证令牌在请求元数据中传递。

```php
namespace App\Endpoint\GRPC\Interceptor;

use App\Attribute\Guarded;
use Spiral\Attributes\ReaderInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;
use Spiral\RoadRunner\GRPC\ContextInterface;

final class GuardedInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly ReaderInterface $reader
    ) {
    }

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $reflection = $context->getTarget()->getReflection();
        
        if ($reflection !== null) {
            $attribute = $this->reader->firstFunctionMetadata($reflection, Guarded::class);
            
            if ($attribute !== null) {
                $ctx = $context->getArguments()[0] ?? null;
                if ($ctx instanceof ContextInterface) {
                    $this->checkAuth($attribute, $ctx);
                }
            }
        }

        return $handler->handle($context);
    }

    private function checkAuth(Guarded $attribute, ContextInterface $ctx): void
    {
        // 元数据始终将值存储为数组
        $token = $ctx->getValue($attribute->tokenField)[0] ?? null;

        // 这里可以实现您自己的认证逻辑
        if ($token !== 'secret') {
            throw new \Exception('Unauthorized.');
        }
    }
}
```

需要认证的方法示例：

```php
use App\Attribute\Guarded;

#[Guarded]
public function ping(GRPC\ContextInterface $ctx, PingRequest $in): PingResponse
{
    // ...
}
````

Guarded 属性示例：

```php
namespace App\Attribute;

use Doctrine\Common\Annotations\Annotation\NamedArgumentConstructor;

#[\Attribute(\Attribute::TARGET_METHOD), NamedArgumentConstructor]
class Guarded
{
    public function __construct(
        public readonly string $tokenField = 'token'
    ) {
    }
}
```

### 注册服务器拦截器

要使用服务器拦截器，请在配置文件 `app/config/grpc.php` 中注册它们：

```php app/config/grpc.php
return [    
    'interceptors' => [
        \App\Endpoint\GRPC\Interceptor\LogInterceptor::class,
        \App\Endpoint\GRPC\Interceptor\ExceptionHandlerInterceptor::class,
        \App\Endpoint\GRPC\Interceptor\GuardedInterceptor::class,
    ]
];
```

## 客户端拦截器

客户端拦截器允许您修改或扩展 gRPC 客户端请求的行为。
它们可以添加日志记录、超时配置、重试和认证等功能。

> **查看更多**
> 有关 gRPC 客户端配置和使用的基本信息，请参阅 [客户端](./client.md) 文档。

### 内置客户端拦截器

Spiral 通过 `spiral/grpc-client` 包提供了几个内置的客户端拦截器：

#### SetTimeoutInterceptor

为每个 gRPC 请求设置超时时间（毫秒）：

```php
use Spiral\Grpc\Client\Interceptor\SetTimoutInterceptor;

// 添加 5 秒超时
SetTimoutInterceptor::createConfig(5_000)
```

#### RetryInterceptor

使用指数退避算法为失败的请求实现重试逻辑：

```php
use Spiral\Grpc\Client\Interceptor\RetryInterceptor;

// 配置具有自定义参数的重试
RetryInterceptor::createConfig(
     // 初始退避间隔（毫秒）（默认：50ms）
    initialInterval: 50,
     // 资源耗尽问题的初始间隔（默认：1000ms）
    congestionInitialInterval: 1000,
     // 计算下一次重试退避的系数（默认：2.0）
    backoffCoefficient: 2.0,
     // 最大退避间隔（默认：initialInterval 的 100 倍）
    maximumInterval: null,
     // 最大重试次数（默认：0 表示无限制）
    maximumAttempts: 3,
     // 应用于间隔的最大抖动（默认：0.1）
    maximumJitterCoefficient: 0.1,
)
```

`RetryInterceptor` 会自动重试因以下状态码失败的请求：
- `ResourceExhausted`
- `Unavailable`
- `Unknown`
- `Aborted`

#### ConnectionsRotationInterceptor

轮换多个连接直到第一个成功响应：

```php
use Spiral\Grpc\Client\Interceptor\ConnectionsRotationInterceptor;

// 只需添加拦截器类
ConnectionsRotationInterceptor::class
```

#### ExecuteServiceInterceptors

这个特殊的拦截器调用在服务配置中定义的特定于服务的拦截器。
如果全局拦截器列表中不包含 `ExecuteServiceInterceptors` 拦截器，则不会执行特定于服务的拦截器。

```php
use Spiral\Grpc\Client\Interceptor\ExecuteServiceInterceptors;

// 需要执行特定于服务的拦截器
ExecuteServiceInterceptors::class
```

#### TracingInterceptor

`Spiral\RoadRunnerBridge\GRPC\Client\TracingInterceptor` 随 `spiral/roadrunner-bridge` 5.0 提供，并会自动添加：桥接器把它放在全局拦截器的第一位。
每个调用都会成为[遥测](../advanced/telemetry.md)跟踪器的 `CLIENT` span，名称和属性与服务器 span 相同，跟踪上下文通过
`traceparent` 元数据发送给服务器，因此两端处于同一个跟踪中。

放在第一位时，每个调用（包括重试）只有一个 span。如果需要每次尝试一个 span，请自行把它放在 `RetryInterceptor` 之后；桥接器会保留您选择的位置。
只把它放在全局列表中：如果同时加入某个服务的拦截器，每个调用会被包裹两次。

```php
use Spiral\Grpc\Client\Interceptor\RetryInterceptor;
use Spiral\RoadRunnerBridge\GRPC\Client\TracingInterceptor;

// 每次尝试一个 span
RetryInterceptor::createConfig(maximumAttempts: 3),
TracingInterceptor::class,
```

### 使用 gRPC 上下文字段

编写自定义客户端拦截器时，您经常需要访问或修改 gRPC 请求上下文的元素。
`Spiral\Grpc\Client\Interceptor\Helper` 类提供了安全操作这些上下文字段的方法：

```php
use Spiral\Grpc\Client\Interceptor\Helper;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final class AuthInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly string $authToken,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        // 获取当前元数据
        $metadata = Helper::getMetadata($context);

        // 在元数据中添加认证令牌
        $metadata['auth-token'] = [$this->authToken];

        // 用新元数据更新上下文
        $context = Helper::withMetadata($context, $metadata);

        // 继续请求流程
        return $handler->handle($context);
    }
}
```

`Helper` 类提供以下方法：
- `getConnections()`/`withConnections()` - 获取/设置可用连接
- `getCurrentConnection()`/`withCurrentConnection()` - 获取/设置当前连接
- `getMessage()`/`withMessage()` - 获取/设置请求消息
- `getMetadata()`/`withMetadata()` - 获取/设置请求元数据
- `getReturnType()`/`withReturnType()` - 获取/设置预期响应类型
- `getOptions()`/`withOptions()` - 获取/设置请求选项

### 拦截器顺序

拦截器的顺序很重要。拦截器按照它们在配置中定义的顺序执行。
例如：

```php
[
    SetTimoutInterceptor::createConfig(10_000), // 10 秒全局超时
    RetryInterceptor::createConfig(maxAttempts: 3), // 最多 3 次重试
    SetTimoutInterceptor::createConfig(3_000), // 每次尝试 3 秒超时
]
```

在此配置中：
1. 为整个请求（包括所有重试）设置 10 秒的全局超时
2. 重试拦截器将在需要时进行最多 3 次重试尝试
3. 每个单独的尝试有 3 秒的超时

`ExecuteServiceInterceptors` 拦截器通常应放在全局拦截器列表的最后，以确保特定于服务的拦截器在全局拦截器之后运行。

### 配置客户端拦截器

有关配置 gRPC 客户端及其拦截器的详细信息，请参阅 [客户端](./client.md) 文档。以下是一个基本示例：

```php app/config/grpc.php
use Spiral\Grpc\Client\Config\GrpcClientConfig;
use Spiral\Grpc\Client\Config\ServiceConfig;
use Spiral\Grpc\Client\Config\ConnectionConfig;
use Spiral\Grpc\Client\Interceptor\SetTimoutInterceptor;
use Spiral\Grpc\Client\Interceptor\RetryInterceptor;
use Spiral\Grpc\Client\Interceptor\ExecuteServiceInterceptors;

return [
    // ... 服务器配置 ...
    
    'client' => new GrpcClientConfig(
        interceptors: [
            // 全局拦截器
            SetTimoutInterceptor::createConfig(5_000),
            RetryInterceptor::createConfig(maximumAttempts: 3),
            // 调用特定于服务的拦截器
            ExecuteServiceInterceptors::class,
        ],
        services: [
            new ServiceConfig(
                connections: ConnectionConfig::createInsecure('localhost:9001'),
                interfaces: [
                    \GRPC\MyService\ServiceInterface::class,
                ],
                // 特定于服务的拦截器
                interceptors: [
                    SetTimoutInterceptor::createConfig(2_000), // 为此服务覆盖超时
                ],
            ),
        ],
    )
];
```