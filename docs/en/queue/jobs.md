# Queue and Jobs - Running Jobs

You can run queue jobs in your application right after the installation of the application server. The Web and GRPC bundles
come with a pre-configured `queue` service capable of running your tasks using an `ephemeral` broker.

## Create Handler

To run a job, you must create a proper job handler. The handler must implement `Spiral\Queue\HandlerInterface`. Handlers
are responsible for job payload serialization and execution. Use `Spiral\Queue\JobHandler` to simplify your abstraction
and perform dependency injection in your handler method `invoke`:

```php
namespace App\Jobs;

use Spiral\Queue\JobHandler;

class SampleJob extends JobHandler
{
    public function invoke(): void
    {
        // do something
    }
}
```

You can freely use the method injection in your handler:

```php
namespace App\Jobs;

use Spiral\Queue\JobHandler;

class SampleJob extends JobHandler
{
    public function invoke(MyService $service): void
    {
        // Do something with service
    }
}
```

> **Note**
> You can define handlers as singletons for better performance.

## Dispatch Job

You can dispatch your job via `Spiral\Queue\QueueInterface` or via the prototype property `queue`. The method `push` of
`QueueInterface` accepts a job name, the payload in the array form, and additional options.

```php
use App\Jobs\SampleJob;
use Spiral\Queue\QueueInterface;

public function createJob(QueueInterface $queue): void
{
    $queue->push(SampleJob::class);
}
``` 

You can use your handler name as the job name. It will be automatically converted into `-` identifier, for example,
`App\Jobs\SampleJob` will be presented as `app-jobs-sampleJob`.

## Passing Parameters

Job handlers can accept any number of job parameters via the second argument of `QueueInterface->push()`. The parameters
provided in an array form. No objects are supported (see how to bypass it below) to ensure compatibility with consumers
written in other languages.

```php
use App\Jobs\SampleJob;
use Spiral\Queue\QueueInterface;

public function createJob(QueueInterface $queue): void
{
    $queue->push(SampleJob::class, ['value' => 123]);
}
```

You can receive the passed payload in the handler using the parameter `payload` of the `invoke` method:

```php
use Spiral\Queue\JobHandler;

class SampleJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        dump($payload);
    }
}
```

In addition to that, the default `Spiral\Queue\JobHandler` implementation will pass all values of the payload as method
arguments:

```php
use Spiral\Queue\JobHandler;

class SampleJob extends JobHandler
{
    public function invoke(string $value): void
    {
        dump($value);
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

```php
<?php

declare(strict_types=1);

return [
    'registry' => [
        'handlers' => [
            'sample::job' => App\Jobs\SampleJob::class
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
        $registry->setHandler('sample::job', \App\Jobs\SampleJob::class);
    }
}
```

## Job Payload serialization

When a job pushed into a queue, the job payload is serialized via [Serializer component](../component/serializer.md).

### Configure default serializer
```php
// file app/config/queue.php

declare(strict_types=1);

use Spiral\Core\Container\Autowire;
use Spiral\Serializer\Serializer\JsonSerializer;
use Spiral\Serializer\Serializer\PhpSerializer;

return [
    // ...

    // via class name
    'defaultSerializer' => JsonSerializer::class,
    
    // via instance
    'defaultSerializer' => new JsonSerializer(),
    
    // via Autowire
    'defaultSerializer' => new Autowire(PhpSerializer::class)
];
```

### Changing serializer

There are several ways to change the serializer. You can globally change the default serializer for the application. 
You can read more about configuring the Serializer [here](../component/serializer.md).

Or you can set a specific serializer for the job type. A specific serializer is selected by the `Spiral\Serializer\SerializerRegistryInterface`.

You can configure the serializer for a specific job type in the `app/config/queue.php` configuration file.

```php
use Spiral\Core\Container\Autowire;

return [
    'registry' => [
        'serializers' => [
            ObjectJob::class => 'json',
            TestJob::class => 'serializer',
            OtherJob::class => CustomSerializer::class,
            FooJob::class => new CustomSerializer(),
            BarJob::class => new Autowire(CustomSerializer::class),
        ]
    ],
];
```

A serializer can be a `key string` under which the serializer is registered in the `Serializer` component, 
a `fully-qualified class name`, a `serializer instance`, an `Autowire instance`.

> **Note**
> The serializer class must implement the `Spiral\Serializer\SerializerInterface` interface.

Or, register a serializer using the `setSerializer` method of the `Spiral\Queue\QueueRegistry` class.

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container\Autowire;
use Spiral\Queue\QueueRegistry;

class AppBootloader extends Bootloader
{
    public function boot(QueueRegistry $registry): void
    {
        $registry->setSerializer(ObjectJob::class, 'json');
        $registry->setSerializer(TestJob::class, 'serializer');
        $registry->setSerializer(OtherJob::class, CustomSerializer::class);
        $registry->setSerializer(FooJob::class, new CustomSerializer());
        $registry->setSerializer(BarJob::class, new Autowire(CustomSerializer::class));
    }
}
```

## Handle failed jobs

By default, all failed jobs will be sent into the spiral log. But you can change the default behavior. At first, you need to
create your own implementation for `Spiral\Queue\Failed\FailedJobHandlerInterface`.

### Custom handler example

```php
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

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\RoadRunnerBridge\Queue\Failed\FailedJobHandlerInterface;

final class QueueFailedJobsBootloader extends Bootloader
{
    protected const SINGLETONS = [
        FailedJobHandlerInterface::class => \App\Jobs\DatabaseFailedJobsHandler::class,
    ];
}
```

And register this bootloader after `QueueFailedJobsBootloader` in your application

```php
protected const APP = [
    // ...
    App\Bootloader\QueueFailedJobsBootloader::class,
];
```

## Events

| Event                            | Description                                                 |
|----------------------------------|-------------------------------------------------------------|
| Spiral\Queue\Event\JobProcessing | The Event will be fired `before` the job handler is executed. |
| Spiral\Queue\Event\JobProcessed  | The Event will be fired `after`  the job handler is executed.  |
