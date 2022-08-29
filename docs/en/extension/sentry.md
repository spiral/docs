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

```php
protected const LOAD = [
    // ...
    
    Spiral\Bootloader\DebugBootloader::class,
    Spiral\Bootloader\Debug\LogCollectorBootloader::class,
    Spiral\Bootloader\Debug\HttpCollectorBootloader::class,   
    
    Spiral\Sentry\Bootloader\SentryBootloader::class,
    
    // ...
];
```

> **Note**
> Better place for the extension bootloaders is before `Spiral\Bootloader\SnapshotsBootloader`.
