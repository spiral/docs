# Advanced — Application Metrics

As a professional, you know the importance of keeping track of key metrics for your application.
With [Prometheus](https://prometheus.io/), you can collect and store time series data such as application metrics, and
use its powerful
query language to analyze and visualize that data in real-time using a tool like [Grafana](https://grafana.com/) saving
you the time and effort of building your own dashboard from scratch.

![Grafana dashboard](https://user-images.githubusercontent.com/773481/205066017-ecddefc4-1d07-4428-b3ad-af49baadad0a.png)

Spiral and [RoadRunner metrics](https://roadrunner.dev/docs/plugins-metrics) plugin provide the ability to collect
application metrics and expose them to Prometheus.

> **Note**
> Here you can find out more about [Prometheus metrics](https://prometheus.io/docs/concepts/data_model/).

## Installation

At first, you need to install the [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) package.

> **Note**
> The `spiral/roadrunner-bridge` package allows you to use RoadRunner
> [metrics plugin](https://roadrunner.dev/docs/lab-metrics) with Spiral. This package provides RPC api for metrics
> and a bootloader for your application.

Once the package is installed, you can add the `Spiral\RoadRunnerBridge\Bootloader\MetricsBootloader` to the list of
bootloaders:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\MetricsBootloader::class,
    // ...
];
```

## Configuration

The metrics service does not require configuration in the application. However, you must activate the service
in `.rr.yaml`:

```yaml
rpc:
  listen: tcp://127.0.0.1:6001

# ...

metrics:
  # prometheus client address (path /metrics added automatically)
  address: 127.0.0.1:2112
```

> **Note**
> You can view defaults metrics on http://127.0.0.1:2112

## Usage

### Application metrics declaration

There are two ways to declare application-specific metrics in Spiral application:

:::: tabs
::: tab RoadRunner
Using the `.rr.yaml` file:

```yaml .rr.yaml
metrics:
  address: 127.0.0.1:2112

  collect:
    registered_users:
      type: counter
      help: "Total registered users counter."
```

:::

::: tab PHP
Declare metrics in PHP code

```php app/src/Application/Bootloader/MetricsBootloader.php
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Metrics\Collector;

class MetricsBootloader extends Bootloader
{
    //...

    public function boot(MetricsInterface $metrics): void
    {
        $metrics->declare(
            'registered_users',
            Collector::counter()->withHelp('Total registered users counter.')
        );
    }
}
```

:::

::::

To populate a metric from the application, use `Spiral\RoadRunner\Metrics\MetricsInterface`:

```php
use Spiral\RoadRunner\Metrics\MetricsInterface; 

class UserRegistrationHandler
{
    public function __construct(
        private readonly MetricsInterface $metrics
    ) {
    }

    public function handle(User $user): void
    {
        // Store user in database

        $this->metrics->add('registered_users', 1);
    }
}
```

> **See more**
> Supported types: gauge, counter, summary, histogram. Read more about metrics types in
> the [official Prometheus documentation](https://prometheus.io/docs/concepts/metric_types/).

### Tagged metrics

Using tagged (also known as labeled) metrics allows you to attach additional metadata to your metrics, which can be
useful for filtering, grouping, and aggregating the data.

**Some benefits of using labeled metrics include:**

- **Increased granularity**: You can attach multiple labels to a metric, allowing you to slice and dice the data in
  various ways.
- **Better organization**: Labels can help you group and organize your metrics, making it easier to find and understand
  the data you are looking for.
- **Simplified querying**: You can use labels to filter and aggregate your metric data, making it easier to extract
  meaningful insights from the data.

:::: tabs
::: tab RoadRunner
Using the `.rr.yaml` file:

```yaml .rr.yaml
metrics:
  address: 127.0.0.1:2112

  collect:
    registered_users:
      type: histogram
      help: "Total registered users counter."
      labels: [ "type" ]
```

:::

::: tab PHP
Declare metrics in PHP code

```php app/src/Application/Bootloader/MetricsBootloader.php
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Metrics\Collector;

class MetricsBootloader extends Bootloader
{
    //...

    public function boot(MetricsInterface $metrics): void
    {
        $metrics->declare(
            'registered_users',
            Collector::counter()->withHelp('Total registered users counter.')->withLabels('type')
        );
    }
}
```

:::

::::

In the example, the `registered_users` metric is declared with a label called `type`. When adding data to the
metric, you can specify the value for the type label, such as `customer`, `admin`, etc. This allows you to differentiate
between different types of users when analyzing the metric data.

```php
use Spiral\RoadRunner\Metrics\MetricsInterface; 

class UserRegistrationHandler
{
    public function __construct(
        private readonly MetricsInterface $metrics
    ) {
    }

    public function handle(User $user): void
    {
        // Store user in database

        $this->metrics->add('registered_users', 1, ['customer']);
        
        // or
        
        $this->metrics->add('registered_users', 1, ['admin']);
    }
}
```

## Example Application

There is a good example [**Demo ticket booking system**](https://github.com/spiral/ticket-booking) application built
on Spiral Framework, that follows the principles of microservices and allows developers to create reusable,
independent, and easy-to-maintain components.

In this demo application, you can find an example of using RoadRunner's metrics plugin.

Overall, our demo ticket booking system is a great example of how Spiral and other tools can be used to build
a modern and efficient application. We hope you have a blast using it and learning more about the capabilities of
our framework and the other tools we've used.

**Happy (fake) ticket shopping!**