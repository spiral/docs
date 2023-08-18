# Configuring RoadRunner Queue Pipelines

Job queues have become an indispensable component in modern PHP applications, handling complex, resource-intensive tasks
and delivering substantial enhancements to application performance.

One solution that harnesses the power of Go for PHP applications is RoadRunner, in conjunction with the Spiral
Framework. This tutorial, will guide you through configuring queue pipelines for RoadRunner in Spiral applications,
offering a high-performing, robust queue service solution.

> **Note**
> This tutorial covers the basics of the components and approaches. For more detailed information, we suggest referring
> to the relevant sections.

Our application will consist of two types of applications:

1. **Producer** - will push jobs into a queue
2. **Consumer** - will receive queued tasks and handle them

## Producer

To get started with building **producer** application, you can easily install the default `spiral/app` bundle with most
of the required components by running the following command:

```terminal
composer create-project spiral/app my-app
```

During the installation process, you will be prompted to select various options with the Spiral installer, such as the
application preset, whether to use Cycle ORM, which collections to use, which validator component to use, and so on.
In our example we will use CLi application that will push a task into a queue from console command.

For this tutorial, we recommend choosing the options shown above:

```output
✔ Which application preset do you want to install? > Cli
✔ Create a default application structure and demo data? > No
✔ Do you need Cycle ORM? > No
✔ Do you want to use Queue component? > Yes
✔ Do you want to use Cache component? > No
✔ Do you want to use Mailer Component? > No
✔ Do you want to use Storage component? > No
✔ Do you need RoadRunner? > Yes
✔ Do you need the RoadRunner Metrics? > No
✔ Do you need the Temporal? > No
```

Once the installation is complete, you need to configure RoadRunner queue pipelines, where you will push your jobs.

First of all you need to configure RoadRunner server to use `jobs` plugin:

```yaml .rr.yaml
amqp:
  addr: amqp://guest:guest@127.0.0.1:5672

jobs:
  consume: [ ]
  pipelines: { }
```

> **Note**
> You can read more about RoadRunner Jobs plugin configuration
> in the [official documentation](https://roadrunner.dev/docs/queues-overview)

As you can see we didn't specify any pipelines in our configuration. RoadRunner provides the ability to create pipelines
dynamically, so we will create them later in our application.

Once the configuration is complete, you can start the server. RoadRunner
uses [RPC](https://roadrunner.dev/docs/php-rpc/2023.x/en) to communicate between PHP application and RoadRunner, so we
need to start it before pushing jobs into a queue.

Let's check if everything work fine using the following command:

```terminal
./rr serve
```

### Configuration

The configuration of Spiral applications is accomplished through configuration files located in the `app/config`
directory.

:::: tabs

::: tab PHP

Let's define our first pipeline in the `app/config/queue.php` file:

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'pipelines' => [
         'default' => [
             'connector' => new AMQPCreateInfo(
                  name: 'default',
                  priority: 100,
                  queue: 'default',
             ),
             // Do not consume jobs for this pipeline on startup
             'consume' => false,
         ],
    ],
    
    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'pipeline' => 'default',
        ],
    ],
];
```

> **Note**
> You can read more about roadrunner queue configuration in the [Queue — RoadRunner integration](../queue/roadrunner.md)
> section.

:::

::: tab YAML

Let's define our first pipeline in the `.rr.yaml` file:

```yaml
amqp:
  addr: amqp://guest:guest@127.0.0.1:5672

jobs:
  consume: [ ]
  pipelines:
    default:
      driver: amqp
      priority: 100
      queue: default
```

And select defined pipeline in the `app/config/queue.php` file:

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'pipelines' => [],
    
    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'pipeline' => 'default',
        ],
    ],
];
```

> **Note**
> You can read more about roadrunner queue configuration in the [Queue — RoadRunner integration](../queue/roadrunner.md)
> section.

:::

::::

When you run the `./rr serve` command, RoadRunner will create a pipeline with the name `default` and will use the
`roadrunner` connection to push jobs into the queue by default.

### Pushing jobs

Let's add some logic to it:

:::: tabs

::: tab Array payload

```php app/src/Endpoint/Console/PingCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Command;
use Spiral\Queue\QueueInterface;

#[AsCommand(name: 'ping')]
final class PingCommand extends Command
{
    public function __invoke(QueueInterface $queue): int
    {
        $queue->push('ping', [
            'url' => 'https://spiral.dev',
        ]);

        return self::SUCCESS;
    }
}
```

In case if we use an `array` payload we can use simple serializer, like `json`.

> **Note**
> Read more about available serializers in the [Component — Serializer](../advanced/serializer.md) section.

Let's define a default serializer in the `app/config/queue.php` file:

```php app/config/queue.php
return [
    // ...
    'defaultSerializer' => 'json',
];
```

> **Note**
> Read more about job payload serialization in the [Queue — Running Jobs](../queue/jobs.md#job-payload-serialization)
> section.

:::

::: tab Object payload

Queue component provides the ability to use complex objects as a payload. To use object payload we need to install
the `spiral-packages/symfony-serializer` package first. It will be helpful to serialize our payload:

```terminal
composer require spiral-packages/symfony-serializer
```

After package install you need to register bootloader:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Serializer\Symfony\Bootloader\SerializerBootloader::class,
];
```

And define default serializer in the `app/config/queue.php` file:

```php app/config/queue.php
return [
    // ...
    'defaultSerializer' => 'symfony-json',
];
```

> **Note**
> Read more about job payload serialization in the [Queue — Running Jobs](../queue/jobs.md#job-payload-serialization)
> section.

That's it! Now you can push jobs into a queue using object payload and it will be automatically serialized and
sent to a queue as a JSON string.

Let's create first a DTO class that will carry all the data we need:

```php app/src/DTO/Ping.php
namespace App\DTO;

final class Ping 
{
    public function __construct(
        public readonly string $url,
    ) {}
}
```

Now we can use it as a payload to push a job into a queue:

```php app/src/Endpoint/Console/PingCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Command;
use Spiral\Queue\QueueInterface;
use App\DTO\PingDTO;

#[AsCommand(name: 'ping')]
final class PingCommand extends Command
{
    public function __invoke(QueueInterface $queue): int
    {
        $queue->push('ping', new Ping(url: 'https://spiral.dev'));

        return self::SUCCESS;
    }
}
```

:::

::::

Once we created a console command `ping`, we can push a job into a queue.

First, we need to start the RoadRunner server:

```terminal
./rr serve
```

and then run our command:

```terminal
php app.php ping
```

## Consumer

To get started with building **consumer** application, you can easily install the default `spiral/app` bundle with most
of the required components by running the following command:

```terminal
composer create-project spiral/app my-consumer-app
```

For consumer, we need also `Queue component` and `RoadRunner`. Other components are optional and you can choose which
one you need during the installation process.

```output
✔ Which application preset do you want to install? > Cli
✔ Create a default application structure and demo data? > No
✔ Do you want to use Queue component? > Yes
✔ Do you need RoadRunner? > Yes
```

Once the installation is complete, you need to configure RoadRunner queue pipelines, where you will push your jobs.

First of all you need to configure RoadRunner server to use `jobs` plugin:

```yaml .rr.yaml
amqp:
  addr: amqp://guest:guest@127.0.0.1:5672

jobs:
  consume: [ ]
  pipelines: { }
```

> **Note**
> `amqp` section should be the same as in the producer application. **Consumer and Producer should use the same AMQP
> server.**

### Configuration

:::: tabs

::: tab PHP

Let's define our pipeline in the `app/config/queue.php` file:

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'pipelines' => [
         'default' => [
             'connector' => new AMQPCreateInfo(
                  name: 'default',
                  priority: 100,
                  queue: 'default',
             ),
             'consume' => true, // <===== Enables consuming
         ],
    ],
    
    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'pipeline' => 'default',
        ],
    ],
];
```

> **Note**
> The only difference between consumer and producer configuration is that consumer should have `consume` option set to
> `true`. In this case, RoadRunner will automatically consume jobs from the AMQP server.

:::

::: tab YAML

Let's define our pipeline in the `.rr.yaml` file:

```yaml
amqp:
  addr: amqp://guest:guest@127.0.0.1:5672

jobs:
  consume: [ default ]  # <===== Enables consuming
  pipelines:
    default:
      driver: amqp
      priority: 100
      queue: default
```

> **Note**
> The only difference between consumer and producer configuration is that consumer should have `jobs.consume` option
> contains `default` pipeline name. In this case, RoadRunner will automatically consume jobs from the AMQP server.

:::

::::

### Job

When a job is going to be consumed, it will be passed to the job handler class that has all the logic to handle it.

> **Note**
> Read more about job handlers in the [Queue — Job Handlers](../queue/jobs.md) section.

Let's create our first job using scaffolder:

```terminal
php app.php create:jobHandler Ping
```

> **Note**
> You can read more about scaffolding in the [Basics — Scaffolding](../basics/scaffolding.md) section.

After running this command, you will see the following output:

```output
Declaration of 'PingJob' has been successfully written into '~/my-app/app/src/Endpoint/Job/PingJob.php'.
```

We've just created a job handler that will be used to handle our jobs.

Let's add some logic to it:

:::: tabs

::: tab Array payload

If you are using a JSON payload, you can use simple serializer like `json` to deserialize a payload received from a
queue.

Let's define a default serializer in the `app/config/queue.php` file:

```php app/config/queue.php
return [
    // ...
    'defaultSerializer' => 'json',
];
```

Now let's add some logic to our job handler:

```php app/src/Endpoint/Job/PingJob.php
namespace App\Endpoint\Job;

use Psr\Log\LoggerInterface;
use Spiral\Queue\JobHandler;
use App\DTO\Ping;

final class PingJob extends JobHandler
{
    public function invoke(
        LoggerInterface $logger, 
        string $id, 
        Ping $payload, 
        array $headers,
    ): void {
        $logger->info('Ping job received', [
            'id' => $id,
            'url' => $payload->url,
            'headers' => $headers,
        ]);
    }
}
```

Our job handler will just log all the data received from the queue.

:::

::: tab Object payload

Spiral Queue component provides several way to understand what class should be used to deserialize a payload.

1. If you push an object into a queue, a component will also send a class name as a header `x-job-class`. In this case,
   you can use `spiral-packages/symfony-serializer` package we mentioned above to deserialize a payload.
2. It there is no `x-job-class` header, but you specified class type for `payload` argument in the `invoke` method, a
   component will try to deserialize a payload using the specified type.

> **Warning**
> Please pay attention that if you use header `x-job-class` to specify a class name, you should have this class with
> the same namespace in both producer and consumer applications. Probably you need to consider using Shared Repository
> to share your DTO between applications.

Let's define a default serializer in the `app/config/queue.php` file:

```php app/config/queue.php
return [
    // ...
    'defaultSerializer' => 'symfony-json',
];
```

And create a class in which we will deserialize our payload:

```php app/src/DTO/Ping.php
namespace App\DTO;

final class Ping 
{
    public function __construct(
        public readonly string $url,
    ) {}
}
```

Now let's add some logic to our job handler:

```php app/src/Endpoint/Job/PingJob.php
namespace App\Endpoint\Job;

use Psr\Log\LoggerInterface;
use Spiral\Queue\JobHandler;

final class PingJob extends JobHandler
{
    public function invoke(
        LoggerInterface $logger, 
        string $id, 
        array $payload,
        array $headers,
    ): void {
        $logger->info('Ping job received', [
            'id' => $id,
            'payload' => $payload,
            'headers' => $headers,
        ]);
    }
}
```

:::

::::

When we push a job into a queue using `push` method, we specified a task name `ping` in first argument. Now we need to
tell consumer that if it receives a job with this task name, it should be handled by `PingJob` handler.

Let's connect our job handler with the task name in the `app/config/queue.php` file:

```php app/config/queue.php
return [
    // ...
    'registry' => [
        'handlers' => [
            'ping' => App\Endpoint\Job\PingJob::class
        ],
    ],
];
```

> **Note**
> You can read more about job handlers registry in the [Queue — Job Handlers](../queue/jobs.md#job-handler-registry)
> section.


After all these steps, we are ready to consume jobs from the queue.

Let's run RoadRunner server:

```terminal
./rr serve
```

And push a job into a queue from producer application:

```terminal
php app.php ping
```

### Retry policy

Let's imagine that we have a job that should be retried if it fails. For example, we have a job that sends a request to
a remote server. If the server is not available, we should retry this job after some time.

In this case we can use job headers and queue interceptors to implement retry policy.

> **Note**
> Read more about queue interceptors in the [Queue — Interceptors](../queue/interceptors.md) section.

Let's create an interceptor that will catch all the exceptions in a job handler and try to retry it after some time:

```php app/src/Endpoint/Job/Interceptor/RetryPolicyInterceptor.php
namespace App\Endpoint\Job\Interceptor;

use Carbon\Carbon;
use Psr\Log\LoggerInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Exceptions\ExceptionReporterInterface;
use Spiral\Queue\Exception\FailException;
use Spiral\Queue\Exception\RetryException;
use Spiral\Queue\Options;

final class RetryPolicyInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
        private readonly ExceptionReporterInterface $reporter,
        private readonly int $maxAttempts = 3,
        private readonly int $delayInSeconds = 5,
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        try {
        
            // Try to execute a job handler
            return $core->callAction($controller, $action, $parameters);
            
        } catch (\Throwable $e) {
        
            // Report an exception
            $this->reporter->report($e);
            
            $headers = $parameters['headers'] ?? [];
            
            // Get attempts count from headers or if it the first attempt, use max attempts count
            $attempts = (int)($headers['attempts'] ?? $this->maxAttempts);
            
            // If attempts are over, throw a FailException
            if ($attempts === 0) {
                $this->logger->warning('Job handling failed: ['.$e->getMessage().']');
                
                throw new FailException($e->getMessage(), $e->getCode(), $e);
            }

            throw new RetryException(
                reason: $e->getMessage(),
                options: (new Options())->withDelay($this->delay)->withHeader('attempts', (string)($attempts - 1))
            );
        }
    }
}
```

Now we need to register our interceptor in the `app/config/queue.php` file:

```php app/config/queue.php
use App\Endpoint\Job\Interceptor\RetryPolicyInterceptor;

return [    
    // ...
    'interceptors' => [
        'consume' => [
            RetryPolicyInterceptor::class,
        ],
    ],
];
```

Now if our job handler fails, it will be retried after 5 seconds. After 3 attempts, the job will be marked as failed.

## Want more? Unlock the Power of Advanced Workflow Orchestration

Spiral Framework provides an integration with [Temporal](https://temporal.io/), a powerful workflow orchestration tool
that allows you to build complex workflows. Now, if you’re familiar with queue services like RoadRunner, you’re in for a
treat because Temporal IO takes workflow management to a whole new level. It’s like having superpowers for handling
complex workflows in a simple and elegant manner.

In the world of PHP development, we often find ourselves juggling various tasks and processes that need to be executed
in a specific order. That’s where Temporal shines. It allows us to write expressive and powerful workflows in a way
that’s easy to understand and maintain.

### Let’s dive into an example that showcases the beauty of Temporal

Imagine you have a task of handling user subscriptions on a monthly basis. With Temporal, it becomes a straightforward
process. Here’s a simple example to illustrate it:

```php
<?php
/**
 * This file is part of Temporal package.
 *
 * For the full copyright and license information, please view the LICENSE
 * file that was distributed with this source code.
 */
declare(strict_types=1);

namespace Temporal\Samples\Subscription;

use Carbon\CarbonInterval;
use Temporal\Activity\ActivityOptions;
use Temporal\Exception\Failure\CanceledFailure;
use Temporal\Workflow;

/**
 * Demonstrates a long-running process to represent a user subscription workflow.
 */
class SubscriptionWorkflow implements SubscriptionWorkflowInterface
{
    private $account;

    // Workflow logic goes here...

    public function subscribe(string $userID)
    {
        yield $this->account->sendWelcomeEmail($userID);

        try {
            $trialPeriod = true;
            while (true) {
                // Lower the period duration to observe workflow behavior
                yield Workflow::timer(CarbonInterval::days(30));

                if ($trialPeriod) {
                    yield $this->account->sendEndOfTrialEmail($userID);
                    $trialPeriod = false;
                    continue;
                }

                yield $this->account->chargeMonthlyFee($userID);
                yield $this->account->sendMonthlyChargeEmail($userID);
            }
        } catch (CanceledFailure $e) {
            yield Workflow::asyncDetached(
                function () use ($userID) {
                    yield $this->account->processSubscriptionCancellation($userID);
                    yield $this->account->sendSorryToSeeYouGoEmail($userID);
                }
            );
        }
    }
}
```

In this example, the `subscribe` method represents the workflow logic for managing monthly subscriptions. The magic lies
in the `Workflow::timer` function, which allows you to schedule a specific duration for each iteration of the loop.

By setting the timer to `CarbonInterval::months(1)`, you can ensure that the subscription tasks are executed every 
month. Temporal takes care of the scheduling and coordination, freeing you from the hassle of managing it manually.

Moreover, Temporal provides built-in fault tolerance and scalability. If an exception occurs, such as 
a `CanceledFailure` indicating a subscription cancellation, you can handle it gracefully within the workflow.

With Temporal, managing complex periodic workflows like monthly subscriptions becomes a breeze.