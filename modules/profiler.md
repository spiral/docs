# Profiler Panel
The default Spiral application comes with a pre-installed profiler module. If for some reason, you don't have profiler panel installed, simply execute "composer require spiral/profiler".

## Enbable/Disable Profiler panel
Profiler panel only works with HttpDispatcher and is treated as middleware. If you want to enable or disable profiler, simply edit the http middleware configuration located in "application/config/http.php" file.

```php
'middlewares'  => [
    \Spiral\Profiler\Profiler::class,
    Middlewares\DispatcherHeaders::class,
    Http\Cookies\CookieManager::class,
    Middlewares\CsrfFilter::class,
    Middlewares\JsonParser::class,
    \Spiral\Session\Http\SessionStarter::class,
],
```

Once enabled, the profiler will create an icon located at the lower right section of your screen. This icon gives you access to various profiler panels and information about current script timing, memory consuption and response code.

![Profiler Panel](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/panel.png)

## Environment Overview
The first profiler panel will give you an overview of the current application environment. It will provide information about the current view namespaces, http routes, container bindings, loaded extensions and a list of loaded classes. In addtion, profiler will assign a color to the loaded classes based on their location. For example, every application class will be highlighted blue or every vendor class will be yellow.

![Environment](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/environment.png)

## Request/Response Data
Because the profiler is assigned to application as middleware, it has full access to sent request and the generated response. The second panel will give you the ability to check the different parameters of these objects.

![Variables](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/variables.png)

## Benchmarks
The Spiral application provides a simple mechanism to track time spent for potentially long operations. You can read more about these benchmarking and tracking tools  [here] (/components/debug.md). The third panel will convert the collected data/information into the application timeline:

![Benchmarks](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/benchmarks.png)

## Logging
If you enable [global logging] (/components/debug.md) in your application, profile will show these log messages on within the last tab. In addition, profiler will colorize different log level messages and highlight SQL statements.

![Logging](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/logging.png)
