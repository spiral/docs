# Queue and Jobs - Installation and Configuration

The Web and GRPC bundles of Spiral Framework support background PHP processing and a queue out of the box. You can work with
one or multiple message brokers such as Beanstalk, AMQP (RabbitMQ), or Amazon SQS.

To install the extensions in alternative bundles:

```bash
composer require spiral/queue
```

Make sure to add `Spiral\Queue\Bootloader\QueueBootloader` to your application kernel.

## Configuration

By default, the queue configuration is located in the `app/config/queue.php` file. The configuration includes a set of
options for each queue driver and aliases.

```php
<?php

declare(strict_types=1);

return [
    /**
     *  Default queue connection name
     */
    'default' => env('QUEUE_CONNECTION', 'sync'),

    /**
     *  Aliases for queue connections, if you want to use domain specific queues
     */
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

To create new queue connection, add a new section or alter existed options in the `connections` section of your
configuration.


### Aliases

Your application and modules can access the queue in several different ways. Queue aliasing allows you to use
separate connections with relation to one physical queue.

```php
<?php

declare(strict_types=1);

return [
    'aliases' => [
        'mailQueue' => 'roadrunner',
        'ratingQueue' => 'sync',
    ],
    
];
```

```php
use Spiral\Queue\QueueInterface;

public function __construct(QueueInterface $mailQueue, QueueInterface $ratingQueue)
{
    // ...
}
```

To point `mailQueue` and `ratingQueue` to a specific queue instance:

Or by using `Spiral\Queue\QueueConnectionProviderInterface`

```php
use Spiral\Queue\QueueConnectionProviderInterface;

$container->bind(MyService::class, function(QueueConnectionProviderInterface $provider) {
    return new MyService($provider->getConnection('mailQueue'));
})
```

```php
class MyService
{
    public function __construct(QueueInterface $queue)
    {
        // ...
    }
}
```

### Job handlers

You can register handlers for your job classes to associate which handler should be handled for a specific job.
