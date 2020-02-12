# 框架 - 内核与环境

每个 Spiral 的构建都包含特定于域的内核以及一组特定于应用的服务。与 Symfony 不同之处在于， 对于所有的调度方法（HTTP, 队列、GRPC, 控制台），Spiral 只需要一个内核。这个内核会根据被连接的 `Spiral\Boot\DispatcherInterface` 实例自动选择调度器。

The base kernel implementation is located in `spiral/boot` repository.

## 内核职责

`Spiral\Boot\AbstractKernel` 类只处理应用的以下方面：

- 通过一组特定于应用的引导加载器进行容器初始化
- 环境变量和目录结构的初始化
- 选择合适的调度方法

你可以通过扩展 `Spiral\Boot\AbstractKernel` 来创建自己的内核类：

```php
namespace App;

use Spiral\Boot\AbstractKernel;
use Spiral\Boot\Exception\BootException;

class MyApp extends AbstractKernel
{
    protected const LOAD = [
        // 要初始化的引导加载器
    ];

    public function get(string $service)
    {
        return $this->container->get($service);
    }

    protected function bootstrap()
    {
        // 自定义初始化代码
        // 此方法将在所有引导加载器都加载之后执行
    }

    protected function mapDirectories(array $directories): array
    {
        if (!isset($directories['root'])) {
            throw new BootException('Missing required directory `root`');
        }

        if (!isset($directories['app'])) {
            $directories['app'] = $directories['root'] . '/app/';
        }

        return array_merge(
            [
                // public 目录
                'public'    => $directories['root'] . '/public/',

                // 依赖库目录
                'vendor'    => $directories['root'] . '/vendor/',

                // 数据目录
                'runtime'   => $directories['root'] . '/runtime/',
                'cache'     => $directories['root'] . '/runtime/cache/',

                // 应用相关目录
                'config'    => $directories['app'] . '/config/',
                'resources' => $directories['app'] . '/resources/',
            ],
            $directories
        );
    }
}
```

> `Spiral\Framework\Kernel` 定义了默认的目录数组。

然后，调用静态方法 `init` 来初始化内核：

```php
$myapp = MyApp::init(
    [
        'root' => __DIR__
    ],
    null, // 使用默认环境
    false // 不挂载错误处理器
);      

// 输出所有定义的目录路径
dump($myapp->get(\Spiral\Boot\DirectoriesInterface::class)->getAll());
```

## 环境

通过 `Spiral\Boot\EnvironmentInterface` 可以访问环境变量。默认配置下，框架依赖于系统级别的环境变量。如果要重新定义环境变量，可以在初始化内核的时候传递自行实现的 `EnvironmentInterface` 给 `init` 方法。

```php
$myapp = MyApp::init(
    [
        'root' => __DIR__
    ],
    new Environment(['key' => 'value']),
    false // 不挂载错误处理器
);

// 列出所有定义的目录路径
dump($myapp->get(\Spiral\Boot\EnvironmentInterface::class)->getAll());
```

> 可以通过上述方法来引导应用，以用于测试。
