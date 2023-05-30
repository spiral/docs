# Getting started — First background job

In this guide, I'll walk you through the process of creating and running a background job using Spiral and the
[RoadRunner](https://roadrunner.dev/) application server. This will enable you to execute tasks asynchronously, allowing
your application to continue its operations while the background job runs seamlessly in the background.

Let's dive into the basic steps you need to follow:

## Creating a Job

To create your first job effortlessly, use the scaffolding command:

```terminal
php app.php create:jobHandler PingSite
```

> **Note**
> Read more about scaffolding in the [Basics — Scaffolding](../basics/scaffolding.md#job-handler) section.

After executing this command, the following output will confirm the successful creation:

```output
Declaration of '[32mPingSiteJob[39m' has been successfully written into '[33mapp/src/Endpoint/Job/PingSiteJob.php[39m'.
```

Now, let's inject some logic into our freshly created job handler.

Here's an example of a job that sends a `GET` request to a given site:

```php app/src/Endpoint/Job/PingSiteJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class PingSiteJob extends JobHandler
{
    public function invoke(HttpClientInterface $client, string $site): void
    {
        $response = $client->request('GET', $site);
        
        // do something with response ...
    }
}
```

## Configuration

Ensure that the jobs plugin is enabled in the RoadRunner configuration file `.rr.yaml`:

```yaml .rr.yaml
rpc:
  listen: 'tcp://127.0.0.1:6001'

jobs:
  consume: { }

# ...
```

Next, we need to configure our application to send jobs to RoadRunner. Open the `app/config/queue.php` configuration
file and apply the changes below:

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\MemoryCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    pipelines' => [
        'memory' => [
            'connector' => new MemoryCreateInfo('local'),
            'consume' => true,
        ]
    ],
            
    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'default' => 'memory',
        ],
    ],
];
```

These adjustments configure our application to establish a new `in-memory` pipeline for the RoadRunner server. Whenever
we push a job to this pipeline, it will be added to the `in-memory` queue. RoadRunner will then send it to a consumer
for handling.

## Running the Job

Now that our job and RoadRunner are configured, we can create a console command to push a job to the queue.

Let's create a command that will push a `PingSiteJob` to the queue:

```terminal
php app.php create:command PingSite
```

```php app/src/Endpoint/Console/PingSiteCommand.php
namespace App\Endpoint\Console;

use App\Endpoint\Job\PingSiteJob;
use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Command;
use Spiral\Queue\QueueInterface;

#[AsCommand(name: 'ping:site', description: 'Ping site')]
final class PingSiteCommand extends Command
{
    #[Argument(description: 'Site to ping')]
    public string $site;

    public function __invoke(): int
    {
        $id = $queue->push(PingSiteJob::class, [
            'site' => $this->site,
        ]);

        $this->writeln(\sprintf('Job %s pushed', $id));

        return self::SUCCESS;
    }
}
```

In the provided example, we inject the `QueueInterface` into the method. This allows the dependency injection container
to automatically resolve it and provide an instance of the default queue connection specified in the configuration file.

#### Start the RoadRunner server

To run the job, we first need to start the RoadRunner server with the following command:

```terminal
./rr serve
```

#### Run the console command

Now, it's time to run our console command and push a job to the queue. Use the following command:

```terminal
php app.php ping:site "https://google.com"
```

You should see the following output:

```output
Job [32m3332e595-9774-434c-908c-3c419f80c967[39m pushed
```

Once the job is pushed to the queue, it will be picked up by RoadRunner, which will subsequently hand it over to a
consumer for processing.

That's it! Congratulations on creating your first background job using Spiral and RoadRunner. With this setup, you can
effortlessly incorporate more jobs and execute tasks asynchronously to ensure smooth operation of your application.

<hr>

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Queue and Jobs](../queue/configuration.md)
* [Queue Interceptors](../queue/interceptors.md)
* [Create console command](../console/commands.md)
* [Scaffolding](../basics/scaffolding.md)