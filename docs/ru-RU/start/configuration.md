# Начало работы — Конфигурация

Spiral делает настройку вашего приложения с помощью файлов конфигурации невероятно простой. Все файлы конфигурации находятся в директории `app/config` и позволяют настраивать такие вещи, как подключение к базе данных, настройка хранилища кэша, управление очередями и так далее.

Вам даже не нужно создавать файлы конфигурации самостоятельно, поскольку Spiral поставляется с настройками по умолчанию. Но при желании вы также можете использовать переменные окружения для изменения основных параметров вашего приложения.

> **Примечание**
> Кэширование файлов конфигурации не требуется, поскольку они загружаются только один раз во время инициализации приложения и не перезагружаются, пока приложение не будет перезапущено.

## Переменные окружения

Использование переменных окружения — отличный способ отделить конфигурацию вашего приложения от самого кода. Это упрощает хранение конфиденциальной информации, такой как учетные данные базы данных, API-ключи и другие конфигурации, которые вы не хотите жестко кодировать в приложении.

Spiral интегрируется с [Dotenv](https://github.com/vlucas/phpdotenv) через класс `Spiral\DotEnv\Bootloader\DotenvBootloader`. Этот загрузчик (Bootloader) отвечает за загрузку переменных окружения из файла `.env` и обеспечение их доступности для приложения.

Общепринятой практикой является включение файла `.env.sample` в новый проект, который может использоваться в качестве руководства для настройки переменных окружения.

<details>
  <summary>Нажмите, чтобы показать .env.sample</summary>

```dotenv .env
# Environment (prod or local)
APP_ENV=local

# Debug mode set to TRUE disables view caching and enables higher verbosity
DEBUG=true
VERBOSITY_LEVEL=verbose # basic, verbose, debug

# Set to an application specific value, used to encrypt/decrypt cookies etc
ENCRYPTER_KEY=...

# Monolog
MONOLOG_DEFAULT_CHANNEL=default
MONOLOG_DEFAULT_LEVEL=DEBUG # DEBUG, INFO, NOTICE, WARNING, ERROR, CRITICAL, ALERT, EMERGENCY

# Queue
QUEUE_CONNECTION=roadrunner

# Cache
CACHE_STORAGE=roadrunner

# Telemetry
TELEMETRY_DRIVER=null

# Serializer
DEFAULT_SERIALIZER_FORMAT=json # csv, xml, yaml

# Session
SESSION_LIFETIME=86400
SESSION_COOKIE=sid

# Authorization
AUTH_TOKEN_TRANSPORT=cookie
AUTH_TOKEN_STORAGE=session

# Mailer
MAILER_DSN=
MAILER_FROM="My site <no-reply@site.com>"
```

</details>

> **Предупреждение**
> Переменная, определенная в суперглобальных массивах `$_SERVER` или `$_ENV`, будет иметь приоритет над значением той же переменной, определенной в файле `.env`.

Значения из файла `.env` будут скопированы в окружение вашего приложения и доступны через `Spiral\Boot\EnvironmentInterface` или функцию `env`.

### Доступные переменные

| Переменная                  | Описание                                                                                     |
|-----------------------------|----------------------------------------------------------------------------------------------|
| `APP_ENV`                   | Текущее окружение приложения. (`prod`, `stage`, `testing`, `local`). По умолчанию: `local`  |
| `DEBUG`                     | Режим отладки. По умолчанию: `false`                                                        |
| `TOKENIZER_CACHE_TARGETS`   | Кэш целей для токенизатора. (Bool) По умолчанию: `false`                                    |
| `ENCRYPTER_KEY`             | Ключ шифрования.                                                                             |
| `VERBOSITY_LEVEL`           | Уровень детализации. (`basic`, `verbose`, `debug`). По умолчанию: `verbose`                 |
| `MONOLOG_DEFAULT_CHANNEL`   | Имя канала по умолчанию из `app/config/monolog.php`. По умолчанию: 'default'                |
| `QUEUE_CONNECTION`          | Имя подключения очереди из `app/config/queue.php`. По умолчанию: `sync`                     |
| `BROADCAST_CONNECTION`      | Подключение трансляции. (`log`, `null`, `centrifugo`). По умолчанию: `null`                 |
| `CACHE_STORAGE`             | Хранилище кэша из `app/config/cache.php`                                                    |
| `TELEMETRY_DRIVER`          | Драйвер телеметрии. (`null`, `log`, `otel`). По умолчанию: `null`                           |
| `LOCALE`                    | Локаль по умолчанию. По умолчанию: `en`                                                     |
| `DEFAULT_SERIALIZER_FORMAT` | Формат сериализатора по умолчанию. (`json`, `serializer`) По умолчанию: `json`              |
| `AUTH_TOKEN_TRANSPORT`      | Транспорт токена авторизации. (`cookie`, `header`). По умолчанию: `cookie`                  |
| `AUTH_TOKEN_STORAGE`        | Хранилище токена авторизации. (`session`, `cycle`). По умолчанию: `session`                 |
| `SESSION_LIFETIME`          | Время жизни сессии. По умолчанию: `86400`                                                   |
| `SESSION_COOKIE`            | Имя cookie сессии. По умолчанию: `sid`                                                      |
| `VIEW_CACHE`                | Кэш представлений. По умолчанию: `DEBUG !== true`                                           |
| `MAILER_DSN`                | DSN почтового сервиса.                                                                      |
| `MAILER_FROM`               | Отправитель почты. По умолчанию: `Spiral <sendit@local.host>`                               |
| `MAILER_QUEUE_CONNECTION`   | Подключение очереди для почты. По умолчанию: значение из переменной `QUEUE_CONNECTION`      |

### Доступ к переменным окружения

Вы можете получить доступ к переменным окружения, используя `Spiral\Boot\EnvironmentInterface`

```php
use Spiral\Boot\EnvironmentInterface;

final class GithubClient
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {}
    
    public function getAccessToken(): ?string
    {
        return $this->env->get('GITHUB_ACCESS_TOKEN');
    }
}
```

или через короткую функцию `env()`

```php 
return [
    'access_token' => env('GITHUB_ACCESS_TOKEN'),
    // ...
];
```

### Предварительная обработка

Помните, что значения в `.env` будут предварительно обработаны, произойдут следующие изменения:

| Значение | PHP значение |
|----------|--------------|
| true     | true         |
| (true)   | true         |
| false    | false        |
| (false)  | false        |
| null     | null         |
| (null)   | null         |
| empty    | ''           |

> **Примечание**
> Кавычки вокруг строк будут автоматически удалены.

## Конфигурация

Хотя переменные окружения — отличный способ настройки определенных параметров вашего приложения, могут быть случаи, когда вам нужно внести более сложные или детализированные изменения в конфигурацию, которые невозможно или нецелесообразно выполнять только с помощью переменных окружения. В таких случаях вы можете напрямую изменить файлы конфигурации для конкретных компонентов, которые вы хотите изменить.

Например, вы можете изменить HTTP-заголовки по умолчанию:

```php app/config/http.php
return [
    'basePath'   => '/',
    'headers' => [
        'Server' => 'Spiral',
        'Content-Type' => 'text/html; charset=UTF-8'
    ],
    'middleware' => [],
];
```

### Доступ к значениям конфигурации

### Объекты конфигурации

В Spiral все объекты конфигурации являются инъектируемыми, что упрощает их использование в вашем приложении.

```php
use Spiral\Http\Config\HttpConfig;

final class HttpClient 
{
    private readonly string $basePath;

    public function __construct(
        HttpConfig $config // <-- Контейнер автоматически загрузит значения из app/config/http.php
    ) {
        $this->basePath = $this->config->getBasePath();
    }
}
```

Каждый инъектируемый класс конфигурации в Spiral содержит [константу CONFIG](https://github.com/spiral/http/blob/master/src/Config/HttpConfig.php#L19), которая определяет имя соответствующего файла конфигурации. Когда контейнер разрешает инъектируемый объект конфигурации, он автоматически загружает все значения из файла конфигурации и присваивает их свойству `$config` объекта конфигурации.

> **Примечание**
> Когда объект конфигурации загружает связанный с ним файл конфигурации, он автоматически объединяет его с настройками по умолчанию, определенными в классе конфигурации. Это означает, что вам не нужно включать каждый параметр в файл конфигурации, а только те, которые вы хотите изменить. Настройки по умолчанию используются как резервные, если значение отсутствует в файле конфигурации.

Например, для **HTTP конфигурации**:

```php spiral/framework/src/Http/src/Config/HttpConfig.php
final class HttpConfig extends InjectableConfig
{
    const CONFIG = 'http';
    
    // ...
}
```

> **Примечание**
> См. справочную информацию по конфигурации каждого компонента в соответствующем разделе документации.

### Определение окружения приложения

Текущее окружение приложения определяется через переменную `APP_ENV`. Вы можете получить доступ к этому значению, используя инъектируемый класс перечисления `Spiral\Boot\Environment\AppEnvironment`.

> **Узнать больше**
> Подробнее об инъектируемых перечислениях читайте в разделе [Продвинутые — Инжекторы (Injectors) контейнера](../container/injectors.md#enum-injectors).

Когда вы запрашиваете `AppEnvironment` из контейнера, он автоматически внедрит перечисление с правильным значением.

```php
use Spiral\Boot\Environment\AppEnvironment;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly AppEnvironment $env
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->env->isProduction()) {
                // ...
            }
            
            // ...
        }
    }
}
```

### Определение режима отладки

Текущий режим отладки определяется через переменную `DEBUG`. Вы можете получить доступ к этому значению, используя инъектируемый класс перечисления `Spiral\Boot\Environment\DebugMode`.

```php
use Spiral\Boot\Environment\DebugMode;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly DebugMode $debug
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->debug->isEnabled()) {
                // ...
            }
            
            // ...
        }
    }
}
```

### Определение уровня детализации

Текущий уровень детализации определяется через переменную `VERBOSITY_LEVEL`. Вы можете получить доступ к этому значению, используя инъектируемый класс перечисления `Spiral\Exceptions\Verbosity`.

```php
use Spiral\Exceptions\Verbosity;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly Verbosity $verbosity
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->verbosity === Verbosity::BASIC)
                // ...
            }
            
            if ($this->verbosity === Verbosity::VERBOSE)
                // ...
            }
            
            // ...
        }
    }
}
```

<hr>

## Что дальше?

Теперь углубитесь в основы, прочитав следующие статьи:

* [Объекты конфигурации](../framework/config.md)
* [Продвинутые — Инжекторы (Injectors) контейнера](../container/injectors.md)
* [Ядро и окружение](../framework/kernel.md)