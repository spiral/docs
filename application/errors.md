# Error Handling
Spiral will defined set of error, exception and shutdown handles while core initiation, you can read about it more [here] (startup.md). By default spiral will work in 
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
using `SnaphotInterface` (by default binded with `Spiral\Debug\Snapshot)`). Snapshots used to report, describe and render exceptions. Every generated snapshot
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