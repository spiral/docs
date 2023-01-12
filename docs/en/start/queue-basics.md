# Getting started — First background job

In this guide, we'll go over how to create and run a background job using the Spiral Framework and the RoadRunner
application server. This will allow you to perform tasks asynchronously, so that your application can continue to
perform other tasks while the background job is running.

Here are the basic steps you'll need to follow:

## Creating a Job

First, let's create a new job by creating a new class `PingSiteJob`:

```php app/src/Interface/Jobs/PingSiteJob.php
namespace App\Interface\Jobs;

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

Make sure that the jobs plugin is enabled in the RoadRunner config file `.rr.yaml`:

```yaml .rr.yaml
version: '2.7'

rpc:
  listen: 'tcp://127.0.0.1:6001'

jobs:
  consume: { }

# ...
```

Next, we need to configure our application to send jobs to RoadRunner. We can do this in the `app/config/queue.php`
config file:

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\MemoryCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'default' => 'memory',
            'pipelines' => [
                'memory' => [
                    'connector' => new MemoryCreateInfo('local'),
                    'consume' => true,
                ]
            ],
        ],
    ],
];
```

This configures our application to create a new `in-memory` pipeline for the RoadRunner server. When we push a job to
this pipeline, it will be added to the in-memory queue and RoadRunner will send it to a consumer to handle.

## Running the Job

Now that our job and RoadRunner are configured, we can create a console command to push a job to the queue.

```php app/src/Interface/Console/PingSiteCommand.php
namespace App\Interface\Console;

use App\Interface\Jobs\PingSiteJob;
use Spiral\Console\Command;
use Spiral\Queue\QueueInterface;

class PingSiteCommand extends Command
{
    protected const SIGNATURE = 'ping:site {site : Site to ping}';
    protected const DESCRIPTION = 'Ping site';

    public function __invoke(QueueInterface $queue): void
    {
        $id = $queue->push(PingSiteJob::class, [
            'site' => $this->argument('site'),
        ]);

        $this->writeln(\sprintf('Job %s pushed', $id));
    }
}
```

In the example I provided, the `QueueInterface` is being injected into the method, so when the dependency injection
container resolves it, it will provide an instance of the `default` queue connection specified in the config file.

#### Start the RoadRunner server

To run the job, we first need to start the RoadRunner server with the command:

```bash
./rr serve
```

#### Run the console command

Then, we can run our console command to push a job to the queue.

```bash
php app.php ping:site https://google.com
```

You should see the following output:

```output
Job [32m3332e595-9774-434c-908c-3c419f80c967[39m pushed
```

When the job is pushed to the queue, it will be sent to RoadRunner. Then, RoadRunner will send it to a consumer to
handle.

That's it! You've now created your first background job using the Spiral Framework and RoadRunner. With this setup, you
can easily add more jobs and perform tasks asynchronously to keep your application running smoothly.

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Queue and Jobs](../queue/configuration.md)
* [Queue Interceptors](../queue/interceptors.md)