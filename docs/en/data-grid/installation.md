# Installation

To install the Data Grid component, you need to install both the core component and the bridge:

```terminal
composer require spiral/data-grid-bridge spiral/cycle-bridge
```

> **Warning:**
> Spiral does not provide Cycle ORM out of the box. To use this component, you need to install `spiral/cycle-bridge`
> component. You can find more information about Cycle ORM Bridge in
> the [The Basics — Database and ORM](../basics/orm.md)section.

## Bootloader Registration

Activate the bootloader `Spiral\DataGrid\Bootloader\GridBootloader` in your application after Cycle bootloaders:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\DataGrid\Bootloader\GridBootloader::class,
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
    \Spiral\DataGrid\Bootloader\GridBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

## Configuration

After installation, set up the writers for your data sources in the `app/config/dataGrid.php` config file.

**Here's a basic setup for Cycle ORM Bridge:**

```php app/config/dataGrid.php
return [
    'writers' => [
        \Spiral\Cycle\DataGrid\Writer\QueryWriter::class,
        \Spiral\Cycle\DataGrid\Writer\PostgresQueryWriter::class,
        \Spiral\Cycle\DataGrid\Writer\BetweenWriter::class,
    ],
];
```

> **Note**
> As you can see, we can register multiple writers for different specifications. The order of writers is important, as
> the compiler will use them in the same order as they are registered.