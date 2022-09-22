# Filters from Spiral Framework 2.x

If you are migrating from Spiral Framework 2.x and want to continue using old filters you can
use [spiral/filters-bridge](https://github.com/spiral/filters-bridge) package.

The package `spiral/filters-bridge` provides support for request validation, composite validation, an error message
mapping and locations, etc.

## Requirements

Make sure that your server is configured with following PHP version and extensions:

- PHP 8.1+
- Spiral framework 3.0+

## Installation

To install the package:

```bash
composer require spiral/filters-bridge
```

> **Note**
> The package will automatically install and configure `spiral/validator` package.

The package does not require any configuration and can be activated using the
bootloader `Spiral\Filters\Bootloader\FiltersBootloader`:

```php
[
    // ...
    \Spiral\Filters\Bootloader\FiltersBootloader::class
    // ...
]
```

There are no difference between using `spiral/filters` component and `spiral/filters-bridge` package, so you can use
[documentation](https://spiral.dev/docs/filters-configuration/2.14/en#create-filter) for previous the Spiral Framework 
version.