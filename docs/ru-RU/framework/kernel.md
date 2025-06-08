# Фреймворк — Ядро и окружение

Spiral использует объект ядра, который содержит набор специфичных для приложения сервисов. В отличие от Symfony, Spiral требует только одно ядро для всех методов диспетчеризации, таких как HTTP, Queue, GRPC Console и т.д. Ядро автоматически выбирает подходящий метод диспетчеризации на основе подключенного [Dispatcher](../framework/dispatcher.md).

> **Примечание**
> Базовая реализация ядра находится в репозитории `spiral/boot`.

## Обязанности ядра

Класс Spiral\Boot\AbstractKernel отвечает за следующие аспекты приложения:

- Инициализация контейнера через набор специфичных для приложения загрузчиков (Bootloader)
- Инициализация загрузчиков (Bootloader)
- Инициализация окружения и структуры директорий
- Инициализация обработчика исключений (при необходимости)
- Выбор подходящего диспетчера

Для создания ядра приложения необходимо расширить класс `Spiral\Boot\AbstractKernel`. Пример этого можно увидеть в следующем фрагменте кода:

```php app/src/Application/MyApp.php
namespace App\Application;

use Spiral\Boot\AbstractKernel;
use Spiral\Boot\Exception\BootException;

final class MyApp extends AbstractKernel
{
    protected const LOAD = [
        // загрузчики для инициализации
    ];

    protected function bootstrap(): void
    {
        // пользовательский код инициализации
        // вызывается после загрузки всех загрузчиков
    }

    protected function mapDirectories(array $directories): array
    {
        if (!isset($directories['root'])) {
            throw new BootException('Missing required directory `root`');
        }

        if (!isset($directories['app'])) {
            $directories['app'] = $directories['root'] . '/app/';
        }

        return \array_merge(
            [
                // публичный корень
                'public'    => $directories['root'] . '/public/',

                // библиотеки поставщиков
                'vendor'    => $directories['root'] . '/vendor/',

                // директории данных
                'runtime'   => $directories['root'] . '/runtime/',
                'cache'     => $directories['root'] . '/runtime/cache/',

                // директории приложения
                'config'    => $directories['app'] . '/config/',
                'resources' => $directories['app'] . '/resources/',
            ],
            $directories
        );
    }
}
```

> **Примечание**
> `Spiral\Framework\Kernel` определяет карту директорий по умолчанию.

## Инициализация ядра

Для инициализации ядра должен быть вызван статический метод `create`. Пример этого можно увидеть в следующем фрагменте кода:

```php app.php
$myapp = MyApp::create(
    directories: [
        'root' => __DIR__,
    ],
    handleErrors: false // не устанавливать обработчик ошибок
);

$myapp->run(environment: null); // использовать окружение по умолчанию

\dump($myapp->get(\Spiral\Boot\DirectoriesInterface::class)->getAll());
```

> **Примечание**
> Во время инициализации `MyApp` будет привязан к `Spiral\Boot\KernelInterface` в контейнере как синглтон.

### Колбэки

Класс `Spiral\Boot\AbstractKernel` предоставляет несколько колбэков, которые выполняются на разных этапах инициализации приложения. Эти колбэки: `running`, `booting`, `booted` и `bootstrapped`. Класс `Spiral\Framework\Kernel`, который расширяет `AbstractKernel`, добавляет дополнительные колбэки `appBooting` и `appBooted`. Это позволяет разработчикам выполнять пользовательские действия на определенных этапах процесса инициализации приложения.

> **Примечание**
> В пакете приложения класс по умолчанию `App\Application\Kernel` расширяет класс `Spiral\Framework\Kernel` и использует эти колбэки.

#### Running

Колбэк `running` является первым колбэком, выполняемым во время процесса инициализации приложения. Он выполняется при вызове метода `run`, сразу после привязки `EnvironmentInterface` в контейнере приложения.

Вот пример колбэка `running`:

```php app.php
$app = MyApp::create(directories: ['root' => __DIR__]);

$app->running(static function (): void {
    // Выполнить что-то
});

$app->run();
```

> **Примечание**
> Колбэки могут быть вызваны несколько раз для регистрации нескольких колбэков, они будут вызываться в порядке их регистрации.

#### Booting

Колбэк `booting` выполняется перед загрузкой всех загрузчиков фреймворка в секции `LOAD`.

Есть два способа зарегистрировать колбэк для этапа `booting`:

:::: tabs

::: tab Инициализация ядра

Метод `booting` может быть вызван на экземпляре приложения после его создания.

```php app.php
$app = MyApp::create(
    directories: ['root' => __DIR__]
);

$app->booting(function () {
    // ...
});

$app->run();
```

:::

::: tab Загрузчики ядра

Метод `init` загрузчика также может использоваться для регистрации колбэка booting. Этот метод выполняется перед вызовом колбэка.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

class AppBootloader extends Bootloader
{
    public function init(KernelInterface $app): void
    {
        $app->booting(function () {
            // ...
        });
    }
}
```

:::

::::

#### Booted

Колбэк `booted` выполняется после того, как все загрузчики фреймворка в секции `LOAD` завершили свой процесс инициализации.

```php
$app->booted(function () {
    // ...
});
```

#### AppBooting

Колбэк `appBooting` выполняется перед загрузкой всех загрузчиков приложения в секции `APP`.

```php
$app->appBooting(function () {
    // ...
});
```

#### AppBooted

Колбэк `appBooted` выполняется после того, как все загрузчики приложения в секции `APP` завершили свой процесс инициализации.

```php
$app->appBooted(function () {
    // ...
});
```

## Окружение

Spiral интегрируется с [Dotenv](https://github.com/vlucas/phpdotenv) через класс `Spiral\DotEnv\Bootloader\DotenvBootloader`. Этот загрузчик отвечает за загрузку переменных окружения из файла `.env` и предоставление их приложению.

### Переменные окружения

`Spiral\Boot\EnvironmentInterface` используется для доступа к списку переменных окружения (ENV vars). По умолчанию фреймворк полагается на переменные окружения системного уровня. Однако можно переопределить эти значения при инициализации ядра, передав пользовательский объект `Spiral\Boot\Environment` методу `run`.

> **Подробнее**
> Читайте больше об окружении приложения в разделе [Начало работы — Конфигурация](../start/configuration.md).

Пример этого можно увидеть в следующем фрагменте кода:

```php app.php
use \Spiral\Boot\Environment;

// Создать экземпляр приложения ...

$app->run(new Environment(['DEBUG' => true]));

\dump($app->get(\Spiral\Boot\EnvironmentInterface::class)->getAll());
```

> **Примечание**
> Этот подход может быть использован для инициализации приложения в целях тестирования.

### Расположение файла .env

По умолчанию загрузчик ищет файл `.env` в корне проекта, но вы можете изменить его расположение, определив переменную окружения `DOTENV_PATH` при запуске ядра:

```php app.php
use Spiral\Boot\Environment;

$app = App\Application\Kernel::create(...);

$app->run(new Environment(['DOTENV_PATH' => __DIR__ . '/.env.production']));
```

> **Примечание**
> Кроме того, вы также можете создать собственную реализацию класса `DotenvBootloader`. Это позволяет настроить поведение загрузки переменных окружения, например изменить расположение, где ищется файл .env, или добавить дополнительную функциональность. Это может быть полезно в случаях, когда загрузчик по умолчанию не соответствует специфическим требованиям вашего приложения.

### Перезапись переменных

По умолчанию Spiral не перезаписывает ранее установленные переменные окружения при загрузке новых из файла `.env`. Однако это поведение можно изменить, установив параметр `overwrite` в `true` при инициализации класса `Environment`.

```php app.php
use Spiral\Boot\Environment;

$app = App\Application\Kernel::create(...);

$app->run(new Environment([
    'APP_ENV' => 'production'
], overwrite: true));
```

## События

| Событие                              | Описание                                                                                                                         |
|--------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Spiral\Boot\Event\Bootstrapped       | Событие будет вызвано `после` инициализации всех загрузчиков из секций `SYSTEM`, `LOAD` и `APP`.                                |
| Spiral\Boot\Event\Serving            | Событие будет вызвано `перед` поиском диспетчера для обработки входящих запросов в текущем окружении.                           |
| Spiral\Boot\Event\DispatcherFound    | Событие будет вызвано, когда найден диспетчер для обработки входящих запросов в текущем окружении.                              |
| Spiral\Boot\Event\DispatcherNotFound | Событие будет вызвано, когда диспетчер приложения не найден.                                                                     |
| Spiral\Boot\Event\Finalizing         | Событие будет вызвано при выполнении финализаторов `перед` запуском финализаторов.                                               |

> **Примечание**
> Чтобы узнать больше о диспетчеризации событий, см. раздел [События](../advanced/events.md) в нашей документации.
