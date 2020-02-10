# 工作进程及应用生命周期
Spiral 可以和通过传统的 Nginx/PHP-FPM 配置来运行。但是借助内置的应用服务器（基于 [RoadRunner(https://roadrunner.dev/)]）可以达到更高的性能。应用服务器为每一种调度/通讯方式（HTTP, GRPC, Queue 等等）创建一组 PHP 进程。

每个 PHP 进程只在一个请求或者任务内工作。这使得你可以继续按以前在传统的框架下的方法去写代码而不用考虑进程间共享的问题。你也可以按照以前的习惯去使用各种类库，只有极少数例外。

通过把应用程序保留在驻留内存，可以大大提高性能，还可以把一部分功能从业务代码移到应用服务器。

> 想了解更多有关框架与应用服务器共生的信息，可以看看[这里](/framework/design.md)。

## 应用服务器
你可以在 `.rr.yaml` 文件中配置工作进程数、内存限制以及其他扩展：

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

要配置 HTTP 服务的工作进程数：

```yaml
http:
  address: 0.0.0.0:8080
  workers:
    command:         "php app.php"
    pool.numWorkers: 4
```

### 开发模式
要强制工作进程在每次请求时进行重载（完整调试模式）以及限制只使用单个工作进程：

```yaml
http:
  address: 0.0.0.0:8080
  workers:
    command:         "php app.php"
    pool.numWorkers: 1
    pool.maxJobs:    1
```

关于 RoadRunner 的更多信息，请查阅 [RoadRunner 文档](https://roadrunner.dev/docs)

## 应用内核
每个工作进程会包含一个应用实例。默认的应用架构基于 [spiral/boot](https://github.com/spiral/boot) 这个包。

这个包让开发者能够通过静态方法 `init` 快速实例化应用程序：

```php
$app = \App\App::init(['root' => __DIR__]);
```

应用内核用一组引导加载程序（Bootloader）来初始化你的应用状态和 IoC 配置：

```php
class App extends Kernel
{
    /*
     * 在应用启动时自动注册到系统容器中的组件和扩展。
     */
    protected const LOAD = [
        // 环境配置
        DotEnv\DotenvBootloader::class,
        
        // 核心服务
        Bootloader\DebugBootloader::class,
        Bootloader\SnapshotsBootloader::class,
        Bootloader\I18nBootloader::class,
        
        // 安全和验证
        Bootloader\Security\EncrypterBootloader::class,
        
        // ...
    ];
}
```

引导加载程序只会在首次初始化时调用一次，所以它们没有请求/任务的上下文。初始化之后应用会驻留在进程的内存中。由于应用的程序引导在大量请求下只发生一次，因此你可以添加很多组件和扩展而不会造成性能的损失（当然，你要注意内存的占用）。

## 注意
有一些限制需要特别注意。

#### 内存泄露
由于应用将在内存中长期驻留，即使是很小的内存泄露也可能导致进程重启。RoadRunner 会持续监控内存占用并执行软重置，但最好还是要避免应用的业务代码中出现内存泄露。

尽管 Spiral 框架和所有官方组件在开发时都已经考虑到了内存管理的问题，但你依然要确保你自己的业务代码中没有内存泄露。


#### 应用状态
任何定义为单例模式的服务都会驻留在应用内存中，直到进程终止。因此要尽量避免在这些服务中存储任何用户数据，也不要在这些服务中打开资源。

> Spiral 框架包含了一组能够简化开发过程，避免内存/状态泄露的工具，例如 IoC 作用域，Cycle ORM，不可变配置，域核心，路由和中间件等。
