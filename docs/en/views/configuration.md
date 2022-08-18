# Views - Installation and Configuration

The `spiral/views` component is available in Web bundle by default, to install it in alternative builds:

```bash
composer require spiral/views
```

> **Note**
> Please note that the spiral/framework >= 2.6 already includes this component.

To activate the component use bootloader `Spiral\Views\Bootloader\ViewsBootloader`:

```php
[
    // ...
    Spiral\Views\Bootloader\ViewsBootloader::class,
    // ...
]
```

The component can work with multiple rendering engines (**Twig**, **Stempler** or **native PHP** templates) and
store view files in multiple namespaces.

## Configuration

To change the default component configuration create and edit file `app/config/views.php`:

```php
<?php
declare(strict_types=1);

use Spiral\Views\Engine\Native\NativeEngine;

return [
    'cache' => [
        'enabled' => !env('DEBUG', false),
        'directory' => directory('cache') . 'views'
    ],
    'namespaces' => [
        'default' => [
            directory('views')
        ]
    ],
    'dependencies' => [],
    'engines' => [
        NativeEngine::class
    ],
    'globalVariables' => [
        'some_var' => env('SOME_VALUE')
    ]
];
```

## Auto-Configuration

You can alter some of the component settings using bootloader `Spiral\Bootloaders\Views\ViewsBootloader`.

### View Namespace

To assign a new directory to view namespace:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Views\Bootloader\ViewsBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ViewsBootloader $views): void
    {
        $views->addDirectory('default', directory('root') . 'views');
    }
}
```

### View Engine

To register a new view engine make sure to implement `Spiral\Views\EngineInterface`:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Views\Bootloader\ViewsBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ViewsBootloader $views): void
    {
        $views->addEngine(MyEngine::class);
    }
}
```

## Caching

By default, view caching is turned off if the env variable `DEBUG` is set to true. The cache files are stored in
`runtime/cache/views`.

## Purge

Run the console command `views:reset` to delete view cache:

```bash
php app.php views:reset
```

## Warmup

To warmup cache before run `views:compile` or `configure`:

```bash
php app.php views:compile
``` 

> **Note**
> Always warmup view cache before switching worker pool under load.
