# Extensions - Sentry

You can automatically send the exceptions to the Sentry service.

## Installation

To install the extension:

```bash
composer require spiral/sentry-bridge
```

After package install you need to add bootloader `Spiral\Sentry\Bootloader\SentryBootloader` from the package in your
application.

```php
protected const LOAD = [
    // ...
    Spiral\Sentry\Bootloader\SentryBootloader::class,
    // ...
];
```

> **Note**
> Don't forget to disable `Spiral\Bootloader\SnapshotsBootloader` bootloader in the bootloaders list.

The component will look at `SENTRY_DSN` env value.

```dotenv
SENTRY_DSN=https://...@sentry.io/...
```

## Additional Data

To expose current application logs, PSR-7 request state, etc. you can enable additional debug extensions

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
