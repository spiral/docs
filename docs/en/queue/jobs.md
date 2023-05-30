# Queue — Running Jobs

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

You can configure the serializer for a specific job type in the `app/config/queue.php` configuration file.

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

You can do it via the `app/config/queue.php` config:

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

```php  app/src/Application/Kernel.php
protected const APP = [
    // ...
    \App\Application\Bootloader\QueueFailedJobsBootloader::class,
];
```

## Events

| Event                            | Description                                                   |
|----------------------------------|---------------------------------------------------------------|
| Spiral\Queue\Event\JobProcessing | The Event will be fired `before` the job handler is executed. |
| Spiral\Queue\Event\JobProcessed  | The Event will be fired `after`  the job handler is executed. |

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
