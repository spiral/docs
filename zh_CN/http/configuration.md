# HTTP - 安装配置

Spiral 官方提供的 WEB 应用模板（`spiral/app`）已经预先配置好了 HTTP 组件。如果不使用官方模板而是自己构建一个项目，那么除了 HTTP 组件以外，还需要安装和启用几个其它的扩展，才能让项目正常运行。

## 安装

要安装所需的扩展组件，可以执行以下命令：

```bash
$ composer require spiral/http spiral/router spiral/nyholm-bridge
```

安装完成后，添加两个引导程序来启用这些扩展：

```php
class App extends Kernel
{
    /*
     * 在应用程序首次启动时，在系统容器中自动注册的扩展和组件列表
     */
    protected const LOAD = [
        // ...

        // Nyholm 是一个高效的对于 PSR-7 规范的实现
        \Spiral\Nyholm\Bootloader\NyholmBootloader::class,

        // HTTP 核心组件
        Spiral\Bootloader\Http\HttpBootloader::class,

        // 实现 PSR-15 规范的处理器
        Spiral\Bootloader\Http\RouterBootloader::class,

        // ...
    ];
}
```

> 如果想自行按照 PSR-15 规范实现处理器，可以参考[这份文档](/zh_CN/http/psr-15.md)。

别忘了配置[路由](/zh_CN/http/routing.md)。

## 配置

HTTP 组件可以通过 `app/config/http.php` 这个配置文件进行配置：

```php
<?php

declare(strict_types=1);

return [
    // 默认的根路径
    'basePath'   => '/',

    // 默认响应头信息
    'headers'    => [
        'Content-Type' => 'text/html; charset=UTF-8'
    ],

    // 应用级（全局）中间件
    'middleware' => [
        // 中间件的类名
    ],
];
```

> 提示：此配置文件不是必须的，如果配置文件不存在，Spiral 会使用默认配置。

在应用程序的引导（初始化）阶段，可以通过 `HttpBootloader` 引导程序来注册中间件：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;
use Spiral\Http\Middleware\JsonPayloadMiddleware;

class AppBootloader extends Bootloader
{
    public function boot(HttpBootloader $http): void
    {
        // 添加一个解析 json 请求体的中间件
        $http->addMiddleware(JsonPayloadMiddleware::class);
    }
}
```

## 中间件

HTTP 组件包含了部分中间件，你可以在项目中根据需要自行启用：

| 引导程序                                      | 中间件                                                        |
| --------------------------------------------- | ------------------------------------------------------------- |
| Spiral\Bootloader\Http\ErrorHandlerBootloader | 在非调试模式下隐藏系统错误并渲染友好的错误页面                |
| Spiral\Bootloader\Http\JsonPayloadsBootloader | 解析 JSON 请求的请求体                                        |
| Spiral\Bootloader\Http\PaginationBootloader   | 通过请求中的查询参数自动配置分页                              |
| Spiral\Bootloader\Http\DiactorosBootloader    | 使用 Zend/Diactoros 作为 PSR-7 的实现（旧版遗留，不建议使用） |
