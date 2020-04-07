# 应用指标

Spiral 应用服务器中内置了 [prometheus](https://prometheus.io/) 服务器，可以利用它来公开一些应用指标。

## 配置

指标服务不需要对应用进行特别的配置。但是必须在 `.rr.yaml` 里启用这个服务：

```yaml
metrics:
  # prometheus 客户端地址（会自动添加 /metrics 路径）
  address: localhost:2112
```

> 可以访问 http://localhost:2112/metrics 查看默认指标

## 自定义应用指标

也可以发布特定于应用的指标。首先在配置文件里注册需要的指标：

```yaml
metrics:
  address: localhost:2112
  collect:
    app_metric_counter:
      type: counter
      help: "Application counter."
```

> 可用的类型有：gauge, counter, summary, histogram.

借助 `Spiral\RoadRunner\MetricsInterface` 可以填充应用中的指标：

```php
use Spiral\RoadRunner\MetricsInterface; 

// ...

public function index(MetricsInterface $metrics)
{
    $metrics->add('app_metric_counter', 1);
}
```

> 在中间件中可以调用 MetricsInterface.

## 标记的指标

可以通过指标的标记（标签）来对值分组：

```yaml
metrics:
  address: localhost:2112
  collect:
    app_type_duration:
      type: histogram
      help: "Application counter."
      labels: ["type"]
```

推送指标的时候需要给值指定标签：

```php
use Spiral\RoadRunner\MetricsInterface; 

// ...

public function index(MetricsInterface $metrics)
{
    $metrics->add('app_type_duration', 0.5, ['some-type']);
}
```
