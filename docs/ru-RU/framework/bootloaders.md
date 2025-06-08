# Framework — Загрузчики (Bootloader)

Spiral использует классы загрузчиков (bootloader) во время процесса начальной загрузки для управления конфигурацией приложения. Одной из ключевых особенностей является то, что загрузчики выполняются только один раз во время загрузки приложения, гарантируя, что добавление дополнительного кода в них не влияет негативно на производительность во время выполнения.

Классы Bootloader играют важную роль в настройке различных аспектов приложения.

**Вот несколько примеров:**

- **Конфигурация контейнера**: Загрузчики могут использоваться для настройки контейнера внедрения зависимостей. Это включает такие задачи, как регистрация сервисов и связывание интерфейсов с их соответствующими реализациями.
- **Конфигурация приложения**: Загрузчики отвечают за установку различных настроек уровня приложения, включая окружение, режим отладки и обработку ошибок.
- **Конфигурация базы данных**: Загрузчики участвуют в настройке и конфигурировании подключения к базе данных приложения. Это включает указание драйвера базы данных, деталей подключения и любых необходимых миграций.
- **Конфигурация маршрутизации**: Загрузчики могут использоваться для настройки правил маршрутизации приложения, например, для определения того, какие контроллеры должны обрабатывать какие URL.
- **Инициализация сервисов**: Загрузчики заботятся об инициализации любых сервисов, необходимых приложению, таких как кэширование, логирование и системы событий.

![Application Control Phases](https://user-images.githubusercontent.com/773481/180768689-c711e6f0-3523-4330-a496-f78088504b29.png)

## Простой Bootloader

Для простого создания загрузчика используйте команду создания каркаса:

```terminal
php app.php create:bootloader GithubClient
```

> **Примечание**
> Подробнее о создании каркасов читайте в разделе [Основы — Создание каркасов](../basics/scaffolding.md#bootloader).

После выполнения этой команды следующий вывод подтвердит успешное создание:

```output
Declaration of '[32mGithubClientBootloader[39m' has been successfully written into '[33mapp/src/Application/Bootloader/GithubClientBootloader.php[39m'.
```

Теперь вы можете найти класс `GithubClientBootloader` в директории `app/src/Application/Bootloader`.

В настоящее время новый Bootloader не выполняет никаких действий. Немного позже мы добавим к нему некоторую функциональность.

## Регистрация загрузчика

Каждый загрузчик должен быть активирован в ядре вашего приложения `app/src/Application/Kernel.php`.

В Spiral вы можете регистрировать загрузчики в вашем ядре, используя два разных подхода: константы и методы.

1. **Константы:** Ядро предоставляет константы, такие как `Kernel::SYSTEM`, `Kernel::LOAD` и `Kernel::APP`. Эти константы позволяют вам указать, какие загрузчики должны быть выполнены на разных этапах процесса инициализации приложения. Используя эти константы, вы можете достичь четкого разделения ответственности и легко понять, какие загрузчики обрабатывают конкретные задачи.
2. **Методы:** Ядро также предлагает методы, такие как `defineBootloaders`, `defineAppBootloaders` и `defineSystemBootloaders`. Эти методы позволяют более сложную логику и регистрацию объектов, анонимных классов или других продвинутых случаев использования. С этими методами у вас есть большая гибкость и контроль над процессом инициализации вашего приложения.

> **Предупреждение**
> Вы не можете использовать методы и константы одновременно. Если вы выберете подход с методами, регистрация через константы не будет действовать.

:::: tabs

::: tab Использование методов

Добавьте ссылку на класс в методы `defineBootloaders` или `defineAppBootloaders` вашего класса `App\Application\Kernel`:

```php app/src/Application/Kernel.php
namespace App\Application;

use App\Application\Bootloader\RoutesBootloader;
use App\Application\Bootloader\LoggingBootloader;
use App\Application\Bootloader\MyBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // ...
           RoutesBootloader::class,
        ];
    }

    public function defineAppBootloaders(): array
    {
        return [
           LoggingBootloader::class,
           MyBootloader::class,
           
           // анонимный загрузчик через экземпляр объекта
           new class extends Bootloader {
               // ...
           },
       ];
    }
}
```

> **Примечание**
> Загрузчики в методе `defineAppBootloaders` всегда загружаются после загрузчиков в методе `defineBootloaders`. Храните в нем загрузчики, специфичные для домена.

:::

::: tab Использование констант

Добавьте ссылку на класс в константы `LOAD` или `APP` вашего класса `App\Application\Kernel`:

```php app/src/Application/Kernel.php
namespace App\Application;

use App\Application\Bootloader\RoutesBootloader;
use App\Application\Bootloader\LoggingBootloader;
use App\Application\Bootloader\MyBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    protected const LOAD = [
        // ...
        RoutesBootloader::class,
    ];

    protected const APP = [
        // ...
        LoggingBootloader::class,
        MyBootloader::class,
    ];
}
```

> **Примечание**
> Загрузчики в константе `APP` всегда загружаются после загрузчиков в константе `LOAD`. Храните в ней загрузчики, специфичные для домена.

:::

::::

### Управление загрузкой загрузчиков

Существует также функция, которая позволяет разработчикам управлять тем, как загрузчики настраиваются и используются. Эта функциональность особенно выгодна для адаптации приложений к различным окружениям, таким как HTTP, интерфейсы командной строки или любые другие контексты. Она позволяет селективную активацию или деактивацию загрузчиков на основе конкретных требований окружения, повышая как эффективность, так и производительность.

Существует новый DTO класс `Spiral\Boot\Attribute\BootloadConfig`, который позволяет включение или исключение загрузчиков, передачу параметров, которые будут переданы в методы init и boot загрузчика, и динамическую настройку загрузки загрузчика на основе переменных окружения.

Чтобы использовать это в ядре, вы должны использовать полное имя класса загрузчика в качестве ключа в массиве загрузчиков, с соответствующим значением в виде объекта `BootloadConfig`.

```php app/src/Application/Kernel.php
namespace App\Application;

use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Prototype\Bootloader\PrototypeBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // ...
            PrototypeBootloader::class => new BootloadConfig(allowEnv: ['APP_ENV' => ['local', 'dev']]),
            // ...
        ];
    }
}
```

В этом примере мы указали, что `PrototypeBootloader` должен загружаться только если переменная окружения `APP_ENV` определена и имеет значение `local` или `dev`.

Вместо создания объекта `BootloadConfig` напрямую, вы можете определить функцию, которая возвращает объект `BootloadConfig`. Эта функция может принимать аргументы, которые могут быть получены из контейнера.

```php app/src/Application/Kernel.php
namespace App\Application;

use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Environment\AppEnvironment;
use Spiral\Prototype\Bootloader\PrototypeBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // ...
            PrototypeBootloader::class => static fn (AppEnvironment $env) => new BootloadConfig(enabled: $env->isLocal()),
            // ...
        ];
    }
}
```

Вы также можете использовать класс `BootloadConfig` как атрибут для управления поведением загрузчика. Этот метод особенно полезен, потому что он позволяет вам настроить конфигурацию непосредственно в классе загрузчика, делая это более прямолинейным и легким для понимания.

**Вот простой пример того, как вы можете использовать атрибут для настройки загрузчика:**

```php app/src/Application/Bootloader/SomeBootloader.php
use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

#[BootloadConfig(allowEnv: ['APP_ENV' => 'local'])]
final class SomeBootloader extends Bootloader
{
}
```

Атрибуты - отличный выбор, когда вы хотите держать конфигурацию близко к коду загрузчика. Это более интуитивный способ настройки загрузчиков, особенно в случаях, когда конфигурация проста и не требует сложной логики.

#### Расширение для пользовательских предварительных условий

Расширяя `BootloadConfig`, вы можете создавать пользовательские классы, которые инкапсулируют конкретные условия, при которых загрузчики должны работать. Этот подход упрощает использование загрузчиков, абстрагируя детали конфигурации в эти пользовательские классы.

```php app/src/Application/Bootloader/TargetRRWorker.php
namespace App\Application\Bootloader;

use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

class TargetRRWorker extends BootloadConfig 
{
    public function __construct(array $modes)
    {
        parent::__construct(
            env: ['RR_MODE' => $modes],
        );
    }
}
```

Теперь вы можете использовать это в ваших загрузчиках

```php app/src/Application/Bootloader/SomeBootloader.php
use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

#[TargetRRWorker(modes: ['http', 'grpc'])]
final class SomeBootloader extends Bootloader
{
}
```

или использовать в ядре:

```php app/src/Application/Kernel.php
namespace App\Application;

use Spiral\Framework\Kernel;

class Kernel extends Kernel
{
    public function defineBootloaders(): array
    {
        return [
            HttpBootloader::class => new TargetRRWorker(['http']),
            RoutesBootloader::class => new TargetRRWorker(['http']),

            GrpcBootloader::class => new TargetRRWorker(['grpc']),

            TemporalBootloader::class => new TargetRRWorker(['temporal']),
            
            // Другие загрузчики...
        ];
    }
}
```

Возможность расширения `BootloadConfig` открывает мир возможностей для настройки поведения загрузчиков.

## Доступные методы

Загрузчики предоставляют два метода `init` и `boot`, которые выполняются при инициализации приложения.

### Метод Init

Этот метод будет выполнен **первым**.

Это хорошая идея - установить значения по умолчанию для файлов конфигурации перед продолжением. Вы можете использовать специальные методы загрузчика для изменения файлов конфигурации по мере необходимости. После этого вы можете продолжить и выполнить любую другую логику, которая не требует чтения файлов конфигурации и не зависит от выполнения кода в методах `init` и `boot` других загрузчиков.

Это также хорошее время для добавления обратных вызовов инициализации и настройки привязок контейнера, пока это не требует доступа к конфигурации приложения.

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Config\ConfiguratorInterface;
use App\Service\Github\GithubConfig;

final class GithubClientBootloader extends Bootloader
{
    public function __construct(
        private readonly ConfiguratorInterface $config
    ) {
    }

    public function init(EnvironmentInterface $env): void 
    {
        $this->config->setDefaults(
            GithubConfig::CONFIG,
            [
                'access_token' => $env->get('GITHUB_ACCESS_TOKEN'),
                'secret' => $env->get('GITHUB_SECRET'),
            ]
        );
    }
}
```

> **Примечание**
> 1. Узнайте больше о `Spiral\Boot\EnvironmentInterface` в разделе [Конфигурация](../start/configuration.md).
> 2. Узнайте больше о классе `Spiral\Boot\AbstractKernel` (также известном как 'Kernel') в разделе [Ядро и окружение](../framework/kernel.md).
> 3. Узнайте больше о классе `Spiral\Config\ConfiguratorInterface` в разделе [Объекты конфигурации](../framework/config.md).

### Метод Boot

Этот метод будет выполнен после того, как метод `init` во всех загрузчиках был выполнен. Причина этого в том, что вам могут понадобиться результаты инициализации загрузчика для продолжения. Например, скомпилированные файлы конфигурации.

Просто имейте в виду, что он должен быть запущен после того, как все эти методы `init` завершились.

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use App\Service\Github\GithubConfig;

final class GithubClientBootloader extends Bootloader
{
    // См. код выше ...
    
    public function boot(GithubConfig $config): void 
    {
        $token = $config->getAccessToken();
        // ...
    }
}
```

## Настройка контейнера

Загрузчики обычно используются для настройки контейнера, например, если мы хотим связать несколько реализаций с их интерфейсами или создать некоторый сервис. Мы можем использовать как метод `init`, так и `boot` для этого, что позволяет нам запрашивать любые необходимые сервисы, используя внедрение методов.

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use Spiral\Core\BinderInterface;
use App\Service\Github\GithubConfig;
use App\Service\Github\ClientInterface;
use App\Service\Github\Client;

final class GithubClientBootloader extends Bootloader
{
    // См. код выше ...
    
    public function boot(BinderInterface $binder): void 
    {
        $binder->bindSingleton(
            ClientInterface::class, 
            static fn (GithubConfig $config) => new Client(
                $config->getAccessToken(),
                $config->getSecret(),
            )
        );
    }
}
```

> **Примечание**
> Замыкание, предоставленное в качестве аргумента методу `bindSingleton`, будет вызвано контейнером внедрения зависимостей (DI), когда ему нужно создать экземпляр `MyService`. Когда замыкание вызывается, контейнер DI автоматически разрешит и внедрит любые зависимости, которые требуются замыканию.
>
> Если вы хотите узнать больше о DI, вы можете ознакомиться с разделом [Контейнер и фабрики](../container/overview.md) документации. В нем должна быть вся необходимая информация.

Загрузчики также предоставляют возможность упростить определение привязок контейнера

:::: tabs

::: tab Использование методов

Вы можете использовать методы `defineBindings` и `defineSingletons` для определения привязок контейнера декларативным способом.

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use Spiral\Core\BinderInterface;
use App\Service\Github\GithubConfig;
use App\Service\Github\ClientInterface;
use App\Service\Github\Client;

final class GithubClientBootloader extends Bootloader
{
    public function defineBindings(): array
    {
        return [
            MyInterface::class => MyClass::class
        ];
    }

    public function defineSingletons(): array
    {
        return [
            ClientInterface::class => [self::class, 'createClient'],
            
            // или
            
            ClientInterface::class => static fn(GithubConfig $config) => new Client(
                $config->getAccessToken(),
                $config->getSecret(),
            );
        ];
    }

    // См. код выше ...
    
    public function createClient(GithubConfig $config): ClientInterface 
    {
        return new Client(
            $config->getAccessToken(),
            $config->getSecret(),
        );
    }
}
```

:::

::: tab Использование констант

Вы можете использовать константы `BINDINGS` и `SINGLETONS` для определения привязок контейнера декларативным способом.

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use Spiral\Core\BinderInterface;
use App\Service\Github\GithubConfig;
use App\Service\Github\ClientInterface;
use App\Service\Github\Client;

final class GithubClientBootloader extends Bootloader
{
    const BINDINGS = [
        MyInterface::class => MyClass::class
    ];
    
    const SINGLETONS = [
        ClientInterface::class => [self::class, 'createClient'],
    ];
    
    // См. код выше ...
    
    public function createClient(GithubConfig $config): ClientInterface 
    {
        return new Client(
            $config->getAccessToken(),
            $config->getSecret(),
        );
    }
}
```

:::

::::

## Настройка приложения

Другой распространенный случай использования загрузчиков - это настройка фреймворка перед запуском приложения. Например, мы можем объявить новый маршрут для нашего приложения или модуля:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;
use Spiral\Router\Route;

final class RoutesBootloader extends Bootloader 
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'my-route',
            new Route('/<action>', new Controller(MyController::class))
        );
    }
}
```

> **Примечание**
> Вы можете использовать загрузчики для настройки ваших компонентов только во время фазы загрузки (также известной как через другой загрузчик). Фреймворк не позволит вам изменить какое-либо значение конфигурации после инициализации компонента.

## Зависимость от других загрузчиков

Зависимость от других загрузчиков может быть действительно полезной в определенных ситуациях. Например, если вы хотите убедиться, что определенный загрузчик инициализирован перед вашим, вы можете использовать один из двух основных подходов: внедрение класса загрузчика в метод init или boot в качестве аргумента, или использование константы `Bootloader::DEPENDENCIES` в вашем классе загрузчика.

Это может быть хорошим способом управления инициализацией вашего приложения и обеспечения того, что все необходимые ресурсы и зависимости доступны, когда они вам нужны. Просто имейте в виду, что зависимые загрузчики будут инициализированы только один раз, даже если от них зависят несколько других загрузчиков.

Некоторые загрузчики фреймворка могут использоваться как простой способ настройки параметров приложения. Например, мы можем использовать `Spiral\Bootloader\Http\HttpBootloader` для добавления глобального PSR-15 промежуточного ПО:

**Существует два способа определения зависимых загрузчиков:**

1. Внедрение класса загрузчика в метод `init` или `boot` в качестве аргумента вашего класса загрузчика

**Например:**

```php app/src/Application/Bootloader/MyBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\HttpBootloader;
use App\Middleware\MyMiddleware;

class MyBootloader extends Bootloader 
{
    public function boot(HttpBootloader $http): void
    {
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

2. Использование константы `Bootloader::DEPENDENCIES` в вашем классе загрузчика. Это может быть удобным способом определения зависимых загрузчиков, когда вам не нужно обращаться к ним напрямую в вашем классе загрузчика.

**Например:**

```php app/src/Application/Bootloader/MyBootloader.php
namespace App\Application\Bootloader;

class MyBootloader extends Bootloader 
{
    protected const DEPENDENCIES = [
        \Spiral\Bootloader\Http\HttpBootloader::class
    ];
    
    public function boot(): void
    {
        // ...
    }
}
```

Spiral автоматически разрешит и инициализирует зависимый загрузчик перед тем, как зависящий загрузчик будет инициализирован.

Оба этих подхода позволяют вам определять зависимые загрузчики декларативным способом, что может облегчить управление инициализацией вашего приложения и обеспечить, что все необходимые ресурсы и зависимости доступны, когда они нужны.

> **Примечание**
> Зависимые загрузчики будут инициализированы только один раз, даже если несколько других загрузчиков зависят от них.

## Каскадная загрузка

Вы можете управлять процессом загрузки, используя сам Bootloader, просто запросив `Spiral\Boot\BootloadManager`:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\BootloadManager;
use Spiral\Bootloader\DebugBootloader;
use Spiral\Boot\EnvironmentInterface;

class AppBootloader extends Bootloader
{
    public function boot(BootloadManager $bootloadManager, EnvironmentInterface $env): void
    {
        if ($env->get('DEBUG')) {
            $bootloadManager->bootload([
                DebugBootloader::class
            ]);
        }
    }
}
```

<hr>

## Что дальше?

Теперь углубитесь в основы, прочитав несколько статей:

* [HTTP — Перехватчики (Interceptor)](../http/interceptors.md)
* [Создание каркасов](../basics/scaffolding.md)
