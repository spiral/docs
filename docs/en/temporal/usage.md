# Temporal — Usage

In Temporal, a workflow is a long-running process that is composed of a series of interconnected activities. Workflows
allow developers to define and execute complex processes that may span multiple services or even systems.

Workflows are executed by the Temporal workflow engine. Workflows can be triggered by external events (such as a user
request or a message on a message queue) or they can be scheduled to run at specific intervals.

Let's take a look at a simple example of a workflow that ping a website every 5 minutes and send a notification if
website is down.

## Workflow Definition

Workflow is a PHP class that contains a single method annotated with `#[WorkflowMethod]` attribute. This method will be
used as an entry point for starting the workflow.

> **Read more**
> For more information about workflows, see the [Temporal documentation](https://docs.temporal.io/workflows).

To create a workflow run the following command:

```terminal
php app.php create:workflow WebsiteStatus
```

This command will create a new workflow class in the `app/src/Endpoint/Temporal/Workflow` directory with the following
content:

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Temporal\Workflow;

use Temporal\Workflow\WorkflowInterface;
use Temporal\Workflow\WorkflowMethod;

#[WorkflowInterface]
class WebsiteStatusWorkflow
{
    #[WorkflowMethod]
    public function handle()
    {
        // TODO: Implement handle method
    }
}
```

Let's add some logic to the workflow:

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
namespace App\Endpoint\Temporal\Workflow;

use Carbon\CarbonInterval;
use Temporal\Workflow\WorkflowInterface;
use Temporal\Workflow\WorkflowMethod;
use Temporal\Workflow;

#[WorkflowInterface]
class WebsiteStatusWorkflow
{
    private bool $isDownNotified = false;
    private bool $isRecoveryNotified = false;
    private int $downTime = 0;

    #[WorkflowMethod]
    public function handle(string $url, int $intervalInMinutes = 5)
    {
        while (true) {
            // here we will ping the website and get the status
            $status = ...

            if ($status === false) {
                // Send notification only once when the website goes down
                if (!$this->isDownNotified) {
                    // here we will send a notification about downtime
                }

                $this->isDownNotified = true;
                // increase downtime by 5 minutes
                $this->downTime += $intervalInMinutes;
            } else {
                // Send notification only once when the website goes up
                if (!$this->isRecoveryNotified) {
                    // here we will send a notification about recovery with total downtime
                }

                $this->downTime = 0;
                $this->isRecoveryNotified = true;
            }

            // wait for 5 minutes
            yield Workflow::timer(CarbonInterval::minutes($intervalInMinutes));
        }
    }
}
```

As you can see, our workflow is a simple loop that will ping the website every 5 minutes and send a notification if
website is down and when it goes up. We also keep track of the total downtime and send it in the notification when the
website goes up.

To ping the website and send a notification we will use activities. Let's create them.

> **Note**
> Workflow classes will be automatically registered in the Temporal server when you run the application. Spiral will
> look for all classes that has an attribute `Temporal\Workflow\WorkflowInterface` and register them in the Temporal
> server.

> **Warning**
> You cannot use DI, io operations or any other blocking operations in the workflow. If you need to use any of these
> operations, you need to use activities.

## Activity Definition

Activities are the building blocks of workflows. Activities execute a single, well-defined action (either short or
long running), such as calling another service, transcoding a media file, sending an email message, etc.

> **Read more**
> For more information about activities, see the [Temporal documentation](https://docs.temporal.io/activities).

Let's create an activity that will ping the website:

```terminal
php app.php create:activity PingWebsite --method=ping:bool
```

This command will create a new activity class in the `app/src/Endpoint/Temporal/Activity` directory with the following
content:

```php app/src/Endpoint/Temporal/Activity/PingWebsiteActivity.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Temporal\Activity;

use React\Promise\PromiseInterface;
use Temporal\Activity\ActivityInterface;
use Temporal\Activity\ActivityMethod;

#[ActivityInterface]
class PingWebsiteActivity
{
    /**
     * @return PromiseInterface<bool>
     */
    #[ActivityMethod(name: 'ping')]
    public function ping(): bool
    {
        // TODO: Implement activity method
    }
}
```

> **Note**
> Activity classes will be automatically registered in the Temporal server when you run the application. Spiral will
> look for all classes that has an attribute `Temporal\Activity\ActivityInterface` and register them in the Temporal
> server.

Let's add some logic to the activity:

```php app/src/Endpoint/Temporal/Activity/PingWebsiteActivity.php
namespace App\Endpoint\Temporal\Activity;

use React\Promise\PromiseInterface;
use Temporal\Activity\ActivityInterface;
use Temporal\Activity\ActivityMethod;

#[ActivityInterface]
class PingWebsiteActivity
{
    /**
     * @return PromiseInterface<bool>
     */
    #[ActivityMethod(name: 'ping')]
    public function ping(string $domain): bool
    {
        // here we will ping the website and get the status
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $domain);
        curl_setopt($ch, CURLOPT_HEADER, TRUE);
        curl_setopt($ch, CURLOPT_NOBODY, TRUE); // remove body
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
        $head = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);

        return $httpCode === 200;
    }
}
```

To use the activity in the workflow, we need to initialize it in the workflow constructor:

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
use Temporal\Internal\Workflow\ActivityProxy;
use Temporal\Activity\ActivityOptions;
use App\Endpoint\Temporal\Activity\PingWebsiteActivity;

#[WorkflowInterface]
class WebsiteStatusWorkflow
{
    // ...
    
    private PingWebsiteActivity|ActivityProxy $pingActivity;

    public function __construct()
    {
        $this->pingActivity = Workflow::newActivityStub(
            PingWebsiteActivity::class,
            ActivityOptions::new()
                ->withStartToCloseTimeout(5)
        );
    }
    
    //...
}
```

`Workflow::newActivityStub` method will create a proxy class `Temporal\Internal\Workflow\ActivityProxy` that will be
used to call the activity. Despite the fact that the activity is a PHP class, every activity's method call will
be sent to the Temporal server and server will execute the real activity by its name.

Now we can use the activity in the workflow:

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
// ...
    #[WorkflowMethod]
    public function handle(string $url, int $intervalInMinutes = 5)
    {
        while (true) {
            // here we will ping the website and get the status
            $status = yield $this->pingActivity->ping($url);

            if ($status === false) {

// ...
```

When you call an activity method a promise object will be returned. This promise will be resolved when the activity is
completed. We use `yield` to wait for the promise to be resolved and will return the result of the activity.

```php
yield $this->pingActivity->ping($url)
```

An Activity invocation synchronously blocks until the Activity completes, fails, or times out. Even if Activity
Execution takes a few months, the Workflow code still sees it as a single synchronous invocation.

In some cases, you may want to execute multiple Activities in parallel. For example, you may want to call
services to ping your site like:

```php
$status = yield $this->pingActivity->pingFromEurope($url);
$status = yield $this->pingActivity->pingFromAsia($url);
$status = yield $this->pingActivity->pingFromAmerica($url);
```

If you use `yield` to call activities, they will be executed sequentially. To execute activities in parallel, you need
to use `\Temporal\Promise\Promise::all` method:

```php
use Temporal\Promise\Promise;

[$statusEurope, $statusAsia, $statusAmerica] = yield Promise::all([
    $this->pingActivity->pingFromEurope($url),
    $this->pingActivity->pingFromAsia($url),
    $this->pingActivity->pingFromAmerica($url),
]);
```

In this case, all activities will be executed in parallel and the result will be returned as an array.

### Sending Notifications

To send notifications we will use another activity:

```terminal
php app.php create:activity SendNotification --method=sendFailedNotification:void --method=sendRecoveryNotification:void
```

This command will create a new activity class in the `app/src/Endpoint/Temporal/Activity` directory with the following
content:

```php app/src/Endpoint/Temporal/Activity/SendNotificationActivity.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Temporal\Activity;

use React\Promise\PromiseInterface;
use Temporal\Activity\ActivityInterface;
use Temporal\Activity\ActivityMethod;

#[ActivityInterface]
class SendNotificationActivity
{
    /**
     * @return PromiseInterface<void>
     */
    #[ActivityMethod(name: 'sendFailedNotification')]
    public function sendFailedNotification(): void
    {
        // TODO: Implement activity method
    }
    
    /**
     * @return PromiseInterface<void>
     */
    #[ActivityMethod(name: 'sendRecoveryNotification')]
    public function sendRecoveryNotification(): void
    {
        // TODO: Implement activity method
    }
}
```

Let's add some logic to the activity:

```php app/src/Endpoint/Temporal/Activity/SendNotificationActivity.php
namespace App\Endpoint\Temporal\Activity;

use React\Promise\PromiseInterface;
use Spiral\Mailer\MailerInterface;
use Temporal\Activity\ActivityInterface;
use Temporal\Activity\ActivityMethod;

#[ActivityInterface]
class SendNotificationActivity
{
    public function __construct(
        private readonly MailerInterface $mailer,
    ) {
    }

    /** @return PromiseInterface<void> */
    #[ActivityMethod(name: 'sendFailedNotification')]
    public function sendFailedNotification(string $domain): void
    {
        $text = "Website {$domain} is down.";

        // $this->mailer->send(...);
    }

    /** @return PromiseInterface<void> */
    #[ActivityMethod(name: 'sendRecoveryNotification')]
    public function sendRecoveryNotification(string $domain, int $downTime): void
    {
        $text = "Website {$domain} is up after {$downTime} minutes of downtime";

        // $this->mailer->send(...);
    }
}
```

We also can specify task queue for the activity. Task queue is a logical grouping of activities. By default, all the
workflow and activities are assigned to the `default` task queue. You can specify task queue for the activity using PHP
attributes:

```php
use Spiral\TemporalBridge\Attribute\AssignWorker;

#[AssignWorker('mailer')]
#[ActivityInterface]
class SendNotificationActivity
{
}
```

And then we can tell the workflow to use this task queue for the activity:

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
$this->mailActivity = Workflow::newActivityStub(
    SendNotificationActivity::class,
    ActivityOptions::new()
        ->withStartToCloseTimeout(5)
        ->withTaskQueue('mailer')
);
```

To use the activity in the workflow, we need to initialize it in the workflow constructor:

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
use Temporal\Internal\Workflow\ActivityProxy;
use Temporal\Activity\ActivityOptions;
use App\Endpoint\Temporal\Activity\SendNotificationActivity;

#[WorkflowInterface]
class WebsiteStatusWorkflow
{
    // ...
    
    private SendNotificationActivity|ActivityProxy $mailActivity;

    public function __construct()
    {
        // ...
        
        $this->mailActivity = Workflow::newActivityStub(
            SendNotificationActivity::class,
            ActivityOptions::new()
                ->withStartToCloseTimeout(5)
                ->withTaskQueue('mailer')
        );
    }
    
    //...
}
```

Now we can use the activity in the workflow:

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
if ($status === false) {
    // Send notification only once when the website goes down
    if (!$this->isDownNotified) {
        yield $this->mailActivity->sendFailedNotification($url);
    }

    $this->isDownNotified = true;
    // increase downtime
    $this->downTime += $intervalInMinutes;
} else {
    // Send notification only once when the website goes up
    if (!$this->isRecoveryNotified) {
        yield $this->mailActivity->sendRecoveryNotification($url, $this->downTime);
    }

    $this->downTime = 0;
    $this->isRecoveryNotified = true;
}
```

That's it. Now we can start the workflow.

## Starting the Workflow

### Temporal development server

Before we run the workflow, we need to start the Temporal server.

To start the server, run the following command:

```terminal
temporal server start-dev
```

### RoadRunner server

To run the application, we need to start the RoadRunner server with pre-configured Temporal plugin:

```yaml .rr.yaml
version: '3'
rpc:
  listen: 'tcp://127.0.0.1:6001'

server:
  command: 'php app.php'
  relay: pipes

temporal:
  address: localhost:7233
  activities:
    num_workers: 10
```

and then run the following command:

```terminal
./rr serve
```

### Running the Workflow

We can run the workflow using `Temporal\Client\WorkflowClientInterface` interface. Let's create a console command that
will start the workflow:

```terminal
php app.php create:command CheckStatus
```

This command will create a new console command class in the `app/src/Endpoint/Console` directory with the following
content:

```php app/src/Endpoint/Console/CheckStatusCommand.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'check:status')]
final class CheckStatusCommand extends Command
{
    public function __invoke(): int
    {
        // Put your command logic here
        $this->info('Command logic is not implemented yet');

        return self::SUCCESS;
    }
}
```

Let's add some logic to the command:

```php app/src/Endpoint/Console/CheckStatusCommand.php
#[AsCommand(name: 'check:status')]
final class CheckStatusCommand extends Command
{
    #[Argument(description: 'Domain to check')]
    #[Question(question: 'What domain do you want to check?')]
    private string $domain;

    #[Option(name: 'interval', shortcut: 'i', description: 'Interval in minutes')]
    private int $intervalInMinutes = 5;

    public function __invoke(WorkflowClientInterface $workflowClient): int
    {
        $workflow = $workflowClient->newWorkflowStub(
            WebsiteStatusWorkflow::class,
        );

        $workflowClient->start(
            $workflow,
            $this->domain,
            $this->intervalInMinutes
        );

        return self::SUCCESS;
    }
}
```

And now we can run the workflow using the following command:

```terminal
php app.php check:status https://spiral.dev -i 5
```

That's it. Now you can open the Temporal UI http://127.0.0.1:8233 and see the workflow execution.

## Scheduled Workflows

[Temporal Schedules](https://docs.temporal.io/workflows#schedule) are a replacement for traditional cron jobs for task
scheduling because the Schedules provide a more durable way to execute tasks, allow insight into their progress, enable
observability of schedules and workflow runs, and let you start, stop, and pause them.

To schedule a workflow, you need to use `Temporal\Client\ScheduleClientInterface` interface:

```php app/src/Endpoint/Console/CheckStatusCommand.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;
use Temporal\Client\ScheduleClientInterface;
use Temporal\Client\Schedule;

#[AsCommand(name: 'check:status')]
final class CheckStatusCommand extends Command
{
    #[Argument(description: 'Domain to check')]
    #[Question(question: 'What domain do you want to check?')]
    private string $domain;

    #[Option(name: 'interval', shortcut: 'i', description: 'Interval in minutes')]
    private int $intervalInMinutes = 5;

    public function __invoke(ScheduleClientInterface $client): int
    {
        $client->createSchedule(
            Schedule\Schedule::new()->withAction(
                Schedule\Action\StartWorkflowAction::new(WebsiteStatusWorkflow::class)
                    ->withRetryPolicy(\Temporal\Common\RetryOptions::new()->withMaximumAttempts(3))
                    ->withWorkflowExecutionTimeout('40m')
            )->withSpec(
                Schedule\Spec\ScheduleSpec::new()
                    ->withIntervalList(5 * 60) // every 5 minutes
                    ->withJitter(60) // with jitter of 1 minute
            ),
        );

        return self::SUCCESS;
    }
}
```

> **Note**
> You can find examples of how to use schedules in
> the [Temporal PHP samples repository](https://github.com/temporalio/samples-php/tree/master/app/src/Schedule).

## Console Commands

There are several console commands that can be used to manage workflows and activities:

### List Available Workflows and Activities

To list all available workflows and activities, run the following command:

```terminal
php app.php temporal:info
```

Here is an example output:

```bash
Workflows
=========

+-----------------+------------------------------------------------------+------------------+
| Name            | Class                                                | Task Queue       |
+-----------------+------------------------------------------------------+------------------+
| fooWorkflow     | Spiral\TemporalBridge\Tests\Commands\Workflow        | worker2          |
|                 | src/Commands/InfoCommandTest.php                     |                  |
| AnotherWorkflow | Spiral\TemporalBridge\Tests\Commands\AnotherWorkflow | default, worker2 |
|                 | src/Commands/InfoCommandTest.php                     |                  |
+-----------------+------------------------------------------------------+------------------+
```

You can also use `--with-activities` or `-a` option to list all available activities:

```terminal
php app.php temporal:info --with-activities
```

```bash
Workflows
=========

+-----------------+------------------------------------------------------+------------------+
| Name            | Class                                                | Task Queue       |
+-----------------+------------------------------------------------------+------------------+
| fooWorkflow     | Spiral\TemporalBridge\Tests\Commands\Workflow        | worker2          |
|                 | src/Commands/InfoCommandTest.php                     |                  |
| AnotherWorkflow | Spiral\TemporalBridge\Tests\Commands\AnotherWorkflow | default, worker2 |
|                 | src/Commands/InfoCommandTest.php                     |                  |
+-----------------+------------------------------------------------------+------------------+

Activities
==========

+------------------------+---------------------------------------------+------------+
| Name                   | Class                                       | Task Queue |
+------------------------+---------------------------------------------+------------+
| fooActivity            | ActivityInterfaceWithWorker::foo            | worker1    |
| bar                    | ActivityInterfaceWithWorker::bar            |            |
+------------------------+---------------------------------------------+------------+
| fooActivity__construct | ActivityInterfaceWithoutWorker::__construct | default    |
| fooActivitybaz         | ActivityInterfaceWithoutWorker::baz         |            |
+------------------------+---------------------------------------------+------------+
```