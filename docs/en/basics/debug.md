# The Basics — Debugging

Spiral does not come with built-in tools for variable dumping, but fear not! There are third-party packages available to
help us out.

## Spiral Dumper

With the `spiral/dumper` package, developers can effortlessly inspect and analyze variable values during the development
process. This package is an invaluable asset for debugging and troubleshooting in both web and CLI applications.

The component acts as a wrapper over
the [symfony/var-dumper](https://symfony.com/doc/current/components/var_dumper.html) library. It allows you to send
dumps directly to the browser within HTTP workers or to the STDERR output in other environments.

### Installation

By default, the `spiral/dumper` package is already included in the `spiral/app` skeleton. However, if you're using a
different skeleton, you can easily install the package using the following command:

```terminal
composer require spiral/dumper
```

### Usage

To dump variables, simply utilize the helper function `dump()` provided by the package.

```php
dump($variable);
```

## Symfony VarDumper

Alternatively, you can opt for the [symfony/var-dumper](https://symfony.com/doc/current/components/var_dumper.html)
package as a debugging solution. This package offers a straightforward server that collects all the dumped data. You can
start the server using a command, and it will listen for data sent by the `dump()` function.

### Installation

To install the package, execute the following command:

```terminal
composer require symfony/var-dumper
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

By doing this, you instruct the package to send the dumped data to the server instead of displaying it in the output.

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

<hr />

## Buggregator

[Buggregator](https://github.com/buggregator/spiral-app) is a beautiful, lightweight standalone server built on Spiral
Framework, NuxtJs and RoadRunner underhood. It helps debugging mostly PHP applications without extra packages.

It supports collecting and displaying data from the following tools and services:

- [Xhprof](https://github.com/buggregator/spiral-app#1-xhprof-profiler),
- [Symfony var-dumper](https://github.com/buggregator/spiral-app#2-symfony-vardumper-server),
- [Monolog](https://github.com/buggregator/spiral-app#5-monolog-server),
- [Sentry](https://github.com/buggregator/spiral-app#4-compatible-with-sentry-reports),
- [Catching SMTP mails](https://github.com/buggregator/spiral-app#3-fake-smtp-server-for-catching-mail).

#### Installation

You can run Buggregator via docker from
the [Github Packages](https://github.com/buggregator/spiral-app/pkgs/container/server)

```bash Latest dev release
docker run --pull always ghcr.io/buggregator/server:dev
    -p 8000:8000 
    -p 1025:1025 
    -p 9912:9912 
    -p 9913:9913 
```

```bash Latest stable release
docker run --pull always ghcr.io/buggregator/server:latest
    -p 8000:8000 
    -p 1025:1025 
    -p 9912:9912 
    -p 9913:9913 
```

#### Using buggregator with docker compose

```yaml docker-compose.yaml
version: "2"
services:
  # ...
  buggregator:
    image: ghcr.io/buggregator/server:dev
    ports:
      - 8000:8000
      - 1025:1025
      - 9912:9912
      - 9913:9913
```

That's it. Now you open http://127.0.0.1:8000 url in your browser and collect dumps from your application.

<hr />

## Xhprof

XHProf is a tool that helps you figure out how your PHP code is running and where it might be slow. It keeps track of
how many times different parts of your code get called, and how long they take. It also can help you figure out how much
memory your code is using. Spiral has a [spiral/profiler](https://github.com/spiral/profiler) package that
makes it easy to use XHProf in your PHP application. It provides a simple and convenient way to use the XHProf profiler
during the development or profiling period, so you can quickly identify and optimize performance bottlenecks in your
code.

### Installation

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

### Configuration

Use the following environment variables to configure the profiler to send data to
the [Buggregator](https://github.com/buggregator/spiral-app) server:

```dotenv .env
PROFILER_ENDPOINT=http://127.0.0.1:8000/api/profiler/store
PROFILER_APP_NAME=My super app
```

### Usage

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