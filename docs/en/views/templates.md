# Views — Plain PHP

You can use plain php files as your views without any compilation or caching layer. This engine is enabled by default and is
available in the Web application skeleton.

## Creating view

To create a plain PHP view, simply put a file with `.php` extension into the `views` directory. The template can be rendered by it's filename:

```php
<?php // test.php
echo "hello world";
```

```php
public function index(ViewsInterface $views): string
{
    return $views->render('test'); // no need to specify the extension
}
```

All the passed parameters will be available as PHP variables:

```php
public function index(ViewsInterface $views): string
{
    return $views->render('test', ['name' => 'Antony']); 
}
```

Where `test.php`:

```php
Hello , <?= $name ?>!
```

## Container

You can access the container in your view files via `$this->container`:

```php
Hello world, <?= $name ?>!

<?php dump($this->container->get(MyService::class)); ?>
```


# Views - Twig

The framework provides deep integration with the [Twig Template](https://twig.symfony.com/) engine including access to IoC
scopes, i18n integration, and caching.

## Installation and Configuration

To install Twig:

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

```php
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

```twig
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

```twig
Hello, {{ name }}!

{{ get("Spiral\\Http\\Request\\InputManager").attribute('csrfToken') }}
```
