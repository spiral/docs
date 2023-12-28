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
    'temporalNamespace' => 'default',
    'workers' => [
        'someTaskQueue' => WorkerOptions::new(),
        // ...
    ],
    'interceptors' => [],
    'clientOptions' => null,
];
```

#### Client Options

Client options are used to configure various aspects of a Temporal client's behavior.

1. **Namespace Specification:** Allows the specification of a namespace in which the client operations will be
   performed.
2. **Client Identity Setting:** This class enables the setting of an identity for the client. This identity is useful
   for tracking and logging purposes, helping in identifying which client performed certain operations in the Temporal
   system.
3. **Query Rejection Condition Configuration:** It offers the ability to set conditions under which query requests to
   the Temporal server can be rejected.

For example:

```php app/config/temporal.php
use Temporal\Client\ClientOptions;

return [
    // ...
    'clientOptions' => (new ClientOptions())
       ->withNamespace('default')
       ->withIdentity('customer-service'),
];
```

> **Note**
> `temporalNamespace` option will be used only if `clientOptions` is not specified. If `clientOptions` is not specified
> then ClientOptions will be created with `temporalNamespace` option value.

#### Worker Options

Worker options are used to configure various aspects of a Temporal worker's behavior.

1. **Customization of Worker Behavior:** The class provides a range of options to fine-tune how workers handle activity
   and workflow tasks. This includes setting concurrency limits, rate limiting, and managing task polling behavior.
2. **Resource Management:** By allowing developers to set limits on the number and rate of activities executed,
   the `WorkerOptions` class helps in effectively managing system resources. This is crucial for maintaining system
   stability and efficiency, especially in high-load scenarios.
3. **Control Over Task Processing:** The class gives control over how tasks are retrieved and processed by the worker.
   Options like the maximum number of concurrent tasks and pollers help balance workload and optimize task execution.
4. **Enhanced Flexibility:** It offers advanced options like sticky schedules and graceful stop timeouts, providing
   greater control over task execution and worker shutdown processes.
5. **Support for Session-Based Activities:** With options to enable session workers and set session-related parameters,
   it facilitates the execution of activities within a session context, enhancing the scope of what can be achieved with
   Temporal workflows.

You can define worker options for each task queue. For example:

```php app/config/temporal.php
use Temporal\Worker\WorkerOptions;

return [
    // ...
    'workers' => [
        'workerName' => WorkerOptions::new(),
        'default' => WorkerOptions::new()
           ->withMaxConcurrentActivityExecutionSize(10)
           ->withWorkerActivitiesPerSecond(100),
    ],
];
```

There is also an ability to use alternative way to define worker options. For example:

```php app/config/temporal.php
use Temporal\Worker\WorkerOptions;

return [
    // ...
    'workers' => [
        'workerName' => [
            'options' => WorkerOptions::new(),
            'exception_interceptor' => new ExceptionInterceptor(),
        ]
    ],
];
```

Using this way you can additionally define an exception interceptor. Read more about interceptors in the
[Temporal — Interceptors](interceptors.md#exception-interceptor) section.

#### Interceptors

Interceptors are used to intercept workflow and activity invocations. They can be used to add custom logic to the
invocation process, such as logging or metrics collection.

Read more about interceptors in the [Temporal — Interceptors](interceptors.md) section.

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

