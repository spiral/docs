# The Basics — Debugging

When you're developing a long-running application using the Spiral Framework and RoadRunner, there are specific
challenges to consider while debugging your code. Here's what you need to know:

## Common Debugging Techniques

### Avoid Using `die` and `exit` functions

In traditional PHP development, you might be used to using `die` or `exit` functions to halt script execution, often in
combination with a dump function like `var_dump`. However, in a Spiral environment, using `die` or `exit` can break
your application because it will stop the RoadRunner worker completely, not just the current request.

For instance, using the `dd` function from the `symfony/var-dumper` package can cause issues as this function dumps a
variable's contents and then calls `die`, thus breaking your RoadRunner worker.

### Handling `PHP_SAPI` Mismatch

RoadRunner doesn't use the traditional PHP SAPI (Server API), and some dumpers are built with the assumption that they
are working in a CLI (Command Line Interface) environment. This discrepancy can lead to unexpected behavior when you are
trying to debug your application.

## Spiral Dumper

We’ve developed the `spiral/dumper` package to solve this problems. This package acts as a wrapper
around [symfony/var-dumper](https://symfony.com/doc/current/components/var_dumper.html) library and allows you to send
variable dumps directly to the browser within HTTP workers, or to the `STDERR` output in other environments. It is
designed to play nicely with RoadRunner's long-running approach.

With the `spiral/dumper` package, developers can effortlessly inspect and analyze variable values during the development
process. This package is an invaluable asset for debugging and troubleshooting in both web and CLI applications.

### Installation

By default, the `spiral/dumper` package is already included in the `spiral/app` skeleton. However, if you're using a
different skeleton, you can easily install the package using the following command:

```terminal
composer require --dev spiral/dumper
```

Once installed, you need to add the package's bootloader to your application.

```php app/src/Application/Kernel.php
use Spiral\Debug\Bootloader\DumperBootloader;

protected const SYSTEM = [
    // ...
    DumperBootloader::class,
];
```

### Usage

To dump variables, simply utilize the helper function `dump()` provided by the package.

```php
dump($variable);
```

With the package you can use the `dd` function just like you would in a traditional PHP application, but without the
risk of halting your entire RoadRunner worker.

```php
dd($variable);
```

<hr />

## Symfony VarDumper

Alternatively, for a more traditional debugging approach, you can opt for the symfony/var-dumper package.

This package offers a standalone server that collects all the dumped data. You start the server using a command and it
will listen for data sent by the dump() function. Any variable dumps you send to this function will be displayed
in a separate console window, not in your main application output.

Here is an example of console output:

```terminal
./vendor/bin/var-dump-server

Symfony Var Dumper Server
=========================
 [OK] Server listening on tcp://127.0.0.1:9912
 // Quit the server with CONTROL-C.

$ app.php
---------
 -------- ---------------------------------------------------------
  date     Fri, 18 Aug 2023 11:54:44 +0000
  source   SimpleController.php on line 36
  file     app/src/Interfaces/Http/Controller/SimpleController.php
 -------- ---------------------------------------------------------
null

 -------- ---------------------------------------------------------
  date     Fri, 18 Aug 2023 11:54:44 +0000
  source   SimpleController.php on line 37
  file     app/src/Interfaces/Http/Controller/SimpleController.php
 -------- ---------------------------------------------------------
App\Service\Site\Site^ {#1260
  -theme: "default"
  -docs: App\Service\Site\Docs^ {#1269
    -defaultVersion: "3.5"
    -defaultLanguage: "en"
  }
  -host: "127.0.0.1"
}
```

### Installation

To install the package, execute the following command:

```terminal
composer require --dev symfony/var-dumper
```

### Usage

To start the server, run the following command:

```terminal
./vendor/bin/var-dump-server
```

To use this feature, you also need to define the `VAR_DUMPER_FORMAT` environment variable in your `.env` file
as follows:

```dotenv .env
VAR_DUMPER_FORMAT=server
```

### Known Issues

If an object has numerous properties, or those properties contain substantial data, the console output can
become overwhelming. In these cases, it becomes extremely difficult to sift through the massive amount of text in the
console to locate the specific information you are interested in.

This problem is exacerbated when you are working with complex objects—like ORM entities with many relations, or big
arrays—that have deep and wide structures. The console, being a linear and limited-size output, can struggle to present
this information in a readable and navigable way.

<hr />

## Advanced Debugging with Buggregator

[Buggregator](https://github.com/buggregator/spiral-app) is a powerful, dockerized web application and server designed
to significantly enhance your debugging experience in PHP development. It listens on both TCP and HTTP ports, allowing
it to handle a variety of incoming requests, including variable dumps, exceptions, application logs, SMTP mails, and
more.

![var-dumper](https://user-images.githubusercontent.com/773481/208727353-b8201775-c360-410b-b5c8-d83843d388ff.png)

### Key Features

1. **Catch and Display PHP Variables:** Easily integrates with tools
   like [Symfony var-dumper](https://github.com/buggregator/spiral-app#2-symfony-vardumper-server) to capture and
   display dumped variables in an organized, readable format.

2. **Exception Handling:** It can catch and display exceptions, including those sent by error tracking platforms like
   [Sentry](https://github.com/buggregator/spiral-app#4-compatible-with-sentry-reports), giving you a clear, centralized
   view of issues as they arise.

3. **SMTP Mail Catcher:** It can act as
   a [mock SMTP server](https://github.com/buggregator/spiral-app#3-fake-smtp-server-for-catching-mail), catching and
   displaying emails sent by your application during development, so you can view and test emails without actually
   sending them.

4**User-Friendly Interface:** Provides a clean, intuitive web UI that organizes and displays your debugging data in a
way that’s easy to navigate and understand.

5**Dockerized for Easy Setup:** Buggregator is packaged as a Docker container, making it incredibly simple to get up
and running in any development environment.

When dealing with complex objects with numerous properties and substantial data, sifting through console output or log
files can be overwhelming and time-consuming. Buggregator addresses this problem by presenting this information in a
structured, collapsible, and searchable web interface. This way, you can quickly and efficiently find the exact piece of
data you need, without having to scroll through hundreds of lines of text.

### Installation

To start using Buggregator, simply pull the Docker image and run the container:

```bash Latest stable release
docker run --pull always ghcr.io/buggregator/server:latest
    -p 8000:8000 
    -p 1025:1025 
    -p 9912:9912 
    -p 9913:9913 
```

Or, if you want to use it with docker-compose, add the following service to your `docker-compose.yaml` file:

```yaml docker-compose.yaml
services:
  # ...
  buggregator:
    image: ghcr.io/buggregator/server:latest
    ports:
      - 8000:8000
      - 1025:1025
      - 9912:9912
      - 9913:9913
```

Once Buggregator is running, navigate to http://127.0.0.1:8000 in your web browser to access the Buggregator interface
and start monitoring your application’s debugging data in real time.

> **Note**
> Information about configuring your application to send data to Buggregator can be found in
> the [GitHub repository](https://github.com/buggregator/server#features)

### XHProf Integration

Buggregator not only serves as a comprehensive tool for catching and displaying debugging data, but it also shines as a
valuable partner for application profiling. It can act as an observer
for [Xhprof](https://github.com/buggregator/spiral-app#1-xhprof-profiler) profiles, providing developers with an
intuitive and efficient way to analyze performance data, identify bottlenecks, and spot memory leaks in their
PHP applications.

> **Note**
> XHProf is a tool that helps you figure out how your PHP code is running and where it might be slow. It keeps track of
> how many times different parts of your code get called, and how long they take. It also can help you figure out how much
> memory your code is using. Spiral has a [spiral/profiler](https://github.com/spiral/profiler) package that
> makes it easy to use XHProf in your PHP application. It provides a simple and convenient way to use the XHProf profiler
> during the development or profiling period, so you can quickly identify and optimize performance bottlenecks in your
> code.

![xhprof](https://user-images.githubusercontent.com/773481/208724383-3790a3e1-9ebe-4616-8d4d-d1869f8f2b7c.png)

**It can be useful in the following ways:**

1. **Convenient Bottleneck Identification:** Buggregator presents XHProf data in a way that makes it easy to spot
   performance bottlenecks. With sortable tables and graphical representations, developers can quickly understand which
   parts of the application are consuming the most time and resources.

2. **Memory Leak Detection:** Buggregator’s detailed profiling view helps developers to pinpoint memory usage spikes,
   making it a valuable tool for identifying and resolving memory leaks.

#### Installation

To begin, you need to install the Xhprof extension. One way to do this is by using the PECL package.

```terminal
pear channel-update pear.php.net
pecl install xhprof
```

Next, install the profiler package:

```terminal
composer require --dev spiral/profiler:^3.0
```

Once the package is installed, add the bootloader from the package to your application.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Profiler\ProfilerBootloader::class,
    // ...
];
```

#### Configuration

Use the following environment variables to configure the profiler to send data to
the [Buggregator](https://github.com/buggregator/spiral-app) server:

```dotenv .env
PROFILER_ENDPOINT=http://127.0.0.1:8000/api/profiler/store
PROFILER_APP_NAME=My super app
```

#### Usage

There are two ways to use profiler:

- Profiler as an interceptor
- Profiler as a middleware

#### Profiler as an interceptor

Interceptor will be useful if you want to profile some specific part of your application which supports using
interceptors.

- [Controllers](../http/interceptors.md),
- [GRPC](../grpc/interceptors.md),
- [Queue jobs](../queue/interceptors.md).
- TCP
- [Events](../advanced/events.md#interceptors).

> **See more**
> Read more about interceptors in the [Framework — Interceptors](../framework/interceptors.md) section.

To use the profiler as an interceptor, you just need to register `Spiral\Profiler\ProfilerInterceptor` class.

Here is an example of how to use the profiler as an interceptor in the http layer:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        \Spiral\Profiler\ProfilerInterceptor::class
    ];
}
```

#### Profiler as a middleware

Middleware will be useful if you want to profile all requests to your application. To use profiler as a middleware you
need to add it to your router.

> **See more**
> Read more about middleware in the [HTTP — Routing](../http/routing.md) section.

##### Global middleware

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Profiler\ProfilerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            ProfilerMiddleware::class,  // <================
            // ...
        ];
    }
    
    // ...
}
```

##### Route group middleware

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Profiler\ProfilerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                // ...
            ],
            'profiler' => [                  // <================
                ProfilerMiddleware::class,
                'middleware:web',
            ],
        ];
    }
    
    // ...
}
```

##### Route middleware

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Router\Annotation\Route;

final class UserController
{
    #[Route(route: '/users', name: 'user.store', methods: ['POST'], middleware: \Spiral\Profiler\ProfilerMiddleware::class)]
    public function store(...): void 
    {
        // ...
    }
}
```

<hr />

## XDebug

Debugging the Spiral application is as feasible as debugging any other classic PHP application when utilizing the xDebug
extension.

### IDE Configuration

Ad first, you need to configure your IDE to work with xDebug.

> **See more**
> Read more about the IDE configuration in the
> official [documentation](https://roadrunner.dev/docs/php-debugging).

### On-Demand

It is more convenient to start RoadRunner with xDebug enabled only when it's needed. Add the following env variables to
`.rr.yaml` to properly configure xDebug:

```yaml .rr.yaml
env:
  PHP_IDE_CONFIG: serverName=application.loc
  XDEBUG_CONFIG: remote_host=localhost max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
```

> **Note**
> Alter values according to your environment.

To enable xDebug, run the application server with `-o` (overwrite flag) for the required service:

```terminal
./rr serve -o "server.command=php -d zend_extension=xdebug app.php"
```

### In Docker

To alter workers config in docker, use the following or similar config for your container:

```yaml docker-compose.yaml
version: "2"
services:
  ...
  app:
    ...
    command:
      - /usr/local/bin/rr
      - serve
      - -o
      - server.command=php -d zend_extension=xdebug.so app.php
    environment:
      PHP_IDE_CONFIG: serverName=application.loc
      XDEBUG_CONFIG: remote_host=host.docker.internal max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
```