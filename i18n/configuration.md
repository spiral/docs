# Internationalization - Installation and Configuration
The spiral i18n component is based on the `symfony/translator` package and provides compatible interfaces. The web bundle includes
this extension by default.

To enable the extension in alternative builds:

```bash
$ composer require spiral/translator
```

> Please note that the spiral/framework >= 2.6 already includes this component.

Activate the bootloader `Spiral/Bootloader/I18nBootloader` in your application.

## Usage
By default, the translator component will automatically load all locale files located in `app/locale/{lang}` directories.

To get a list of available locales:

```php
namespace App\Controller;

use Spiral\Translator\TranslatorInterface;

class HomeController
{
    public function index(TranslatorInterface $translator)
    {
        dump($translator->getCatalogueManager()->getLocales());
    }
}
```

To change the locale, use the `setLocale` method. 

> It's recommended to use the `setLocale` method inside middleware or domain core and properly reset the original locate after.

## Default Configuration
The framework supplies the default configuration automatically, inside the `I18nBootloader`. To alter the configuration
create `app/config/translator.php` file:

```php
<?php
use Symfony\Component\Translation\Dumper;
use Symfony\Component\Translation\Loader;

return [
    'locale'         => env('LOCALE', 'en'),
    'fallbackLocale' => env('LOCALE', 'en'),
    'directory'      => directory('locale'),
    'autoRegister'   => env('DEBUG', true),
    // available locale loaders (the key is extension)
    'loaders'        => [
        'php'  => Loader\PhpFileLoader::class,
        'po'   => Loader\PoFileLoader::class,
        'csv'  => Loader\CsvFileLoader::class,
        'json' => Loader\JsonFileLoader::class
    ],
    // export methods
    'dumpers'        => [
        'php'  => Dumper\PhpFileDumper::class,
        'po'   => Dumper\PoFileDumper::class,
        'csv'  => Dumper\CsvFileDumper::class,
        'json' => Dumper\JsonFileDumper::class,
    ],
    'domains'        => [
        // by default we can store all messages in one domain
        'messages' => ['*']
    ]
];
```
