# Profiler Panel
The Skeleton spiral application ships with `spiral/profiler` module to help you profile and debug internal flow of your application.

## Enable/Disable Profiler panel
Profiler module can be enable and disabled by simply booting `ProfilerBootloader` at beginning of your application. By default, such boot will only happen in DEBUG mode defined by the enviroment:


```php
/**
 * Application core bootloading, you can configure your environment here.
 */
protected function bootstrap()
{
    env('DEBUG') && $this->enableDebugging();
}

/**
 * Debug packages.
 */
private function enableDebugging()
{
    //Initiating all needed binding (no need to use memory caching)
    $this->getBootloader()->bootload([
        \Spiral\Profiler\ProfilerBootloader::class,
        \Spiral\Snapshotter\Bootloaders\FileSnapshotterBootloader::class
    ]);
}
```

> This code is located in your App class.

Once booted profiler will create floating panel at bottom right of your screen:

![Profiler Panel](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/panel.png)

## Environment Overview
First profiler panel will show what classes, extensions being loaded. Indicate container bindings, http routes and ect.

![Environment](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/environment.png)

## Request/Response Data
View incoming request and response data on second tab.

![Variables](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/variables.png)

> Please note, the currently profiler indicates state of response before it's being processed by application middlewares.

## Timeline
Profiler is able to indicate your application timeline using `BenchmarkerInterface` as source of data:

![Benchmarks](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/benchmarks.png)

> Read about benchmarker [here](/debug/profiling.md).

## Logging
Every use of LoggerTrait or logger created via `LogsInterface` (including global `LoggerInterface` instance) will be captured on this tab:

![Logging](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/logging.png)


## Performance Notes
Please note that profiler is executed in a same process as your application, keeping profiler enabled will slow down your application a lot.

> Consider alternative profiling techniques like [xDebug](https://xdebug.org/) or Blackfire(https://blackfire.io/).