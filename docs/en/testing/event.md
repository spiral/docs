# Testing — Events Tests

The Spiral Framework has a convenient way to test the [spiral/event](../advanced/events.md) component. When testing code
that dispatches events, you may want to prevent the event's listeners from actually executing. You can use the
`fakeEventDispatcher` method to do this.

```php
private \Spiral\Testing\Events\FakeEventDispatcher $events;
    
protected function setUp(): void
{
    parent::setUp();
    $this->events = $this->fakeEventDispatcher();
}
```

It allows you to replace the default event dispatcher in the application container with a fake one. This fake event
dispatcher, `Spiral\Testing\Events\FakeEventDispatcher`, records all events that are dispatched and provides assertion
methods that you can use to check if specific events were dispatched and how many times.

This method allows you to pass an array of event classes as an argument. When you pass a list of specific event classes,
the fake event dispatcher will only record and provide the assertion methods for those specific events. Other events
that are dispatched will be handled by the default event dispatcher. This feature is useful when you only want to test
a specific part of your code that dispatches certain events, and not all events that are dispatched in the application.

```php
private \Spiral\Testing\Events\FakeEventDispatcher $events;
    
protected function setUp(): void
{
    parent::setUp();
    $this->events = $this->fakeEventDispatcher([UserRegistered::class]);
}
```

After you execute the code you want to test, you can then use the `assertDispatched`, `assertDispatchedTimes`,
`assertNotDispatched`, and `assertNothingDispatched` methods to check which events were dispatched by your application.

```php
use Tests\TestCase;

final class UserServiceTest extends TestCase
{
    private \Spiral\Testing\Events\FakeEventDispatcher $events;

    protected function setUp(): void
    {
        parent::setUp();
        $this->events = $this->fakeEventDispatcher();
    }


    public function testUserShouldBeRegistered(): void
    {
        // Perform user registration ...

        $this->events->assertDispatched(UserRegistered::class);
    }
}
```

In addition, you can also pass a closure to the `assertDispatched` or `assertNotDispatched` methods to check if an event
was dispatched that meets a certain condition. For example, you can check if an event's parameter equals a certain
value. If at least one event was dispatched that meets the condition, then the assertion will be successful.

### Asserting Events Were Dispatched

```php
// Assert if an event dispatched one or more times
$this->events->assertDispatched(SomeEvent::class);

// Assert if an event dispatched one or more times based on a truth-test callback.
$this->events->assertDispatched(
    UserRegistered::class, 
    static function(UserRegistered $event): bool {
        return $event->username === 'john_smith';
    }
);
```

### Asserting Events Were dispatched a Specific Number of Times

```php
$this->events->assertDispatchedTimes(UserRegistered::class, 1);
```

### Asserting Events Were Not Dispatched

```php
$this->events->assertNotDispatched(UserRegistered::class);

$this->events->assertNotDispatched(
    UserRegistered::class, 
    static function(UserRegistered $event): bool {
        return $event->username === 'john_smith';
    }
);
```

### Asserting No Events Were Dispatched

```php
$this->events->assertNothingDispatched();
```

### Getting All Dispatched Events

Get all the events matching a truth-test callback.

```php
$events = $this->events->dispatched(UserRegistered::class);

// or

$events =  $this->events->dispatched(
    UserRegistered::class, 
    static function(UserRegistered $event): bool {
        return $event->someParam === 100;
    }
);
```