# Application Lifecycle
Spiral Framework can work using classic Nginx/PHP-FPM setup. But the highest effectiveness can be achieved using an embedded application server (based on [RoadRunner](https://roadrunner.dev/)). The server creates a set of php processes for each dispatching/communication method (HTTP, GRPC, Queue, etc.).

Every PHP process will only work within a single request/task. It allows you to write code as if you would typically do in classic frameworks (share nothing approach). You can also use every library as in the past with only minor exceptions.

By keeping the application in resident memory, you can drastically increase performance and also offload part of functionality to the application server.

> Read more about Framework and application server symbiosis [here](/framework/design.md). Read about PSR-7 request flow [here](/http/lifecycle.md).

## Application Server
You can configure the number of workers, memory limits, and other extensions using `.rr.yaml` file:

```yaml
http:
  address: 0.0.0.0:8080
  workers:
    command: "php app.php"

jobs:
  dispatch:
    app-job-*.pipeline: "local"
  pipelines:
    local.broker: "ephemeral"
  consume: ["local"]
  workers:
    command: "php app.php"
    pool.numWorkers: 2

static:
  dir:    "public"
  forbid: [".php", ".htaccess"]

limit:
  services:
    http.maxMemory: 100
    jobs.maxMemory: 100
```

To set the number of workers for HTTP:

```yaml
http:
  address: 0.0.0.0:8080
  workers:
    command:         "php app.php"
    pool.numWorkers: 4
```

### Developer Mode
To force worker reload after every request (full debug mode) and limit processing to single worker:

```yaml
http:
  address: 0.0.0.0:8080
  workers:
    command:         "php app.php"
    pool.numWorkers: 1
    pool.maxJobs:    1
```

You can read more about RoadRunner [here](https://roadrunner.dev/docs).

## Application Kernel
Every worker will contain a single application instance. Default application skeleton(s) are based 
on [spiral/boot](https://github.com/spiral/boot).

The package allows quick application instantiation via static factory method `init`:

```php
$app = \App\App::init(['root' => __DIR__]);
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

The bootloaders will only be invoked once, without request/task context after that application will stay in the process memory permanently. Since the application bootload only happens once for many requests, you can add many components and extensions without the performance penalty (still, watch the memory consumption).

## Gotchas
There are a couple of limitations to be aware of.

#### Memory Leaks
Since the application stays in memory for a long time, even a small memory leak might lead to the process restart. RoadRunner
will monitor memory consumption and perform a soft reset, but it is best to avoid memory leaks in your application source code.

Though Framework and all of its components are written with memory management in mind, you still have to make sure that your domain code is not leaking.

#### Application State
Any service declared as singleton will remain in the application memory till the process end. Try to avoid storing any user data
or open resources in such services. 

> Framework includes a set of instruments to simplify the development process and avoid memory/state leaks such as 
IoC Scopes, Cycle ORM, Immutable Configs, Domain Cores, Routes, and Middleware.
