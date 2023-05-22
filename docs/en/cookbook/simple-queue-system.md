# Queue System

## Introduction

In the modern era of web development, the way we handle asynchronous tasks and manage heavy loads has become a
game-changer. Job queues have emerged as an indispensable part of PHP applications, tackling various challenges and
increasing overall efficiency. They enable us to efficiently process complex tasks without requiring immediate return
data. As PHP applications continue to grow in complexity and scale, the need for a robust, high-performance queue
service (QoS) has become increasingly evident. The right QoS can drastically enhance the efficiency and performance of
PHP applications. This article aims to shed light on why QoS is essential in today's PHP development landscape and how
employing QoS implemented in high-performance languages can significantly improve your PHP application's performance.

**Here are some example of task:**

1. Sending bulk emails
2. Processing complex tasks that don't require immediate feedback
3. Handling resource-intensive operations in the background
4. Managing scheduled tasks
5. Handling delayed executions and asynchronous processing
6. Balancing system load

## PHP QoS

PHP, with its simplicity and wide usage, has been a go-to language for developers for decades.

**Its strong points include:**

- Rich frameworks and continuous enhancement processes
- Unique community packages
- Simple deployment methods and tool assistance
- Extensive community support

However, PHP may not be the ideal choice for implementing infrastructure tools like queue services. This is primarily
due to its single-threaded nature which limits its ability to handle multiple tasks simultaneously.

**Other reasons include:**

1. **Not Optimized for Infrastructure Tools:** PHP was initially designed for web development, and its core strength
   lies in handling HTTP requests and generating HTML content. Implementing infrastructure tools like queue services,
   which require long-running processes and efficient resource management, is not PHP's primary focus.

2. **Interpreted Language:** PHP is an interpreted language, which means the PHP interpreter reads and executes code
   line by line. While this feature makes PHP easy to use and deploy, it can slow down execution speed, particularly for
   CPU-intensive tasks, which are common in queue services.

3. **Non-Optimized Code:** PHP’s flexibility allows developers to write code in different ways to achieve the same
   result. However, this can often lead to non-optimized code, which can affect performance, especially in a queue
   service where performance is critical.

4. **Blocked I/O Operations:** PHP, by default, follows a synchronous or blocking I/O model. This means that when a PHP
   application initiates an I/O operation (like reading from a file or querying a database), the execution thread is
   blocked or put on hold until the operation completes. This blocking nature can significantly hamper the performance
   and scalability of a queue service, which often needs to handle multiple I/O operations concurrently. Languages that
   support non-blocking or asynchronous I/O operations, like Go, are better suited for implementing high-performance
   queue services.

In PHP, communication with a queue broker is usually handled through sockets. However, PHP's handling of sockets is less
efficient compared to other languages like Go. In a queue service, efficient socket communication is key for tasks like
pushing and pulling messages from the queue, acknowledging the processing of messages, and monitoring the health of the
queue. The overhead of managing socket connections in PHP can lead to bottlenecks, affecting the overall performance and
reliability of the queue service. This issue is amplified in distributed systems where there is a high volume of socket
communication between different services. Languages that offer efficient, low-level control over network operations,
like Go, can handle socket communications more efficiently, resulting in a more performant and reliable queue service.

## Golang QoS

In contrast, Go, developed by Google, is designed to build simple, reliable, and speedy software. It comes with built-in
concurrency mechanisms that make it easier to write programs that achieve more in less time and are less prone to
crashing.

**Go particularly excels in developing:**

- Cloud-native applications
- Distributed networked services
- Fast and elegant command-line interfaces (CLIs)
- DevOps and SRE support tools
- Web development tools
- Stand-alone utilities

Given these strengths, Go has emerged as an excellent choice for implementing services like queue systems.

## RoadRunner

The Spiral Framework, in conjunction with the RoadRunner server, is a prime example of harnessing Go's power for PHP
applications. It leverages RoadRunner's QoS (jobs plugin), a high-performance queue service written in Go. This
combination offers a compelling alternative to PHP packages traditionally used to implement queue services.

One of the distinct advantages of the RoadRunner jobs plugin is its ease of configuration. You don't need to mess around
with configuring queue brokers like AMQP or Redis on the PHP side, or bother with adding any extra PHP extensions.
RoadRunner sorts all that out on its own. So, you get to focus on writing your app, while RoadRunner handles the heavy
lifting.

### Consuming and Producing Jobs

RoadRunner supports both single-instance and distributed setups for handling queue pipelines.

#### Single RoadRunner Instance

When working with a single RoadRunner instance, you can push and consume tasks using the same instance. This approach is
suitable for small projects where you don't need a distributed queue but only want to handle local jobs in the
background.

> **Note**
> RoadRunner provides an `memory` queue driver that can handle a large number of tasks in seconds.

**Here is an example configuration:**

```yaml .rr.yaml
version: "3"

rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: php consumer.php
  relay: pipes

jobs:
  consume: [ "local" ]
  pipelines:
    local:
      driver: memory
      config:
        priority: 10
        prefetch: 10
```

**In this configuration:**

- `jobs.consume` defines the pipelines that the RoadRunner instance will consume. In this case, it is consuming
  the `local` pipeline.

#### Multiple RoadRunner Instances

If you need a distributed setup using a queue broker like **RabbitMQ**, you can configure multiple RoadRunner
instances to handle the job processing. In this setup, one or more instances will act as producers, pushing jobs into a
queue, while other instances will act as consumers, consuming tasks from the queue.

**Here is an example configuration of producer:**

```yaml .rr.yaml
version: "3"

rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: php consumer.php
  relay: pipes

amqp:
  addr: amqp://guest:guest@127.0.0.1:5672

jobs:
  consume: [ ]
  pipelines:
    email:
      driver: amqp
      queue: email
      ...
    default:
      driver: amqp
      queue: default
      ...
```

**In this configuration:**

- `amqp.addr` specifies the address of the RabbitMQ broker to connect to.
- `jobs.consume` is an empty array, indicating that the RoadRunner instances will act as producers and only push tasks
  into the queue.
- `pipelines.email` and `pipelines.default` define two pipelines that will use the AMQP queue driver. Each pipeline is
  associated with a specific queue name (`email` and `default` in this case).

To configure a RoadRunner instance as a consumer, you need to define the pipelines it will consume. The `jobs.consume`
section specifies the pipelines that the consumer instance will process.

In a distributed setup, you can have multiple consumer instances connected to a queue broker. Each consumer instance
can be configured to consume tasks from one or more pipelines. This allows for parallel and scalable job processing.

**To configure multiple consumer instances:**

```yaml .rr.yaml
jobs:
  consume: [ "email" ]
  pipelines:
    email:
      driver: amqp
      queue: email
      ...
```

In this configuration, the consumer instances are set to consume tasks from the `email` pipeline.

You can have multiple consumer instances consuming tasks from different pipelines or a combination of shared and
dedicated pipelines, depending on your application's requirements.

One of the benefits of using multiple consumer instances is the ability to scale the job processing capacity
horizontally. By adding more consumer instances, you can increase the number of tasks processed simultaneously and
achieve higher throughput.

To scale consumer instances, you can replicate the consumer configuration for each new instance and ensure they connect
to the same queue broker and consume the desired pipelines. RoadRunner's design allows for seamless integration of
additional consumers without compromising the stability or performance of the job processing system.

### Monitoring

RoadRunner goes beyond just job processing and provides valuable metrics specifically designed for monitoring purposes.
It integrates seamlessly with Prometheus. By leveraging metrics plugin, you can collect detailed metrics about job
processing, worker performance, and resource utilization.

Additionally, RoadRunner provides a pre-configured Grafana dashboard that allows you to visualize the collected metrics
in a user-friendly and customizable way. With Grafana, you can create rich visualizations, set up alerts based on
defined thresholds, and gain valuable insights into the performance and health of your job processing system.

## Task payload

The payload's size significantly impacts the efficiency of communication between the application and the QoS. Large
payloads can cause various problems, including:

- **Increased Network Latency:** Larger payloads take longer to transmit over the network, increasing latency and
  slowing down the overall application performance.
- **Higher Memory Usage:** Larger payloads consume more memory, both in the queue service and in the PHP application.
  This increased memory usage can affect the performance of other parts of the application.
- **Increased CPU Usage:** Processing larger payloads can consume more CPU cycles, affecting the application's
  performance and scalability.
- **Reduced Throughput:** The larger the payload, the fewer the number of messages that can be processed per unit of
  time, reducing the overall throughput of the queue service.

Given these drawbacks, it's important to minimize the payload size when communicating between the PHP application and
the queue service. One effective way to do this is by using a compact serialization format like Protocol Buffers (
protobuf). In comparison to other serialization formats, protobuf offers smaller payload sizes which result in improved
performance and efficiency.

| Data Type                               | Size in Bytes |
|-----------------------------------------|---------------|
| **JSON array**                          | 430           |
| **Ig Binary array**                     | 348           |
| **Ig Binary object**                    | 482           |
| **Native serializer array**             | 635           |
| **Native serializer object**            | 848           |
| **Protobuf object**                     | 193           |
| **SerializableClosure array**           | 961           |
| **SerializableClosure object**          | 1174          |
| **SerializableClosure array w/secret**  | 1111          |
| **SerializableClosure object w/secret** | 1325          |

Spiral Framework enhances this aspect by offering the flexibility to use any serializer you want, allowing for efficient
serialization and deserialization of payloads. This feature, combined with Spiral's ability to efficiently handle socket
communication and manage queue services, makes it an attractive choice for PHP developers looking to optimize their
applications.

## Spiral Framework queue component

Spiral Framework offers an array of seamlessly integrated components, which makes it an ideal choice for building
complex applications. In this tutorial, we will guide you on creating a simple application using Spiral Framework and
RoadRunner.

> **Note**
> This tutorial covers the basics of the components and approaches. For more detailed information, we suggest referring
> to the relevant sections.

Our application will consist of two types of applications:

1. **Producer** - will push jobs into a queue
2. **Consumer** - will receive queued tasks and handle them

### Producer

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

#### Configuration

The configuration of Spiral applications is accomplished through configuration files located in the `app/config`
directory.

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

When you run the `./rr serve` command, RoadRunner will create a pipeline with the name `default` and will use the
`roadrunner` connection to push jobs into the queue by default.

#### Pushing jobs

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
    ...
    
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
    ...
    
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

### Consumer

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

#### Configuration

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
             'consume' => true, // <===== Enable consuming
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

#### Job

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
    ...
    
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
    ...
    
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
    ...

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

- Retry policy
- Interceptors
- 