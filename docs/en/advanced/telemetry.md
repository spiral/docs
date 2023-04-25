# Advanced — Application Telemetry

Spiral is a powerful tool for building microservices. One of its key features is the `spiral/telemetry`
component, which enables you to collect and send application metrics to a telemetry server or logs. This component
provides a flexible and robust solution for gathering performance data and monitoring your microservices.

![OpenTelemetry](https://user-images.githubusercontent.com/773481/213914208-cd944ca8-f218-4baf-8a54-5a4e42a1ed40.jpg)

The collected traces can then be sent to a third-party service for rendering, providing a clear and detailed
visualization of the performance of your microservices.

## Example of usage

When a customer places an order on the website, the request would be traced from the frontend service, through the order
processing service, to the inventory management service, and finally to the shipping service.

You could use the **trace ID** to correlate all the traces related to the same request, so you can see the entire
path of the request, and how it was handled by each service. With that data, you can monitor the execution time, and
if there is any latency in any of the service, they can investigate further by looking at the trace of each service. You
can also monitor the number of requests each service is handling and see if any service is overloaded or underutilized.

In addition, you could also trace database queries to identify slow-performing queries that are impacting the overall
performance of the system. And also, you can trace external service calls to identify any issues with third-party APIs
that the platform relies on.

## Open Telemetry integration

By default, the component uses a `null` driver and does not perform any action. However, it also offers integration
with the [OpenTelemetry](https://opentelemetry.io/) service through
the [spiral/otel-bridge](https://github.com/spiral/otel-bridge) package. This allows you to trace requests as they move
through your microservices, using a **trace ID** that is passed along through headers from one service to the next.
This enables you to gain a comprehensive understanding of how requests are processed, and how different microservices
are interacting.

### Installation

To install the `spiral/otel-bridge` package, you can use the following command:

```terminal
composer require spiral/otel-bridge open-telemetry/exporter-otlp
```

> **Note**
> In our example, we are using the `open-telemetry/exporter-otlp` package to send traces to the OpenTelemetry collector.
> If you want to use a different exporter, you can check
> [Exporters section](https://opentelemetry.io/docs/instrumentation/php/exporters/) in the OpenTelemetry documentation.

After installing the package, you need to register the bootloader in your application's kernel:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\OpenTelemetry\Bootloader\OpenTelemetryBootloader::class,
    // ...
];
```

### Configuration

To fully configure the package, you will need to update your application's `.env` file with the appropriate settings.

```dotenv .env
# Telemetry driver [log, null, otel]
TELEMETRY_DRIVER=otel

# OpenTelemetry
OTEL_SERVICE_NAME=php # Your application name
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318
OTEL_PHP_TRACES_PROCESSOR=simple
```

> **See more**
> You can find more information about the configuration options in 
> the [OpenTelemetry documentation](https://opentelemetry.io/docs/concepts/sdk-configuration/general-sdk-configuration/).

To run the [OpenTelemetry collector](https://opentelemetry.io/docs/collector/) server and
the [Zipkin](https://zipkin.io/) tracing system, you can use the example `docker-compose.yaml` file provided:

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

and `otel-collector-config.yml` config file

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

You should also configure RoadRunner to send traces to the OpenTelemetry collector server:

```yaml .rr.yaml
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

This will enable the integration and allow you to start tracing requests through your application using the
OpenTelemetry service.

#### Monolog integration

The component does not require any specific configuration within the application, but it offers the ability to configure
the Monolog to add trace context to log messages. This can be done by adding the
`\Spiral\Telemetry\Monolog\TelemetryProcessor::class` as a processor in the `monolog.php` configuration file.

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

This allows the **trace ID** to be stored with the log information. This makes it easier to find the trace for a
specific log and investigate issues as it allows to correlate log and trace data.

## Usage

The component provides a `Spiral\Telemetry\TracerInterface` interface which can be used to send traces to the collector.

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
        SpanInterface $span,
        HttpClientInterface $httpClient
    ) use($url): string {
        // The code inside the callback will be executed in the span context and information about the span will be
        // sent to the collector
        
        $response = $httpClient->get($url);
        
        // Attributes that will be added to the span object
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

The `trace` method is called with the following parameters:

- `name` - The name of the span. This name will be used to identify the span in the trace.
- `callback` - The callback that will be executed in the span context. The callback will receive the current span object
  and the container as parameters. The callback can return any value. You can use dependency injection in the
  callback function, which can be useful for injecting services or other dependencies that your function needs to
  execute.
- `attributes` - The attributes that will be added to the span object.
- `scoped` - If `true`, all spans inside the callback will be related to the current span.
- `traceKind` - a constant indicating the kind of the span (client, server, etc).

The `SpanInterface` is passed as an argument to the callback function can be used to manipulate the current span:

- `updateName(string $name)`: updates the name of the current span`
- `setStatus(string|int $code, string $description = null)`: sets a status for the current span
- `setAttributes(array $attributes)`: sets the current span attributes
- `setAttribute(string $name, mixed $value)`: sets an attribute on the current span

These methods can be used to add more information to the span, such as attributes, status, and update the name of the
span. This allows you to add more context to the trace and have more information about the execution of the code inside
the callback function.

### Send trace context

The trace context is a set of key-value pairs that contain information about the current trace, such as the trace ID,
span ID and other attributes. This context is used to link together multiple spans that make up a trace.

When you want to send the trace context to another application, you can get it from
the `Spiral\Telemetry\TracerInterface` by calling the `getContext()` method. This method returns an associative array
of the trace context. You can then loop through the context and add the key-value pairs as headers to the response
which is being sent to another application.

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

This allows the other application to access the trace context and link it to the trace that the request is part of. It
makes it possible to trace requests across different services, which can be helpful in understanding the flow of
requests and identifying issues.

### Create trace from context

When you have the trace context from another application and you want to create a trace based on it, you can use the
`Spiral\Telemetry\TracerFactoryInterface`. This interface provides a createTracer method, which takes an array of
context key-value pairs and returns an instance of `Spiral\Telemetry\TracerInterface`. This instance can be used to
create new spans and link them to the trace that the context belongs to.

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

## Example Application

There is a good example [**Demo ticket booking system**](https://github.com/spiral/ticket-booking) application built
on the Spiral Framework, that follows the principles of microservices and allows developers to create reusable,
independent, and easy-to-maintain components.

In this demo application, you can find an example of using OpenTelemetry.

Overall, it is a great example of how Spiral and other tools can be used to build a modern and efficient application.
We hope you have a blast using it and learning more about the capabilities of Spiral and the other tools we've used.

**Happy (fake) ticket shopping!**