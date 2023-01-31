# Views — Twig templating

The framework provides deep integration with the [Twig Template](https://twig.symfony.com/) engine including access to IoC
scopes, i18n integration, and caching.

## Installation and Configuration

To install Twig bridge component, run the following command:

```terminal
composer require spiral/twig-bridge
```

The extension can be enabled using `Spiral\Twig\Bootloader\TwigBootloader`.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Views\Bootloader\ViewsBootloader::class,
    \Spiral\Bootloader\Views\TranslatedCacheBootloader::class, // keep localized views in separate cache files
    \Spiral\Twig\Bootloader\TwigBootloader::class,
    // ...
];
```

You can add any custom extension to Twig via the `addExtension` method of `TwigBootloader`:

```php app/src/Application/Bootloader/TwigExtensionBootloader.php
class TwigExtensionBootloader extends Bootloader
{
    public function boot(TwigBootloader $twig): void
    {
        $twig->addExtension(MyExtension::class);
    
        // custom options
        $twig->setOption('name', 'value');
    }
}
```

## Usage

You can use twig views immediately. Create a view with `.twig` extension in the `app/views` directory.

```twig app/views/filename.twig
Hello, {{ name }}!
```

You can use this view without an extension in your controllers:

```php
public function index(): string
{
    return $this->views->render('filename', ['name' => 'User']);
}
```

> **Note**
> You can freely use twig's `include` and `extends` directives.

To access the value from the IoC scope:

```twig app/views/filename.twig
Hello, {{ name }}!

{{ get("Spiral\\Http\\Request\\InputManager").attribute('csrfToken') }}
```

## Debug

In twig you can use the `dump` function to display information about a variable, including its type and value. Its useful for debugging purposes in your templates.

To enable the function create a new Bootloader: `TwigDebugBootloader`:

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Twig\Bootloader\TwigBootloader;
use Twig\Extension\DebugExtension;

class TwigDebugBootloader extends Bootloader
{
    public function boot(TwigBootloader $twig)
        {
            $twig->addExtension(new DebugExtension());
            $twig->setOption('debug', true);
        }
}
```

make sure your created `Bootloader` is loaded in your `App.php`:

```php
protected const LOAD = [
    // ...
    TwigDebugBootloader::class
    // ...
]
```

Afterwards you can use the `{{ dump() }}` function in your template.

```twig
<pre>
    {{ dump(cats) }}
</pr>
```
`cats` in this case is the variable we would like to debug.

> **Note**
> You can assign more variables by passing them as arguments:

```twig
    {{ dump(cats, dogs, birds) }}
```

If we dont pass any value to the function, all variables from the current context will be dumped.

> **Note**
> For more informations, visit the official [twig documentation](https://twig.symfony.com/doc/3.x/functions/dump.html)
