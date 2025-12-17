# Queue — Job Handlers

Job handlers are an essential part of Spiral, providing a structured approach to executing jobs efficiently and managing
their payloads. Job handlers are classes responsible for performing specific tasks or actions within the system.

## Create Handler

To create a job handler, you need to implement the `Spiral\Jobs\HandlerInterface` interface. This interface defines the
required methods for handling jobs, executing the job, and handling any potential errors that may occur during the
process. Spiral also provides a convenient abstract class called `Spiral\Queue\JobHandler` that you can extend to
simplify the implementation of your job handlers.

To create a job handler effortlessly, use can the scaffolding command:

```terminal
php app.php create:jobHandler Sample
```

> **Note**
> Read more about scaffolding in the [Basics — Scaffolding](../basics/scaffolding.md#job-handler) section.

After executing this command, the following output will confirm the successful creation:

```output
Declaration of '[32mSampleJob[39m' has been successfully written into '[33mapp/src/Endpoint/Job/SampleJob.php[39m'.
```

Now you can find the `SampleJob` class in the `app/src/Endpoint/Job` directory.

```php app/src/Endpoint/Job/SampleJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class SampleJob extends JobHandler
{
    public function invoke(string $id, array $payload, array $headers): void
    {
        // Do something with service
    }
}
```

Currently, a new job handler doesn't perform any actions.

## Dispatch Job

### Pushing to the default queue

You can dispatch your job via `Spiral\Queue\QueueInterface` or via the prototype property `queue`. When you request
the `Spiral\Queue\QueueInterface` from the container, you will receive an instance of the default queue connection.

The method `push` of `QueueInterface` accepts a job name, the payload, and additional options.

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueInterface;

public function createJob(QueueInterface $queue): void
{
    $queue->push(SampleJob::class);
}
``` 

You can use your handler name as the job name. It will be automatically converted into `-` identifier, for example,
`App\Endpoint\Job\SampleJob` will be presented as `app-jobs-sampleJob`.

### Pushing to a specific queue

If you need to push a job using a specific queue connection, you can
use `Spiral\Queue\QueueConnectionProviderInterface`.

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueConnectionProviderInterface;

final class MyService
{
    public function __construct(
        private readonly QueueConnectionProviderInterface $provider
    ) {
    }

    public function createJob(): void
    {
        $this->provider->getConnection('sync')->push(SampleJob::class);
    }
}
```

## Passing Parameters

The second argument of the `QueueInterface->push()` method can accept any type of variable, such as **arrays**, *
*objects**, **strings**, etc. However, it's important to note that the default serializer used by the framework
is `json`.

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueInterface;

public function createJob(QueueInterface $queue): void
{
    // Array payload
    $queue->push(SampleJob::class, ['value' => 123]);
    
    // Object payload
    $queue->push(SampleJob::class, new User(id: 123, name: 'John'));
    
    // Some strig payload
    $queue->push(SampleJob::class, 'some string');
}
```

## Handling Jobs

When the job is dispatched, the queue service will automatically find the handler for the job and execute it.
The `invoke` method is responsible for handling the queued tasks that are received by the job handler.

```php app/src/Endpoint/Job/SampleJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

class SampleJob extends JobHandler
{
    public function invoke(string $id, array $payload): void
    {
        // Do something with service
    }
}
```

The method accepts a number of arguments, which are described below:

#### Payload

The `$payload` argument contains the data that was added to the queue when the task was queued. This can be
of any type, such as an `array`, `object`, `string`, etc.

> **Warning**
> The `payload` parameter should have the same type as the payload you passed to the `push` method.

#### Task ID

The `$id` argument is an optional string that contains the unique identifier for the job. This can be used to track the
job's progress within the application.

#### Task Headers

The `$headers` argument is an optional array of additional headers or context that can be
added when the task is pushed to the queue. This can be useful for providing additional information about the task or
for passing context data to the invoke method.

**Some examples of context data that can be added to the headers include:**

- **Retry Attempts:** The number of times the task has been retried. This can be useful for determining whether the task
  has failed multiple times and needs to be handled differently.

- **Priority:** The priority level of the task. This can be useful for ensuring that important tasks are handled first,
  or for prioritizing tasks based on their importance.

- **Timestamp:** The timestamp when the task was added to the queue. This can be useful for tracking the progress of the
  task or for logging purposes.

- **User ID:** The ID of the user who initiated the task. This can be useful for tracking the actions of individual
  users or for enforcing user-specific policies.

#### Dependency Injection

You can freely use the method injection in your handler's `invoke` method. When the job handler is called, the
dependency injection container will automatically provide the specified dependencies to the method.

```php app/src/Endpoint/Job/SampleJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;
use Psr\Log\LoggerInterface;

class SampleJob extends JobHandler
{
    public function invoke(LoggerInterface $logger, array $payload): void
    {
        $logger->debug('Job processing...', ['id' => $id]);
        
        // Do something with service
        
        $logger->debug('Job processed', ['id' => $id]);
    }
}
```

> **Note**
> Define handlers as singletons for better performance.

It's important to note that the `invoke` method must always have a `void` return type, as it does not return any value.

## Job Payload serialization

The queue component supports the use of a serializer for converting objects to and from a serialized form suitable for
storage in a queue. This allows you to easily enqueue and dequeue complex objects without having to manually serialize
and deserialize them.

> **See more**
> The [Serializer component](../advanced/serializer.md) is used to serialize the job payload when it is added to the
> queue and deserialize when it is retrieved from the queue and passed to a job handler for processing.

### Configure default serializer

The default serializer for the queue component can be specified via the `queue.php` configuration file.

**Example:**

```php app/config/queue.php
use Spiral\Core\Container\Autowire;
use Spiral\Serializer\Serializer\JsonSerializer;
use Spiral\Serializer\Serializer\PhpSerializer;

return [
    // via serializer name
    'defaultSerializer' => 'json',

    // via class name
    'defaultSerializer' => JsonSerializer::class,
    
    // via instance
    'defaultSerializer' => new JsonSerializer(),
    
    // via Autowire
    'defaultSerializer' => new Autowire(PhpSerializer::class)
];
```

> **Note**
> This allows you to easily customize the serialization strategy for the queue and choose the approach that best fits
> your needs. Read more about available serializers in the [Component — Serializer](../advanced/serializer.md).

### Changing serializer

There are several ways to change the serializer. You can globally change the default serializer for the application.
Or you can set a specific serializer for the job type. A specific serializer is selected by
the `Spiral\Serializer\SerializerRegistryInterface`.

You can configure the serializer for a specific job type using `Spiral\Queue\Attribute\Serializer` attribute:

```php
use Spiral\Queue\Attribute\Serializer;
use Spiral\Queue\JobHandler;

#[Serializer('json')]
final class Ping extends JobHandler
{
    public function invoke(array $payload): void
    {
        // ...
    }
}
```

or using `app/config/queue.php` configuration file.

```php app/config/queue.php
use Spiral\Core\Container\Autowire;

return [
    'registry' => [
        'serializers' => [
            'ping.job' => 'json',
            TestJob::class => 'serializer',
            OtherJob::class => CustomSerializer::class,
            FooJob::class => new CustomSerializer(),
            BarJob::class => new Autowire(CustomSerializer::class),
        ]
    ],
];
```

Or, register a serializer using the `setSerializer` method of the `Spiral\Queue\QueueRegistry` class.

```php app/src/ApplicationBootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container\Autowire;
use Spiral\Queue\QueueRegistry;

class AppBootloader extends Bootloader
{
    public function boot(QueueRegistry $registry): void
    {
        $registry->setSerializer('ping.job', 'json');
        $registry->setSerializer(TestJob::class, 'serializer');
        $registry->setSerializer(OtherJob::class, CustomSerializer::class);
        $registry->setSerializer(FooJob::class, new CustomSerializer());
        $registry->setSerializer(BarJob::class, new Autowire(CustomSerializer::class));
    }
}
```

## Job handler registry

If you don't want to use the job handler class name as the queue job name as in the example below:

```php
use Spiral\Queue\QueueInterface;

public function createJob(QueueInterface $queue): void
{
    $queue->push('sample::job');
}
```

you need to tell the queue how to handle a job with the name `sample::job`.

You can do it via attribute `Spiral\Queue\Attribute\JobHandler`:

```php
use Spiral\Queue\Attribute\JobHandler as Handler;
use Spiral\Queue\JobHandler;

#[Handler('sample::job')]
final class Ping extends JobHandler
{
    public function invoke(array $payload): void
    {
        // ...
    }
}
```

or via the `app/config/queue.php` config:

```php app/config/queue.php
return [
    'registry' => [
        'handlers' => [
            'sample::job' => App\Endpoint\Job\SampleJob::class
        ],
    ],
];
```

or via `Spiral\Queue\QueueRegistry`:

```php
use Spiral\Boot\Bootloader\Bootloader;

class MyBootloader extends Bootloader
{
    public function boot(\Spiral\Queue\QueueRegistry $registry): void
    {
        $registry->setHandler('sample::job', \App\Endpoint\Job\SampleJob::class);
    }
}
```

## Job Options

The `Spiral\Queue\Options` class allows you to specify additional context for a job when pushing it to a queue using the
`QueueInterface::push()` method.

#### `withHeader(string $name, string|array $value)`

This method allows you to set a header value for the job. Headers can be used to pass additional metadata about the job
to the consumer server.

```php
$options = new \Spiral\Queue\Options();

$queue->push(
    SampleJob::class, 
    ['value' => 123], 
    $options->withHeader('user_id', 123)
);
```

#### `withQueue(?string $queue)`

This method allows you to specify the name of the queue to which the job should be pushed. If no queue is specified,
the job will be pushed to the default queue.

```php
$options = new \Spiral\Queue\Options();

$queue->push(
    SampleJob::class, 
    ['value' => 123], 
    $options->withQueue('high_priority')
);
```

#### `withDelay(?int $delay)`

This method allows you to specify a delay in seconds before the job will be available for processing. If no delay is
specified, the job will be processed after a default delay period.

```php
$options = new Options();

$queue->push(
    SampleJob::class, 
    ['value' => 123], 
    $options->withDelay(3600) // job will be available for processing in 1 hour
);
```

## Handle failed jobs

By default, all failed jobs will be sent into the spiral log. But you can change the default behavior. At first, you
need to create your own implementation for `Spiral\Queue\Failed\FailedJobHandlerInterface`.

### Custom handler example

```php app/src/Infrastructure/Queue/DatabaseFailedJobsHandler.php
namespace App\Infrastructure\Queue;

use Spiral\Queue\Failed\FailedJobHandlerInterface;
use Cycle\Database\DatabaseInterface;
use Spiral\Queue\SerializerInterface;

class DatabaseFailedJobsHandler implements FailedJobHandlerInterface
{
    private DatabaseInterface $database;
    private SerializerInterface $serializer;
    
    public function __construct(DatabaseInterface $database, SerializerInterface $serializer)
    {
        $this->database = $database;
        $this->serializer = $serializer;
    }

    public function handle(string $driver, string $queue, string $job, array $payload, \Throwable $e): void
    {
        $this->database
            ->insert('failed_jobs')
            ->values([
                'driver' => $driver,
                'queue' => $queue,
                'job_name' => $job,
                'payload' => $this->serializer->serialize($payload),
                'error' => $e->getMessage(),
            ])
            ->run();
    }
}
```

Then you need to bind your implementation to the `Spiral\Queue\Failed\FailedJobHandlerInterface` interface.

```php app/src/ApplicationBootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\RoadRunnerBridge\Queue\Failed\FailedJobHandlerInterface;

final class QueueFailedJobsBootloader extends Bootloader
{
    protected const SINGLETONS = [
        FailedJobHandlerInterface::class => \App\Infrastructure\Queue\DatabaseFailedJobsHandler::class,
    ];
}
```

And register this bootloader after `QueueFailedJobsBootloader` in your application

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\QueueFailedJobsBootloader::class,
        // ...
    ];
}
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\QueueFailedJobsBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

## Retrying Failed Jobs

In distributed systems, jobs often fail due to temporary issues like network timeouts, rate limits, or database
deadlocks. Spiral's retry mechanism ensures these jobs are automatically retried without data loss.

### Quick Start

Add the `RetryPolicy` attribute to your job handler:

```php app/src/Endpoint/Job/SendEmailJob.php
use Spiral\Queue\Attribute\RetryPolicy;
use Spiral\Queue\JobHandler;

#[RetryPolicy(maxAttempts: 3, delay: 5, multiplier: 2)]
final class SendEmailJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        // If this throws an exception, the job will be retried
        $this->mailer->send($payload['email'], $payload['message']);
    }
}
```

This configuration will retry the job up to 3 times with exponential backoff:

- First retry: after 5 seconds
- Second retry: after 10 seconds (5 * 2^1)
- Third retry: after 20 seconds (5 * 2^2)

### How Retries Work

The retry mechanism consists of three components:

1. **RetryPolicyInterceptor** — Catches exceptions during job execution and evaluates retry policies
2. **Retry Policy** — Determines whether to retry and calculates delay
3. **Queue Driver** — Re-queues the job with updated options

When a job fails:

1. The interceptor catches the exception
2. It checks for a retry policy (from attribute or exception)
3. If retryable, it increments the attempt counter
4. It calculates the delay based on the policy
5. It throws a `RetryException` with the new delay
6. The consumer re-queues the job with the updated delay and attempt count

### Configuration

The `RetryPolicyInterceptor` is enabled by default. Verify it's configured in your `app/config/queue.php`:

```php app/config/queue.php
use Spiral\Queue\Interceptor\Consume\RetryPolicyInterceptor;
use Spiral\Queue\Interceptor\Consume\ErrorHandlerInterceptor;

return [    
    'interceptors' => [
        'consume' => [
            ErrorHandlerInterceptor::class,    // Handles failed jobs
            RetryPolicyInterceptor::class,     // Handles retries
        ],
    ],
];
```

> **See more**
> Read more about interceptors in the [Queue — Interceptors](./interceptors.md) section.

### Retry Methods

Spiral provides multiple ways to configure retry behavior:

#### Using RetryPolicy Attribute

The `RetryPolicy` attribute is the recommended approach for most use cases:

```php
use Spiral\Queue\Attribute\RetryPolicy;
use Spiral\Queue\JobHandler;

#[RetryPolicy(maxAttempts: 5, delay: 10, multiplier: 2.0)]
final class ApiCallJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        $this->api->call($payload['endpoint']);
    }
}
```

**Parameters:**

- `maxAttempts` — Maximum number of retry attempts (0 disables retries)
- `delay` — Initial delay in seconds between retries
- `multiplier` — Exponential backoff multiplier (1.0 for constant delay)

#### Using RetryableExceptionInterface

For fine-grained control based on exception types:

```php app/src/Exception/ApiException.php
use Spiral\Queue\Exception\RetryableExceptionInterface;
use Spiral\Queue\RetryPolicy;
use Spiral\Queue\RetryPolicyInterface;

final class ApiException extends \RuntimeException implements RetryableExceptionInterface
{
    public function isRetryable(): bool
    {
        // Only retry server errors (5xx)
        return $this->getCode() >= 500 && $this->getCode() < 600;
    }

    public function getRetryPolicy(): ?RetryPolicyInterface
    {
        return new RetryPolicy(
            maxAttempts: 5,
            delay: 10,
            multiplier: 2.0
        );
    }
}
```

Then use it in your job:

```php app/src/Endpoint/Job/ApiCallJob.php
use Spiral\Queue\JobHandler;

final class ApiCallJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        try {
            $response = $this->api->call($payload['endpoint']);
        } catch (\Exception $e) {
            // Transform to retryable exception
            throw new ApiException(
                $e->getMessage(),
                $e->getCode(),
                $e
            );
        }
    }
}
```

#### Using RetryException

For manual retry control with custom options:

```php app/src/Endpoint/Job/RateLimitedJob.php
use Spiral\Queue\Exception\RetryException;
use Spiral\Queue\Options;
use Spiral\Queue\JobHandler;

final class RateLimitedJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        if ($this->api->isRateLimited()) {
            throw new RetryException(
                reason: 'Rate limit exceeded',
                options: (new Options())->withDelay(300) // Retry in 5 minutes
            );
        }
        
        $this->api->call($payload['endpoint']);
    }
}
```

### Tracking Retry Attempts

Access the current attempt number via job headers:

```php
use Spiral\Queue\JobHandler;

final class ProgressiveJob extends JobHandler
{
    public function invoke(array $payload, array $headers): void
    {
        $attempt = (int) ($headers['attempts'][0] ?? 0);
        
        $this->logger->info('Processing job', [
            'attempt' => $attempt + 1,
            'payload' => $payload,
        ]);
        
        // Adjust strategy based on attempt
        if ($attempt >= 2) {
            $this->processCautiously($payload);
        } else {
            $this->process($payload);
        }
    }
}
```

### Best Practices

**Make Handlers Idempotent**

Ensure handlers can be safely retried:

```php
final class ProcessOrderJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        $orderId = $payload['orderId'];
        
        // Check if already processed
        if ($this->orders->isProcessed($orderId)) {
            return; // Skip duplicate
        }
        
        // Process and mark atomically
        $this->orders->processAndMark($orderId);
    }
}
```

**Log Retry Context**

Track retry behavior for monitoring:

```php
public function invoke(array $payload, string $id, array $headers): void
{
    $attempt = (int) ($headers['attempts'][0] ?? 0);
    
    if ($attempt > 0) {
        $this->logger->warning('Job retry', [
            'job_id' => $id,
            'attempt' => $attempt + 1,
            'max_attempts' => 3,
        ]);
    }
    
    try {
        $this->process($payload);
    } catch (\Exception $e) {
        $this->logger->error('Job failed', [
            'job_id' => $id,
            'error' => $e->getMessage(),
        ]);
        throw $e;
    }
}
```

> **See more**
> For understanding retry lifecycle, see [Queue — Job Lifecycle](./lifecycle.md).

## Events

The Queue component dispatches events at key points during job processing, enabling monitoring, logging, and custom
processing logic.

### Available Events

| Event                              | Description                          | Dispatched When           |
|------------------------------------|--------------------------------------|---------------------------|
| `Spiral\Queue\Event\JobProcessing` | Job execution is about to start      | Before handler invocation |
| `Spiral\Queue\Event\JobProcessed`  | Job execution completed successfully | After handler completes   |

### JobProcessing Event

Dispatched immediately before a job handler is invoked.

```php
use Spiral\Queue\Event\JobProcessing;
use Psr\EventDispatcher\ListenerProviderInterface;

final class JobStartedListener
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}
    
    public function __invoke(JobProcessing $event): void
    {
        $this->logger->info('Job processing started', [
            'job' => $event->name,
            'id' => $event->id,
            'queue' => $event->queue,
            'driver' => $event->driver,
            'payload' => $event->payload,
            'headers' => $event->headers,
        ]);
    }
}
```

### JobProcessed Event

Dispatched after a job handler completes successfully.

```php
use Spiral\Queue\Event\JobProcessed;

final class JobCompletedListener
{
    public function __construct(
        private readonly MetricsInterface $metrics
    ) {}
    
    public function __invoke(JobProcessed $event): void
    {
        $this->metrics->increment('jobs.completed', [
            'job' => $event->name,
            'queue' => $event->queue,
        ]);
    }
}
```

### Registering Event Listeners

Register event listeners in your bootloader:

```php
use Spiral\Boot\Bootloader\Bootloader;
use Psr\EventDispatcher\ListenerProviderInterface;
use Spiral\Queue\Event\JobProcessing;
use Spiral\Queue\Event\JobProcessed;

final class QueueEventsBootloader extends Bootloader
{
    public function boot(ListenerProviderInterface $provider): void
    {
        $provider->listen(JobProcessing::class, JobStartedListener::class);
        $provider->listen(JobProcessed::class, JobCompletedListener::class);
    }
}
```

> **See more**
> For understanding when events are dispatched, see [Queue — Job Lifecycle](./lifecycle.md).
> To learn more about the event system, see [Events](../advanced/events.md).
