# Начало работы — Долгоживущие приложения

Spiral разработан для содействия разработке долгоживущих приложений, обеспечивая при этом эффективное управление памятью и предотвращение утечек памяти. Это достигается за счет использования продвинутых [техник управления памятью](../container/scopes.md). Кроме того, он работает в паре с RoadRunner, что еще больше повышает общую производительность и масштабируемость приложения.

RoadRunner — это высокопроизводительный сервер приложений PHP и менеджер процессов, который обеспечивает возможности долгого выполнения для PHP-приложений. Он обеспечивает эффективное управление ресурсами, такими как использование CPU и памяти, что помогает поддерживать плавную и эффективную работу приложения в течение длительного периода времени.

Он предназначен для обработки широкого спектра типов запросов, включая [HTTP](../http/lifecycle.md), [gRPC](../grpc/configuration.md), TCP, потребление [Queue Job](../queue/roadrunner.md) и Temporal. Он работает, запуская воркеры только один раз при инициализации, а затем направляя запросы к [диспетчеру](../framework/dispatcher.md) в зависимости от их типа. Это означает, что каждый воркер изолирован и работает независимо, следуя подходу "ничего не делить", где ресурсы не разделяются между воркерами.

Использование RoadRunner может значительно улучшить скорость и эффективность, устраняя необходимость для приложения многократно проходить процесс инициализации. Это может сэкономить ресурсы CPU и памяти и сократить время отклика.

> **Узнать больше**
> Читайте больше о симбиозе фреймворка и сервера приложений в разделе [Фреймворк — Жизненный цикл приложения](../framework/lifecycle.md).

## Установка

Использование RoadRunner относительно простое. После загрузки бинарного файла вы можете использовать его для запуска вашего PHP-приложения.

Существует несколько способов его загрузки:

:::: tabs

::: tab Composer

Лучший способ — использовать composer-пакет `spiral/roadrunner-cli`. Он поможет вам автоматически загрузить сервер.

Просто установите пакет в вашем проекте и выполните следующую команду:

```terminal
composer require spiral/roadrunner-cli
```

И выполните следующую команду для загрузки последней версии RoadRunner:

```terminal
./vendor/bin/rr get
```

> **Внимание**
> Для автоматической загрузки RoadRunner требуются расширения PHP `php-curl` и `php-zip`.
:::

::: tab cURL

Загрузите последний стабильный релиз RoadRunner с помощью cURL.

```bash
curl --proto '=https' --tlsv1.2 -sSf  https://raw.githubusercontent.com/roadrunner-server/roadrunner/master/download-latest.sh | sh
```
:::

::: tab Docker

RoadRunner предоставляет предварительно скомпилированные бинарные файлы сервера RoadRunner в Docker-образе.

```docker Dockerfile
FROM spiralscout/roadrunner as roadrunner
# ИЛИ
# FROM ghcr.io/roadrunner-server/roadrunner as roadrunner

FROM php:8.1-cli

# Копируем бинарный файл RoadRunner из образа roadrunner в локальный каталог bin
COPY --from=roadrunner /usr/bin/rr /usr/local/bin/rr

# Запускаем команду сервера RoadRunner
CMD ["rr", "serve"]
```

**Образ доступен на:**

- **Github** — [ghcr.io/roadrunner-server/roadrunner](https://github.com/roadrunner-server/roadrunner/pkgs/container/roadrunner)
- **Docker Hub** — [spiralscout/roadrunner](https://hub.docker.com/r/spiralscout/roadrunner)

:::

::: tab Linux
Вариант установки для производных Debian (Ubuntu, Mint, MX и т.д.)

```bash
wget https://github.com/roadrunner-server/roadrunner/releases/download/v2.X.X/roadrunner-2.X.X-linux-amd64.deb
sudo dpkg -i roadrunner-2.X.X-linux-amd64.deb
```
:::

::: tab Github

Если ни один из других вариантов установки не подходит вам, вы всегда можете загрузить бинарный файл RoadRunner непосредственно с GitHub.

Перейдите к [последнему релизу RoadRunner](https://github.com/roadrunner-server/roadrunner/releases/latest), прокрутите вниз до "Assets" и выберите бинарный файл, соответствующий вашей операционной системе.

:::

::::

## Конфигурация

Вы можете настроить количество воркеров, лимиты памяти и другие плагины, используя файл `.rr.yaml`:

```yaml .rr.yaml
rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: "php app.php"
  relay: pipes

# Настройки HTTP-плагина
http:
  address: 0.0.0.0:8080
  middleware: [ "gzip", "static" ]
  static:
    dir: "public"
    forbid: [ ".php", ".htaccess" ]
  pool:
    num_workers: 2
    supervisor:
      max_worker_memory: 100
```

Чтобы установить количество воркеров для HTTP:

```yaml .rr.yaml
http:
  pool:
    num_workers: 4
```

> **Узнать больше**
> Читайте больше о конфигурации сервера приложений в официальной [документации](https://roadrunner.dev/docs).

## Запуск сервера

:::: tabs

::: tab Linux

Используйте следующую команду для запуска сервера приложений на **Linux**

```terminal
./rr serve
```

> **Внимание**
> Убедитесь, что бинарный файл `rr` является исполняемым.

:::

::: tab Windows
Используйте следующую команду для запуска сервера приложений на **Windows**

```terminal
./rr.exe serve
```

:::

::::

> **Узнать больше**
> Читайте больше о командах сервера в [документации RoadRunner](https://roadrunner.dev/docs/app-server-cli).

## RoadRunner bridge

Пакет [spiral/roadrunner-bridge](https://github.com/spiral/roadrunner-bridge) обеспечивает полную интеграцию между Spiral и RoadRunner. Этот пакет позволяет разработчикам использовать различные плагины RoadRunner, включая `http`, `grpc`, `jobs`, `tcp`, `kv`, `locks`, `centrifugo`, `app-logger` и `metrics`.

> **Примечание**
> Компонент доступен по умолчанию в [пакете приложения](https://github.com/spiral/app).

### Установка

Для установки пакета выполните следующую команду:

```terminal
composer require spiral/roadrunner-bridge
```

После установки вам нужно добавить загрузчики (Bootloader) пакета в ваше приложение в `Kernel`, выбрав конкретные загрузчики (Bootloader), которые соответствуют плагинам, которые вы хотите использовать:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
use Spiral\RoadRunnerBridge\Bootloader as RoadRunnerBridge;

public function defineBootloaders(): array
{
    return [
        RoadRunnerBridge\HttpBootloader::class, // Опционально, если нужно работать с http-плагином
        RoadRunnerBridge\QueueBootloader::class, // Опционально, если нужно работать с jobs-плагином
        RoadRunnerBridge\CacheBootloader::class, // Опционально, если нужно работать с KV-плагином
        RoadRunnerBridge\GRPCBootloader::class, // Опционально, если нужно работать с GRPC-плагином
        RoadRunnerBridge\CommandBootloader::class,
        RoadRunnerBridge\TcpBootloader::class, // Опционально, если нужно работать с TCP-плагином
        RoadRunnerBridge\MetricsBootloader::class, // Опционально, если нужно работать с metrics-плагином
        RoadRunnerBridge\LoggerBootloader::class, // Опционально, если нужно работать с app-logger-плагином
        // ...
    ];
}
```

Читайте больше о загрузчиках (Bootloader) в разделе [Фреймворк — Загрузчики (Bootloader)](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
use Spiral\RoadRunnerBridge\Bootloader as RoadRunnerBridge;

protected const LOAD = [
    RoadRunnerBridge\HttpBootloader::class, // Опционально, если нужно работать с http-плагином
    RoadRunnerBridge\QueueBootloader::class, // Опционально, если нужно работать с jobs-плагином
    RoadRunnerBridge\CacheBootloader::class, // Опционально, если нужно работать с KV-плагином
    RoadRunnerBridge\GRPCBootloader::class, // Опционально, если нужно работать с GRPC-плагином
    RoadRunnerBridge\CommandBootloader::class,
    RoadRunnerBridge\TcpBootloader::class, // Опционально, если нужно работать с TCP-плагином
    RoadRunnerBridge\MetricsBootloader::class, // Опционально, если нужно работать с metrics-плагином
    RoadRunnerBridge\LoggerBootloader::class, // Опционально, если нужно работать с app-logger-плагином
    // ...
];
```

Читайте больше о загрузчиках (Bootloader) в разделе [Фреймворк — Загрузчики (Bootloader)](../framework/bootloaders.md).
:::

::::

## Остерегайтесь подводных камней

Есть несколько ограничений, о которых следует знать при использовании долгоживущих приложений.

### Состояние приложения

При запуске вашего приложения с использованием сервера RoadRunner любые изменения в файлах не повлияют на ваше приложение, поскольку после запуска оно загружается в память. Чтобы увидеть изменения, вам нужно перезапустить сервер. Это может привести к некоторым неудобствам.

Чтобы принудительно перезагружать воркер после каждого запроса (полный режим отладки) и ограничить обработку одним воркером, добавьте опцию `debug`. Это может быть полезно для отладки и целей разработки.

```yaml .rr.yaml
http:
  pool:
    debug: true
```

> **Внимание**
> Важно отметить, что эта функция может повлиять на производительность сервера приложений, поэтому лучше использовать ее только в режиме разработки.

### Утечки памяти

Поскольку приложение остается в памяти долгое время, даже небольшая утечка памяти может привести к перезапуску процесса. RoadRunner отслеживает потребление памяти и выполняет мягкий сброс, но лучше избегать утечек памяти в исходном коде вашего приложения.

> **Примечание**
> Фреймворк включает набор инструментов для упрощения процесса разработки и избежания утечек памяти/состояния, таких как области видимости (Scope) IoC, Cycle ORM, неизменяемые конфигурации, ядра доменов, маршруты и посредники (Middleware).

<hr>

## Что дальше?

Теперь изучите основы более глубоко, прочитав некоторые статьи:

* [Жизненный цикл приложения](../framework/lifecycle.md)
* [Диспетчеры](../framework/dispatcher.md)
* [Финализаторы (Finalizer)](../framework/finalizers.md)
* [Статическая память](../advanced/memory.md)
* [Пользовательский диспетчер](../cookbook/custom-dispatcher.md)
