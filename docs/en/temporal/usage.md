# Temporal — Usage

In Temporal, a workflow is a long-running process that is composed of a series of interconnected activities. Workflows
allow developers to define and execute complex processes that may span multiple services or even systems.

Workflows are executed by the Temporal workflow engine. Workflows can be triggered by external events (such as a user
request or a message on a message queue) or they can be scheduled to run at specific intervals.

> **See more**
> Read more about workflows in
> the [Temporal documentation](https://docs.temporal.io/application-development/foundations#develop-workflows).

Here is an example of a simple Temporal workflow interface:

```php app/src/Endpoint/Temporal/Workflow/MonthlySubscription/MonthlySubscriptionInterface.php
namespace App\Endpoint\Temporal\Workflow\MonthlySubscription;

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

```php app/src/Endpoint/Temporal/Workflow/MonthlySubscription/SubscriptionWorkflow.php
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


> **See more**
> You can find examples of Temporal workflows in the [temporalio/samples-php](https://github.com/temporalio/samples-php)
> repository.