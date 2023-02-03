# Testing — Queue Tests

When testing code that sends out jobs, you typically want to make sure a specific job was sent out, but you don't
actually want the job to run. This is because the job's execution can be tested in a separate test.

Spiral has a handy way to test the job sending [spiral/queue](../queue/configuration.md) component. You can use 
the `fakeQueue` method to prevent jobs from being sent to the actual queue.

The `fakeQueue` method allows you to replace the default queue connection provider in the application container with a
fake one, specifically the `Spiral\Testing\Queue\FakeQueueManager` class. This fake queue connection provider creates
fake queues that record all jobs that are pushed to it.

You can then use the assertion methods provided by the class, such as `assertPushed`, `assertPushedOnQueue`, 
`assertPushedTimes`, `assertNotPushed`, and `assertNothingPushed`, to check if specific jobs were pushed, and how many 
times they were pushed. These methods allow you to test your application's behavior when it comes to dispatching jobs 
without actually having to run the jobs themselves.

```php
use Spiral\Mailer\Message;
use Tests\TestCase;

final class UserServiceTest extends TestCase
{
    private \Spiral\Testing\Queue\FakeQueueManager $connection;
    private \Spiral\Testing\Queue\FakeQueue $queue;

    protected function setUp(): void
    {
        parent::setUp();
        $this->connection = $this->fakeQueue();
        $this->queue = $this->connection->getConnection('default');
    }

    public function testUserShouldBeRegistered(): void
    {
        // Perform user registration ...

        $this->queue->assertPushed('mail.job', function (array $data) {
            return $data['handler'] instanceof \Spiral\SendIt\MailJob
                && $data['options']->getQueue() === 'mail'
                && $data['payload']['foo'] === 'bar';
        });
    }
}
```

### Asserting Jobs Were Pushed

The `assertPushed` method can be used to check if a specific job was pushed to the queue. You can also pass a closure to
this method, which will be used as a "truth test" to check if the job that was pushed meets certain conditions.

```php
$this->queue->assertPushed('mail.job', function (array $data) {
    return $data['handler'] instanceof \Spiral\SendIt\MailJob
        && $data['options']->getQueue() === 'mail'
        && $data['payload']['foo'] === 'bar';
});
```

### Asserting Jobs Were Pushed to a Specific Queue

Additionally, you can use the `assertPushedOnQueue` method to check if a job was pushed to a specific queue.

```php
$this->queue->assertPushedOnQueue('mail', 'mail.job', function (array $data) {
    return $data['handler'] instanceof \Spiral\SendIt\MailJob
        && $data['payload']['foo'] === 'bar';
});
```

### Asserting Jobs Were Pushed a Specific Number of Times

And you can use the `assertPushedTimes` method to check if a job was pushed a certain number of times.

```php
$this->queue->assertPushedTimes('mail.job', 2);
```

### Asserting Jobs Were not Pushed

You can also use the `assertNotPushed` method to check if a specific job was not pushed to the queue

```php
$this->queue->assertNotPushed('mail.job', function (array $data) {
    return $data['handler'] instanceof \Spiral\SendIt\MailJob
        && $data['options']->getQueue() === 'mail'
        && $data['payload']['foo'] === 'bar';
});
```

And the `assertNothingPushed` method to check if no jobs were pushed to the queue. These methods are useful for making
sure that certain jobs or queues are not being used when they shouldn't be.

```php
$this->queue->assertNothingPushed();
```
