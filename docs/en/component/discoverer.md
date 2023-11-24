# Component — Discoverer

The `spiral-packages/discoverer` package is a useful tool for the Spiral framework. It enhances the framework by
allowing the discovery of bootloaders and tokenizer directories from sources beyond the Application kernel. This feature
simplifies the process of managing and integrating various packages in your Spiral application.

**Features**

1. **Automatic Discovery of Bootloaders**: Automates the process of discovering and registering bootloaders from
   installed packages, significantly simplifying the integration and setup process in Spiral applications.

2. **Composer.json Integration**: The package leverages the `composer.json` file of other packages to define bootloaders
   and tokenizer directories. This integration streamlines the configuration process, making it easier for developers to
   manage package settings.

3. **Custom Registry Support**: The package allows for the creation of custom bootloader and tokenizer registries. This
   flexibility enables developers to tailor the discovery process to their specific needs, enhancing the customization
   and scalability of their Spiral applications.

## Requirements

Make sure that your server is configured with following PHP version and extensions:

- PHP 8.1+
- Spiral Framework version 3.10 or higher

## Installation

Install the package via Composer with the following command:

```terminal
composer require spiral-packages/discoverer
```

After installation, you need to register the package's bootloader. Add the `DiscovererBootloader` to your system's bootloaders array:

```php app/src/Application/Kernel.php
protected const SYSTEM = [
    // ...
    \Spiral\Discoverer\DiscovererBootloader::class,
];
```

That's it.

## Registries

Discoverer can search for bootloaders or tokenizer directories in different types of registries:

### Composer.json Registry

When a Spiral application developer installs an additional package, they typically need to register the package's bootloader. To streamline this process, packages can define their bootloaders in the `extra` section of their `composer.json` file.

**Like this:**

```json vendor/spiral/dotenv/composer.json
{
  "extra": {
    "spiral": {
      "bootloaders": [
        "Spiral\\DotEnv\\Bootloader\\MonologBootloader"
      ],
      "directories": [
        "src/Entities"
      ]
    }
  }
}
```

In some cases, you may want to disable package discovery for specific packages. You can do this by listing the package names in the `extra` section of your application's `composer.json` file:

```json composer.json
{
  "extra": {
    "spiral": {
      "dont-discover": [
        "spiral/dotenv",
        "spiral-packages/bar"
      ]
    }
  }
}
```

Discoverer will automatically register the bootloaders and directories of the package upon installation.

### Custom Bootloader Registry

You can create a custom source for bootloaders by implementing the `Spiral\Discoverer\Bootloader\BootloaderRegistryInterface`.

Here's an example:

```php
use Spiral\Discoverer\Bootloader\BootloaderRegistryInterface;
use Spiral\Core\Container;
use Spiral\Files\FilesInterface;

final class JsonRegistry implements BootloaderRegistryInterface
{
    private array $bootloaders = [];

    public function __construct(private string $jsonPath) {}

    public function init(Container $container): void {
        $files = $container->get(FilesInterface::class);
        $data = json_decode($files->read($this->jsonPath), true);

        $this->bootloaders = $data['bootloaders'] ?? [];
    }

    public function getBootloaders(): array {
        return $this->bootloaders;
    }

    public function getIgnoredBootloaders(): array {
        return [];
    }
}
```

Register your custom registry in the `discoverer.php` configuration file:

```php app/config/discoverer.php
<?php

use Spiral\Discoverer\Bootloader as BootloaderRegistry;
use Spiral\Discoverer\Tokenizer as TokenizerRegistry;

return [
    'registries' => [
        'bootloaders' => [
            BootloaderRegistry\ComposerRegistry::class,
            BootloaderRegistry\ConfigRegistry::class,
            JsonRegistry::class,
        ],
        'directories' => [
            TokenizerRegistry\ComposerRegistry::class,
        ],
    ],
];
```

### Custom Tokenizer Registry

Similarly, you can create a custom source for tokenizer directories by implementing the `Spiral\Discoverer\Tokenizer\DirectoryRegistryInterface`.

Here's an example:

```php
use Spiral\Discoverer\Tokenizer\DirectoryRegistryInterface;
use Spiral\Core\Container;
use Spiral\Files\FilesInterface;

final class JsonRegistry implements DirectoryRegistryInterface
{
    private array $directories = [];

    public function __construct(private string $jsonPath) {}

    public function init(Container $container): void {
        $files = $container->get(FilesInterface::class);
        $data = json_decode($files->read($this->jsonPath), true);

        $this->directories = $data['directories'] ?? [];
    }

    public function getDirectories(): array {
        return $this->directories;
    }
}
```

Register your custom registry in the `discoverer.php` configuration file:

```php app/config/discoverer.php
<?php

use Spiral\Discoverer\Bootloader as BootloaderRegistry;
use Spiral\Discoverer\Tokenizer as TokenizerRegistry;

return [
    'registries' => [
        'bootloaders' => [
            BootloaderRegistry\ComposerRegistry::class,
            BootloaderRegistry\ConfigRegistry::class,
        ],
        'directories' => [
            TokenizerRegistry\ComposerRegistry::class,
            JsonRegistry::class,
        ],
    ],
];
```

As you can see these features not only save time but also introduce a level of flexibility and scalability that is invaluable in modern web application development.