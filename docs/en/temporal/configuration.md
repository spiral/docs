# Temporal — Installation and Configuration

[Temporal](https://temporal.io/) is an open-source workflow engine that manage and execute reliable, durable and
fault-tolerant workflows in a distributed manner. But the only way to use it in PHP is
with [RoadRunner](https://roadrunner.dev/docs/workflow-temporal). It makes it super easy to integrate your PHP app with
Temporal, so you can start using it right away.

One cool thing about Temporal is that you can write your workflows and activities in any supported language. So for
example, you could have a workflow written in PHP, but handle some of the activities with code written in Go or vice
versa. This can be really helpful if you have a team with different language preferences, or if you want to take
advantage of the strengths of different languages for different tasks.

Use Temporal when you have to manage complex data flows or ensure reliable transaction processing across multiple
business domains. it provides timers, retry mechanisms and much more.

## Installation

The Spiral framework offers seamless integration with the Temporal workflow engine. By installing the
`spiral/temporal-bridge` package, you can easily use Temporal within your application.

Install the `spiral/temporal-bridge` package by running the following composer command:

```terminal
composer require spiral/temporal-bridge
```

After the package is installed, you will need to activate the component using the bootloader:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\TemporalBridge\Bootloader\TemporalBridgeBootloader::class,
];
```

### Temporal Server

You can run temporal server via docker by using the example below:

> **Note**
> You can find official docker compose files [here](https://github.com/temporalio/docker-compose).

```yaml
version: '3.5'

services:
  postgresql:
    container_name: temporal-postgresql
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: temporal
      POSTGRES_USER: temporal
    ports:
      - 5432:5432

  temporal:
    container_name: temporal
    image: temporalio/auto-setup:1.14.2
    depends_on:
      - postgresql
    environment:
      - DB=postgresql
      - DB_PORT=5432
      - POSTGRES_USER=temporal
      - POSTGRES_PWD=temporal
      - POSTGRES_SEEDS=postgresql
      - DYNAMIC_CONFIG_FILE_PATH=config/dynamicconfig/development.yaml
    ports:
      - 7233:7233
    volumes:
      - ./temporal:/etc/temporal/config/dynamicconfig

  temporal-web:
    container_name: temporal-web
    image: temporalio/web:1.13.0
    depends_on:
      - temporal
    environment:
      - TEMPORAL_GRPC_ENDPOINT=temporal:7233
      - TEMPORAL_PERMIT_WRITE_API=true
    ports:
      - 8088:8088
```

> **Warning**
> Please make sure that a configuration file for temporal server
> exists. `mkdir temporal && touch temporal/development.yaml`.

## Configuration

Configure temporal address via env variables .env

```dotenv .env
TEMPORAL_ADDRESS=127.0.0.1:7233
```

Or create a `temporal.php` configuration file in the `app/config` directory. This file allows you to specify options such
as the `address`, `namespace`, default worker, and individual worker configurations.

> **Note**
> The default configuration should work for most cases, so you only need to modify this file if you need to change the
> default settings.

Here is an example configuration:

```php app/config/temporal.php
use Temporal\Worker\WorkerFactoryInterface;
use Temporal\Worker\WorkerOptions;

return [
    'address' => env('TEMPORAL_ADDRESS', '127.0.0.1:7233'),
    'namespace' => 'App\\Application\\Workflow',
    'defaultWorker' => WorkerFactoryInterface::DEFAULT_TASK_QUEUE,
    'workers' => [
        'workerName' => WorkerOptions::new()
    ],
];
```

In your RoadRunner configuration file `.rr.yaml`, add a temporal plugin section. This will allow you to specify the
`address` of your Temporal server, as well as the number of workers for your activities:

```yaml .rr.yaml
...

temporal:
  address: localhost:7233
  activities:
    num_workers: 10
```

> **Note**
> For more information on configuring the Temporal plugin in RoadRunner, check out the
> documentation [here](https://roadrunner.dev/docs/workflow-temporal).

That's it! Happy workflow building!