# Application telemetry

The telemetry service allows you to collect and send application metrics to the telemetry server or to the logs. By
default, it uses `null` driver and do nothing.

## Installation

To enable the component, you just need to add `Spiral\Telemetry\Bootloader\TelemetryBootloader` to the
top of bootloader's `LOAD` section.

There are two drivers out of the box:

- `null` - Allows you to totally disable the component.
- `log` - Allows you to send metrics to the log stream.

## Configuration

The telemetry service does not require configuration in the application.
However, there is an ability to configure `Monolog` to add trace context to the log messages.

```php
// config/monolog.php
<?php

return [
    ...

    /**
     * Processors allows adding extra data for all records.
     * @see https://github.com/Seldaek/monolog/blob/main/doc/02-handlers-formatters-processors.md#processors
     */
    'processors' => [
        'default' => [
            \Spiral\Telemetry\Monolog\TelemetryProcessor::class,
        ],
    ],
];
```

## Usage

The component provides a `Spiral\Telemetry\TracerInterface` interface. You can use it to send traces to the driver
collector.

Example of usage:

```php
use Spiral\Telemetry\TracerInterface;
use Spiral\Telemetry\TraceKind;
use Spiral\Telemetry\SpanInterface;

$tracer = $this->container->get(TracerInterface::class);
$url = 'https://example.com';

$result = $tracer->trace(
    name: 'some.function'
    callback: static function(
        SpanInterface $span, // Current span object
        HttpClientInterface $httpClient // You can use autowiring here
    ) use($url): string {
        // The code inside the callback will be executed in the span context and information about the span will be
        // sent to the driver collector
        
        $response = $httpClient->get($url);
        
        // Attributes that will be added to the span object
        $span->setAttribute('http.response.code', $response->getStatusCode());
        $span->setAttribute('http.response.length', \strlen($response->getContent()));
        
        return $response->getContent();
    },
    attributes: [ // Span attributes
        'http.url' => $url,
    ],
    scoped: true, // If true, all spans inside the callback will be related to the current span
    traceKind: TraceKind::CLIENT, // Span kind
);
```

### Send trace context

If you want to send trace context to another application, you can get it from `Spiral\Telemetry\TracerInterface`.

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

### Create trace from context

If you want to create trace from context, you can use `Spiral\Telemetry\TracerFactoryInterface`.

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

# Open Telemetry integration

![OpenTelemetry](https://user-images.githubusercontent.com/773481/202469153-f0b6458c-535c-4cb7-8570-bab8c27abd29.png)


Integration with OpenTelemetry is available via [spiral/otel-bridge](https://github.com/spiral/otel-bridge) package.

## Installation

You can install the package via composer:

```bash
composer require spiral/otel-bridge
```

After package install you need to register bootloader from the package.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\OpenTelemetry\Bootloader\OpenTelemetryBootloader::class,
    // ...
];
```

## Configuration

You need to configure the package via `.env` file:

```dotenv
# Telemetry driver [log, null, otel]
TELEMETRY_DRIVER=otel

# OpenTelemetry
OTEL_SERVICE_NAME=php # Your application name
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318
OTEL_PHP_TRACES_PROCESSOR=simple
```

You can run OpenTelemetry collector server and Zipkin tracing system via docker by using the example below:

```yaml
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

and `otel-collector-config.yml` config file

```yaml
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
      key: # your datadog api key

  otlp:
    endpoint: https://otlp.eu01.nr-data.net:443
    headers:
      api-key: # your new relic api key

service:
  pipelines:
    traces:
      receivers: [ otlp ]
      processors: [ batch ]
      # Here you can set exporters where you want to send traces
      exporters: [ zipkin, datadog, otlp, logging ]
```

You can also configure RoadRunner to send traces to the OpenTelemetry collector server:

```yaml
http:
  address: 0.0.0.0:8080
  middleware: [ "otel" ]
  otel:
    insecure: true
    compress: false
    client: http
    exporter: otlp
    service_name: rr-blog # your app name
    service_version: 1.0.0 # your app version
    endpoint: 127.0.0.1:4318 # otel collector server address
```
