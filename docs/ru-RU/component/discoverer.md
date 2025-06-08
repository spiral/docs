# Компонент — Discoverer

Пакет `spiral-packages/discoverer` является полезным инструментом для Spiral framework. Он расширяет возможности фреймворка, позволяя обнаруживать загрузчики (bootloaders) и директории токенизатора из источников помимо ядра приложения. Эта функция упрощает процесс управления и интеграции различных пакетов в вашем Spiral приложении.

**Возможности**

1. **Автоматическое обнаружение загрузчиков**: Автоматизирует процесс обнаружения и регистрации загрузчиков из установленных пакетов, значительно упрощая процесс интеграции и настройки в Spiral приложениях.

2. **Интеграция с Composer.json**: Пакет использует файл `composer.json` других пакетов для определения загрузчиков и директорий токенизатора. Эта интеграция упрощает процесс конфигурации, облегчая разработчикам управление настройками пакетов.

3. **Поддержка пользовательских реестров**: Пакет позволяет создавать пользовательские реестры загрузчиков и токенизаторов. Эта гибкость позволяет разработчикам настраивать процесс обнаружения под свои специфические потребности, улучшая возможности настройки и масштабируемости их Spiral приложений.

## Требования

Убедитесь, что ваш сервер настроен со следующей версией PHP и расширениями:

- PHP 8.1+
- Spiral Framework версии 3.10 или выше

## Установка

Установите пакет через Composer следующей командой:

```terminal
composer require spiral-packages/discoverer
```

После установки необходимо зарегистрировать загрузчик пакета. Добавьте `DiscovererBootloader` в массив системных загрузчиков:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineSystemBootloaders(): array
{
    return [
        // ...
        \Spiral\Discoverer\DiscovererBootloader::class,
    ];
}
```

Подробнее о загрузчиках читайте в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const SYSTEM = [
    // ...
    \Spiral\Discoverer\DiscovererBootloader::class,
];
```

Подробнее о загрузчиках читайте в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

Вот и всё.

## Реестры

Discoverer может искать загрузчики или директории токенизатора в различных типах реестров:

### Реестр Composer.json

Когда разработчик Spiral приложения устанавливает дополнительный пакет, ему обычно необходимо зарегистрировать загрузчик пакета. Для упрощения этого процесса пакеты могут определять свои загрузчики в секции `extra` файла `composer.json`.

**Вот так:**

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

В некоторых случаях вы можете захотеть отключить обнаружение пакетов для определенных пакетов. Вы можете сделать это, перечислив имена пакетов в секции `extra` файла `composer.json` вашего приложения:

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

Discoverer автоматически зарегистрирует загрузчики и директории пакета при установке.

### Пользовательский реестр загрузчиков

Вы можете создать пользовательский источник для загрузчиков, реализуя интерфейс `Spiral\Discoverer\Bootloader\BootloaderRegistryInterface`.

Вот пример:

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

Зарегистрируйте ваш пользовательский реестр в конфигурационном файле `discoverer.php`:

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

### Пользовательский реестр токенизатора

Аналогично, вы можете создать пользовательский источник для директорий токенизатора, реализуя интерфейс `Spiral\Discoverer\Tokenizer\DirectoryRegistryInterface`.

Вот пример:

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

Зарегистрируйте ваш пользовательский реестр в конфигурационном файле `discoverer.php`:

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

Как видите, эти функции не только экономят время, но и вводят уровень гибкости и масштабируемости, который неоценим в современной разработке веб-приложений.