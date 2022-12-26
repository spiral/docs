# Debug - Handling Exceptions

During the development process, it is common for errors and exceptions to arise. Debugging these exceptions can be a challenging and time-consuming task, but it is a critical aspect of the development process. The Spiral Framework offers a range of tools and techniques for debugging exceptions and identifying the underlying cause of issues.

## Using Yii Error Handler

The Yii Error Handler is a bridge package for Spiral Framework that provides integration with the Yii framework's error handling mechanism. This allows developers to use the Yii error handling system within their Spiral application.

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

The `YiiErrorHandlerBootloader` will register all available renderers during initialization. If you wish to register specific renderers, you can refer to the [Exceptions](../component/exceptions.md) documentation.

### Built-in renderers

The bridge provides several built-in renderers for displaying errors:

- `HtmlRenderer`: Renders error pages as HTML.
- `JsonRenderer`: Renders error pages as JSON. This can be useful for handling errors in API requests.
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
