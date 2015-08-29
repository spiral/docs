# Events and Event Dispatchers
In spiral framework i tried to limit events usage as much as i can, at this moment events are generally used only in DataEntity type classes (ORM and ODM models), every other place trying to utilize interfaces and dependencies. This made me possible to create very light and simple event dispatching mechanism without overloading application
and framework with hidden logic.

> There is only few framework specific events: "notFound" in `Spiral\Core\Components\Loader` and "statement" event in `Spiral\Database\Entities\Database`. In 99% cases you will never met events in your application.

## Events Dispatchers
First of all let's take a look at class which is reposinble for raising and registering listeners ot our events:

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

Event itself represented by simple object which must implement `EventInterface`:

```php
interface EventInterface
{
    /**
     * Event name. Shorted for convenience. You can't change name anyway.
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

As you can see such even provides method named `context()` which will return reference to event payload information with ability to alter such structure, this gives us ability to not onbly listen to something, but modify payload and event result. 

## Event Trait
Spiral events provides ability to create and maintain event dispached on class (not object basis, meaning every class instance will share such event dispatcher), hovewer raised events with become an instace of `ObjectEvent` with reference to parent. We can use EventsTrait for such purposes:

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

As you can see we defined our event (using `fire`) "something" in method doSomething. As event context we provide our input data, same data will be returned when event passed thought every listener.

Let's try to create some listeners:
```php
public function index()
{
    self::events()->listen('something', function (ObjectEvent $event) {
        dump($event->context());
    });

    dump($this->doSomething(['abc']));
}
```

Now we have event listener which will dump our context data (["abc"]) and object with raised an event, if we wish to modify our context we can create different listener:

```php
public function index()
{
    self::events()->listen('something', function (ObjectEvent $event) {
        dump($event->parent());
        dump($event->context());
    });

    self::events()->listen('something', function (ObjectEvent $event) {
        $event->context()[] = 'cba';
    });

    dump($this->doSomething(['abc']));
}
```

Result of our `doSomething` method will be modified (['abc', 'cba']). If you wish to stop every other listener from perforoming, you can call stopPropagnation() method of provided event object:

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
