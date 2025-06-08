# Продвинутые возможности — Атомарные блокировки

Spiral имеет полную интеграцию с плагином [RoadRunner locks](https://roadrunner.dev/docs/plugins-locks), который позволяет управлять блокировками ресурсов в вашем приложении. С этим компонентом вы можете легко управлять критическими секциями вашего приложения и предотвращать состояния гонки (race conditions), повреждение данных и другие проблемы синхронизации, которые могут возникнуть в многопроцессных средах.

Для включения интеграции с RoadRunner, Spiral предоставляет встроенную поддержку через пакет [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge).

> **Предупреждение**
> Блокировки доступны только в пакете `spiral/roadrunner-bridge` версии `3.2` и выше.

## Установка

Чтобы начать, вам нужно установить пакет [Roadrunner bridge](../start/server.md#roadrunner-bridge). После установки добавьте `Spiral\RoadRunnerBridge\Bootloader\LockBootloader` в список загрузчиков (bootloaders) в вашем классе Kernel:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\LockBootloader::class,
        // ...
    ];
}
```

Читайте больше о загрузчиках в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\LockBootloader::class,
    // ...
];
```

Читайте больше о загрузчиках в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

Это все, что нужно! Дополнительная конфигурация на стороне RoadRunner не требуется.

## Использование

Вот краткий пример того, как вы можете использовать блокировки в вашем приложении:

```php
$locks = $container->get(\RoadRunner\Lock\LockInterface::class);
$id = $lock->lock('pdf:create');

// Ваша логика для создания PDF файла

$lock->release('pdf:create', $id);
```

> **Предупреждение**
> Плагин RoadRunner lock использует хранилище в памяти для хранения информации о блокировках в настоящий момент. Когда используется несколько экземпляров RoadRunner, каждый экземпляр будет иметь свое собственное хранилище блокировок в памяти. В результате, если процесс получает блокировку на одном экземпляре RoadRunner, он не будет знать о состоянии блокировки на других экземплярах.

### Получение блокировок

Блокировка ресурса гарантирует, что только один процесс может получить к нему доступ одновременно, предотвращая конфликты данных и обеспечивая согласованность.

#### Базовое получение блокировки

Получает эксклюзивную блокировку на указанном ресурсе.

```php
$id = $lock->lock('pdf:create');
```

Это простейшая форма получения блокировки. Она пытается заблокировать ресурс, идентифицированный как `pdf:create`. При успехе возвращает идентификатор блокировки.

#### Блокировка с временем жизни (TTL)

```php
$id = $lock->lock('pdf:create', ttl: 10);
// или
$id = $lock->lock('pdf:create', ttl: new \DateInterval('PT10S'));
```

Эти строки демонстрируют, как получить блокировку с TTL (Time-to-Live). Первая строка устанавливает TTL в 10 микросекунд, в то время как вторая строка использует объект DateInterval для установки TTL в 10 секунд. TTL используется для указания, как долго блокировка должна удерживаться, прежде чем она автоматически освободится.

#### Блокировка с временем ожидания

```php
$id = $lock->lock('pdf:create', wait: 5);
// или
$id = $lock->lock('pdf:create', wait: new \DateInterval('PT5S'));
```

Эти строки показывают, как получить блокировку с указанным временем ожидания. Если блокировка не доступна немедленно, процесс будет ждать данное время (5 микросекунд в первой строке и 5 секунд во второй строке), прежде чем сдаться.

#### Блокировка с идентификатором

```php
$id = $lock->lock('pdf:create', id: '14e1b600-9e97-11d8-9f32-f2801f1b9fd1');
```

Эта строка демонстрирует получение блокировки с конкретным идентификатором. Это может быть полезно для отслеживания или логирования, позволяя указать пользовательский идентификатор для блокировки.

### Получение блокировок чтения

В параллельном программировании получение блокировки чтения позволяет нескольким процессам получать доступ к ресурсу одновременно для чтения, в то же время предотвращая эксклюзивный доступ для записи.

Существуют аналогичные методы для получения блокировок чтения:

```php
$id = $lock->lockRead('pdf:create', ttl: 100000);
// или
$id = $lock->lockRead('pdf:create', ttl: new \DateInterval('PT10S'));

// Получить блокировку и ждать 5 микросекунд, пока блокировка не будет освобождена
$id = $lock->lockRead('pdf:create', wait: 5);
// или
$id = $lock->lockRead('pdf:create', wait: new \DateInterval('PT5S'));

// Получить блокировку с id - 14e1b600-9e97-11d8-9f32-f2801f1b9fd1
$id = $lock->lockRead('pdf:create', id: '14e1b600-9e97-11d8-9f32-f2801f1b9fd1');
```

### Освобождение блокировки

Освобождение блокировок — это важная часть работы с контролем параллелизма. Оно гарантирует, что ресурсы освобождаются для использования другими процессами.

> **Предупреждение**
> Вы всегда должны освобождать блокировку после завершения задачи, требующей эксклюзивного или совместного доступа к ресурсу. Это гарантирует, что другие процессы смогут получить блокировку при необходимости.

Чтобы освободить ранее полученную эксклюзивную блокировку или блокировку чтения:

```php
$id = $lock->lock('pdf:create');

// Ваша логика для создания PDF файла

$lock->release('pdf:create', $id);
```

#### Принудительное освобождение блокировки

В некоторых случаях вам может потребоваться освободить блокировку, не зная идентификатора блокировки. Например, если процесс завершается аварийно, удерживая блокировку, блокировка не будет освобождена.

```php
$lock->forceRelease('pdf:create');
```

### Проверка блокировки

Чтобы проверить, существует ли блокировка на ресурсе, используйте метод `exists()`:

```php
$status = $lock->exists('pdf:create');

if ($status) {
    // Блокировка существует
} else {
    // Блокировка не существует
}
```

### Обновление TTL

В некоторых случаях вам может потребоваться обновить TTL блокировки. Например, если процесс выполняет длительную задачу, вы можете захотеть продлить TTL, чтобы предотвратить освобождение блокировки до завершения задачи.

```php
// Добавить 10 микросекунд к ttl блокировки
$lock->updateTTL('pdf:create', $id, 10);
// или
$lock->updateTTL('pdf:create', $id, new \DateInterval('PT10S'));
```

## Интеграция с Symfony

Мы также предоставляем пакет, который добавляет драйвер блокировок к компоненту [Symfony lock](https://symfony.com/doc/current/components/lock.html).

Чтобы установить пакет, выполните следующую команду:

```terminal
composer require roadrunner-php/symfony-lock-driver
```

После установки вам нужно создать класс загрузчика (bootloader), который зарегистрирует драйвер блокировок в контейнере:

```php app/src/Application/Bootloader/LockBootloader.php
<?php

declare(strict_types=1);

namespace App\Application\Bootloader;

use RoadRunner\Lock\LockInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\Environment\AppEnvironment;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Goridge\RPC\RPCInterface;
use Spiral\RoadRunner\Symfony\Lock\RoadRunnerStore;
use Spiral\RoadRunnerBridge\Bootloader\RoadRunnerBootloader;
use Symfony\Component\Lock\LockFactory;
use Symfony\Component\Lock\PersistingStoreInterface;
use Symfony\Component\Lock\Store\InMemoryStore;
use Symfony\Component\Lock\Store\RedisStore;

final class LockBootloader extends Bootloader
{
    public function defineDependencies(): array
    {
        return [
            \Spiral\RoadRunnerBridge\Bootloader\LockBootloader::class
        ];
    }

    public function defineSingletons(): array
    {
        return [
            LockFactory::class => [self::class, 'initLockFactory'],
        ];
    }

    protected function initLockFactory(LockInterface $rrLock, EnvironmentInterface $env): LockFactory
    {
        $driver = $env->get('LOCK_DRIVER', 'roadrunner');
        $defaultTtl = $env->get('LOCK_DRIVER_TTL', 100);

        $store = match ($driver) {
            'memory' => new InMemoryStore(), // для целей тестирования
            'roadrunner' => new RoadRunnerStore($rrLock, initialWaitTtl: $defaultTtl),
            default => throw new \InvalidArgumentException("Unknown lock driver: {$driver}"),
        };

        return new LockFactory($store);
    }
}
```

Теперь вы можете использовать компонент Symfony lock в вашем приложении:

```php
use Symfony\Component\Lock\LockFactory;

$factory = $container->get(LockFactory::class);
$lock = $factory->createLock('pdf-creation');

if ($lock->acquire()) {
    // Ресурс "pdf-creation" заблокирован.
    // Вы можете безопасно вычислить и сгенерировать счет здесь.

    $lock->release();
}
```

> **Примечание**
> Читайте больше о компоненте Symfony lock в [документации Symfony](https://symfony.com/doc/current/components/lock.html).
