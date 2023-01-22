# Queue and Jobs — RoadRunner integration

Roadrunner queues provide a unified queueing API across a variety of different queue backends. You can read full 
information about supported pipelines on the [official site](https://roadrunner.dev/docs/plugins-jobs).

## Installation

At first, you need to install the [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) package.

Once the package is installed, you can add the `Spiral\RoadRunnerBridge\Bootloader\QueueBootloader` to the list of
bootloaders:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\QueueBootloader::class,
    // ...
];
```

## Configuration

You can create a config file `app/config/queue.php` if you want to configure Queue connections:

```php
<?php

declare(strict_types=1);

use Spiral\RoadRunner\Jobs\Queue\MemoryCreateInfo;
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;
use Spiral\RoadRunner\Jobs\Queue\BeanstalkCreateInfo;
use Spiral\RoadRunner\Jobs\Queue\SQSCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'aliases' => [
        // 'mail-queue' => 'roadrunner',
        // 'rating-queue' => 'sync',
    ],
    
    'connections' => [
        'sync' => [
            // Job will be handled immediately without queueing
            'driver' => 'sync',
        ],
        'roadrunner' => [
            'driver' => 'roadrunner',
            'default' => 'local',
            'pipelines' => [
                'local' => [
                    'connector' => new MemoryCreateInfo('local'),
                    // Run consumer for this pipeline on startup (by default)
                    // You can pause consumer for this pipeline via console command
                    // php app.php queue:pause local
                    'consume' => true 
                ],
                // 'amqp' => [
                //     'connector' => new AMQPCreateInfo('bus', ...),
                //     // Don't consume jobs for this pipeline on start
                //     // You can run consumer for this pipeline via console command
                //     // php app.php queue:resume local
                //     'consume' => false 
                // ],
                // 
                // 'beanstalk' => [
                //     'connector' => new BeanstalkCreateInfo('bus', ...),
                // ],
                // 
                // 'sqs' => [
                //     'connector' => new SQSCreateInfo('amazon', ...),
                // ],
            ]
        ],
    ],
    
    'driverAliases' => [
        'sync' => \Spiral\Queue\Driver\SyncDriver::class,
        'roadrunner' => \Spiral\RoadRunnerBridge\Queue\Queue::class,
    ],
];
```

> **Note**
> Connections with the roadrunner driver will automatically declare pipelines without configuring on the RoadRunner 
> side. If the pipeline is declared via the RoadRunner config, Queue manager will just connect to it (without declaring).


