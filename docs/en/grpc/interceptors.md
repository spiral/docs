# GRPC — Interceptors

Spiral provides interceptors for gRPC services that allow you to intercept and modify requests and responses at various 
points in the request lifecycle.

> **See more**
> Read more about interceptors in the [Framework — Interceptors](../framework/interceptors.md) section.

There are two types of interceptors:

1. Server interceptors
2. Client interceptors

## Server Interceptors

Server interceptors are used to intercept and modify requests and responses received by a server. They are typically
used to add cross-cutting functionality such as logging, authentication, or monitoring to the server.

### Logging Interceptor

Here is an example of a simple interceptor that logs a request before and after processing:

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

### Exception Handler Interceptor

Here is an example of a simple interceptor that handles exceptions thrown by the server. It will catch all exceptions
and convert them to a gRPC exception.

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

### Tracing

You don't need an interceptor to trace incoming calls: since `spiral/roadrunner-bridge` 5.0, every call runs inside
a `SERVER` span of the [telemetry](../advanced/telemetry.md) tracer, and your interceptors run inside that span. The
trace continues from the `traceparent` metadata of the request, and the response headers carry the span context
back to the caller.

The span is named after the method, such as `ping.dto.PingService/Ping`, and carries the OpenTelemetry gRPC
attributes `rpc.system.name`, `rpc.method` and `rpc.response.status_code`. A call that ends with any status other than
`OK` is marked as an error, with the status in `error.type`; an exception that is not a `GRPCExceptionInterface`
puts its class there instead.

### Guard Interceptor

Here is an example of a simple interceptor that checks if the user is authenticated. It will use PHP attributes to
determine which methods require authentication. An authentication token is passed in the request metadata.

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
        // Metadata always stores values as array. 
        $token = $ctx->getValue($attribute->tokenField)[0] ?? null;

        // Here you can implement your own authentication logic
        if ($token !== 'secret') {
            throw new \Exception('Unauthorized.');
        }
    }
}
```

And example of a method that requires authentication:

```php
use App\Attribute\Guarded;

#[Guarded]
public function ping(GRPC\ContextInterface $ctx, PingRequest $in): PingResponse
{
    // ...
}
````

And example of Guarded attribute:

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

### Registering Server Interceptors

To use server interceptors, register them in the configuration file `app/config/grpc.php`:

```php app/config/grpc.php
return [    
    'interceptors' => [
        \App\Endpoint\GRPC\Interceptor\LogInterceptor::class,
        \App\Endpoint\GRPC\Interceptor\ExceptionHandlerInterceptor::class,
        \App\Endpoint\GRPC\Interceptor\GuardedInterceptor::class,
    ]
];
```

## Client Interceptors

Client interceptors allow you to modify or extend the behavior of gRPC client requests.
They can add functionality such as logging, timeout configuration, retries, and authentication.

> **See more**
> See the [Client](./client.md) documentation for basic gRPC client configuration and usage.

### Built-in Client Interceptors

Spiral provides several built-in client interceptors through the `spiral/grpc-client` package:

#### SetTimeoutInterceptor

Sets a timeout for each gRPC request in milliseconds:

```php
use Spiral\Grpc\Client\Interceptor\SetTimoutInterceptor;

// Adds a 5-second timeout
SetTimoutInterceptor::createConfig(5_000)
```

#### RetryInterceptor

Implements retry logic with exponential backoff for failed requests:

```php
use Spiral\Grpc\Client\Interceptor\RetryInterceptor;

// Configure retries with custom parameters
RetryInterceptor::createConfig(
     // Initial backoff interval in milliseconds (default: 50ms)
    initialInterval: 50,
     // Initial interval for resource exhaustion issues (default: 1000ms)
    congestionInitialInterval: 1000,
     // The coefficient for calculating next retry backoff (default: 2.0)
    backoffCoefficient: 2.0,
     // Maximum backoff interval (default: 100x of initialInterval)
    maximumInterval: null,
     // Maximum number of retry attempts (default: 0 means unlimited)
    maximumAttempts: 3,
     // Maximum jitter to apply to intervals (default: 0.1)
    maximumJitterCoefficient: 0.1,
)
```

The `RetryInterceptor` automatically retries requests that fail with one of these status codes:
- `ResourceExhausted`
- `Unavailable`
- `Unknown`
- `Aborted`

#### ConnectionsRotationInterceptor

Rotates through multiple connections until the first successful response:

```php
use Spiral\Grpc\Client\Interceptor\ConnectionsRotationInterceptor;

// Simply add the interceptor class
ConnectionsRotationInterceptor::class
```

#### ExecuteServiceInterceptors

This special interceptor calls service-specific interceptors defined in the service configuration.
If the `ExecuteServiceInterceptors` interceptor is not included in the global interceptors list, service-specific interceptors will not be executed.

```php
use Spiral\Grpc\Client\Interceptor\ExecuteServiceInterceptors;

// Required to execute service-specific interceptors
ExecuteServiceInterceptors::class
```

#### TracingInterceptor

`Spiral\RoadRunnerBridge\GRPC\Client\TracingInterceptor` comes with `spiral/roadrunner-bridge` 5.0 and is added
for you: the bridge puts it first among the global interceptors. Each call becomes a `CLIENT` span of the
[telemetry](../advanced/telemetry.md) tracer, with the same name and attributes as a server span, and the trace
context goes to the server in the `traceparent` metadata, so both sides end up in one trace.

Being first, it makes one span per call, retries included. For a span per attempt, list it yourself after
`RetryInterceptor`; the bridge keeps the position you chose. Keep it in the global list only: added to a service's
interceptors as well, it wraps each call twice.

```php
use Spiral\Grpc\Client\Interceptor\RetryInterceptor;
use Spiral\RoadRunnerBridge\GRPC\Client\TracingInterceptor;

// A span per attempt
RetryInterceptor::createConfig(maximumAttempts: 3),
TracingInterceptor::class,
```

### Working with gRPC Context Fields

When writing custom client interceptors, you'll often need to access or modify elements of the gRPC request context.
The `Spiral\Grpc\Client\Interceptor\Helper` class provides methods to safely work with these context fields:

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
        // Get current metadata
        $metadata = Helper::getMetadata($context);

        // Add authentication token to metadata
        $metadata['auth-token'] = [$this->authToken];

        // Update context with new metadata
        $context = Helper::withMetadata($context, $metadata);

        // Continue the request pipeline
        return $handler->handle($context);
    }
}
```

The `Helper` class provides the following methods:
- `getConnections()`/`withConnections()` - Get/set available connections
- `getCurrentConnection()`/`withCurrentConnection()` - Get/set current connection
- `getMessage()`/`withMessage()` - Get/set request message
- `getMetadata()`/`withMetadata()` - Get/set request metadata
- `getReturnType()`/`withReturnType()` - Get/set expected response type
- `getOptions()`/`withOptions()` - Get/set request options

### Order of Interceptors

The order of interceptors matters. Interceptors are executed in the order they are defined in the configuration.
For example:

```php
[
    SetTimoutInterceptor::createConfig(10_000), // 10 second global timeout
    RetryInterceptor::createConfig(maxAttempts: 3), // Up to 3 retries
    SetTimoutInterceptor::createConfig(3_000), // 3 second per-attempt timeout
]
```

In this configuration:
1. A 10-second global timeout is set for the entire request (including all retries)
2. The retry interceptor will make up to 3 retry attempts if needed
3. Each individual attempt has a 3-second timeout

The `ExecuteServiceInterceptors` interceptor should typically be placed last in the global interceptors list to ensure that service-specific interceptors run after global ones.

### Configuring Client Interceptors

For detailed information on configuring the gRPC client and its interceptors, see the [Client](./client.md) documentation. Below is a basic example:

```php app/config/grpc.php
use Spiral\Grpc\Client\Config\GrpcClientConfig;
use Spiral\Grpc\Client\Config\ServiceConfig;
use Spiral\Grpc\Client\Config\ConnectionConfig;
use Spiral\Grpc\Client\Interceptor\SetTimoutInterceptor;
use Spiral\Grpc\Client\Interceptor\RetryInterceptor;
use Spiral\Grpc\Client\Interceptor\ExecuteServiceInterceptors;

return [
    // ... server configuration ...
    
    'client' => new GrpcClientConfig(
        interceptors: [
            // Global interceptors
            SetTimoutInterceptor::createConfig(5_000),
            RetryInterceptor::createConfig(maximumAttempts: 3),
            // Calls service-specific interceptors
            ExecuteServiceInterceptors::class,
        ],
        services: [
            new ServiceConfig(
                connections: ConnectionConfig::createInsecure('localhost:9001'),
                interfaces: [
                    \GRPC\MyService\ServiceInterface::class,
                ],
                // Service-specific interceptors
                interceptors: [
                    SetTimoutInterceptor::createConfig(2_000), // Override timeout for this service
                ],
            ),
        ],
    )
];
```
