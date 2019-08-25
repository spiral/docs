# Application Metrics
You can expose some of application metrics using Prometheus service embedded to RoadRunner application server.

## Configuration
Metrics service does not require configuration in application, however, you must active this service in `.rr.yaml`:

```yaml
metrics:
  # prometheus client address (path /metrics added automatically)
  address: localhost:2112
```

> You can view defaults metrics on http://localhost:2112/metrics

## Custom Application metrics
You can also publish application specific metrics. First you have to register metric in your configuration file:

```yaml
metrics:
  address: localhost:2112
  collect:
    app_metric_counter:
      type: counter
      help: "Application counter."
```

> Supported types: gauge, counter, summary, histogram.

To populate metric from application use `Spiral\RoadRunner\MetricsInterface`:

```php
use Spiral\RoadRunner\MetricsInterface; 

// ...

public function index(MetricsInterface $metrics)
{
    $metrics->add('app_metric_counter', 1);
}
```

> You can call MetricsInterface in middleware.

## Tagged metrics
You can use tagged (labels) metrics to group values:

```yaml
metrics:
  address: localhost:2112
  collect:
    app_type_duration:
      type: histogram
      help: "Application counter."
      labels: ["type"]
```

You should specify values for your labels while pushing the metric:

```php
use Spiral\RoadRunner\MetricsInterface; 

// ...

public function index(MetricsInterface $metrics)
{
    $metrics->add('app_type_duration', 0.5, ['some-type']);
}
```