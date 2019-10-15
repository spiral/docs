# Views - Twig
Framework provides deep integration with [Twig Template](https://twig.symfony.com/) engine including access to IoC
scopes, i18n integration and caching.

## Installation and Configuration
To install Twig:

```bash
$ composer require spiral/twig-bridge
```

The extension can be enabled using `Spiral\Twig\Bootloader\TwigBootloader`.

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\Views\ViewsBootloader::class,
    Spiral\Bootloader\Views\TranslatedCacheBootloader::class, // to keep translated views in different cache files
    Spiral\Twig\Bootloader\TwigBootloader::class,
    // ...
];
```

You can add any custom extension to Twig via `addExtension` method of `TwigBootloader`:

```php
class TwigExtensionBootloader extends Bootloader
{
    public function boot(TwigBootloader $twig)
    {
        $twig->addExtension(MyExtension::class);
    
        // custom options
        $twig->setOption('name', 'value');
    }
}
```

## Usage
You can use twig views immediately. Create view with `.twig` extension in `app/views` directory.

```twig
Hello, {{ name }}!
```

You can use this view without an extension in your controllers:

```php
public function index()
{
    return $this->views->render('filename', ['name' => 'User']);
}
```

> You can freely use twig include and extends directives.

To access the value from the IoC scope:

```php
Hello, {{ name }}!

{{ get("Spiral\\Http\\Request\\InputManager").attribute('csrfToken') }}
```