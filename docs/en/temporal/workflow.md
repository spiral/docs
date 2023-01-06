# Temporal - Creating Workflows

Temporal is a distributed, fault-tolerant workflow engine that helps developers build, manage, and monitor long-running
processes. It is designed to handle complex workflows with many steps and dependencies, and can scale horizontally to
support large workloads.

> **Note**
> You can find examples of Temporal workflows in the [temporalio/samples-php](https://github.com/temporalio/samples-php)
> repository.

## Workflows

In Temporal, a workflow is a long-running process that is composed of a series of interconnected activities. Workflows
allow developers to define and execute complex processes that may span multiple services or even systems.

Workflows are executed by the Temporal workflow engine. Workflows can be triggered by external events (such as a user
request or a message on a message queue) or they can be scheduled to run at specific intervals.

> **Note**
> Read more about workflows in
> the [Temporal documentation](https://docs.temporal.io/application-development/foundations#develop-workflows).

Here is an example of a simple Temporal workflow interface:

```php
namespace App\Workflow\MonthlySubscription;

use Temporal\Workflow\WorkflowInterface;
use Temporal\Workflow\WorkflowMethod;

#[WorkflowInterface]
interface MonthlySubscriptionInterface
{
    #[WorkflowMethod]
    public function subscribe(string $userId);
}
```

The workflow interface defines the workflow's public API.

And here is an example of a workflow implementation:

```php
namespace App\Workflow\MonthlySubscription;

use Carbon\CarbonInterval;
use Temporal\Activity\ActivityOptions;
use Temporal\Exception\Failure\CanceledFailure;
use Temporal\Workflow;

/**
 * Demonstrates long running process to represent user subscription process.
 */
class SubscriptionWorkflow implements MonthlySubscriptionInterface
{
    private $account;
    private string $userId;
    private int $totalMonths = 0;
    
    public function __construct()
    {
        $this->account = Workflow::newActivityStub(
            AccountActivityInterface::class,
            ActivityOptions::new()
                ->withScheduleToCloseTimeout(CarbonInterval::seconds(2))
        );
    }

    public function subscribe(string $userId)
    {
        $this->userId = $userId;

        yield $this->account->sendWelcomeEmail($userId);

        try {
            $trialPeriod = true;
            while (true) {
                // Lower period duration to observe workflow behaviour
                yield Workflow::timer(CarbonInterval::month());
                $this->totalMonths++;

                if ($trialPeriod) {
                    yield $this->account->sendEndOfTrialEmail($userId);
                    $trialPeriod = false;
                    continue;
                }

                yield $this->account->chargeMonthlyFee($userId);
                yield $this->account->sendMonthlyChargeEmail($userId);
            }
        } catch (CanceledFailure $e) {
            yield Workflow::asyncDetached(
                function () use ($userId) {
                    yield $this->account->processSubscriptionCancellation($userId);
                    yield $this->account->sendSorryToSeeYouGoEmail($userId);
                }
            );
        }
    }
}
```

Each workflow consists of a series of steps, called activities, which are executed in a specific order. Activities can
be asynchronous, meaning they can be called and then later waited on for a result, or they can be called synchronously,
where the calling code blocks until the result is returned.

Here is an example of an activity interface:

```php
namespace App\Workflow\MonthlySubscription;

use Temporal\Activity\ActivityInterface;

#[ActivityInterface]
interface AccountActivityInterface
{
    public function sendWelcomeEmail(string $userID): void;
    public function chargeMonthlyFee(string $userID): void;
    public function sendEndOfTrialEmail(string $userID): void;
    public function sendMonthlyChargeEmail(string $userID): void;
    public function sendSorryToSeeYouGoEmail(string $userID): void;
    public function processSubscriptionCancellation(string $userID): void;
}
```

### Signals

Temporal allows you to use signal methods in your workflows to allow external entities to send signals to the workflow.
A signal method is a special method that can be called from outside the workflow to send a signal to the workflow, which
can be used to trigger some action or update the state of the workflow.

Here is an example of a signal method:

```php
namespace App\Workflow\MonthlySubscription;

use Temporal\Workflow\WorkflowInterface;
use Temporal\Workflow\SignalMethod;
use Temporal\Workflow\WorkflowMethod;

#[WorkflowInterface]
interface MonthlySubscriptionInterface
{
    // ...
    
    #[SignalMethod]
    public function cancelSubscription(): void;
}
```

And an implementation of the signal method:

```php
public function cancelSubscription(): void
{
    yield $this->account->processSubscriptionCancellation($this->userId);
    yield $this->account->sendSorryToSeeYouGoEmail($this->userId);
}
```

### Queries

Temporal also supports query methods in your workflows. A query method is a special method that can be called from
outside the workflow to retrieve information from the workflow. Query methods allow you to access the state of the
workflow and retrieve information without modifying the state of the workflow.

Here is an example of a query method:

```php
namespace App\Workflow\MonthlySubscription;

use Temporal\Workflow\WorkflowInterface;
use Temporal\Workflow\QueryMethod;
use Temporal\Workflow\WorkflowMethod;

#[WorkflowInterface]
interface MonthlySubscriptionInterface
{
    // ...
    
    #[QueryMethod]
    public function totalMonths(): int;
}
```

And an implementation of the query method:

```php
public function totalMonths(): int
{
    return $this->totalMonths;
}
```

The Spiral framework uses the `WorkflowInterface` and `ActivityInterface` attributes to identify workflow and activity
implementations and automatically registers them with the default worker. You can also specify a custom worker for a
particular workflow or activity using the `Spiral\TemporalBridge\Attribute\AssignWorker` attribute. This attribute
allows you to specify a worker name, which can be used to route a workflow or activity to a specific worker.

Here is an example of how to use the AssignWorker attribute to specify a custom worker for a workflow:

```php
use App\Workflow\MyWorkflow;
use Spiral\TemporalBridge\Attribute\AssignWorker;
use Temporal\Workflow\WorkflowInterface;

#[WorkflowInterface]
#[AssignWorker(nameL 'customWorker')]
interface MyCustomWorkflow
{
    // Implementation of the workflow
}
```

By using Temporal to manage and execute workflows, developers can build complex, reliable processes that can run for an
extended period of time. Temporal provides built-in support for fault tolerance, error handling, and retry logic, which
makes it easier to build resilient systems.