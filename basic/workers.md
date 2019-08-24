# Workers and Application Lifecycle
Though Framework can work using classic Nginx/PHP-FPM setup the highest effectiveness can be achieved using 
embedded application server (based on RoadRunner). The server creates the set of php processes for each of dispatching 
method (HTTP, GRPC, Queue).

Every PHP process will only work within single request/task. It allows you to write code as if you would normally do. 
By keeping application state intact between requests you can drastically increase performance and also offload part
of functionality to application server.

## Application Server
You can configure the number of workers, memory limits and other extensions using `.rr.yaml` file:

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
Every worker will contain single application instance. Default application skeleton(s) are based 
on [spiral/boot](https://github.com/spiral/boot).

The package allows quick application instantiation via static factory:

```php
$app = \App\App::init(['root' => __DIR__]);
```

Application Kernel initiates your application state and IoC configuration using set of bootloaders:

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

The bootloaders will only be invoked once, without request/task context. After that application will stay in the process 
memory permanently. Since the application bootload only happens once for many requests you can add many components and extension
without performance penalty (still, watch memory).

## Limitations
There is multiple limitations to be aware of.

#### Memory Leaks
Since application stays in memory for long even small memory leak might lead to the process restart. RoadRunner
will monitor memory consumption but it is the best to avoid memory leaks in your application source code.

#### Application State
Any service declared as singleton will remain in application memory till process end. Try to avoid storing any user data
or resources in such services. 

> Framework includes a set of instruments to simplify the development process and avoid memory/state leaks such as 
IoC Scopes, Cycle ORM, Immutable Configs and Routes, Middleware.
