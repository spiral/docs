# Основы — Логирование

Spiral предлагает компонент `spiral/logger`, который соответствует стандарту [PSR-3](https://www.php-fig.org/psr/psr-3/).
Этот компонент может использоваться для логирования различных типов информации, таких как ошибки, предупреждения и отладочные сообщения,
что может помочь в идентификации и решении проблем в приложении.

По умолчанию фреймворк не предоставляет свою собственную реализацию, однако доступен компонент `spiral/monolog-bridge`,
который полностью интегрируется с пакетом [Seldaek/monolog](https://github.com/Seldaek/monolog) и предлагает
поддержку различных мощных обработчиков логов.

Фреймворк упрощает настройку этих обработчиков, позволяя настраивать обработку логов с помощью использования
различных каналов.

## Конфигурация

Для настройки этого компонента он может быть настроен по вашему предпочтению через конфигурационный файл или bootloader. Конфигурационный файл для
этого компонента обычно расположен в `app/config/monolog.php`. Через этот файл вы можете выбрать обработчик по умолчанию,
установить глобальный уровень логирования и настроить обработчики и процессоры для удовлетворения ваших конкретных потребностей.

Вот пример конфигурационного файла:

```php app/config/monolog.php
use Monolog\Handler\ErrorLogHandler;
use Monolog\Handler\SyslogHandler;
use Monolog\Logger;
use Monolog\Processor\PsrLogMessageProcessor;

return [
    /**
     * -------------------------------------------------------------------------
     *  Обработчик Monolog по умолчанию
     * -------------------------------------------------------------------------
     */
    'default' => env('MONOLOG_DEFAULT_CHANNEL', 'default'),

    /**
     * -------------------------------------------------------------------------
     *  Глобальный уровень логирования
     * -------------------------------------------------------------------------
     *
     * Monolog поддерживает уровни логирования, описанные в RFC 5424.
     *
     * @see https://seldaek.github.io/monolog/doc/01-usage.html#log-levels
     */
    'globalLevel' => Logger::toMonologLevel(
        env('MONOLOG_DEFAULT_LEVEL', \Monolog\Logger::DEBUG)
    ),

    /**
     * -------------------------------------------------------------------------
     *  Обработчики
     * -------------------------------------------------------------------------
     *
     * @see https://seldaek.github.io/monolog/doc/02-handlers-formatters-processors.html#handlers
     */
    'handlers' => [
        'default' => [
            [
                'class' => 'log.rotate',
                'options' => [
                    'filename' => directory('runtime') . 'logs/app.log',
                    'level' => \Monolog\Logger::DEBUG,
                ],
            ],
        ],
        'stderr' => [
            ErrorLogHandler::class,
        ],
        'stdout' => [
            [
                'class' => SyslogHandler::class,
                'options' => [
                    'ident' => 'app',
                    'facility' => LOG_USER,
                ],
            ],
        ],
    ],

    /**
     * -------------------------------------------------------------------------
     *  Процессоры
     * -------------------------------------------------------------------------
     *
     * Процессоры позволяют добавлять дополнительные данные для всех записей.
     *
     * @see https://seldaek.github.io/monolog/doc/02-handlers-formatters-processors.html#processors
     */
    'processors' => [
        'default' => [
            [
                'class' => PsrLogMessageProcessor::class,
                'options' => [
                    'dateFormat' => 'Y-m-d\TH:i:s.uP',
                ],
            ],
        ],
    ],
];
```

> **Примечание**
> Используйте переменную окружения `MONOLOG_DEFAULT_CHANNEL` для указания обработчика по умолчанию, который должен использоваться в вашем приложении.

### Формат лога

По умолчанию обработчик будет форматировать сообщение лога, используя следующую
структуру `[%datetime%] %level_name%: %message% %context%\n`.

Если вы хотите изменить структуру сообщения лога, вы можете установить переменную окружения `MONOLOG_FORMAT`, как в примере
ниже:

```dotenv .env
MONOLOG_FORMAT=[%datetime%] %level_name%: %message% %context%\n
```

> **Узнать больше**
> Прочитайте больше о доступных заполнителях в
> [документации Monolog](https://seldaek.github.io/monolog/doc/message-structure.html).


## Регистрация обработчика

Помимо настройки компонента логирования через конфигурационный файл или bootloader, вы также можете регистрировать обработчики через
класс `Spiral\Monolog\Bootloader\MonologBootloader`.

### Обработчик ротации логов

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            'my-channel',
            $monolog->logRotate(directory('runtime') . 'logs/my-channel.log')
        );
    }
}
```

> **Предупреждение**
> Не забудьте добавить этот bootloader в начало списка bootloaders в `app/src/Application/Kernel.php`:

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\LoggingBootloader::class,
        // ...
    ];
}
```

Прочитайте больше о bootloaders в разделе [Фреймворк — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\LoggingBootloader::class,
    // ...
];
```

Прочитайте больше о bootloaders в разделе [Фреймворк — Bootloaders](../framework/bootloaders.md).
:::

::::

### RoadRunner обработчик

Пакет моста RoadRunner предоставляет обработчик `Spiral\RoadRunnerBridge\Logger\Handler` для отправки логов в
[логгер приложения RoadRunner](https://roadrunner.dev/docs/plugins-applogger).

Вам просто нужно добавить `Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader` в начало списка bootloaders:

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader::class,
        // ...
    ];
}
```

Прочитайте больше о bootloaders в разделе [Фреймворк — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader::class,
    // ...
];
```

Прочитайте больше о bootloaders в разделе [Фреймворк — Bootloaders](../framework/bootloaders.md).
:::

::::

> **Предупреждение**
> Убедитесь, что у вас установлен пакет [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge).
> Этот пакет предоставляет необходимые классы для интеграции RoadRunner с Monolog.

И измените канал по умолчанию на `roadrunner`:

:::: tabs

::: tab Environment

```dotenv .env
MONOLOG_DEFAULT_CHANNEL=roadrunner
```

:::

::: tab Config

```php app/config/monolog.php
return [
    'default' => 'roadrunner',
    // ...
];
```

:::

::::

## Использование

### Отправка логов в канал по умолчанию

Для использования компонента логирования фреймворк использует класс `Psr\Log\LoggerInterface`, который может использоваться для
логирования сообщений в канал по умолчанию.

```php
use Psr\Log\LoggerInterface;

final class UserService
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}

    public function register(string $email, string $password): void
    {
        // Регистрация пользователя ...
        
        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

### Отправка логов в определенный канал

Есть несколько способов получить экземпляр логгера с определенным каналом:
- Используя фабрику логгеров, которая реализует `Spiral\Logger\LogsInterface`.
- Используя атрибут `Spiral\Logger\Attribute\LoggerChannel` на параметре `Psr\Log\LoggerInterface`
  во время автоподключения.


:::: tabs

::: tab Используя фабрику

```php
use Psr\Log\LoggerInterface;
use Spiral\Logger\LogsInterface;

final class UserService
{
    private readonly LoggerInterface $logger;

    public function __construct(LogsInterface $logs) 
    {
        $this->logger = $logs->channel('my-channel');
    }

    public function register(string $email, string $password): void
    {
        // Регистрация пользователя ...
        
        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

:::

::: tab Используя атрибут

```php
use Psr\Log\LoggerInterface;
use Spiral\Logger\Attribute\LoggerChannel;

final class UserService
{
    public function __construct(
        #[LoggerChannel('my-channel')]
        private readonly LoggerInterface $logger
    ) {}

    public function register(string $email, string $password): void
    {
        // Регистрация пользователя ...

        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

:::

::::

### Logger trait

Spiral предоставляет удобный способ быстрого назначения Logger любому классу через использование
трейта `Spiral\Logger\Traits\LoggerTrait`. Просто включив этот трейт в класс, вы можете легко получить доступ к экземпляру Logger
и логировать сообщения.

> **Предупреждение**
> **Имя канала**, используемое для логирования, будет именем класса по умолчанию. Этот трейт позволяет вам логировать сообщения из
> любого класса без необходимости явного внедрения Logger в конструктор.

```php
use Spiral\Logger\Traits\LoggerTrait;

final class UserService
{
    use LoggerTrait;

    public function register(string $email, string $password): void
    {
        // Регистрация пользователя ...
        
        $this->getLogger()->info('User has been registered', ['email' => $email]);
    }
}
```

И назначить ему логгер:

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    // ...
    
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            UserService::class,
            $monolog->logRotate(directory('runtime') . 'logs/user-service.log')
        );
    }
}
```

> **Предупреждение**
> LoggerTrait работает только внутри глобальной [области видимости IoC](../framework/scopes.md).

### Обработка только определенных уровней логирования

В некоторых случаях вы можете захотеть логировать только определенные уровни логирования. Например, вы можете захотеть агрегировать только ошибки приложения
в одном файле лога.

**Для этого вы можете подписаться на канал по умолчанию:**

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Monolog\Logger;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    // ...
    
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            'default',
            $monolog->logRotate(directory('runtime') . 'logs/errors.log', Logger::ERROR) // только ERROR и выше
        );
    }
}
```
