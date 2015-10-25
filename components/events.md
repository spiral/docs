# Events and Event Dispatchers
In Spiral framework I tried to limit events usage as much as possible to make the framework core and components as clear as possible. At this moment, events are only used in DataEntity type classes (ORM and ODM models), Core Loader and Database entity class, in every other place I'm trying to utilize interfaces and dependencies. 

> Spiral event interfaces might be implemented using Symfony\Events in future. At this moment you are able to tweak framework behaviour without using events in core classes (some events might be added in future by request).

## Events Dispatchers
First of all let's take a look at the class which is responsible for raising and registering listeners to our events:

```php
interface DispatcherInterface
{
    /**
     * Add listener for specified event.
     *
     * @param string   $event
     * @param callable $listener
     * @return self
     */
    public function listen($event, $listener);

    /**
     * Remove event specific listener.
     *
     * @param string   $event
     * @param callable $listener
     * @return self
     */
    public function remove($event, $listener);

    /**
     * Check if event has specified listener.
     *
     * @param string   $event
     * @param callable $listener
     * @return bool
     */
    public function hasListener($event, $listener);

    /**
     * List of every event listener.
     *
     * @param string $event
     * @return callable[]
     */
    public function listeners($event);

    /**
     * Fire event instance or create default event implementation with specified context.
     *
     * @param EventInterface|string $event   Event instance or event name (default implementation to
     *                                       use).
     * @param mixed                 $context Event context to be passed.
     * @return mixed                         Passed event context.
     * @throws InvalidArgumentException
     */
    public function fire($event, $context = null);
}
```

Event itself is represented by a simple object which must implement `EventInterface`:

```php
interface EventInterface
{
    /**
     * Event name. Shortened for convenience. You can't change name anyway.
     *
     * @return string
     */
    public function name();

    /**
     * Get event context reference. Can point to anything (usually array) and should be editable.
     *
     * @return mixed
     */
    public function &context();

    /**
     * Event being stopped. Listeners chain should be aborted.
     *
     * @return bool
     */
    public function isStopped();

    /**
     * Has to be set by listener to indicate that no other listener has to be executed.
     */
    public function stopPropagation();
}
```

As you can see, events have as method named `context()` which will return a reference to event payload information with the ability to alter the structure. This gives us the ability to not only listen to something, but modify the payload and event chain result.

## Event Trait
The Spiral event component provides the ability to create and maintain an event dispacher on a class level (meaning every class instance will utilize the same dispatcher and set of listements, this is mainly done for performance reasons), however raised events will become an instace of `ObjectEvent` with a reference to the parent. Let's use the `EventsTrait` to do this:

```php
class HomeController extends Controller
{
    use EventsTrait;

    public function index() 
    {
        dump($this->doSomething('abc'));
    }

    public function doSomething($data)
    {
        return $this->fire('something', $data);
    }
}
```

As you can see we, define our event "something" (using the `fire` method) in the "doSomething" action. As event context we will provide our input data variable, same data will be returned when event passed thought every listener. Let's try to create some listeners

```php
public function index()
{
    self::events()->listen('something', function (ObjectEvent $event) {
        dump($event->context());
        dump($event->parent());
    });

    dump($this->doSomething(['abc']));
}
```

Now we have event listener which will dump our context data (["abc"]) and object which raised an event, if we wish to modify our context we can create different listener:

```php
public function index()
{
    self::events()->listen('something', function (ObjectEvent $event) {
        dump($event->parent());
        dump($event->context());
    });

    self::events()->listen('something', function (ObjectEvent $event) {
        $context = &$event->context();
        $context[] = 'cba';
    });

    dump($this->doSomething(['abc']));
}
```

Result of our `doSomething` method now will be modified (['abc', 'cba']). If you wish to stop every other listener from performing, you can call stopPropagnation() method of provided event object:

```php
public function index()
{
    self::events()->listen('something', function (ObjectEvent $event) {
        $event->stopPropagation();
    });

    self::events()->listen('something', function (ObjectEvent $event) {
        $event->context()[] = 'cba';
    });

    dump($this->doSomething(['abc']));
}
```

Second event listener will never be executed.
> At this moment listeners executed in order of how they was registered in dispatcher.
