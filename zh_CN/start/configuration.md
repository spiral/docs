# 默认配置
官方提供的应用程序框架都已经用最佳设置进行了预配置。你可以通过 `app/config/` 目录下的配置文件来对调整任何一个配置项。

> 如果要编辑的配置文件不存在，你可以自行创建。每个配置文件的基础代码都是 `<?php return [];`: 返回一个数组。

## 环境变量
Web 和 GRPC 应用程序模板使用 DotEnv 扩展从项目根目录下的 `.env` 文件中读取环境变量。

```env
# Debug 模式禁用视图缓存，并输出更详细的调试信息。
DEBUG = true

# 设置为特定于应用程序的值，用于加密/解密 cookies 和其它内容的密钥。
ENCRYPTER_KEY = {encrypt-key}

# 设置为 TRUE 时，执行 `migrate` 命令不会提示确认。
SAFE_MIGRATIONS = true
```

要访问这些值，可以通过 `Spiral\Boot\EnvironmentInterface` 或者简单的全局方法 `env`：

```php
public function index(EnvironmentInterface $env)
{
    dump($env->get('ENCRYPTER_KEY'));
    dump(env('ENCRYPTER_KEY'));
}
```

##  配置文件
默认的组件配置都保存在相关的引导程序中。你可以通过其它的引导程序来更改这些配置（参见自动配置），或者在 `app/config` 目录下创建*默认配置*文件。

Web 和 GRPC 应用程序模板默认包含 `app/config/database.php` 配置文件：

```php
use Spiral\Database\Driver;

return [
    'default'   => 'default',
    'databases' => [
        // 连接名 => 数据库驱动
        'default' => ['driver' => 'runtime'],
    ],
    'drivers'   => [
        // 数据库驱动名称 => 驱动选项
        'runtime' => [
            'driver'     => Driver\SQLite\SQLiteDriver::class,
            'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
            'profiling'  => true,
        ],
    ]
];
```

同样的方式，你可以编辑任何组件的任何配置项。例如，我们可以通过 `app/config/http.php` 改变默认的 HTTP 头信息：


```php
return [
    'headers' => [
        'Server'       => 'Spiral',
        'Content-Type' => 'text/html; charset=UTF-8'
    ],
];
```

如果你需要了解哪个配置文件对应你需要修改的 HTTP 组件的配置对象，请查看 [HTTP 组件的配置常量](https://github.com/spiral/http/blob/master/src/Config/HttpConfig.php#L17)

```php
final class HttpConfig extends InjectableConfig
{
    const CONFIG = 'http';
    
    // ...
```

> 请参阅相关文档章节下每个组件的配置参考。
