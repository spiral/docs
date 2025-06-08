# Основы — Обработка ошибок

В процессе разработки часто возникают ошибки и исключения. Отладка этих исключений может быть сложной и трудоемкой задачей, но это критически важный аспект процесса разработки. Spiral предлагает широкий спектр инструментов и техник для отладки исключений и выявления основных причин проблем.

Эта документация проведет вас через функции, доступные в Spiral для обработки исключений, их отображения и настройки.

## Обработчик исключений

Spiral предлагает надежный механизм для обработки исключений, предоставляемый классом `Spiral\Exceptions\ExceptionHandler`.

Он разработан для управления как глобальными, так и runtime ошибками, предлагая структурированный подход к обработке исключений в вашем приложении.

#### Ключевые функции

- **Отображение исключений:** Отображение исключений в различных форматах, включая HTML, JSON и обычный текст.
- **Отчетность по исключениям:** Отправка отчетов об исключениях во внешние сервисы, такие как [Sentry](#sentry-integration) или [S3 storage](#cloud-storage-reporter).
- **Глобальная обработка ошибок:** Класс используется для обработки глобальных ошибок, таких как фатальные ошибки и ошибки завершения работы.
- **Настраиваемость:** Класс позволяет добавлять пользовательские рендереры и репортеры.

### Настройка обработчика исключений

Для приложений, требующих специфических стратегий обработки ошибок, Spiral предлагает гибкость замены этого обработчика по умолчанию на пользовательскую реализацию.

Сначала создайте класс, который расширяет класс `Spiral\Exceptions\ExceptionHandler`:

```php app/src/Application/Exception/Handler.php
<?php

declare(strict_types=1);

namespace App\Application\Exception;

use Spiral\Exceptions\ExceptionHandler;
use Throwable;

final class Handler extends ExceptionHandler
{
    // ...
}
```

Затем укажите класс в файле `app.php`:

```php app.php
use App\Application\Kernel;
use App\Application\Exception\Handler;

// ...

$app = Kernel::create(
    directories: ['root' => __DIR__],
    exceptionHandler: Handler::class, // <--
)->run();

// ...
```

Когда обработчик инициализируется, он вызовет метод `bootBasicHandlers`, который является одним из способов настройки обработчика. Этот метод используется для регистрации базовых рендереров и репортеров.

```php app/src/Application/Exception/Handler.php
final class Handler extends ExceptionHandler
{
    protected function bootBasicHandlers(): void
    {
        parent::bootBasicHandlers();
        
        // Зарегистрируйте здесь ваши рендереры и репортеры
        // $this->addRenderer(new MyRenderer());
        // $this->addRenderer(new MyReporter());
    }
}
```

Handler — отличное место для обработки исключений, которые происходят во время процесса загрузки приложения. Например, если вы хотите пропустить отчетность о некоторых исключениях, вы можете переопределить метод `report` и обработать их там.

```php app/src/Application/Exception/Handler.php
<?php

declare(strict_types=1);

namespace App\Application\Exception;

use Spiral\Exceptions\ExceptionHandler;
use Spiral\Http\Exception\ClientException;

final class Handler extends ExceptionHandler
{
    /**
     * @var class-string<\Throwable>[]
     */
    private array $nonReportableExceptions = [
        ClientException::class,
        // ...
    ];

    public function report(\Throwable $exception): void
    {
        foreach ($this->nonReportableExceptions as $nonReportableException) {
            if ($exception instanceof $nonReportableException) {
                return;
            }
        }

        parent::report($exception);
    }
}
```

## Отображение исключений

Spiral использует форматы для определения того, какой рендерер должен использоваться для обработки данного исключения. Формат может основываться на окружении, таком как `cli` для консольных приложений или `http` для HTTP-запросов. Это позволяет регистрировать и использовать различные рендереры в зависимости от контекста, в котором было обнаружено исключение.

### Как это работает

Иногда вам может потребоваться показать ошибки особым образом, например, в JSON для API. Вот как вы можете это сделать:

1. **Создайте рендерер**

Вот пример того, как реализовать JSON рендерер:

```php app/src/Application/Exception/Renderer/JsonRenderer.php
<?php

namespace Spiral\YiiErrorHandler;

use Spiral\Exceptions\ExceptionRendererInterface;
use Spiral\Exceptions\Verbosity;
use Yiisoft\ErrorHandler\Renderer\JsonRenderer as YiiJsonRenderer;
use Yiisoft\ErrorHandler\ThrowableRendererInterface;

final class JsonRenderer implements ExceptionRendererInterface
{
    public const FORMATS = ['application/json', 'json'];

    public function __construct(
        private readonly ?ThrowableRendererInterface $renderer = new YiiJsonRenderer()
    ) {
    }

    public function render(
        \Throwable $exception,
        ?Verbosity $verbosity = Verbosity::BASIC,
        string $format = null,
    ): string {
        if ($verbosity >= Verbosity::VERBOSE) {
            return (string)$this->renderer->renderVerbose($exception);
        }

        return (string)$this->renderer->render($exception);
    }

    public function canRender(string $format): bool
    {
        return \in_array($format, self::FORMATS, true);
    }
}
```

> **Примечание**
> `Spiral\YiiErrorHandler\JsonRenderer` является частью пакета `spiral-packages/yii-error-handler-bridge`.

2. **Зарегистрируйте ваш рендерер**

Зарегистрируйте пользовательский рендерер, используя bootloader

```php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use Spiral\YiiErrorHandler\JsonRenderer;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->addRenderer(new JsonRenderer());
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
        \App\Application\Bootloader\ExceptionHandlerBootloader::class,
        // ...
    ];
}
```

Читайте больше о bootloaders в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\ExceptionHandlerBootloader::class,
    // ...
];
```

Читайте больше о bootloaders в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

3. **Используйте ваш рендерер**

Чтобы использовать этот рендерер для обработки исключений в веб-приложении, мы можем создать новый middleware, который будет перехватывать все исключения и отображать их с помощью этого рендерера только тогда, когда в запросе присутствует специфический заголовок, такой как `Accept=application/json`. Это позволяет более точно контролировать то, как исключения обрабатываются и отображаются клиенту, в зависимости от желаемого формата.

```php
namespace App\Endpoint\Web\Middleware;

use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as Handler;
use Psr\Http\Message\ResponseFactoryInterface;
use Spiral\Exceptions\ExceptionRendererInterface;
use Spiral\Http\Exception\ClientException;
use Spiral\Router\Exception\RouterException;

class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ExceptionRendererInterface $renderer,
        private readonly ResponseFactoryInterface $responseFactory,
    ) {
    }

    public function process(Request $request, Handler $handler): Response
    {
        try {
            return $handler->handle($request);
        } catch (ClientException|RouterException $e) {
            $code = $e instanceof ClientException ? $e->getCode() : 404;
        } catch (\Throwable $e) {
            $code = 500;
        }
        
        $response = $this->responseFactory->createResponse($code);
        $response->getBody()->write(
            (string) $this->renderer->render(
                exception: $e,
                format: $request->getHeaderLine('Accept') ?? 'application/json'
            )
        );

        return $response;
    }
}
```

Как вы можете видеть, различные рендереры могут использоваться для разных окружений, таких как консольный рендерер для приложений командной строки или JSON рендерер для API ответов. Кроме того, различные рендереры могут использоваться для разных форматов.

### Существующие рендереры

Вот некоторые рендереры, которые Spiral уже предоставляет вам:

| Рендерер                                     | Форматы                                         |
|----------------------------------------------|-------------------------------------------------|
| `Spiral\Exceptions\Renderer\ConsoleRenderer` | `console`, `cli`                                |
| `Spiral\Exceptions\Renderer\JsonRenderer`    | `application/json`, `json`                      | 
| `Spiral\Exceptions\Renderer\PlainRenderer`   | `text/plain`, `text`, `plain`, `cli`, `console` |

В некоторых случаях, например, когда `DEBUG=true`, вы можете предпочесть отображать красивую страницу ошибки с подсветкой кода, такую как [filp/whoops](https://github.com/filp/whoops) или [yiisoft/error-handler](https://github.com/spiral-packages/yii-error-handler-bridge).

### Yii Error Renderer

Yii Error Handler — это bridge пакет для Spiral, который обеспечивает интеграцию с обработчиками ошибок фреймворка Yii.

![screenshot](https://user-images.githubusercontent.com/773481/215085868-a7228f6c-1be0-460d-b910-85fa2cd1195b.png)

#### Установка

Для установки компонента:

```terminal
composer require spiral-packages/yii-error-handler-bridge
```

После установки пакета вам нужно зарегистрировать bootloader из пакета:

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader::class,
        // ...
    ];
}
```

Читайте больше о bootloaders в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader::class,
    // ...
];
```

Читайте больше о bootloaders в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

`YiiErrorHandlerBootloader` зарегистрирует все доступные рендереры во время инициализации. Если вы хотите зарегистрировать определенные рендереры.

#### Встроенные рендереры

Bridge предоставляет несколько встроенных рендереров для отображения ошибок:

- `HtmlRenderer`: Отображает страницы ошибок как HTML.
- `JsonRenderer`: Отображает страницы ошибок как JSON. Это может быть полезно для обработки ошибок в API запросах.
- `PlainTextRenderer`: Отображает страницы ошибок как обычный текст.

### Уровни детализации

Уровень детализации может использоваться для контроля количества информации, которая отображается при отображении исключения. Вы можете установить `VERBOSITY_LEVEL` в файле `.env`:

```dotenv .env
# Уровень детализации
VERBOSITY_LEVEL=verbose # basic, verbose, debug
```

Возможные значения определены перечислением `Spiral\Exceptions\Verbosity`:

#### basic или 0

Указывает, что должна отображаться только базовая информация об исключении. Если произошла ошибка, вы увидите:

```output
[Spiral\Router\Exception\RouteNotFoundException] 
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75
```

#### verbose или 1

Указывает, что должна отображаться более детальная информация об исключении. Если произошла ошибка, вы увидите:

```output

[Spiral\Router\Exception\RouteNotFoundException]
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75

 1. Spiral\Router\Router->Spiral\Router\{closure}() at vendor/spiral/framework/src/Router/src/Router.php:75
 2. Spiral\Router\Router->Spiral\Router\{closure}()
 3. ReflectionFunction->invokeArgs() at vendor/spiral/framework/src/Core/src/Internal/Invoker.php:73
 4. ...
```

#### debug или 2

Указывает, что должна отображаться наиболее детальная информация об исключении. Если произошла ошибка, вы увидите:

```output
[Spiral\Router\Exception\RouteNotFoundException]
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75

 1. Spiral\Router\Router->Spiral\Router\{closure}() at vendor/spiral/framework/src/Router/src/Router.php:75
   73                 if ($route === null) {
   74                     $this->eventDispatcher?->dispatch(new RouteNotFound($request));
>  75                     throw new RouteNotFoundException($request->getUri());
   76                 }
   77 

 2. ...
```

<hr />

## Отчетность по исключениям

В Spiral вы можете использовать репортеры для отслеживания проблем, таких как ошибки, в вашем приложении. Репортеры могут выполнять две основные функции:

- Они могут сохранять информацию об этих проблемах в файл. Таким образом, вы можете позже просмотреть файл, чтобы выяснить, что пошло не так.
- Они также могут отправлять отчеты об этих проблемах в другие сервисы, такие как [Sentry](https://sentry.io). Эти сервисы могут предоставить еще больше деталей о проблемах в вашем приложении.

Представьте, что у вас есть веб-сайт, и иногда что-то работает не так, как должно. Это может происходить из-за ошибок в вашем коде, например, когда файл отсутствует или есть проблема с базой данных. Репортеры помогают вам эффективно обрабатывать эти ошибки.

### Как работают репортеры

#### 1. Реализация `ExceptionReporterInterface` или использование встроенных репортеров:

Вы создадите класс (например, `CustomReporter` в примере), который реализует `Spiral\Exceptions\ExceptionReporterInterface`. Думайте об этом классе как об агенте-репортере, который знает, что делать при возникновении исключения.

> **Примечание**
> Читайте больше о доступных репортерах в разделе [Доступные репортеры](#доступные-репортеры) ниже.

```php app/src/Application/Exception/Reporter/CustomReporter.php
namespace App\Application\Exception\Reporter;

final class CustomReporter implements ExceptionReporterInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function report(\Throwable $exception): void
    {
        // Сохранить информацию об исключении в файл или отправить в внешний сервис
        $this->logger->error($exception->getMessage(), ['exception' => $exception]);
    }
}
```

#### 2. Регистрация репортеров:

Чтобы использовать репортеры, вам сначала нужно зарегистрировать их в классе `ExceptionHandler` аналогично рендерерам, используя метод `addReporter` и предоставляя экземпляр класса, который реализует `Spiral\Exceptions\ExceptionReporterInterface`.

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use App\Application\Exception\Reporter\CustomReporter;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->addReporter(new CustomReporter());
    }
}
```

#### 3. Использование репортеров в вашем коде:

Теперь в коде вашего приложения вы можете использовать эти репортеры всякий раз, когда ожидаете, что может произойти исключение.

Например, в примере `PingSiteJob`, если что-то пойдет не так при попытке пинговать веб-сайт (например, сайт недоступен), исключение перехватывается и сообщается с помощью репортера.

```php
<?php

declare(strict_types=1);

namespace App\Job;

use Spiral\Exceptions\ExceptionReporterInterface;

final class PingSiteJob
{
    public function __construct(
        private PingClient $client,
        private ExceptionReporterInterface $reporter,
    ) {
    }

    public function handle(string $url): void
    {
        try {
            $this->client->ping($url);
        } catch (\Throwble $e) {
            $this->reporter->report($e);
        }
    }
}
```

> **Примечание**
> Репортер отправит исключение через всех зарегистрированных репортеров.

### Доступные репортеры

Spiral поставляется с тремя встроенными репортерами, которые доступны из коробки: `Spiral\Exceptions\Reporter\LoggerReporter`, `Spiral\Exceptions\Reporter\FileReporter` и `Spiral\Exceptions\Reporter\StorageReporter`.

#### Logger Reporter

`Spiral\Exceptions\Reporter\LoggerReporter` включен по умолчанию и позволяет логировать исключения с помощью логгера, зарегистрированного в приложении. Это может быть полезно для отслеживания и анализа ошибок со временем.

#### File Reporter

`Spiral\Exceptions\Reporter\FileReporter` также включен по умолчанию, он позволяет сохранять подробную информацию об исключении в файл, известный как `snapshot` в директории `runtime/snapshots`.

#### Cloud Storage Reporter

Сталкивались ли вы когда-нибудь с проблемами хранения снимков исключений вашего приложения при работе с stateless приложениями? У нас есть хорошие новости. Мы сделали это супер легким для вас.

Интегрируясь с компонентом `spiral/storage`, мы даем вашим stateless приложениям возможность сохранять снимки исключений прямо в облачные хранилища, такие как **S3**.

**Почему это потрясающе для вас?**

1. **Упрощенное хранение:** Больше никаких манипуляций со сложными решениями хранения. Сохраняйте снимки прямо в S3 с легкостью.
2. **Специально для Stateless приложений:** Разработано специально для stateless приложений, делая ваши развертывания более плавными и беспроблемными.
3. **Надежность:** С проверенной репутацией S3 знайте, что ваши снимки хранятся безопасно и могут быть доступны всякий раз, когда вам нужно.

`Spiral\Exceptions\Reporter\StorageReporter` также включен по умолчанию, он позволяет сохранять подробную информацию об исключении в файл, известный как `snapshot` в директории `runtime/snapshots`.

**Чтобы использовать этот репортер, вам нужно:**

1. Настроить компонент `spiral/storage`. Читайте больше об этом в разделе [Component — Storage and Cloud distribution](../advanced/storage.md).
2. Зарегистрировать `Spiral\Bootloader\StorageSnapshotsBootloader`
3. Указать желаемый bucket, используя переменную окружения `SNAPSHOTS_BUCKET`, где вы хотите хранить ваши снимки.
4. Зарегистрировать `Spiral\Exceptions\Reporter\StorageReporter` в классе `Spiral\Exceptions\ExceptionHandler`.

## Интеграция Sentry

Spiral предлагает bridge пакет Sentry, который облегчает легкую интеграцию с сервисом [Sentry](https://sentry.io/). Этот документ проведет вас через процесс интеграции и настройки этого инструмента в вашем Spiral приложении.

### Установка

1. Установите компонент Sentry bridge:

```terminal
composer require spiral/sentry-bridge
```

2. После установки зарегистрируйте bootloader `Spiral\Sentry\Bootloader\SentryReporterBootloader` из пакета в вашем приложении:

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
        // ...
    ];
}
```

Читайте больше о bootloaders в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

Читайте больше о bootloaders в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

Это зарегистрирует `Spiral\Sentry\SentryReporter` в `Spiral\Exceptions\ExceptionHandler`.

### Конфигурация

Все, что вам нужно сделать, это установить переменную окружения `SENTRY_DSN` на ваш Sentry DSN.

```dotenv .env
SENTRY_DSN=https://...
```

Начиная с **v2.2** вы также можете использовать дополнительные переменные окружения для настройки репортера:

- `SENTRY_DSN`: Sentry Data Source Name (DSN).
- `SENTRY_SAMPLE_RATE`: Частота выборки событий (например, `0.4`).
- `SENTRY_TRACES_SAMPLE_RATE`: Частота для выборки трассировки (например, `1.0`).
- `SENTRY_SEND_DEFAULT_PII`: Отправлять ли личную идентифицирующую информацию по умолчанию (`true`/`false`).
- `SENTRY_ENVIRONMENT`: Окружение (например, `develop`). В качестве альтернативы можно использовать `APP_ENV`.
- `SENTRY_RELEASE`: Версия релиза (например, `1.0.0`). В качестве альтернативы используйте `APP_VERSION`.

Вот пример:

```dotenv .env
SENTRY_DSN=https://...
SENTRY_SAMPLE_RATE=0.4
SENTRY_TRACES_SAMPLE_RATE=1.0
SENTRY_SEND_DEFAULT_PII=false

SENTRY_ENVIRONMENT=develop
SENTRY_RELEASE=1.0.0
# или
APP_ENV=develop
APP_VERSION=1.0.0
```

Мы также предоставляем способ настройки репортера с помощью файла `config/sentry.php`:

```php config/sentry.php
return [
  'dsn' => 'http://...',
  'environment' => 'develop',
  'release' => '1.0.0',
  'sample_rate' => 1.0,
  'traces_sample_rate' => null,
  'send_default_pii' => true,
];
```

### Интеграции Sentry [Начиная с **v2.2**]

Начиная с **v2.2** мы добавили поддержку [интеграций Sentry](https://docs.sentry.io/platforms/php/integrations/).

Вы можете регистрировать специфичные для приложения интеграции через `Spiral\Sentry\Bootloader\ClientBootloader`. Это делает простым добавление пользовательских функций, адаптированных под нужды вашего приложения.

**Пример регистрации пользовательской интеграции:**

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Sentry\Bootloader\ClientBootloader;

use Spiral\Boot\Bootloader\Bootloader;

final class AppBootloader extends Bootloader
{
    public function init(ClientBootloader $sentry): void
    {
        $sentry->addIntegration(new ExceptionContextIntegration());
    }
}
```

#### Интеграция HTTP Request

Sentry будет автоматически собирать информацию о текущем запросе, используя встроенную интеграцию `Sentry\Integration\RequestIntegration`. Также есть `Spiral\Sentry\Http\SetRequestIpMiddleware`. Этот middleware критически важен для сбора IP-адресов пользователей, когда включен `send_default_pii`. Это опциональная, но мощная функция для тех, кому нужны подробные сведения о пользователях.

> **Примечание**
> Читайте больше о middleware в разделе [HTTP — Routing](../http/routing.md#add-middleware).

### Доступные привязки контейнера [Начиная с **v2.2**]

Для разработчиков, ищущих более глубокую интеграцию и контроль, мы представили новые привязки контейнера:

- `Sentry\Options`: Контейнер конфигурации для тонкой настройки параметров клиента Sentry.
- `Sentry\State\HubInterface`: Предоставляет доступ к управлению состоянием и контекстом Sentry.
- `Sentry\ClientInterface`: Облегчает прямые взаимодействия с клиентом Sentry.

Эти привязки предлагают детальный контроль над клиентом Sentry, обслуживая продвинутые случаи использования.

### Предоставление дополнительных данных

Чтобы предоставить текущие логи приложения, такие как логи приложения или состояние PSR-7 запроса, включите сборщики отладочной информации. Эти сборщики собирают соответствующие данные о текущем состоянии приложения.

Когда происходит исключение, репортер Sentry запросит класс `Spiral\Debug\StateInterface` из IoC контейнера, и во время создания объект будет заполнен информацией из зарегистрированных сборщиков.

> **Предупреждение**
> Будьте осторожны при запросе `Spiral\Debug\StateInterface` из контейнера. Объект будет создаваться при каждом запросе из контейнера, и вы не можете заполнить его вне сборщиков. Если вам нужно добавить дополнительную информацию к объекту `Spiral\Debug\StateInterface`, вы должны использовать сборщики.

#### Http collector

HTTP сборщик — хороший способ отправки информации о текущем запросе в Sentry.

> **Примечание**
> Начиная с **v2.2** HTTP сборщик можно избежать, потому что репортер будет автоматически собирать информацию о текущем запросе, используя встроенную интеграцию `Sentry\Integration\RequestIntegration` лучшим способом.

Он отправит следующую информацию о текущем запросе:

- `method`
- `url`
- `headers`
- `query params`
- `request body`

Чтобы включить HTTP сборщик, вам сначала нужно зарегистрировать `Spiral\Bootloader\Debug\HttpCollectorBootloader` перед `SentryReporterBootloader`.

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Debug\HttpCollectorBootloader::class,
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
        // ...
    ];
}
```

Читайте больше о bootloaders в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Debug\HttpCollectorBootloader::class,
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

Читайте больше о bootloaders в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

Затем вам нужно зарегистрировать middleware `Spiral\Debug\StateCollector\HttpCollector` в приложении.

> **Смотрите больше**
> Читайте больше о том, как регистрировать middleware в разделе [HTTP — Routing](../http/routing.md#add-middleware).

#### Logs collector

Используйте сборщик логов для отправки всех полученных логов в Sentry.

Чтобы включить сборщик логов, вам просто нужно зарегистрировать `Spiral\Bootloader\Debug\LogCollectorBootloader` перед `SentryBootloader`.

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Debug\LogCollectorBootloader::class,
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
        // ...
    ];
}
```

Читайте больше о bootloaders в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Debug\LogCollectorBootloader::class,
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

Читайте больше о bootloaders в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

#### Создание пользовательских сборщиков

Для специализированного сбора данных вы можете создать пользовательские сборщики. Сборщик должен реализовать интерфейс `Spiral\Debug\StateCollectorInterface`.

Например, рассмотрим SQL сборщик:

```php app/src/Application/Debug/Collector/SqlCollector.php
namespace App\Application\Debug\Collector;

use Spiral\Logger\Event\LogEvent;
use Spiral\Debug\StateCollectorInterface;

final class SqlCollector implements StateCollectorInterface
{
    public function __construct(
        private readonly Database $db
    ) {
    }

    public function collect(\Spiral\Debug\StateInterface $state): void
    {
       foreach($this->db->getQueries() as $query) {
            $state->addLogEvent(new LogEvent(
                time: $query->getTime(),
                channel: 'sql',
                level: 'info',
                message: $query->getQuery(),
                context: $query->getParameters()
            ));
       }
    }
}
```

> **Предупреждение**
> Приведенный выше пример использует несуществующий класс Database, что означает, что вам нужно будет реализовать это самостоятельно.

Вот некоторые полезные методы объекта `Spiral\Debug\StateInterface`:

**Добавить тег**

Метод добавит теги, связанные с текущей областью видимости

```php
$state->addTag('IP address', $currentRequest->getIpAddress());
$state->addTag('Environment', $env->get('APP_ENV'));
```

**Добавить переменную**

Метод добавит дополнительные данные, связанные с текущей областью видимости

```php
$state->setVariable('query', $currentRequest->getQueryParams());
```

**Добавить событие лога**

Метод добавит событие лога как breadcrumb в текущую область видимости.

```php
$state->addLogEvent(new \Spiral\Logger\Event\LogEvent(
    time: new \DateTimeImmutable(),
    channel: 'default',
    level: 'info',
    message: 'Something went wrong',
    context: ['foo' => 'bar']
));
```

#### Регистрация пользовательского сборщика

Вы можете зарегистрировать ваш сборщик в bootloader.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\DebugBootloader;
use App\Application\Exception\Reporter\CustomReporter;

final class AppBootloader extends Bootloader
{
    public function init(DebugBootloader $debug, SqlCollector $sqlCollector): void
    {
        $debug->addStateCollector($sqlCollector);
    }
}
```
