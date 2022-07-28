# Queue and Jobs - RoadRunner integration

Roadrunner queues provide a unified queueing API across a variety of different queue backends. Full information about
supported pipelines you can read on [official site](https://roadrunner.dev/docs/beep-beep-jobs).

The `spiral/roadrunner-bridge` component is included by default in Web and GRPC builds.

## Installation

To install the component in alternative bundles or as a standalone library:

```bash
composer require spiral/roadrunner-bridge
```

Activate the bootloader `Spiral\RoadRunnerBridge\Bootloader\QueueBootloader` in your application:

```php
protected const LOAD = [
    // ...
    Spiral\RoadRunnerBridge\Bootloader\QueueBootloader::class,
    // ...
];
```

## Configuration

You can create config file `app/config/queue.php` if you want to configure Queue connections:

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
> Connections with roadrunner driver will automatically declare pipelines without configuring on the RoadRunner side. If
> pipeline will be declared via RoadRunner config, Queue manager will just connect to it (without declaring).


