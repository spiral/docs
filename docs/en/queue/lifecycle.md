# Queue — Job Lifecycle

Understanding the complete lifecycle of a queued job helps you build reliable asynchronous processing systems. This guide explains how jobs flow through the system from creation to completion.

## Overview

A job passes through several stages during its lifecycle:

1. **Creation** — Job is instantiated with payload data
2. **Push** — Job enters the push interceptor pipeline
3. **Serialization** — Payload is serialized for storage
4. **Queue Storage** — Job is stored in the queue broker
5. **Retrieval** — Consumer retrieves job from queue
6. **Deserialization** — Payload is deserialized back to original form
7. **Consume** — Job enters the consume interceptor pipeline
8. **Execution** — Handler processes the job
9. **Completion** — Job finishes successfully or fails

## Push Phase

When you push a job to the queue, it goes through the following steps:

### 1. Queue Interface Call

```php
use App\Endpoint\Job\SendEmailJob;
use Spiral\Queue\QueueInterface;
use Spiral\Queue\Options;

public function sendWelcomeEmail(QueueInterface $queue, User $user): string
{
    $jobId = $queue->push(
        SendEmailJob::class,
        ['userId' => $user->id, 'template' => 'welcome'],
        Options::onQueue('emails')->withDelay(60)
    );
    
    return $jobId; // Unique job identifier
}
```

### 2. Push Interceptor Pipeline

The job enters the push interceptor chain. Each interceptor can:

- Modify job payload or options
- Add headers or metadata
- Log push operations
- Validate job data
- Cancel push by throwing exceptions

```php
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final class JobValidationInterceptor implements InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $args = $context->getArguments();
        
        // Validate payload before pushing
        if (!isset($args['payload']['userId'])) {
            throw new \InvalidArgumentException('userId is required');
        }
        
        // Continue to next interceptor
        return $handler->handle($context);
    }
}
```

### 3. Serialization

The payload is serialized using the configured serializer:

```php
use Spiral\Queue\Attribute\Serializer;
use Spiral\Queue\JobHandler;

#[Serializer('json')] // Use JSON serializer for this job
final class SendEmailJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        // Payload is automatically deserialized
    }
}
```

> See [Job Handlers — Serialization](./jobs.md#job-payload-serialization) for serializer configuration.

### 4. Queue Storage

The serialized job is sent to the queue broker (RoadRunner, Redis, etc.) with metadata:

- Job name/type
- Unique job ID
- Queue name
- Delay time
- Headers
- Serialized payload

## Consume Phase

When a consumer processes jobs from the queue:

### 1. Job Retrieval

The queue driver retrieves the next available job from the broker. Jobs are processed in order unless priority queues are configured.

### 2. Deserialization

The serialized payload is converted back to its original form using the configured serializer.

### 3. Event: JobProcessing

Before job execution, the `JobProcessing` event is dispatched:

```php
use Spiral\Queue\Event\JobProcessing;
use Psr\EventDispatcher\ListenerProviderInterface;

final class JobProcessingListener
{
    public function __invoke(JobProcessing $event): void
    {
        // Access job information
        $jobName = $event->name;      // Job class name
        $jobId = $event->id;          // Unique identifier
        $queue = $event->queue;       // Queue name
        $driver = $event->driver;     // Driver name (e.g., 'roadrunner')
        $payload = $event->payload;   // Job payload
        $headers = $event->headers;   // Job headers
        
        // Log job start
        $this->logger->info('Job processing started', [
            'job' => $jobName,
            'id' => $jobId,
        ]);
    }
}
```

### 4. Consume Interceptor Pipeline

The job enters the consume interceptor chain. Built-in interceptors include:

#### ErrorHandlerInterceptor

Catches exceptions and sends failed jobs to the failed job handler:

```php
namespace Spiral\Queue\Interceptor\Consume;

final class ErrorHandlerInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly FailedJobHandlerInterface $handler
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        try {
            return $handler->handle($context);
        } catch (\Throwable $e) {
            if (!$e instanceof StateException) {
                // Log to failed jobs
                $args = $context->getArguments();
                $this->handler->handle(
                    $args['driver'],
                    $args['queue'],
                    $context->getTarget()->getPath()[0],
                    $args['payload'],
                    $e
                );
            }
            throw $e;
        }
    }
}
```

#### RetryPolicyInterceptor

Handles job retries based on configured policies:

```php
namespace Spiral\Queue\Interceptor\Consume;

final class RetryPolicyInterceptor implements CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        try {
            return $core->callAction($controller, $action, $parameters);
        } catch (\Throwable $e) {
            $policy = $this->getRetryPolicy($e, new \ReflectionClass($controller));
            
            if ($policy === null) {
                throw $e;
            }
            
            $headers = $parameters['headers'] ?? [];
            $attempts = (int) ($headers['attempts'][0] ?? 0);
            
            if (!$policy->isRetryable($e, $attempts)) {
                throw $e;
            }
            
            // Retry the job
            throw new RetryException(
                reason: $e->getMessage(),
                options: (new Options())
                    ->withDelay($policy->getDelay($attempts))
                    ->withHeader('attempts', (string) ($attempts + 1))
            );
        }
    }
}
```

> See [Interceptors](./interceptors.md) for detailed interceptor documentation.

### 5. Handler Execution

The job handler's `invoke` method is called with the deserialized payload:

```php
use Spiral\Queue\JobHandler;
use Psr\Log\LoggerInterface;

final class SendEmailJob extends JobHandler
{
    public function invoke(
        array $payload,
        string $id,
        array $headers,
        LoggerInterface $logger,
        MailerInterface $mailer
    ): void {
        $logger->info('Sending email', ['userId' => $payload['userId']]);
        
        $user = $this->userRepository->find($payload['userId']);
        $mailer->send($user->email, $payload['template']);
    }
}
```

The handler can:
- Process the job successfully
- Throw exceptions to trigger retry logic
- Use dependency injection for services

### 6. Event: JobProcessed

After successful execution, the `JobProcessed` event is dispatched:

```php
use Spiral\Queue\Event\JobProcessed;

final class JobProcessedListener
{
    public function __invoke(JobProcessed $event): void
    {
        $this->logger->info('Job completed', [
            'job' => $event->name,
            'id' => $event->id,
        ]);
        
        // Track job metrics
        $this->metrics->increment('jobs.completed', [
            'job' => $event->name,
            'queue' => $event->queue,
        ]);
    }
}
```

## Exception Handling

Jobs can throw different types of exceptions that affect lifecycle behavior:

### StateException

Base exception for job state changes. Not reported as failures:

```php
use Spiral\Queue\Exception\StateException;

final class JobSkippedException extends StateException {}

// In handler
if ($this->isDuplicate($payload)) {
    throw new JobSkippedException('Job already processed');
}
```

### RetryException

Signals that the job should be retried with new options:

```php
use Spiral\Queue\Exception\RetryException;
use Spiral\Queue\Options;

if ($this->apiService->isRateLimited()) {
    throw new RetryException(
        reason: 'Rate limited',
        options: (new Options())->withDelay(300) // Retry in 5 minutes
    );
}
```

### FailException

Marks job as permanently failed (extends StateException):

```php
use Spiral\Queue\Exception\FailException;

if (!$user) {
    throw new FailException('User not found');
}
```

### RetryableExceptionInterface

Custom exceptions can implement this interface for retry control:

```php
use Spiral\Queue\Exception\RetryableExceptionInterface;
use Spiral\Queue\RetryPolicy;
use Spiral\Queue\RetryPolicyInterface;

final class ApiException extends \RuntimeException implements RetryableExceptionInterface
{
    public function isRetryable(): bool
    {
        // Only retry on 5xx errors
        return $this->getCode() >= 500;
    }
    
    public function getRetryPolicy(): ?RetryPolicyInterface
    {
        return new RetryPolicy(
            maxAttempts: 5,
            delay: 10,
            multiplier: 2.0 // Exponential backoff
        );
    }
}
```

> See [Retry Policies](./retries.md) for detailed retry documentation.

## Retry Lifecycle

When a job is retried:

1. **Exception Thrown** — Handler throws retryable exception
2. **Policy Check** — RetryPolicyInterceptor evaluates retry policy
3. **Attempt Tracking** — Attempt count is incremented in headers
4. **Delay Calculation** — Delay is calculated based on policy
5. **Re-queue** — Job is pushed back to queue with delay
6. **Wait Period** — Job sits in queue for delay duration
7. **Retrieval** — Job is retrieved again after delay
8. **Retry Execution** — Job goes through consume pipeline again

Example with exponential backoff:

```php
use Spiral\Queue\Attribute\RetryPolicy;
use Spiral\Queue\JobHandler;

#[RetryPolicy(maxAttempts: 5, delay: 10, multiplier: 2.0)]
final class ApiCallJob extends JobHandler
{
    public function invoke(array $payload, array $headers): void
    {
        $attempt = (int) ($headers['attempts'][0] ?? 0);
        
        // Attempt 1: Delay 10s
        // Attempt 2: Delay 20s (10 * 2^1)
        // Attempt 3: Delay 40s (10 * 2^2)
        // Attempt 4: Delay 80s (10 * 2^3)
        // Attempt 5: Delay 160s (10 * 2^4)
        
        try {
            $this->api->call($payload['endpoint']);
        } catch (ApiException $e) {
            // Will be retried up to 5 times
            throw $e;
        }
    }
}
```

## Failed Job Handling

When a job fails permanently (no more retries or non-retryable exception):

1. **ErrorHandlerInterceptor** catches the exception
2. **Failed Job Handler** is called
3. **Job is Logged** (default behavior)
4. **Exception is Re-thrown** to mark job as failed

Custom failed job handling:

```php
use Spiral\Queue\Failed\FailedJobHandlerInterface;
use Cycle\Database\DatabaseInterface;

final class DatabaseFailedJobHandler implements FailedJobHandlerInterface
{
    public function __construct(
        private readonly DatabaseInterface $db,
        private readonly SerializerInterface $serializer
    ) {}
    
    public function handle(
        string $driver,
        string $queue,
        string $job,
        mixed $payload,
        \Throwable $e
    ): void {
        $this->db->insert('failed_jobs')->values([
            'driver' => $driver,
            'queue' => $queue,
            'job' => $job,
            'payload' => $this->serializer->serialize($payload),
            'exception' => $e->getMessage(),
            'failed_at' => new \DateTimeImmutable(),
        ])->run();
    }
}
```

## Lifecycle Monitoring

Track job lifecycle with events and interceptors:

```php
final class JobLifecycleMonitor
{
    public function __construct(
        private readonly MetricsInterface $metrics
    ) {}
    
    public function onJobProcessing(JobProcessing $event): void
    {
        $this->metrics->increment('jobs.started', [
            'job' => $event->name,
            'queue' => $event->queue,
        ]);
        
        $this->metrics->gauge('jobs.queue_depth', 
            $this->getQueueDepth($event->queue)
        );
    }
    
    public function onJobProcessed(JobProcessed $event): void
    {
        $this->metrics->increment('jobs.completed', [
            'job' => $event->name,
            'queue' => $event->queue,
        ]);
    }
}
```

## Best Practices

**Idempotent Handlers**

Design handlers to be safely retried:

```php
final class ProcessOrderJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        $orderId = $payload['orderId'];
        
        // Check if already processed
        if ($this->orders->isProcessed($orderId)) {
            return; // Skip duplicate processing
        }
        
        // Process order
        $this->orders->process($orderId);
        
        // Mark as processed
        $this->orders->markProcessed($orderId);
    }
}
```

**Graceful Degradation**

Handle partial failures gracefully:

```php
final class SendNotificationsJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        $userIds = $payload['userIds'];
        $failed = [];
        
        foreach ($userIds as $userId) {
            try {
                $this->sendNotification($userId);
            } catch (\Exception $e) {
                $failed[] = $userId;
            }
        }
        
        // Retry only failed notifications
        if (!empty($failed)) {
            $this->queue->push(self::class, ['userIds' => $failed]);
        }
    }
}
```

**Timeout Protection**

Prevent infinite job execution:

```php
final class LongRunningJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        $startTime = time();
        $timeout = 300; // 5 minutes
        
        foreach ($payload['items'] as $item) {
            if (time() - $startTime > $timeout) {
                throw new RetryException('Job timeout, will retry');
            }
            
            $this->processItem($item);
        }
    }
}
```

## Debugging Lifecycle

Use lifecycle events to debug job processing:

```php
use Psr\EventDispatcher\ListenerProviderInterface;
use Spiral\Queue\Event\JobProcessing;
use Spiral\Queue\Event\JobProcessed;

final class JobDebugListener
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}
    
    public function onProcessing(JobProcessing $event): void
    {
        $this->logger->debug('Job processing started', [
            'job' => $event->name,
            'id' => $event->id,
            'queue' => $event->queue,
            'driver' => $event->driver,
            'payload' => $event->payload,
            'headers' => $event->headers,
        ]);
    }
    
    public function onProcessed(JobProcessed $event): void
    {
        $this->logger->debug('Job processing completed', [
            'job' => $event->name,
            'id' => $event->id,
        ]);
    }
}
```

> See [Events](./events.md) for event system documentation.
