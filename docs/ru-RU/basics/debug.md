# Основы — Отладка

При разработке долго работающего приложения с использованием Spiral Framework и RoadRunner есть специфические
особенности, которые необходимо учитывать при отладке кода. Вот что вам нужно знать:

## Основные методы отладки

### Избегайте использования функций `die` и `exit`

В традиционной разработке на PHP вы могли привыкнуть использовать функции `die` или `exit` для остановки выполнения скрипта, часто в
сочетании с функцией dump, такой как `var_dump`. Однако в среде Spiral использование `die` или `exit` может сломать
ваше приложение, поскольку это полностью остановит worker RoadRunner, а не только текущий запрос.

Например, использование функции `dd` из пакета `symfony/var-dumper` может вызвать проблемы, поскольку эта функция выводит
содержимое переменной, а затем вызывает `die`, тем самым ломая ваш worker RoadRunner.

### Обработка несоответствия `PHP_SAPI`

RoadRunner не использует традиционный PHP SAPI (Server API), и некоторые dumper построены с предположением, что они
работают в CLI (интерфейс командной строки) среде. Это расхождение может привести к неожиданному поведению при
попытке отладки вашего приложения.

## Spiral Dumper

Мы разработали пакет `spiral/dumper` для решения этих проблем. Этот пакет действует как обёртка
вокруг библиотеки [symfony/var-dumper](https://symfony.com/doc/current/components/var_dumper.html) и позволяет отправлять
дампы переменных непосредственно в браузер в HTTP worker или в выход `STDERR` в других средах. Он
разработан для хорошей работы с долго работающим подходом RoadRunner.

С пакетом `spiral/dumper` разработчики могут легко инспектировать и анализировать значения переменных в процессе разработки.
Этот пакет является бесценным ресурсом для отладки и устранения неполадок как в веб-, так и в CLI приложениях.

### Установка

По умолчанию пакет `spiral/dumper` уже включён в скелет `spiral/app`. Однако, если вы используете
другой скелет, вы можете легко установить пакет, используя следующую команду:

```terminal
composer require --dev spiral/dumper
```

После установки вам нужно добавить bootloader пакета в ваше приложение.

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineSystemBootloaders(): array
{
    return [
        // ...
        \Spiral\Debug\Bootloader\DumperBootloader::class,
    ];
}
```

Читайте больше о bootloader в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const SYSTEM = [
    // ...
    \Spiral\Debug\Bootloader\DumperBootloader::class,
];
```

Читайте больше о bootloader в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

### Использование

Для дампа переменных просто используйте вспомогательную функцию `dump()`, предоставляемую пакетом.

```php
dump($variable);
```

С этим пакетом вы можете использовать функцию `dd` так же, как в традиционном PHP приложении, но без
риска остановки всего worker RoadRunner.

```php
dd($variable);
```

<hr />

## Symfony VarDumper

В качестве альтернативы, для более традиционного подхода к отладке, вы можете выбрать пакет symfony/var-dumper.

Этот пакет предлагает автономный сервер, который собирает все данные дампов. Вы запускаете сервер с помощью команды, и он
будет слушать данные, отправленные функцией dump(). Любые дампы переменных, которые вы отправляете в эту функцию, будут отображены
в отдельном окне консоли, а не в основном выводе вашего приложения.

Вот пример вывода консоли:

```terminal
./vendor/bin/var-dump-server

Symfony Var Dumper Server
=========================
 [OK] Server listening on tcp://127.0.0.1:9912
 // Quit the server with CONTROL-C.

$ app.php
---------
 -------- ---------------------------------------------------------
  date     Fri, 18 Aug 2023 11:54:44 +0000
  source   SimpleController.php on line 36
  file     app/src/Interfaces/Http/Controller/SimpleController.php
 -------- ---------------------------------------------------------
null

 -------- ---------------------------------------------------------
  date     Fri, 18 Aug 2023 11:54:44 +0000
  source   SimpleController.php on line 37
  file     app/src/Interfaces/Http/Controller/SimpleController.php
 -------- ---------------------------------------------------------
App\Service\Site\Site^ {#1260
  -theme: "default"
  -docs: App\Service\Site\Docs^ {#1269
    -defaultVersion: "3.5"
    -defaultLanguage: "en"
  }
  -host: "127.0.0.1"
}
```

### Установка

Для установки пакета выполните следующую команду:

```terminal
composer require --dev symfony/var-dumper
```

### Использование

Для запуска сервера выполните следующую команду:

```terminal
./vendor/bin/var-dump-server
```

Чтобы использовать эту функцию, вам также необходимо определить переменную окружения `VAR_DUMPER_FORMAT` в вашем файле `.env`
следующим образом:

```dotenv .env
VAR_DUMPER_FORMAT=server
```

### Известные проблемы

Если объект имеет множество свойств, или эти свойства содержат существенные данные, вывод консоли может
стать слишком большим. В таких случаях становится крайне сложно просеивать огромное количество текста в
консоли, чтобы найти конкретную информацию, которая вас интересует.

Эта проблема усугубляется при работе со сложными объектами — такими как ORM сущности с множеством связей, или большими
массивами — которые имеют глубокие и широкие структуры. Консоль, будучи линейным и ограниченным по размеру выводом, может с трудом представлять
эту информацию в читаемом и навигируемом виде.

<hr />

## Продвинутая отладка с Buggregator

[Buggregator](https://github.com/buggregator/spiral-app) — это мощное, докеризованное веб-приложение и сервер, разработанное
для значительного улучшения вашего опыта отладки в PHP разработке. Он слушает как TCP, так и HTTP порты, позволяя
ему обрабатывать различные входящие запросы, включая дампы переменных, исключения, логи приложения, SMTP письма и
многое другое.

![var-dumper](https://user-images.githubusercontent.com/773481/208727353-b8201775-c360-410b-b5c8-d83843d388ff.png)

### Ключевые особенности

1. **Перехват и отображение PHP переменных:** Легко интегрируется с инструментами
   такими как [Symfony var-dumper](https://github.com/buggregator/spiral-app#2-symfony-vardumper-server) для перехвата и
   отображения дампов переменных в организованном, читаемом формате.

2. **Обработка исключений:** Может перехватывать и отображать исключения, включая те, которые отправляются платформами отслеживания ошибок, такими как
   [Sentry](https://github.com/buggregator/spiral-app#4-compatible-with-sentry-reports), предоставляя вам чёткое, централизованное
   представление проблем по мере их возникновения.

3. **SMTP Mail Catcher:** Может выступать в роли
   [фиктивного SMTP сервера](https://github.com/buggregator/spiral-app#3-fake-smtp-server-for-catching-mail), перехватывая и
   отображая электронные письма, отправленные вашим приложением во время разработки, так что вы можете просматривать и тестировать электронные письма без фактической их отправки.

4. **Удобный интерфейс:** Предоставляет чистый, интуитивно понятный веб-интерфейс, который организует и отображает ваши данные отладки
   таким образом, чтобы их было легко навигировать и понимать.

5. **Докеризован для лёгкой настройки:** Buggregator упакован как Docker контейнер, что делает его невероятно простым для запуска
   в любой среде разработки.

При работе со сложными объектами с множеством свойств и существенными данными, просеивание вывода консоли или лог-файлов
может быть подавляющим и отнимающим много времени. Buggregator решает эту проблему, представляя эту информацию в
структурированном, сворачиваемом и поисковом веб-интерфейсе. Таким образом, вы можете быстро и эффективно найти именно тот фрагмент
данных, который вам нужен, не прокручивая сотни строк текста.

### Установка

Чтобы начать использовать Buggregator, просто скачайте Docker образ и запустите контейнер:

```bash Latest stable release
docker run --pull always ghcr.io/buggregator/server:latest
    -p 8000:8000 
    -p 1025:1025 
    -p 9912:9912 
    -p 9913:9913 
```

Или, если вы хотите использовать его с docker-compose, добавьте следующий сервис в ваш файл `docker-compose.yaml`:

```yaml docker-compose.yaml
services:
  # ...
  buggregator:
    image: ghcr.io/buggregator/server:latest
    ports:
      - 8000:8000
      - 1025:1025
      - 9912:9912
      - 9913:9913
```

Когда Buggregator запущен, перейдите по адресу http://127.0.0.1:8000 в вашем веб-браузере, чтобы получить доступ к интерфейсу Buggregator
и начать мониторинг данных отладки вашего приложения в реальном времени.

> **Примечание**
> Информацию о настройке вашего приложения для отправки данных в Buggregator можно найти в
> [GitHub репозитории](https://github.com/buggregator/server#features)

### Интеграция XHProf

Buggregator не только служит как всеобъемлющий инструмент для перехвата и отображения данных отладки, но также превосходно показывает себя как
ценный партнёр для профилирования приложений. Он может выступать в роли наблюдателя
для [Xhprof](https://github.com/buggregator/spiral-app#1-xhprof-profiler) профилей, предоставляя разработчикам
интуитивный и эффективный способ анализа данных производительности, выявления узких мест и обнаружения утечек памяти в их
PHP приложениях.

> **Примечание**
> XHProf — это инструмент, который поможет вам выяснить, как работает ваш PHP код и где он может быть медленным. Он отслеживает,
> сколько раз вызываются различные части вашего кода и сколько времени они занимают. Он также может помочь выяснить, сколько
> памяти использует ваш код. Spiral имеет пакет [spiral/profiler](https://github.com/spiral/profiler), который
> упрощает использование XHProf в вашем PHP приложении. Он предоставляет простой и удобный способ использования профилировщика XHProf
> в период разработки или профилирования, так что вы можете быстро выявить и оптимизировать узкие места производительности в вашем
> коде.

![xhprof](https://user-images.githubusercontent.com/773481/208724383-3790a3e1-9ebe-4616-8d4d-d1869f8f2b7c.png)

**Это может быть полезно следующими способами:**

1. **Удобная идентификация узких мест:** Buggregator представляет данные XHProf таким образом, что легко обнаружить
   узкие места производительности. С сортируемыми таблицами и графическими представлениями разработчики могут быстро понять, какие
   части приложения потребляют больше всего времени и ресурсов.

2. **Обнаружение утечек памяти:** Детальное представление профилирования Buggregator помогает разработчикам точно определить пики использования памяти,
   делая его ценным инструментом для выявления и устранения утечек памяти.

#### Установка

Для начала вам нужно установить расширение Xhprof. Один из способов сделать это — использовать пакет PECL.

```terminal
pear channel-update pear.php.net
pecl install xhprof
```

Затем установите пакет профилировщика:

```terminal
composer require --dev spiral/profiler:^3.0
```

После установки пакета добавьте bootloader из пакета в ваше приложение.

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Profiler\ProfilerBootloader::class,
        // ...
    ];
}
```

Читайте больше о bootloader в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Profiler\ProfilerBootloader::class,
    // ...
];
```

Читайте больше о bootloader в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

#### Конфигурация

Используйте следующие переменные окружения для настройки профилировщика для отправки данных на
сервер [Buggregator](https://github.com/buggregator/spiral-app):

```dotenv .env
PROFILER_ENDPOINT=http://127.0.0.1:8000/api/profiler/store
PROFILER_APP_NAME=My super app
```

#### Использование

Есть два способа использования профилировщика:

- Профилировщик как перехватчик (Interceptor)
- Профилировщик как промежуточное ПО (Middleware)

#### Профилировщик как перехватчик (Interceptor)

Interceptor будет полезен, если вы хотите профилировать какую-то определённую часть вашего приложения, которая поддерживает использование
interceptor.

- [Контроллеры](../http/interceptors.md),
- [GRPC](../grpc/interceptors.md),
- [Задачи очереди](../queue/interceptors.md).
- TCP
- [События](../advanced/events.md#interceptors).

> **Читайте больше**
> Читайте больше об interceptor в разделе [Framework — Interceptors](../framework/interceptors.md).

Чтобы использовать профилировщик как interceptor, вам просто нужно зарегистрировать класс `Spiral\Profiler\ProfilerInterceptor`.

Вот пример того, как использовать профилировщик как interceptor в HTTP слое:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        \Spiral\Profiler\ProfilerInterceptor::class
    ];
}
```

#### Профилировщик как промежуточное ПО (Middleware)

Middleware будет полезно, если вы хотите профилировать все запросы к вашему приложению. Чтобы использовать профилировщик как middleware, вам
нужно добавить его в ваш роутер.

> **Читайте больше**
> Читайте больше о middleware в разделе [HTTP — Routing](../http/routing.md).

##### Глобальное промежуточное ПО

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Profiler\ProfilerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            ProfilerMiddleware::class,  // <================
            // ...
        ];
    }
    
    // ...
}
```

##### Промежуточное ПО группы маршрутов

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Profiler\ProfilerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                // ...
            ],
            'profiler' => [                  // <================
                ProfilerMiddleware::class,
                'middleware:web',
            ],
        ];
    }
    
    // ...
}
```

##### Промежуточное ПО маршрута

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Router\Annotation\Route;

final class UserController
{
    #[Route(route: '/users', name: 'user.store', methods: ['POST'], middleware: \Spiral\Profiler\ProfilerMiddleware::class)]
    public function store(...): void 
    {
        // ...
    }
}
```

<hr />

## XDebug

Отладка приложения Spiral так же возможна, как отладка любого другого классического PHP приложения при использовании расширения xDebug.

### Конфигурация IDE

Прежде всего, вам нужно настроить вашу IDE для работы с xDebug.

> **Читайте больше**
> Читайте больше о конфигурации IDE в
> официальной [документации](https://roadrunner.dev/docs/php-debugging).

### По требованию

Удобнее запускать RoadRunner с включённым xDebug только при необходимости. Добавьте следующие переменные окружения в
`.rr.yaml` для правильной настройки xDebug:

```yaml .rr.yaml
env:
  PHP_IDE_CONFIG: serverName=application.loc
  XDEBUG_CONFIG: remote_host=localhost max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
```

> **Примечание**
> Измените значения согласно вашей среде.

Для включения xDebug запустите сервер приложения с флагом `-o` (overwrite flag) для требуемого сервиса:

```terminal
./rr serve -o "server.command=php -d zend_extension=xdebug app.php"
```

### В Docker

Чтобы изменить конфигурацию worker в docker, используйте следующую или похожую конфигурацию для вашего контейнера:

```yaml docker-compose.yaml
version: "2"
services:
  ...
  app:
    ...
    command:
      - /usr/local/bin/rr
      - serve
      - -o
      - server.command=php -d zend_extension=xdebug.so app.php
    environment:
      PHP_IDE_CONFIG: serverName=application.loc
      XDEBUG_CONFIG: remote_host=host.docker.internal max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
```