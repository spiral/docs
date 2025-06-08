# Компонент — События (Events)

Компонент Events предоставляет инструменты, которые позволяют компонентам вашего приложения взаимодействовать друг с другом путем отправки событий и прослушивания их. Компонент упрощает регистрацию обработчиков событий в вашем приложении.

Он также предоставляет простой способ интеграции `PSR-14` совместимого `EventDispatcher`.

## Установка

Чтобы включить компонент, вам нужно добавить `Spiral\Events\Bootloader\EventsBootloader` в список загрузчиков (bootloaders), который находится в классе вашего приложения.

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Events\Bootloader\EventsBootloader::class,
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
    \Spiral\Events\Bootloader\EventsBootloader::class,
    // ...
];
```

Читайте больше о загрузчиках в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

Этот загрузчик (bootloader) регистрирует реализацию `Spiral\Events\ListenerFactoryInterface` и `Spiral\Events\ListenerProcessorRegistry` в контейнере приложения. Он также регистрирует обработчики событий в приложении. Мы опишем ниже, как их добавить.

После этого давайте установим `PSR-14` реализацию совместимого `EventDispatcher`. Мы предоставляем пакет `spiral-packages/league-event`, который интегрирует PSR-14 совместимый пакет `league/event` в приложение на основе Spiral.

```terminal
composer require spiral-packages/league-event
```

После установки пакета вам нужно зарегистрировать загрузчик (bootloader) из пакета.

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Events\Bootloader\EventsBootloader::class,
        \Spiral\League\Event\Bootloader\EventBootloader::class,
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
    \Spiral\Events\Bootloader\EventsBootloader::class,
    \Spiral\League\Event\Bootloader\EventBootloader::class,
    // ...
];
```

Читайте больше о загрузчиках в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

## Конфигурация

Файл конфигурации для компонента Events можно найти в `app/config/events.php`. Этот файл позволяет указать `listeners` (обработчики) и `processors` (процессоры) событий.

Параметр `listeners` представлен в виде `array`, где ключ — это `полное имя класса события`, а значение должно быть `array`, содержащим `полное имя класса обработчика события` или экземпляр `Spiral\Events\Config\EventListener`, который позволяет настроить дополнительные параметры обработчика.

Параметр `processors` представлен в виде `array` и содержит `полное имя класса процессора`.

> **Примечание**
> Подробная информация о `processors` будет предоставлена ниже. Этот раздел описывает только конфигурацию.

Например, файл конфигурации может выглядеть следующим образом:

```php app/config/events.php
use App\Listener\RouteListener;
use Spiral\Events\Config\EventListener;
use Spiral\Events\Processor\AttributeProcessor;
use Spiral\Events\Processor\ConfigProcessor;
use Spiral\Router\Event\RouteMatched;

return [
    /**
     * -------------------------------------------------------------------------
     *  Listeners
     * -------------------------------------------------------------------------
     * 
     * Список обработчиков событий для регистрации в приложении.
     */
    'listeners' => [
        // без дополнительных опций
        RouteMatched::class => [
            RouteListener::class,
        ],

        // ИЛИ

        // с дополнительными опциями
        RouteMatched::class => [
            new EventListener(
                listener: RouteListener::class,
                method: 'onRouteMatched',
                priority: 1
            ),
        ],
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Processors
     * -------------------------------------------------------------------------
     * 
     * Массив всех доступных процессоров.  
     */
    'processors' => [
        AttributeProcessor::class,
        ConfigProcessor::class,
    ],
];
```

## Использование

### Событие

Событие может быть представлено простым классом.

```php
namespace App\Event;

use App\Database\User;

final class UserWasCreated
{
    public function __construct(
        public readonly User $user
    ) {
    }
}
```

Отправка события.

```php
namespace App\Service;

use Psr\EventDispatcher\EventDispatcherInterface;
use App\Event\UserWasCreated;

final class UserService 
{
    public function __construct(
        private readonly EventDispatcherInterface $dispatcher
    ) {
    }

    public function create(string $username): User
    {
        $user = new User(username: $username);
        // ...
        $this->dispatcher->dispatch(new UserWasCreated($user));
        
        return $user;
    }
}
```

### Обработчик

Обработчик может быть представлен простым классом с методом, который будет вызван для обработки события. Имя метода может быть настроено в параметре атрибута Listener или в файле конфигурации (по умолчанию `__invoke`).

```php
namespace App\Listener;

use App\Event\UserWasCreated;
use Spiral\Events\Attribute\Listener;

#[Listener]
class UserWasCreatedListener
{
    public function __invoke(UserWasCreated $event): void
    {
        // ...
    }
}
```

Используя атрибут, вы можете настроить дополнительные параметры.

```php
namespace App\Listener;

use App\Event\UserWasCreated;
use Spiral\Events\Attribute\Listener;

#[Listener(event: UserWasCreated::class, method: 'onUserWasCreated', priority: 1)]
class UserWasCreatedListener
{
    public function onUserWasCreated(UserWasCreated $event): void
    {
        // ...
    }
}
```

Атрибут может быть использован `непосредственно на методе`, тогда имя метода можно опустить.

```php
namespace App\Listener;

use App\Event\UserWasCreated;
use Spiral\Events\Attribute\Listener;

class UserWasCreatedListener
{
    #[Listener(event: UserWasCreated::class, priority: 1)]
    public function onUserWasCreated(UserWasCreated $event): void
    {
        // ...
    }
}
```

Атрибут `Spiral\Events\Attribute\Listener` необходим для автоматической регистрации обработчика событий в приложении. Если вы предпочитаете регистрировать обработчики через файл конфигурации, вы можете удалить этот атрибут и `Spiral\Events\Processor\AttributeProcessor` из файла конфигурации.

## Перехватчики (Interceptors)

Компонент Events предоставляет возможность перехватывать события. Это полезно, когда вам нужно изменить данные события или, например, отправить их через websockets. Для этого вам нужно создать класс перехватчика, который реализует интерфейс `Spiral\Interceptors\InterceptorInterface`.

**Пример**

```php
namespace App\Broadcasting;

use Spiral\Broadcasting\BroadcastInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;
use Spiral\Queue\SerializerRegistryInterface;

final class BroadcastEventInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly BroadcastInterface $broadcast,
        private readonly SerializerRegistryInterface $registry,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $event = $context->getArguments()['event'];

        // Сначала отправляем событие
        $result = $handler->handle($context);

        // Транслируем событие после отправки
        if ($event instanceof ShouldBroadcastInterface) {
            $this->broadcast->publish(
                $event->getBroadcastTopics(),
                $this->registry->getSerializer('json')->serialize(
                    ['event' => $event->getEventName(), 'data' => $event->getPayload()],
                ),
            );
        }

        return $result;
    }
}

```

И затем вам нужно будет зарегистрировать его в файле конфигурации `events.php`, который находится в директории `app/config`.

```php app/config/events.php
return [
    // ...
    'interceptors' => [
        \App\Broadcasting\BroadcastEventInterceptor::class
    ]
];
```

> **Читайте больше**
> Читайте больше о перехватчиках в разделе [Фреймворк — Перехватчики](../framework/interceptors.md).

## Создание диспетчера событий

В качестве реализации диспетчера событий мы рассматривали пакет [The League Event bridge for Spiral](https://github.com/spiral-packages/league-event). Этот пакет предоставляет мост между Spiral и пакетом [The League Event](https://event.thephpleague.com/3.0/).

Вы можете создать свою собственную реализацию, используя `PSR-14` диспетчер событий.

### ListenerRegistry

Давайте создадим класс, который будет реализовывать `Spiral\Events\ListenerRegistryInterface` и предоставлять возможность регистрации обработчиков событий. Метод `addListener` из этого класса вызывается `processors`, передавая `event`, `event listener` и `priority` в качестве параметров.

```php
namespace App\EventDispatcher;

use Spiral\Events\ListenerRegistryInterface;
use Psr\EventDispatcher\ListenerProviderInterface;

final class ListenerRegistry implements ListenerRegistryInterface, ListenerProviderInterface
{
    public function addListener(string $event, callable $listener, int $priority = 0): void
    {
        // ...
    }
    
    public function getListenersForEvent(object $event): iterable
    {
        // ...
    }
}
```

### Диспетчер событий

Создайте реализацию `Psr\EventDispatcher\EventDispatcherInterface`.

```php
namespace App\EventDispatcher;

use Psr\EventDispatcher\EventDispatcherInterface;

final class EventDispatcher implements EventDispatcherInterface
{
    public function dispatch(object $event)
    {
        // ...
    }
}
```

### Загрузчик (Bootloader)

После этого нам нужно создать загрузчик (bootloader), который зарегистрирует реализацию интерфейсов в контейнере приложения.

```php
namespace App\EventDispatcher\Bootloader;

use App\EventDispatcher\ListenerRegistry;
use App\EventDispatcher\EventDispatcher;

use Psr\EventDispatcher\EventDispatcherInterface;
use Psr\EventDispatcher\ListenerProviderInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Events\Bootloader\EventsBootloader;
use Spiral\Events\ListenerRegistryInterface;

final class EventBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        EventsBootloader::class
    ];

    protected const SINGLETONS = [
        ListenerRegistryInterface::class => ListenerRegistry::class,
        ListenerRegistry::class => ListenerRegistry::class,
        ListenerProviderInterface::class => ListenerRegistry::class,
        EventDispatcherInterface::class => EventDispatcher::class,
        EventDispatcher::class => EventDispatcherInterface::class
    ];
}
```

> **Примечание**
> Не забудьте зарегистрировать загрузчик (bootloader) в приложении.

## Процессоры

Процессоры событий позволяют регистрировать обработчики событий из различных источников в вашем приложении, таких как файлы конфигурации или PHP атрибуты. Это может быть полезно для организации и централизации управления обработчиками событий в вашем приложении.

По умолчанию компонент Events предоставляет два процессора, которые вы можете использовать для регистрации обработчиков событий в вашем приложении.

**Эти процессоры:**

- `Spiral\Events\Processor\ConfigProcessor`, который позволяет регистрировать обработчики событий из файла конфигурации.
- `Spiral\Events\Processor\AttributeProcessor`, который позволяет регистрировать обработчики событий, используя PHP атрибут `Spiral\Events\Attribute\Listener`.

> **Примечание**
> Вы можете использовать предоставленные процессоры, использовать только один из них или добавить свои собственные пользовательские процессоры по мере необходимости. Просто обновите файл конфигурации `app/config/events.php`, чтобы указать процессоры, которые вы хотите использовать.

### Создание процессора

Процессор — это класс, который должен реализовывать `Spiral\Events\Processor\ProcessorInterface` и реализовать метод `process`.

```php
namespace App\Processor;

use Spiral\Events\ListenerFactoryInterface;
use Spiral\Events\ListenerRegistryInterface;
use Spiral\Events\Processor\AbstractProcessor;

final class MyCustomProcessor extends AbstractProcessor
{
    public function __construct(
        private readonly ListenerFactoryInterface $factory,
        private readonly ?ListenerRegistryInterface $registry = null,
    ) {
    }

    public function process(): void
    {
        // Если реализация EventDispatcher не зарегистрирована в приложении,
        // эта реализация интерфейса не будет зарегистрирована и процессор перестанет работать.
        if ($this->registry === null) {
            return;
        }

        // Используя реализацию ListenerRegistryInterface, мы можем регистрировать обработчики событий.
        $this->registry->addListener(
            event: $event,
            listener: $this->factory->create($listener, $method),
            priority: $priority
        );
    }
}
```

Реализация `Spiral\Events\ListenerFactoryInterface` позволяет создать экземпляр обработчика из полного имени класса и имени метода, добавляя все необходимые зависимости в конструктор.

После этого нам нужно зарегистрировать процессор в файле конфигурации.

```php app/config/events.php
use App\Processor\MyCustomProcessor;

return [
    // ...
    'processors' => [
        MyCustomProcessor::class,
        // ...
    ],
    // ...
];
```
