# Debug - Handling Exceptions

During the development process, it is common for errors and exceptions to arise. Debugging these exceptions can be a challenging and time-consuming task, but it is a critical aspect of the development process. The Spiral Framework offers a range of tools and techniques for debugging exceptions and identifying the underlying cause of issues. By utilizing these resources, you can efficiently troubleshoot and resolve exceptions in your application.

## Using Yii Error Handler bridge

The Yii Error Handler bridge is a package for Spiral Framework that provides integration with the Yii framework's error handling mechanism. This allows developers to use the Yii error handling system within their Spiral applications.

### Installation
```bash
composer require spiral/yii-error-handler-bridge
```

### Usage
After package install you need to register bootloader from the package:

```php
protected const LOAD = [
    // ...
    \Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader::class,
];
```

The `YiiErrorHandlerBootloader` registers all available renderers during initialization. If you wish to register specific renderers, you can refer to the Exceptions documentation.

### Built-in renderers

The Yii Error Handler bridge provides several built-in renderers for rendering error pages:

- `HtmlRenderer`: Renders error pages as HTML. This is the default renderer used by the yii error handler bridge.
- `JsonRenderer`: Renders error pages as JSON. This can be useful for handling errors in API requests.
- `XmlRenderer`: Renders error pages as XML. This can also be useful for handling errors in API requests.
- `PlainTextRenderer`: Renders error pages as plain text.

## Snapshots

The framework offers a unified approach for managing exception registration, including fatal exceptions, through the `spiral/snapshots` package. This feature is designed to facilitate exception registration in external monitoring solutions such as Sentry.

Custom snapshot providers can be implemented using the `Spiral\Snapshots\SnapshotterInterface` interface.
```php
use Spiral\Snapshots\Snapshot;
use Spiral\Snapshots\SnapshotInterface;
use Spiral\Snapshots\SnapshotterInterface;

class MySnapshotter implements SnapshotterInterface
{
    public function register(\Throwable $e): SnapshotInterface
    {
        // register exception in log, Sentry or etc

        return new Snapshot('unique-id', $e);
    }
}
```

> **Note**
> To enable your implementation, it is necessary to bind it to the `Spiral\Snapshots\SnapshotterInterface` interface.
