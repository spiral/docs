# Extensions - Sentry

You can automatically send the exceptions to the Sentry.

## Installation

To install the extension:

```bash
composer require spiral/sentry-bridge
```

Sentry provides two bootloaders `Spiral\Sentry\Bootloader\SentryReporterBootloader` and
`Spiral\Sentry\Bootloader\SentryBootloader`.

### Sentry as Reporter

The `SentryReporterBootloader` registers the `Spiral\Sentry\SentryReporter` in the `Exceptions` component. 
The Reporter sends the exception to Sentry.

```php
protected const LOAD = [
    // ...
    Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

### Sentry as Snapshotter

The `SentryBootloader` registers the `Spiral\Sentry\SentrySnapshotter` as `SnapshotterInterface` in `Snapshots` component.
The SentrySnapshotter sends the exception to Sentry and creates snapshot with error ID from Sentry. Use this bootloader 
only if you need to save a snapshot of the exception with the ID under which the exception is registered in Sentry.
Also, remove the default `Spiral\Bootloader\SnapshotsBootloader`.

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

## Additional Data

To expose current application logs, PSR-7 request state, etc. enable additional debug extensions:

```php
protected const LOAD = [
    // ...
    Spiral\Sentry\Bootloader\SentryBootloader::class,
  
    // ...
    
    // at the end of the chain
    Spiral\Bootloader\DebugBootloader::class,
    Spiral\Bootloader\Debug\LogCollectorBootloader::class,
    Spiral\Bootloader\Debug\HttpCollectorBootloader::class,   
];
```
