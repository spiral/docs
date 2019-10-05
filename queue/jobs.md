# Queue and Jobs - Running Jobs
You can run queue jobs in your application right after the installation of the application server. Web and GRPC bundles
come with pre-configured `jobs` service capable of running your tasks using `ephemeral` broker.

> Read how to run `spiral/jobs` outside of the framework [here](/queue/standalone.md).

## Create Handler
In order to run a job, you must create a proper job handler. The handler must implement `Spiral\Jobs\HandlerInterface`. Handlers
responsible for job payload serialization and execution. Use `Spiral\Jobs\JobHanlder` to simplify your abstraction
and implement dependency injection in your handler method `invoke`:

```php
namespace App\Jobs;

use Spiral\Jobs\JobHandler;

class SampleJob extends JobHandler
{
    public function invoke()
    {
        // do something
    }
}
```

You can freely use method injection in your handler:

```php
class SampleJob extends JobHandler
{
    public function invoke(MyService $service)
    {
        // do something with service
    }
}
```

> You can define handlers as singletons for higher performance.

## Dispatch Job
You can dispatch your job via `Spiral\Jobs\QueueInterface` or via prototype property `queue`. The method `push` of 
`QueueInterface` accepts job name, the payload in array form and additional options.

```php
public function createJob(QueueInterface $queue)
{
    $queue->push(SampleJob::class);
}
``` 

You can use your handler name as job name, it will be automatically converted into `-` identifier, for example, 
`App\Jobs\SampleJob` will be presented as `app-jobs-sampleJob`.

Use this identification to properly configure dispatching your `.rr` configuration:

```yaml
jobs:
    dispatch:
      app-jobs-sampleJob.pipeline: custom-pipeline
```

You can use `*` to define wildcard dispatching for the handler namespace:

```yaml
jobs:
    dispatch:
      app-jobs-*.pipeline: custom-pipeline
```

## Passing Parameters
Job handlers can accept any number of job parameters via the second argument of `QueueInterface->push()`. Parameters
must be provided in array form. No objects are supported (see below how to bypass it) to ensure compatibility with consumers written on other languages.

```php
public function createJob(QueueInterface $queue)
{
    $queue->push(SampleJob::class, ['value' => 123]);
}
```

You can receive passed payload in handler using the parameter `payload` of `invoke` method:

```php
class SampleJob extends JobHandler
{
    public function invoke(array $payload)
    {
        dumprr($payload);
    }
}
```

In addition to that, the default `Spiral\Jobs\JobHandler` implementation will pass all values of the payload as method arguments:


```php
class SampleJob extends JobHandler
{
    public function invoke(string $value)
    {
        dumprr($value);
    }
}
```

## Managing Payloads
The ability to alter the payload content using `serialize` and `unserialize` methods allows you to implement more complex
task logic.

For example, we can modify the job to accept Cycle ORM entity:

```php
class SampleJob extends JobHandler
{
    public function invoke(User $user)
    {
        dumprr($user);
    }

    public function serialize(string $jobType, array $payload): string
    {
        $payload['user'] = $payload['user']->getID();

        return parent::serialize($jobType, $payload);
    }

    public function unserialize(string $jobType, string $payload): array
    {
        $payload = parent::unserialize($jobType, $payload);
        $payload['user'] = $this->orm->getRepository(User::class)->findByPK($payload['user']);

        return $payload;
    }
}
```

Now we can pass user entity to our job:

```php
public function createJob(QueueInterface $queue, User $user)
{
    $queue->push(SampleJob::class, ['user' => $user]);
}
```

## Delayed Jobs
Use third parameter of `QueueInterface->push()` to specify additional job options. Options must be specified as 
`Spiral\Jobs\Options` object. To run job with 60 seconds delay:

```php
use Spiral\Jobs\Options;

// ...

public function createJob(QueueInterface $queue)
{
    $queue->push(SampleJob::class, ['value' => 123], Options::delayed(60));
}
```

## Debugging
Make sure to user `dumprr` function, output to STDOUT will break the communication with application server. If you MUST write to the STDOUT use alternative relay communication method, such as unix or TCP sockets.
