# Application Profiling
Spiral does not provide deep profiling functionality for your application, use tools like xDebug or BlackFire for such purposes.

Instead, framework shipped with simple benchmark class used to capture information about time consuming tasks.

> All SQL queries, Storage operations and view rendering/compilation utilizes benchmarking.

## User Benchmarking
You can connect benchmark functionality to your controller or service by either requesting `BenchmarkerInterface` or via `BenchmarkTrait`:

```php
use BenchmarkTrait;

public function indexAction()
{
    $b = $this->benchmark('long-operation');
    sleep(2);
    $this->benchmark($b);
}
```

## Visualization
Visualization of collected benchmarks if specific to implementation you used, under default application such timeline can be rendered using profiler module:

![Profiler](https://raw.githubusercontent.com/spiral/guide/09branch/resources/profiler.png)

> Note that running your application enabled profiling will slow down execution.