# Temporal — Getting started

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

To use Temporal in your PHP project, you need to install the `spiral/temporal-bridge` package.

Here's how:

```terminal
composer require spiral/temporal-bridge
```

After the package is installed, you will need to activate the component using the bootloader:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\TemporalBridge\Bootloader\TemporalBridgeBootloader::class,
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
    \Spiral\TemporalBridge\Bootloader\TemporalBridgeBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

### Setting Up a Temporal Server

To start using Temporal quickly, use a development server.

First, install the Temporal CLI - follow the instructions on
the [Temporal website](https://docs.temporal.io/cli#install).

After the CLI is installed, you can start the Temporal server by running the following command:

```terminal
temporal server start-dev
````

> **Note**
> There are other ways to start a Temporal server. For example, you can use Docker. You can find docker-compose files
> in the [Temporal repository](https://github.com/temporalio/docker-compose).

## Configuration

### PHP application

All you need is to specify the address of your Temporal server in the `.env` file:

```dotenv .env
TEMPORAL_ADDRESS=127.0.0.1:7233
```

If you want to precisely configure your application, you can create the `temporal.php` configuration file. There you can
specify options such as a task queue, and individual worker configurations.

Here is an example configuration file:

```php app/config/temporal.php
use Temporal\Worker\WorkerFactoryInterface;
use Temporal\Worker\WorkerOptions;

return [
    'address' => env('TEMPORAL_ADDRESS', '127.0.0.1:7233'),
    'defaultWorker' => WorkerFactoryInterface::DEFAULT_TASK_QUEUE,
    'workers' => [
        'workerName' => WorkerOptions::new()
    ],
];
```

### RoadRunner

In your RoadRunner configuration file `.rr.yaml`, add a section `temporal`. This lets you set the server address and the
number of workers. For example:

```yaml .rr.yaml
...

temporal:
  address: localhost:7233
  activities:
    num_workers: 10
```

For more details on configuring Temporal with RoadRunner, read
the [RoadRunner](https://roadrunner.dev/docs/workflow-temporal) documentation.

That's it! Happy workflow building!