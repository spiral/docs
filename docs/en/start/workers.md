# Application Lifecycle

The Spiral Framework can be used with a traditional Nginx/PHP-FPM setup, where Nginx acts as the web server and PHP-FPM
processes incoming requests and initializes the application. However, this can be resource-intensive as it requires CPU
and memory resources to complete with every incoming request.

![Nginx HTTP bootstraping](https://user-images.githubusercontent.com/773481/211190445-06c17d86-58d6-43d8-995f-36cf448714ae.jpg)

To improve the performance of the Spiral Framework, an alternative solution is to use application
server [RoadRunner](https://roadrunner.dev/).

RoadRunner is a high-performance application server designed to handle a wide range of request types, including HTTP,
gRPC, TCP, Queue Job consuming, and Temporal. It operates by running workers only once when it is initiated and then
directing requests to a [dispatcher](../framework/dispatcher.md) based on their type. This means that each worker is
isolated and works independently, following a "share nothing" approach where resources are not shared between workers.

> **Note**
> The approach of running application is similar to other languages like **Java**, **C#**, etc.

This allows developers to write code in a way that is familiar to them and allows workers to be horizontally scaled,
improving resource utilization and increasing the capacity to handle incoming requests.

![RoadRunner HTTP bootstraping](https://user-images.githubusercontent.com/773481/211197998-96b09ff1-4ede-4db0-9b1d-902e996920be.jpg)

Using RoadRunner can significantly improve the speed and efficiency by eliminating the need for
the application to go through the bootstrapping process repeatedly. This can save on CPU and memory resources and reduce
response time.

Another cool thing is that it allows developers to be a bit more flexible with how they bootstrap the application. For
example, they can load configs or routes from various sources like attributes or files without having to worry about
caching them. This can make it easier to modify and update the application as needed.

> **Note**
> Read more about Framework and application server symbiosis [here](/framework/design.md). Read about PSR-7 request
> flow [here](/http/lifecycle.md).

## Application Server

The RoadRunner Application Server is installed using
the [RoadRunner Bridge](https://github.com/spiral/roadrunner-bridge) integration package. In
the [spiral/app](https://github.com/spiral/app) skeleton, this package is already installed and configured.

> **Note**
> Learn more about the [RoadRunner Bridge](https://spiral.dev/docs/packages-roadrunner-bridge):

You can configure the number of workers, memory limits, and other extensions using `.rr.yaml` file:

```yaml
version: '2.7'

rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: "php app.php"
  relay: pipes

# HTTP plugin settings
http:
  address: 0.0.0.0:8080
  middleware: [ "gzip", "static" ]
  static:
    dir: "public"
    forbid: [ ".php", ".htaccess" ]
  pool:
    num_workers: 2
    supervisor:
      max_worker_memory: 100
```

To set the number of workers for HTTP:

```yaml
http:
  pool:
    num_workers: 4
```

### Developer Mode

To force worker reload after every request (full debug mode) and limit processing to a single worker, add a `debug`
option:

```yaml
http:
  pool:
    debug: true
```

You can read more about RoadRunner [here](https://roadrunner.dev/docs).

## Application Kernel

Every worker will contain a single application instance. Default application skeleton(s) are based
on [spiral/boot](https://github.com/spiral/boot) component.

The package allows quick application instantiation via static factory method `create` and allows you to run the created
application via the `run` method:

```php
$app = \App\App::create(['root' => __DIR__])->run();
```

Application Kernel initiates your application state and IoC configuration using a set of bootloaders:

```php
class App extends Kernel
{
    /*
     * List of components and extensions to be automatically registered
     * within system container on application start.
     */
    protected const LOAD = [
        // Environment configuration
        DotEnv\DotenvBootloader::class,
        
        // Core Services
        Bootloader\DebugBootloader::class,
        Bootloader\SnapshotsBootloader::class,
        Bootloader\I18nBootloader::class,
        
        // Security and validation
        Bootloader\Security\EncrypterBootloader::class,
        
        // ...
    ];
}
```

The bootloaders will only be invoked once, without request/task context. After that, application will stay in the
process memory permanently. Since the application bootload only happens once for many requests, you can add many
components and extensions without performance penalty (still, watch the memory consumption).

## Gotchas

There are a couple of limitations to be aware of.

#### Memory Leaks

Since the application stays in memory for a long time, even a small memory leak might lead to process restart.
RoadRunner will monitor memory consumption and perform a soft reset, but it is best to avoid memory leaks in your
application source code.

Though Framework and all of its components are written with memory management in mind, you still have to make sure that
your domain code is not leaking.

#### Application State

> **Note**
> Framework includes a set of instruments to simplify the development process and avoid memory/state leaks such as
> IoC Scopes, Cycle ORM, Immutable Configs, Domain Cores, Routes, and Middleware.

## Events

| Event                                | Description                                                                                                        |
|--------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Spiral\Boot\Event\Bootstrapped       | The Event will be fired `after` all bootloaders from SYSTEM, LOAD and APP sections initialized.                    |
| Spiral\Boot\Event\Serving            | The Event will be fired `before` looking for a dispatcher for handling incoming requests in a current environment. |
| Spiral\Boot\Event\DispatcherFound    | The Event will be fired when a dispatcher for handling incoming requests in a current environment is found.        |
| Spiral\Boot\Event\DispatcherNotFound | The Event will be fired when an application dispatcher is not found.                                               |
| Spiral\Boot\Event\Finalizing         | The Event will be fired when finalizer are executed `before` running finalizers.                                   |
 
