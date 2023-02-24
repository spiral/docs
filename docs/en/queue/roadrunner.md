# Queue — RoadRunner integration

Spiral has full integration with RoadRunner jobs plugin, which provides a unified queueing API for a variety of queue
brokers such as:

- [Kafka](https://roadrunner.dev/docs/plugins-jobs#kafka-driver),
- [AMQP (RabbitMQ)](https://roadrunner.dev/docs/plugins-jobs#amqp-driver),
- [Amazon SQS](https://roadrunner.dev/docs/plugins-jobs#sqs-driver),
- [Beanstalk](https://roadrunner.dev/docs/plugins-jobs#beanstalk-driver),
- and [memory](https://roadrunner.dev/docs/plugins-jobs#memory-driver) driver.

> **Note**
> For more information on supported brokers, please visit the [official site](https://roadrunner.dev/docs/plugins-jobs).

To enable the integration with RoadRunner, Spiral provides built-in support through
the [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) package.

## Installation

To get started, you need to install [Roadrunner bridge](../start/server.md#roadrunner-bridge) package. Once installed,
add the `Spiral\RoadRunnerBridge\Bootloader\QueueBootloader` to the list of bootloaders in your Kernel class:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\QueueBootloader::class,
    // ...
];
```

By doing this, an additional queue driver named `roadrunner` will be registered automatically. You can then add a new
queue connection for RoadRunner in your `app/config/queue.php` configuration file.

## Configuration

RoadRunner provides two ways to declare pipelines:

- Using a `.rr.yaml` configuration file to declare pipelines and brokers.
- Declaring pipelines on the fly in your application's `app/config/queue.php`.

> **Warning**
> Remember that you should always configure broker connection in `.rr.yaml` file.

### Declaring pipelines in `.rr.yaml`

This is the most common way to declare pipelines. But in this way, you can only declare static pipelines. If you need
to declare dynamic pipelines, you should consider using the second way.

Here's a simple example:

```yaml .rr.yaml
version: "2.7"

amqp:
  addr: amqp://guest:guest@localhost:5672

jobs:
  consume: [ "in-memory", "high-priority" ]
  pipelines:
    in-memory:
      driver: memory
      config:
        priority: 10
    high-priority:
      driver: amqp
      config:
        priority: 1
```

In this case your `app/config/queue.php` configuration file should look like this:

```php app/config/queue.php
return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'default' => 'in-memory', // Pipeline name from .rr.yaml
        ],
    ],
];
```

### Declaring pipelines in configuration file

This method is useful when you need to declare dynamic pipelines or if you just want to keep all your configuration
in one place.

**There are some benefits to this approach:**

- **Flexibility**: With dynamic pipeline declaration, you can create pipelines on-the-fly based on the current state of
  your application.
- **Control**: You have more control over the creation and management of pipelines within your application. You can
  create pipelines only when they are needed and remove them when they are no longer required, reducing unnecessary
  overhead.

You can create dynamic pipelines to distribute workload across multiple queue brokers or to balance the load across
multiple workers. This can be useful in large-scale applications where job processing needs to be distributed across
multiple servers or instances.

Here's a simple example:

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\MemoryCreateInfo;
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'rr-amqp'),

    'connections' => [
        // ...
        'rr-amqp' => [
            'driver' => 'roadrunner',
            'default' => 'low-priority', // Default pipeline
            'pipelines' => [
                 'high-priority' => [
                     'connector' => new AMQPCreateInfo(
                        name: 'high-priority',
                        priority: 1,
                        queue: 'default',
                     ),
                     'consume' => true, // Consume jobs for this pipeline on startup
                 ],
                 'low-priority' => [
                     'connector' => new AMQPCreateInfo(
                        name: 'low-priority',
                        priority: 100,
                        queue: 'default',
                     ),
                     'consume' => false, // Do not consume jobs for this pipeline on startup
                 ],
            ],
        ],
        'rr-memory' => [
            'driver' => 'roadrunner',
            'default' => 'in-memory',
            'pipelines' => [
                'in-memory' => [
                    'connector' => new MemoryCreateInfo(name: 'local'),
                    'consume' => true, 
                ]
            ],
        ],
    ],
];
```

In some cases, you may need to create a pipeline with custom default options or in cases where the options are
mandatory. For example, you may want to create a pipeline for Kafka broker, which requires additional options to be set.

> **Note**
> This feature is available since `spiral/roadrunner-bridge` version `2.5.0`.

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\KafkaCreateInfo;
use Spiral\RoadRunner\Jobs\KafkaOptions;

'pipelines' => [
    'event-bus' => [
        'connector' => new KafkaCreateInfo(name: 'kafka', topic: 'events', ...),
        'options' => new KafkaOptions(topic: 'events', ...),
        'consume' => true, 
    ]
],
```

## Usage

By default, when you push a job to the queue using the `roadrunner` driver, the job will be pushed to the default
pipeline defined in your `app/config/queue.php` configuration file.

```php
'rr-amqp' => [
    'driver' => 'roadrunner',
    'default' => 'low-priority', // Default pipeline
    'pipelines' => [
         'high-priority' => [
             // ...
         ],
         'low-priority' => [
             // ...
         ],
    ],
],
```

If you want to push a job to a specific pipeline, you can specify the pipeline name
using `Spiral\Queue\Options::onQueue` method.

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueInterface;
use Spiral\Queue\Options;

public function createJob(QueueInterface $queue): void
{
    $queue->push(
        SampleJob::class, 
        ['value' => 123],
        Options::onQueue('high-priority')
    );
}
```

### Options

Spiral queue component provides `Spiral\Queue\Options` class, but in some cases you may need to use options specific
to the roadrunner driver. In this case you can use options that implement `Spiral\RoadRunner\Jobs\OptionsInterface`.

> **Note**
> This feature is available since `spiral/roadrunner-bridge` version `2.5.0`.

For example, if you want to pass additional options to Kafka broker, you can use `Spiral\RoadRunner\Jobs\KafkaOptions`
class.

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueInterface;
use Spiral\RoadRunner\Jobs\KafkaOptions;

public function createJob(QueueInterface $queue): void
{
    $queue->push(
        SampleJob::class, 
        ['value' => 123],
        new KafkaOptions(topic: 'events')
    );
}
```

## Custom driver

In some cases, you may need to create a custom driver for a specific queue broker.

For example, you may want to create a driver for specific queue broker.

```php app/src/Infrastructure/Queue/KafkaQueue.php
namespace App\Infrastructure\Queue;

use Spiral\Queue\OptionsInterface;
use Spiral\Queue\QueueInterface;
use Spiral\RoadRunner\Jobs\JobsInterface;
use Spiral\RoadRunner\Jobs\Queue\KafkaCreateInfo;
use Spiral\RoadRunner\Jobs\KafkaOptions;

final class KafkaQueue imlements QueueInterface
{
    private readonly QueueInterface $queue;
    
    public function __construct(JobsInterface $jobs, string $name, string $topic) 
    {
        $this->queue = $jobs->create(
            new KafkaCreateInfo(
                name: $name, 
                topic: $topic
            ), 
            new KafkaOptions($topic)
        );
    }

    public function push(string $name, array $payload = [], OptionsInterface|KafkaOptions $options = null): string
    {
        return $this->queue->push($name, $payload, $options)->getId();
    }
}
```

That's all. Now you can use your driver:

```php app/config/queue.php
'connections' => [
    'mail' => [
        'driver' => \App\Infrastructure\Queue\KafkaQueue::class,
        'name' => 'mail',
        'topic' => 'mail',
    ],
],
```

> **See more**
> Read more about creating custom drivers in the [Queue — Getting started](./configuration.md) section.