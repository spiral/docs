# Error Handling
Exception and shutdown handlers created during [core initiation](/v1.0.0/application/startup.md). By default framework will work in a strict mode, meaning that any undefined variable or deprecated function will thrown an `ErrorException`.

## Error Handler
Default error handler defined in Core method `handleError` and can be redefined in you `App` class. Let's check it's source due it's very simple:

```php
public function handleError($code, $message, $filename = '', $line = 0)
{
    throw new \ErrorException($message, $code, 0, $filename, $line);
}
```

## Exception Handler and Snapshots
When application raises uncaught exception, such exception will be handled by Core method `handleException`, this method, by default, will wrap exception object using `SnaphotInterface` (by default binded with `Spiral\Debug\QuickSnapshot`). 

Every generated snapshot object will later be passed to active application dispatcher to render it's content to user (if allowed, see `HttpConfig->exposeErrors()`).

```php
interface SnapshotInterface
{
    /**
     * Associated exception.
     *
     * @return \Throwable|\Exception
     */
    public function getException(): \Throwable;

    /**
     * Must return formatted exception message including exception class, location and etc.
     *
     * @return string
     */
    public function getMessage(): string;

    /**
     * Get shortened exception description in array form.
     *
     * @return array
     */
    public function describe(): array;

    /**
     * Render snapshot information into string or html.
     *
     * @return string
     */
    public function render(): string;

    /**
     * Report or store snapshot in known location. Used to store exception information for future
     * analysis.
     */
    public function report();
}
```

### Reporting snapshots
Right before sending snapshot to dispatcher, core will execute `report` method of `SnapshotInterface`, such method dedicated to record information about happen error or even send this information somewhere. 

QuickSnapshot implementation will record formatted exception message into error log and render html file with full exception trace in `runtime/snapshots` directory, this html file can be used to analyze exception post factum.

### Describing snapshot
Snapshot `describe` method simply converts error information and trace into array form, this data can be used to make lighter reporting (without rendering) or, for example, to be sent to client in case if browser expects JSON.

### Rendering snapshot
Render method dedicated to convert exception information into more user friendly form, by default it will render view template defined in `config/snaphots.php` config and send it to client browser if `HttpDispatcher` allowing that. Exception view will look like that:

![alt text](https://raw.githubusercontent.com/spiral/guide/master/resources/exception.png)

Snapshots configuration:

```php
return [
    /*
     * View to render snapshot/exception information, please note that "slow" view will dump all
     * track arguments which can lead to much longer executing time.
     */
    'view'      => env('EXCEPTION_VIEW', 'spiral:exceptions/light/slow.php'),

    /*
     * Automatic snapshot reporting options. You can define your own SnapshotInterface in order
     * to create custom handler OR add Monolog handler to default log channel.
     */
    'reporting' => [
        'enabled'      => true,
        'maxSnapshots' => 20,
        'directory'    => directory('runtime') . 'snapshots',
        'filename'     => '{date}-{name}.html',
        'dateFormat'   => 'd.m.Y-Hi.s',
    ]
];
```

### Exception views
Spiral debug snapshot provide an ability to render error using specified view file, at this moment framework ships with 4 different exception views:

View                                | Description
---                                 | ---
spiral:exceptions/light/slow.php    | Light exception view, argument are clickable.
spiral:exceptions/light/fast.php    | Light exception view, argument are not clickable.
spiral:exceptions/dark/slow.php     | Dark exception view, argument are clickable.
spiral:exceptions/dark/fast.php     | Dark exception view, argument are not clickable.

## Connecting [Whoops](https://github.com/filp/whoops)
Let's try to demonstrate how to change or connect custom exception snapshot. For this purposes we are going to use well know Whoops extension:

```
composer require filp/whoops
```

Now we can create our snapshot implementation:

```php
class Whoops extends QuickSnapshot
{
    /**
     * {@inheritdoc}
     */
    public function render()
    {
        $whoops = new \Whoops\Run();
        $whoops->pushHandler(new \Whoops\Handler\PrettyPageHandler());
        return $whoops->handleException($this->exception());
    }
}
```

To make this snapshot work we only have to bind our class to `SnaphotInterface`, we can do it in App class:

```php
//$this->container->bind(Debug\SnapshotInterface::class, Debug\Snapshot::class);
$this->container->bind(Debug\SnapshotInterface::class, \Snapshots\Whoops::class);
```

> You can also simply find [PSR7 middleware for Whoops](https://github.com/oscarotero/psr7-middlewares) to use alternative error handling methodic.

## More control
If you want to get more control over exception handling in your application you use following techniques:
* write middleware
* disable error handling at moment of application initialization
* attach external error handlers
* rewrite `App` methods `handleException` or `getSnapshot` to incorporate custom logic or handle only specific exceptions
