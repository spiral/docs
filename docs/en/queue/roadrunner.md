# Queue — RoadRunner integration

Spiral has full integration with [RoadRunner jobs](https://docs.roadrunner.dev/docs/queues-and-jobs/overview-queues) plugin, which provides 
a unified queueing API for a variety of queue brokers such as:

- [Kafka](https://docs.roadrunner.dev/docs/queues-and-jobs/kafka),
- [AMQP (RabbitMQ)](https://docs.roadrunner.dev/docs/queues-and-jobs/amqp),
- [Amazon SQS](https://docs.roadrunner.dev/docs/queues-and-jobs/sqs),
- [Beanstalk](https://docs.roadrunner.dev/docs/queues-and-jobs/beanstalk),
- and [memory](https://docs.roadrunner.dev/docs/queues-and-jobs/memory) driver.

> **Note**
> For more information on supported brokers, please visit the [official site](https://docs.roadrunner.dev/docs/queues-and-jobs/overview-queues).

To enable the integration with RoadRunner, Spiral provides built-in support through
the [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) package.

> **Warning**
> Actual version of the `spiral/roadrunner-bridge` package is `3.0`. If you are using older version of the package,
> some of the features described in this document may not be available.

## Installation

To get started, you need to install [Roadrunner bridge](../start/server.md#roadrunner-bridge) package. Once installed,
add the `Spiral\RoadRunnerBridge\Bootloader\QueueBootloader` to the list of bootloaders in your Kernel class:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\QueueBootloader::class,
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
    \Spiral\RoadRunnerBridge\Bootloader\QueueBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

By doing this, an additional queue driver named `roadrunner` will be registered automatically. You can then add a new
queue connection for RoadRunner in your `app/config/queue.php` configuration file.

## Configuration

RoadRunner provides two ways to declare pipelines:

- Using a `.rr.yaml` configuration file to declare pipelines and brokers.
- Declaring pipelines on the fly in your application's `app/config/queue.php`.

### Declaring pipelines in `.rr.yaml`

This is the most common way to declare pipelines. But in this way, you can only declare static pipelines.

> **Note**
> If you need to declare dynamic pipelines, you should consider using the second way.

**Here's a simple example:**

```yaml .rr.yaml
amqp:
  default:
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
        connection: default
        priority: 1
        queue:
          name: default
          routing_key: default
```

In this case your `app/config/queue.php` configuration file should look like this:

```php app/config/queue.php
return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'pipeline' => 'in-memory', // Pipeline name from .rr.yaml
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

**Here's a simple example:**

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\MemoryCreateInfo;
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'rr-amqp'),

    'pipelines' => [
         'high-priority' => [
             'connector' => new AMQPCreateInfo(
                  name: 'high-priority',
                  priority: 1,
                  queue: 'default',
                  routingKey: 'default',
                  queueHeaders: ['rr_connection' => 'default'],
             ),
             'consume' => true, // Consume jobs for this pipeline on startup
         ],
         'low-priority' => [
             'connector' => new AMQPCreateInfo(
                  name: 'low-priority',
                  priority: 100,
                  queue: 'default',
                  routingKey: 'default',
                  queueHeaders: ['rr_connection' => 'default'],
             ),
             'consume' => false, // Do not consume jobs for this pipeline on startup
         ],
         'in-memory' => [
            'connector' => new MemoryCreateInfo(name: 'local'),
            'consume' => true, 
         ]
    ],
            
    'connections' => [
        // ...
        'rr-amqp' => [
            'driver' => 'roadrunner',
            'pipeline' => 'low-priority',
        ],
        'rr-memory' => [
            'driver' => 'roadrunner',
            'pipeline' => 'in-memory',
        ],
    ],
];
```

> **Note**
> RoadRunner 3 has no default AMQP connection: each AMQP pipeline must name one of the connections declared in the
> `amqp` section of `.rr.yaml`. Pipelines in `.rr.yaml` set `config.connection`, and pipelines created from PHP pass the
> connection name in the reserved `rr_connection` queue header. Read more in the
> [RoadRunner documentation](https://docs.roadrunner.dev/docs/queues-and-jobs/amqp).

> **Warning**
> All the pipelines defined with `'consume' => true` will be initialized during application bootstrapping. This means
> that all the jobs pushed to these pipelines will be consumed immediately. All the pipelines with `'consume' => false` 
> will be initialized only when you push a job to them.

Since `spiral/roadrunner-bridge` 5.0, the bridge declares these pipelines under a RoadRunner
[lock](../advanced/locks.md), so only one worker declares a pipeline at a time. A custom RoadRunner build must include
the `lock` plugin; it needs no section in `.rr.yaml`. The lock names start with `roadrunner-bridge:jobs:`. If this
clashes with locks of your application, set another prefix in the `lockPrefix` key of `app/config/queue.php`.

The worker that creates a `'consume' => true` pipeline also resumes it. A pipeline that already exists is left as is,
so a pipeline paused with `rr:jobs:pause` stays paused after workers restart. If a pipeline must exist before the first
worker boots, declare it in `.rr.yaml`: RoadRunner 3 fails to start the `jobs` plugin when a `server.on_init` script
declares pipelines.

In some cases, you may need to create a pipeline with custom default options or in cases where the options are
mandatory. For example, you may want to create a pipeline for Kafka broker, which requires additional options to be set.

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

The default `options` of a Kafka pipeline must be `KafkaOptions`: since `spiral/roadrunner-bridge` 5.0, a push to a
Kafka pipeline with other default options throws `InvalidArgumentException`. A Kafka pipeline without `options` whose
connector consumes a single topic pushes jobs to that topic.

## Usage

By default, when you push a job to the queue using the `roadrunner` driver, the job will be pushed to the default
pipeline defined in your `app/config/queue.php` configuration file.

```php
'rr-amqp' => [
    'driver' => 'roadrunner',
    'pipeline' => 'low-priority',
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

RoadRunner options carry no pipeline name, so such a job always goes to the default pipeline of the connection.

### Job priority and auto-ack

Starting from `spiral/roadrunner-bridge` 5.0, the `Spiral\RoadRunnerBridge\Queue\JobOptions` class lets you choose the
pipeline and set RoadRunner-specific job options at the same time. It extends `Spiral\Queue\Options`, so it has
everything that class has (`onQueue()`, `delayed()`, `withDelay()`, `withHeader()`, ...) plus the job priority and
auto-ack:

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueInterface;
use Spiral\RoadRunnerBridge\Queue\JobOptions;

public function createJob(QueueInterface $queue): void
{
    $queue->push(
        SampleJob::class,
        ['value' => 123],
        JobOptions::onQueue('in-memory')->withPriority(1)->withAutoAck(true)
    );
}
```

**Priority.** RoadRunner hands jobs with a **lower** priority value to workers first. A job without a priority gets the
priority of its pipeline: the `priority` option of the pipeline in `.rr.yaml`, or the `priority` of the connector for
pipelines declared in `app/config/queue.php` (`10` by default). RoadRunner treats `0` the same way, so `withPriority()`
accepts positive values only.

The job priority orders jobs inside RoadRunner, not inside the broker. All pipelines share one RoadRunner priority
queue, and each pipeline moves at most its `prefetch` number of jobs into it; the rest waits in the broker in the
broker's own order. So a job priority works well for in-memory pipelines and short queues. When urgent jobs must not
wait behind a large backlog in a broker, push them to a separate pipeline with a lower pipeline `priority`, like the
`high-priority` pipeline above.

**Auto-ack.** An auto-acked job is acknowledged before any worker runs it; broker drivers acknowledge it as soon as
RoadRunner fetches it from the broker. If the handler fails or RoadRunner restarts, the job is lost. Use it only for
jobs you can afford to lose.

`JobOptions` also works as the default `options` of a pipeline in `app/config/queue.php`, except for Kafka pipelines,
which require `KafkaOptions`. Options set on the push take precedence over the pipeline defaults, and an explicit
`withAutoAck(false)` overrides a pipeline that auto-acks. Jobs pushed with `JobOptions` to a Kafka pipeline keep the
topic from the pipeline's `KafkaOptions`.

> **Note**
> When a job is retried with `Spiral\Queue\Exception\RetryException`, only the delay and headers of its options are
> applied: a retry cannot change the job priority.

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