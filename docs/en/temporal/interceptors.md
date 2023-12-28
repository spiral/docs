# Temporal — Interceptors

Interceptors are a mechanism for modifying inbound and outbound SDK calls. Interceptors are commonly used to add tracing
and authorization to the scheduling and execution of Workflows and Activities. You can compare these to "middleware" in
other frameworks.

## Available Interceptors types

- `Temporal\Interceptor\WorkflowClientCallsInterceptor`: Intercepts methods of the WorkflowClient, such as starting or
  signaling a Workflow.
- `Temporal\Interceptor\WorkflowInboundCallsInterceptor`: Intercepts inbound Workflow calls, including execution,
  Signals, and Queries.
- `Temporal\Interceptor\WorkflowOutboundCallsInterceptor`: Intercepts outbound Workflow calls to Temporal APIs, such as
  scheduling Activities and starting Timers.
- `Temporal\Interceptor\ActivityInboundCallsInterceptor`: Intercepts inbound calls to an Activity, such as execute.
- `Temporal\Interceptor\GrpcClientInterceptor`: Intercepts all service client gRPC calls (see ServiceClientInterface).
- `Temporal\Interceptor\WorkflowOutboundRequestInterceptor`: Intercepts all commands sent to the RoadRunner server (see
  RequestInterface implementations).
- `Temporal\Exception\ExceptionInterceptorInterface`: Provides the ability to let workflow know if exception should be
  treated as fatal (stops execution) or must only fail the task execution (with consecutive retry).

There are also traits for each of the above interceptors, which you can use to implement your own interceptors.

> **Warning**
> Some interceptor interfaces will be expanded in the future. To avoid compatibility issues, always use the
> corresponding traits in your implementations. The traits will prevent implementations from breaking when new methods
> are added to the interfaces.

- `Temporal\Interceptor\Trait\Temporal\Interceptor\Trait`
- `Temporal\Interceptor\Trait\Temporal\Interceptor\WorkflowClientCallsInterceptorTrait`
- `Temporal\Interceptor\Trait\Temporal\Interceptor\WorkflowInboundCallsInterceptorTrait`
- `Temporal\Interceptor\Trait\Temporal\Interceptor\WorkflowOutboundCallsInterceptorTrait`
- `Temporal\Interceptor\Trait\Temporal\Interceptor\WorkflowOutboundRequestInterceptorTrait`

## Creating an Interceptor

To create an interceptor, you need to create a class and implement specific interface.

For example, let's create an interceptor that will configure activity options based on attributes.

```php app/Temporal/Interceptors/ActivityAttributesInterceptor.php
<?php

declare(strict_types=1);

namespace App\Temporal\Interceptors;

use React\Promise\PromiseInterface;
use ReflectionAttribute;
use Temporal\Interceptor\Trait\WorkflowOutboundCallsInterceptorTrait;
use Temporal\Interceptor\WorkflowOutboundCalls\ExecuteActivityInput;
use Temporal\Interceptor\WorkflowOutboundCallsInterceptor;
use Temporal\Samples\Interceptors\Attribute;
use Temporal\Samples\Interceptors\Attribute\ActivityOption;

/**
 * implement {@see ActivityOption} interface.
 */
final class ActivityAttributesInterceptor implements WorkflowOutboundCallsInterceptor
{
    use WorkflowOutboundCallsInterceptorTrait;

    public function executeActivity(ExecuteActivityInput $input, callable $next): PromiseInterface
    {
        if ($input->method === null) {
            return $next($input);
        }

        $options = $input->options;

        foreach ($this->iterateOptions($input->method) as $attribute) {
            if ($attribute instanceof Attribute\StartToCloseTimeout) {
                \error_log(\sprintf('Redeclare start_to_close timeout of %s to %s', $input->type, $attribute->timeout));
                $options = $options->withStartToCloseTimeout($attribute->timeout);
            }
        }

        return $next($input->with(options: $options));
    }

    /**
     * @return iterable<int, ActivityOption>
     */
    private function iterateOptions(\ReflectionMethod $method): iterable
    {
        $class = $method->getDeclaringClass();
        foreach ($class->getAttributes(Attribute\ActivityOption::class, ReflectionAttribute::IS_INSTANCEOF) as $attr) {
            yield $attr->newInstance();
        }

        foreach ($method->getAttributes(Attribute\ActivityOption::class, ReflectionAttribute::IS_INSTANCEOF) as $attr) {
            yield $attr->newInstance();
        }
    }
}
```

## Registering an Interceptor

### Via configuration file

To register an interceptor, you need to add it to the `config/temporal.php` configuration file.

```php
use App\Temporal\Interceptors\ActivityAttributesInterceptor;

return [
  // ...
  'interceptors' => [
    ActivityAttributesInterceptor::class,
  ],
];
```

You can define an interceptor as a class, instance or autowire:

```php
use Spiral\Core\Container\Autowire;
use App\Temporal\Interceptors\ActivityAttributesInterceptor;

return [
  'interceptors' => [
    ActivityAttributesInterceptor::class,
    
    new ActivityAttributesInterceptor(),
    
    new Autowire(ActivityAttributesInterceptor::class),
  ],
];
```

### Via bootloaders

You can also register an interceptor via a bootloader.

```php
namespace App\Application\Bootloader;

use App\Temporal\Interceptors\ActivityAttributesInterceptor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\TemporalBridge\Bootloader\TemporalBridgeBootloader;
use Spiral\Core\Container\Autowire;

class AppBootloader extends Bootloader
{
    public function init(TemporalBridgeBootloader $temporal): void
    {
        $temporal->addInterceptor(ActivityAttributesInterceptor::class);
        // or
        $temporal->addInterceptor(new ActivityAttributesInterceptor());
        // or
        $temporal->addInterceptor(new Autowire(ActivityAttributesInterceptor::class));
    }
}
```

## Exception Interceptor

Exception interceptor provides the ability to let workflow know if exception should be treated as fatal (stops
execution) or must only fail the task execution (with consecutive retry).

**Here is an example of an exception interceptor:**

```php app/Temporal/Interceptors/ExceptionInterceptor.php
<?php

declare(strict_types=1);

namespace App\Temporal\Interceptors;

use Temporal\Exception\ExceptionInterceptorInterface;

class ExceptionInterceptor implements ExceptionInterceptorInterface
{
    public function isRetryable(\Throwable $e): bool
    {
        if ($e instanceof ApiRateLimitException) {
            return true;
        }
        
        return false;
    }
}
```

**Registering an exception interceptor:**

```php app/config/temporal.php
use Temporal\Worker\WorkerOptions;
use App\Temporal\Interceptors\ExceptionInterceptor;

return [
    // ...
    'workers' => [
        'workerName' => [
            'exception_interceptor' => new ExceptionInterceptor(),
        ]
    ],
];
```