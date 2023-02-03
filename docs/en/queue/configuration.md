# Queue — Getting started

The Spiral Framework provides support for background PHP processing and a queue system for both the Web and GRPC
bundles. This feature is available out of the box and allows you to work with a variety of message brokers including:

- [Kafka](https://roadrunner.dev/docs/plugins-jobs#kafka-driver),
- [AMQP (RabbitMQ)](https://roadrunner.dev/docs/plugins-jobs#amqp-driver),
- [Amazon SQS](https://roadrunner.dev/docs/plugins-jobs#sqs-driver),
- [Beanstalk](https://roadrunner.dev/docs/plugins-jobs#beanstalk-driver),
- and [memory](https://roadrunner.dev/docs/plugins-jobs#memory-driver) driver.

To install the necessary extensions in an alternative bundle, you can use the following command:

```terminal
composer require spiral/queue
```

Make sure to add `Spiral\Queue\Bootloader\QueueBootloader` to your application kernel.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Queue\Bootloader\QueueBootloader::class,
    // ...
];
```

## Configuration

The default queue configuration can be found in the `app/config/queue.php` file. This configuration file includes a set
of
options for each queue driver as well as aliases, which allow you to specify a default queue driver for your application
to use.

```php app/config/queue.php
return [
    /** Default queue connection name */
    'default' => env('QUEUE_CONNECTION', 'sync'),

    /** Aliases for queue connections, if you want to use domain specific queues */
    'aliases' => [
        // 'mailQueue' => 'roadrunner',
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
    
    'driverAliases' => [
        'sync' => \Spiral\Queue\Driver\SyncDriver::class,
        'null' => \Spiral\Queue\Driver\NullDriver::class,
    ],
];
```

### Declare Connection

To create a new queue connection in your application, you can add a new section to the `connections` section of the queue
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
desired queue, and returns an instance of the `Spiral\Queue\QueueInterface`.

For example, if you want to get a queue instance for a name or alias called `mailQueue``, you could use the 
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
