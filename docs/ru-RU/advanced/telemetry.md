# Продвинутое — Телеметрия приложения

Spiral — это мощный инструмент для создания микросервисов. Одной из его ключевых особенностей является компонент `spiral/telemetry`, который позволяет собирать и отправлять метрики приложения на сервер телеметрии или в логи. Этот компонент предоставляет гибкое и надежное решение для сбора данных о производительности и мониторинга ваших микросервисов.

![OpenTelemetry](https://user-images.githubusercontent.com/773481/213914208-cd944ca8-f218-4baf-8a54-5a4e42a1ed40.jpg)

Собранные трейсы могут быть отправлены в стороннюю службу для рендеринга, обеспечивая четкую и детальную визуализацию производительности ваших микросервисов.

## Пример использования

Когда клиент размещает заказ на веб-сайте, запрос будет отслеживаться от фронтенд-сервиса через сервис обработки заказов к сервису управления запасами и, наконец, к сервису доставки.

Вы можете использовать **идентификатор трейса** для связывания всех трейсов, относящихся к одному запросу, чтобы видеть весь путь запроса и то, как он обрабатывался каждым сервисом. С этими данными вы можете отслеживать время выполнения, и если есть какая-либо задержка в любом из сервисов, можете провести дальнейшее расследование, изучив трейс каждого сервиса. Вы также можете отслеживать количество запросов, обрабатываемых каждым сервисом, и видеть, перегружен ли какой-либо сервис или недоиспользуется.

Кроме того, вы также можете отслеживать запросы к базе данных для выявления медленно выполняющихся запросов, которые влияют на общую производительность системы. А также вы можете отслеживать вызовы внешних сервисов для выявления любых проблем со сторонними API, на которые полагается платформа.

## Интеграция с Open Telemetry

По умолчанию компонент использует драйвер `null` и не выполняет никаких действий. Однако он также предлагает интеграцию с сервисом [OpenTelemetry](https://opentelemetry.io/) через пакет [spiral/otel-bridge](https://github.com/spiral/otel-bridge). Это позволяет отслеживать запросы по мере их прохождения через ваши микросервисы, используя **идентификатор трейса**, который передается через заголовки от одного сервиса к другому. Это дает вам возможность получить полное понимание того, как обрабатываются запросы и как взаимодействуют различные микросервисы.

### Установка

Для установки пакета `spiral/otel-bridge` вы можете использовать следующую команду:

```terminal
composer require spiral/otel-bridge open-telemetry/exporter-otlp
```

> **Примечание**
> В нашем примере мы используем пакет `open-telemetry/exporter-otlp` для отправки трейсов в коллектор OpenTelemetry.
> Если вы хотите использовать другой экспортер, вы можете ознакомиться с [разделом "Экспортеры"](https://opentelemetry.io/docs/instrumentation/php/exporters/) в документации OpenTelemetry.

После установки пакета вам необходимо зарегистрировать загрузчик в ядре вашего приложения:

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\OpenTelemetry\Bootloader\OpenTelemetryBootloader::class,
        // ...
    ];
}
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\OpenTelemetry\Bootloader\OpenTelemetryBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

### Конфигурация

Для полной настройки пакета вам потребуется обновить файл `.env` вашего приложения с соответствующими параметрами.

```dotenv .env
# Драйвер телеметрии [log, null, otel]
TELEMETRY_DRIVER=otel

# OpenTelemetry
OTEL_SERVICE_NAME=php # Имя вашего приложения
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318
OTEL_PHP_TRACES_PROCESSOR=simple
```

> **Узнать больше**
> Вы можете найти дополнительную информацию о параметрах конфигурации в 
> [документации OpenTelemetry](https://opentelemetry.io/docs/concepts/sdk-configuration/general-sdk-configuration/).

Для запуска сервера [коллектора OpenTelemetry](https://opentelemetry.io/docs/collector/) и системы трассировки [Zipkin](https://zipkin.io/) вы можете использовать предоставленный пример файла `docker-compose.yaml`:

```yaml docker-compose.yaml
version: "3.6"

services:
  collector:
    image: otel/opentelemetry-collector-contrib
    command: [ "--config=/etc/otel-collector-config.yml" ]
    volumes:
      - ./otel-collector-config.yml:/etc/otel-collector-config.yml
    ports:
      - "4318:4318"

  zipkin:
    image: openzipkin/zipkin-slim
    ports:
      - "9411:9411"
```

и конфигурационный файл `otel-collector-config.yml`

```yaml otel-collector-config.yml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:
    timeout: 1s

exporters:
  logging:
    loglevel: debug

  zipkin:
    endpoint: "http://zipkin:9411/api/v2/spans"

  datadog:
    api:
      site: datadoghq.eu
      key: # ваш API ключ datadog

  otlp:
    endpoint: https://otlp.eu01.nr-data.net:443
    headers:
      api-key: # ваш API ключ new relic

service:
  pipelines:
    traces:
      receivers: [ otlp ]
      processors: [ batch ]
      # Здесь вы можете настроить экспортеры, куда хотите отправлять трейсы
      exporters: [ zipkin, datadog, otlp, logging ]
```

Вам также следует настроить RoadRunner для отправки трейсов на сервер коллектора OpenTelemetry:

```yaml .rr.yaml
http:
  address: 0.0.0.0:8080
  middleware: [ "otel" ]
  otel:
    insecure: true
    compress: false
    client: http
    exporter: otlp
    service_name: rr-blog # имя вашего приложения
    service_version: 1.0.0 # версия вашего приложения
    endpoint: 127.0.0.1:4318 # адрес сервера коллектора otel
```

Это включит интеграцию и позволит вам начать трассировку запросов через ваше приложение с использованием сервиса OpenTelemetry.

#### Интеграция с Monolog

Компонент не требует специальной конфигурации в приложении, но предоставляет возможность настроить Monolog для добавления контекста трейса к сообщениям логов. Это можно сделать, добавив `\Spiral\Telemetry\Monolog\TelemetryProcessor::class` в качестве процессора в конфигурационном файле `monolog.php`.

```php app/config/monolog.php
return [
    ...

    'processors' => [
        'default' => [
            \Spiral\Telemetry\Monolog\TelemetryProcessor::class,
        ],
    ],
];
```

Это позволяет сохранить **идентификатор трейса** вместе с информацией лога. Это упрощает поиск трейса для конкретного лога и расследование проблем, так как позволяет связать данные логов и трейсов.

## Использование

Компонент предоставляет интерфейс `Spiral\Telemetry\TracerInterface`, который можно использовать для отправки трейсов в коллектор.

Пример использования:

```php
use Spiral\Telemetry\TracerInterface;
use Spiral\Telemetry\TraceKind;
use Spiral\Telemetry\SpanInterface;

$tracer = $this->container->get(TracerInterface::class);
$url = 'https://example.com';

$result = $tracer->trace(
    name: 'some.function'
    callback: static function(
        SpanInterface $span,
        HttpClientInterface $httpClient
    ) use($url): string {
        // Код внутри callback будет выполнен в контексте span'а и информация о span'е будет
        // отправлена в коллектор
        
        $response = $httpClient->get($url);
        
        // Атрибуты, которые будут добавлены к объекту span'а
        $span->setAttribute('http.response.code', $response->getStatusCode());
        $span->setAttribute('http.response.length', \strlen($response->getContent()));
        
        return $response->getContent();
    },
    attributes: [
        'http.url' => $url,
    ],
    scoped: true,
    traceKind: TraceKind::CLIENT,
);
```

Метод `trace` вызывается со следующими параметрами:

- `name` - Имя спана. Это имя будет использоваться для идентификации спана в трейсе.
- `callback` - Callback-функция, которая будет выполнена в контексте спана. Callback получит текущий объект спана и контейнер в качестве параметров. Callback может возвращать любое значение. Вы можете использовать внедрение зависимостей в callback-функции, что может быть полезно для внедрения сервисов или других зависимостей, необходимых вашей функции для выполнения.
- `attributes` - Атрибуты, которые будут добавлены к объекту спана.
- `scoped` - Если `true`, все спаны внутри callback будут связаны с текущим спаном.
- `traceKind` - константа, указывающая тип спана (client, server и т.д.).

`SpanInterface`, передаваемый в качестве аргумента callback-функции, может использоваться для манипуляции текущим спаном:

- `updateName(string $name)`: обновляет имя текущего спана
- `setStatus(string|int $code, string $description = null)`: устанавливает статус для текущего спана
- `setAttributes(array $attributes)`: устанавливает атрибуты текущего спана
- `setAttribute(string $name, mixed $value)`: устанавливает атрибут текущего спана

Эти методы можно использовать для добавления дополнительной информации к спану, такой как атрибуты, статус, и обновления имени спана. Это позволяет добавить больше контекста к трейсу и получить больше информации о выполнении кода внутри callback-функции.

### Отправка контекста трейса

Контекст трейса — это набор пар ключ-значение, содержащих информацию о текущем трейсе, такую как идентификатор трейса, идентификатор спана и другие атрибуты. Этот контекст используется для связывания нескольких спанов, составляющих трейс.

Когда вы хотите отправить контекст трейса в другое приложение, вы можете получить его из `Spiral\Telemetry\TracerInterface`, вызвав метод `getContext()`. Этот метод возвращает ассоциативный массив контекста трейса. Затем вы можете перебрать контекст и добавить пары ключ-значение в качестве заголовков к ответу, который отправляется в другое приложение.

```php
public function handle(ServerRequestInterface $request): ResponseInterface
{
    $response = $responseFactory->createResponse();

    $tracer = $this->container->get(TracerInterface::class);
    
    foreach ($tracer->getContext() as $key => $value) {
        $response = $response->withHeader($key, $value);
    }
    
    return $response;
}
```

Это позволяет другому приложению получить доступ к контексту трейса и связать его с трейсом, частью которого является запрос. Это делает возможным отслеживание запросов между различными сервисами, что может быть полезно для понимания потока запросов и выявления проблем.

### Создание трейса из контекста

Когда у вас есть контекст трейса от другого приложения и вы хотите создать трейс на его основе, вы можете использовать `Spiral\Telemetry\TracerFactoryInterface`. Этот интерфейс предоставляет метод createTracer, который принимает массив пар ключ-значение контекста и возвращает экземпляр `Spiral\Telemetry\TracerInterface`. Этот экземпляр может быть использован для создания новых спанов и связывания их с трейсом, к которому принадлежит контекст.

```php
public function handle(ServerRequestInterface $request): ResponseInterface
{
    $tracerFactory = $this->container->get(\Spiral\Telemetry\TracerFactoryInterface::class);
    $tracer = $tracerFactory->make($request->getHeaders());
    
   $response = $tracer->trace(
        name: \sprintf('%s %s', $request->getMethod(), (string)$request->getUri()),
        callback: $callback,
        attributes: [
            'http.method' => $request->getMethod(),
            'http.url' => $request->getUri(),
            'http.headers' => $request->getHeaders(),
        ],
        scoped: true,
        traceKind: TraceKind::SERVER
    );
    
    ...
}
```

## Пример приложения

Есть хороший пример приложения [**Демо система бронирования билетов**](https://github.com/spiral/ticket-booking), построенного на Spiral Framework, который следует принципам микросервисов и позволяет разработчикам создавать переиспользуемые, независимые и легко поддерживаемые компоненты.

В этом демо-приложении вы можете найти пример использования OpenTelemetry.

В целом, это отличный пример того, как Spiral и другие инструменты могут быть использованы для создания современного и эффективного приложения. Мы надеемся, что вы получите удовольствие от его использования и узнаете больше о возможностях Spiral и других инструментов, которые мы использовали.

**Удачных (фиктивных) покупок билетов!**