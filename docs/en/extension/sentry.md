# Extensions - Sentry

You can automatically send the exceptions to the Sentry.

## Installation

To install the extension:

```bash
composer require spiral/sentry-bridge
```
After package install you need to add bootloader from the package in your application. The package provides two 
bootloaders `Spiral\Sentry\Bootloader\SentryReporterBootloader` and `Spiral\Sentry\Bootloader\SentryBootloader`.

### Sentry as a reporter

The `SentryReporterBootloader` registers the `Spiral\Sentry\SentryReporter` in the `Exceptions` component. 
The Reporter sends the exception to the Sentry.

```php
protected const LOAD = [
    // ...
    Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

### Sentry as a snapshotter

The `SentryBootloader` registers the `Spiral\Sentry\SentrySnapshotter` as `SnapshotterInterface` in `Snapshots` component.
The bootloader sends the exception to the Sentry and also creates a snapshot with error ID received from the 
Sentry. 

Use this bootloader only if you need to save the snapshot of the exception with the ID under which the exception 
is registered in the Sentry. 

Also, you have to remove the default `Spiral\Bootloader\SnapshotsBootloader` bootloader.

```php
protected const LOAD = [
    // ...
    Spiral\Sentry\Bootloader\SentryBootloader::class,
    // ...
];
```

> **Note**
> Use one of the provided bootloaders. Don't use both bootloaders.

The component will look at `SENTRY_DSN` env value.

```dotenv
SENTRY_DSN=https://...@sentry.io/...
```

## Additional Data

To expose current application logs, PSR-7 request state, etc you can enable additional debug extensions

### Http collector

Use HTTP collector to send data about current HTTP request to the Sentry.

It will send the following information about current request:
 - method
 - url
 - headers
 - query params
 - request body

To enable HTTP collector at first you need to register `Spiral\Bootloader\Debug\HttpCollectorBootloader` before 
`SentryBootaloder` or `SentryReporterBootloader`.

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\Debug\HttpCollectorBootloader::class, 
    // ...
];
```

Then you need to register middleware `Spiral\Debug\StateCollector\HttpCollector`. 

> **Note**
> Read more how to register a middleware [here](/http/middleware.md).


### Logs collector

Use Logs collector to send all received application logs to the Sentry.

To enable Logs collector you just need to register `Spiral\Bootloader\Debug\LogCollectorBootloader` before
`SentryBootaloder` or `SentryReporterBootloader`.

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\Debug\LogCollectorBootloader::class,   
    // ...
];
```

### Custom data collector

You can use `Spiral\Debug\StateInterface` class to send custom data to the Sentry.

You can request State object from the container.

```php
$state = $container->get(\Spiral\Debug\StateInterface::class);
```

#### Add a tag

The method will add tags associated to the current scope

```php
$state->addTag('IP address', $currentRequest->getIpAddress());
$state->addTag('Environment', $env->get('APP_ENV'));
```

#### Add a variable

The method will add extra data associated to the current scope

```php
$state->setVariable('query', $currentRequest->getQueryParams());
```

#### Add a log event

The method will add a log event as a breadcrumb to the current scope.

```php
$state->addLogEvent(new \Spiral\Logger\Event\LogEvent(
    time: new \DateTimeImmutable(),
    channel: 'default',
    level: 'info',
    message: 'Something went wrong',
    context: ['foo' => 'bar']
));
```