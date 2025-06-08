# Компонент — Сериализатор

Spiral помогает вам управлять и организовывать ваш код. Одним из аспектов этого является возможность сериализации данных, которая включает в себя преобразование их в формат, который может быть сохранен или передан, а затем позже десериализован, или преобразован обратно в исходную форму. Это полезно в различных ситуациях, например, при передаче данных по сети или сохранении их в базе данных.

Spiral включает некоторые базовые инструменты сериализации по умолчанию, но они не всегда могут быть достаточными для более сложных случаев использования. В таких случаях фреймворк упрощает интеграцию дополнительных инструментов сериализации или разработку пользовательских решений для сериализации данных в вашем приложении. Это позволяет вам выбрать лучший подход для ваших конкретных потребностей.

Компонент доступен по умолчанию в [пакете приложения](https://github.com/spiral/app).

## Установка

Чтобы включить компонент сериализатора в вашем приложении Spiral, вам необходимо добавить `Spiral\Serializer\Bootloader\SerializerBootloader` в список bootloaders.

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Serializer\Bootloader\SerializerBootloader::class,
        // ...
    ];
}
```

Подробнее о bootloaders читайте в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Serializer\Bootloader\SerializerBootloader::class,
    // ...
];
```

Подробнее о bootloaders читайте в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

## Конфигурация

Конфигурация для компонента сериализатора хранится в файле `app/config/serializer.php`. В этом файле вы можете настроить массив доступных сериализаторов и указать сериализатор по умолчанию.

Например, файл конфигурации может выглядеть так:

```php app/config/serializer.php
use Spiral\Core\Container\Autowire;
use Spiral\Serializer\Serializer\JsonSerializer;
use Spiral\Serializer\Serializer\PhpSerializer; 
use Spiral\Serializer\Serializer\CallbackSerializer;

return [
    /**
     * -------------------------------------------------------------------------
     *  Сериализатор по умолчанию
     * -------------------------------------------------------------------------
     * Ключ одного из зарегистрированных сериализаторов.
     */
    'default' => 'json',
    
    /**
     * -------------------------------------------------------------------------
     *  Доступные сериализаторы
     * -------------------------------------------------------------------------
     * Список доступных сериализаторов.  
     */
    'serializers' => [
        // через полное имя класса
        'json' => JsonSerializer::class,
        
        // через Autowire 
        'serializer' => new Autowire(PhpSerializer::class),
        
        // через Autowire с аргументами
        'encrypted_serializer' => new Autowire(EncryptedPhpSerializer::class, ['secret' => env('ENCRYPTION_KEY')]),
        
        // или ручное создание объекта
        'callback' => new CallbackSerializer(
            serializeCallback: fn(mixed $payload): string => \json_encode($payload),
            unserializeCallback: fn(string|\Stringable $payload, string|object|null $type = null) => \json_decode($payload, true)
        )
    ],
];
```

## Доступные сериализаторы

Компонент поставляется с несколькими базовыми сериализаторами из коробки, но также легко интегрировать дополнительные инструменты сериализации или разработать пользовательские решения для ваших конкретных потребностей.

- `Spiral\Serializer\Serializer\JsonSerializer` (`json`) - использует PHP функции `json_encode` и `json_decode`. Не поддерживает гидратацию данных в объект.
- `Spiral\Serializer\Serializer\PhpSerializer` (`serializer`) - использует PHP функции `serialize` и `unserialize`.
- `Spiral\Serializer\Serializer\ProtoSerializer` (`proto`) - использует Google Protobuf для сериализации и десериализации.
- `Spiral\Serializer\Serializer\CallbackSerializer`- использует обратные вызовы для сериализации и десериализации данных.

Также существуют пакеты, которые предоставляют дополнительные сериализаторы:

- [Symfony serializer](https://github.com/spiral-packages/symfony-serializer) - использует компонент сериализатора Symfony.
- [Laravel Serializable Closure](https://github.com/spiral-packages/serializable-closure) - использует пакет Laravel Serializable Closure.

## Использование

### SerializerInterface

Сериализатор может быть внедрен из контейнера с использованием `Spiral\Serializer\SerializerInterface`. Он будет ссылаться на сериализатор по умолчанию.

```php
namespace App\Service;

use Spiral\Serializer\SerializerInterface;

class MyService
{
    public function __construct(
        private readonly SerializerInterface $serializer,
    ) {
    }

    public function someMethod(): void
    {
        // сериализация
        $serialized = $this->serializer->serialize(['some' => 'data']);
        
        // десериализация
        $array = $this->serializer->unserialize($serialized);
    }
}
```

### SerializerManager

Вы можете использовать `Spiral\Serializer\SerializerManager` для получения конкретного сериализатора по его строковому ключу из конфигурации.

```php
namespace App\Service;

use Spiral\Serializer\SerializerManager;

class MyService
{
    public function __construct(
        private readonly SerializerManager $serializer,
    ) {
    }

    public function someMethod(): void
    {
        $serialized = $this->serializer->getSerializer('json')->serialize(['some' => 'data']);
        $array = $this->serializer->getSerializer('json')->unserialize($serialized);

        $serialized = $this->serializer->getSerializer('serializer')->serialize(['some' => 'data']);
        $array = $this->serializer->getSerializer('serializer')->unserialize($serialized);
    }
}
```

## Создание сериализатора

### Класс сериализатора

Чтобы создать пользовательский сериализатор в Spiral, вам необходимо реализовать `Spiral\Serializer\SerializerInterface`. Этот интерфейс определяет два метода, которые должен реализовать ваш класс сериализатора: `serialize(...)` и `unserialize(...)`.

Вот пример пользовательского класса сериализатора, который реализует `SerializerInterface`:

```php
<?php

declare(strict_types=1);

namespace App\Application\Serializer;

use Google\Protobuf\Internal\Message;
use Spiral\Serializer\SerializerInterface;

final class ProtoSerializer implements SerializerInterface
{
    public function serialize(mixed $payload): string|\Stringable
    {
        \assert($payload instanceof Message);

        return $payload->serializeToString();
    }

    public function unserialize(\Stringable|string $payload, object|string|null $type = null): mixed
    {
        \assert(
            $type !== null
            && \class_exists($type)
            && \is_a($type, Message::class, true),
        );

        $object = new $type();
        $object->mergeFromString((string)$payload);

        return $object;
    }
}

```

Метод `serialize` вашего пользовательского класса сериализатора должен принимать один параметр `$payload`, который представляет данные для сериализации. Этот метод должен вернуть сериализованные данные в виде строки.

Метод `unserialize` вашего пользовательского класса сериализатора должен принимать два параметра: `$payload`, который представляет сериализованные данные в виде строки, и `$type`, который является опциональным параметром, который указывает имя класса или объекта, в который должен быть десериализован payload. Этот метод должен вернуть десериализованные данные.

### Регистрация нового сериализатора

Существует два способа регистрации пользовательского сериализатора:

:::: tabs
::: tab Реестр
Используя `Spiral\Serializer\SerializerRegistryInterface`:

```php
namespace App\Application\Bootloader;

use App\Application\Serializer\ProtoSerializer;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Serializer\SerializerRegistryInterface;

class SerializerBootloader extends Bootloader
{
    public function boot(SerializerRegistryInterface $registry): void
    {
        $registry->register('proto', new ProtoSerializer());
    }
}
```
:::

::: tab Конфигурация
Используя файл конфигурации:

```php app/config/serializer.php
use App\Application\Serializer\ProtoSerializer;

return [
    'serializers' => [
        'proto' => ProtoSerializer::class,
        // другие сериализаторы
    ],
];
```
:::
::::