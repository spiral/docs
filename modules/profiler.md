# Profiler Panel
Default spiral application ships with pre-installed profiler module, if, for some reason, you don't have profiler panel installed, simply execute "composer require spiral/profiler".

## Enbable/Disable Profiler panel
Profiler panel can work only with HttpDispatcher and treated as middleware, if you wish to enable or disable profiler globally simply edit http middleware configuration located in "application/config/http.php" file.

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

Once enabled profiler will create icon at the lower right of your screen, such icon will provide you access to various profiler panels and information about current script timing, memory consuption and response code.

## Environment Overview
First profiler panel will give an overview of current application environment, it will proviode you information about current view namespaces, http routes, container bindings, loaded extensions and list of loaded classes. In addtion to that profiler will colozier loaded classes based on their location, for example every application class will be highlighted with blue, every vendor class will get yellow color.

![Environment](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/environment.png)

## Request/Response Data
Due profiler assigned to application as middleware it has full access to sent request and generated response, second panel will give you ability to check various parameters of such objects.

![Variables](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/variables.png)

## Benchmarks
Spiral application provides simplistic mechanism to track time spend for potentially long operations, you can read about benchmarking more [here] (/components/debug.md). Third panel will convert collected benchmarks information into application timeline:

![Benchmarks](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/benchmarks.png)

## Logging
If you application has enabled [global logging] (/components/debug.md), profile will show such log messages on it's last forth tab. In additional to that profiler will colorize different log level messages and highlight SQL statements.

![Logging](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler/logging.png)
