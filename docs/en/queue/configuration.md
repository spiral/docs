# Queue — Getting started

Spiral provides support for background PHP processing and a queue system. This feature is available out of the box and
allows you to work with a variety of message brokers.

Make sure to add `Spiral\Queue\Bootloader\QueueBootloader` to your application kernel.

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Queue\Bootloader\QueueBootloader::class,
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
    \Spiral\Queue\Bootloader\QueueBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

## Configuration

The default queue configuration can be found in the `app/config/queue.php` file. This configuration file includes a set
of options for each queue driver as well as aliases, which allow you to specify a default queue driver for your
application to use.

```php app/config/queue.php
return [
    /** Default queue connection name */
    'default' => env('QUEUE_CONNECTION', 'sync'),

    /** Aliases for queue connections, if you want to use domain specific queues */
    'aliases' => [
        // 'mailQueue' => 'null',
        // 'ratingQueue' => 'sync',
    ],
    
    'connections' => [
        'sync' => [
            // Job will be handled immediately without queueing
            'driver' => 'sync',
        ],
        'null' => [
            // Do nothing
            'driver' => 'null',
        ],
    ],
    
    'registry' => [
        'handlers' => [
            'sample::job' => App\Jobs\SampleJob::class
        ],
        'serializers' => [
            ObjectJob::class => 'json',
        ]
    ],
    
    'driverAliases' => [
        'sync' => \Spiral\Queue\Driver\SyncDriver::class,
        'null' => \Spiral\Queue\Driver\NullDriver::class,
    ],
];
```

### Declare Connection

To create a new queue connection in your application, you can add a new section to the `connections` section of the
queue
configuration file.

### Aliases

Queue aliasing is a feature that allows your application and modules to access the queue system in a variety of ways,
using separate connections that are related to a single physical queue.

```php app/config/queue.php
return [
    'aliases' => [
        'mailQueue' => 'roadrunner',
        'ratingQueue' => 'sync',
    ],
];
```

To obtain a queue instance by its name or alias, you can use the `getConnection` method of the
`Spiral\Queue\QueueConnectionProviderInterface`. This method takes a string argument representing the name of the
desired queue connection, and returns an instance of the `Spiral\Queue\QueueInterface`.

For example, if you want to get a queue instance for a name or alias called `mailQueue`, you could use the
following code:

```php
use Spiral\Queue\QueueConnectionProviderInterface;

$container->bind(MyService::class, function(QueueConnectionProviderInterface $provider) {
    return new MyService($provider->getConnection('mailQueue'));
})
```

```php
final class MyService
{
    public function __construct(
        private readonly QueueInterface $queue
    ) {
    }
}
```

### Job handlers

You can register handlers for your job classes to associate which handler should be handled for a specific job.

```php app/config/queue.php
'registry' => [
    'handlers' => [
        'sample::job' => App\Jobs\SampleJob::class
    ],
],
```

## Custom driver

If the available queue driver does not meet your specific needs, you can create your own custom driver by implementing
the `Spiral\Queue\QueueInterface` interface.

Here is an example implementation of a fake queue driver:

```php app/src/Infrastructure/Queue/RedisQueue.php
namespace App\Infrastructure\Queue;

use Spiral\Queue\SerializerRegistryInterface;
use Spiral\Queue\QueueInterface;

final class RedisQueue implements QueueInterface
{
    public function __construct(
        private readonly SerializerRegistryInterface $redis,
        private readonly Redis $redis,
        private readonly string $server,
        private readonly string $queueName,
    ) {
    }

    public function push(string $name, array $payload = [], OptionsInterface $options = null): string
    {
        $payload = $this->serializerRegistry->getSerializer($name)->serialize($payload);
        
        $id = // generate job id
        
        // Push job to the redis broker and return job id
        $this->redis->...
        
        return $id;
    }
}
```

Once you have defined your custom driver, you can register it in the configuration file:

```php app/config/queue.php
'connections' => [
    'mail' => [
        'driver' => \App\Infrastructure\Queue\RedisQueue::class,
        'server' => 'redis://localhost:6379',
        'queueName' => 'mail',
    ],
],
```

So when you use the `mail` connection in your application, Spiral IoC container will create an instance of the
`RedisQueue` class and pass the `redis://localhost:6379` and `mail` values for the `$server` and `$queueName`
parameters, respectively.

You can also register your driver as an alias in the `driverAliases` section of the queue configuration file.

```php app/config/queue.php
'driverAliases' => [
    'redis' => \App\Infrastructure\Queue\RedisQueue::class,
    // ...
],
```

Or register a driver alias via Bootloader:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use App\Infrastructure\Queue\RedisQueue;
use Spiral\Queue\Bootloader\QueueBootloader;

class AppBootloader extends Bootloader
{
    public function init(QueueBootloader $queue): void
    {
        $queue->registerDriverAlias(RedisQueue::class, 'redis');
    }
}
```

> **See more**
> Read more about job handler in the [Queue — Running Jobs](./jobs.md) section.
