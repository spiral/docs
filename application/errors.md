# Error Handling
Spiral will define set of error, exception and shutdown handles while core initiation, you can read about it more [here] (startup.md). By default spiral will work in 
strict mode, meaning that using undefined variables or deprecated functions will thrown an `ErrorException`.

## Error Handler
Default error handler defined in Core method `handleError` and can be redefined in you `Application` class. Let's check it's source due it's very simple:

```php
public function handleError($code, $message, $filename = '', $line = 0)
{
    throw new \ErrorException($message, $code, 0, $filename, $line);
}
```

As you can see that only thing this method is doing - converting generic php error into catchable exception.

## Exception Handler
When application raises uncached exception, such exception will be handled by Core method `handleException`, this method, by default will wrap exception object
using `SnaphotInterface` (by default binded with `Spiral\Debug\Snapshot`). Snapshots used to report, describe and render exceptions. Every generated snapshot
object will be passed to active application dispatcher to display it's content (to be rendered) to use (if allowed, see `HttpDispatcher` [Errors and Isolation] (/http/errors.md)).

### Reporting snapshots
Right before sending snapshot to dispatcher, core will execute `report` method of `SnapshotInterface`, such method dedicated to record information about happen error
or even send this information somewere. Default debug snaoshot implementation will record exception error message into error log and render html file with full exception trace in `runtime/logging/snapshots` directory, this html file can be used to analyze exception post factum.

### Describing snapshot
Snspahot `describe` method simply convers exception information and trace into array, this data can be used to make lighter reporting (without rendering) or, for example,
to be sent to client in case if browser expects JSON.

### Redering snapshot
Render method dedicated to convert exception information into more user friendly form, by default it will render view template defined in `config/debug.php` and send
it to client browser if HttpDispatcher allowing that. Exception view will look like that:

![alt text](https://raw.githubusercontent.com/spiral/guide/master/resources/exception.png)

> You are able to click on method arguments in stack trace to preview their content.

You can simply create your own exception snapshot handler, but impelementing `SnaphotInterface` and writing your own render, describe and report method.

You can edit snaphot options in `debug.php` configuration file:

```php
'snapshots'     => [
    'view'      => 'spiral:exception',
    'reporting' => [
        'enabled'      => true,
        'maxSnapshots' => 20,
        'directory'    => directory('runtime') . '/logging/snapshots',
        'filename'     => '{date}-{exception}.html',
        'dateFormat'   => 'd.m.Y-Hi.s',
    ]
]
```

## Connecting [Whoops](https://github.com/filp/whoops)
Let's try to demonstrate how to change or connect custom exception snaphot. For this purposes we are going to use well know Whoops extension:

```
composer require filp/whoops
```

Once package connected, let's write a simple class in our application:

```php
class Whoops extends QuickSnapshot
{
    /**
     * {@inheritdoc}
     */
    public function render()
    {
        $whoops = new \Whoops\Run;
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

Since this moment every exeception will be rendered using Whoops (as with spiral snaphots you can turn your handler only for your enviroment).

> You can also simply find PSR7 middleware for Whoops.

## More control
If you want to get more control over exception handling in your application you use following techniques:
* write middleware
* disable error handling at moment of application initialization
* attach external error handlers
* rewrite App methods `handleException` or `getSnapshot`
