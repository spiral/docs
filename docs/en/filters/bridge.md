# Filters — Migration from Spiral Framework 2.x

If you are migrating from Spiral Framework 2.x, and you want to continue using the old filters you can
use the [spiral/filters-bridge](https://github.com/spiral/filters-bridge) package in this case.

The package `spiral/filters-bridge` provides support for request validation, composite validation, an error message
mapping and locations, etc.

## Requirements

Make sure that your server is configured with the following PHP version and extensions:

- PHP 8.1+
- Spiral framework 3.0+

## Installation

To install the package:

```terminal
composer require spiral/filters-bridge
```

> **Note**
> The package will automatically install and configure the `spiral/validator` package.

The package does not require any configuration and can be activated using the
bootloader `Spiral\Filters\Bootloader\FiltersBootloader`:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Filters\Bootloader\FiltersBootloader::class,
        // ...
    ];
}
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Filters\Bootloader\FiltersBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

There is no difference between using the `spiral/filters` component and the `spiral/filters-bridge` package, so you can use
[documentation](https://spiral.dev/docs/filters-configuration/2.14/en#create-filter) for the previous Spiral version.
