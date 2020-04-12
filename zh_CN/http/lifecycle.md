# HTTP - 请求生命周期

与其它大部分的 PHP 框架不同，Spiral 中的 HTTP 请求的生命周期是在 PHP 应用程序之外，从 Golang 实现的应用服务器 —— [RoadRunner](https://roadrunner.dev/) 开始的。

![请求生命周期](https://user-images.githubusercontent.com/796136/67088146-1bd39c80-f1ad-11e9-9d5e-6b2499654395.png)

> 响应是按照图中的流程反方向流转。

## PSR 规范

Spiral 框架基于一系列 PSR 规范构建而成，这让它可以和其它的框架、路由、中间件等相互兼容。如果需要，请参阅以下 HTTP 组件用到的 PSR 规范：

- [PSR-7: HTTP 消息接口规范](https://www.php-fig.org/psr/psr-7/)
- [PSR-15: HTTP 服务端请求处理器规范](https://www.php-fig.org/psr/psr-15/)
- [PSR-17: HTTP 消息工厂规范](https://www.php-fig.org/psr/psr-17/)

## 请求处理流程描述

用户请求首先到达 RoadRunner 应用服务器。应用服务器会让请求经过服务器中间件层的一系列 Golang 开发的中间件处理，这些中间件通常负责支持 WebSocket 协议、响应静态文件或者[实现特定业务逻辑](/zh_CN/http/golang.md)。

在经过了上述中间件的处理之后，`net/http` 请求被转换为 `PSR-7` 格式的请求，并发送给第一个可用的 PHP 工作进程。

PHP 工作进程通过 `spiral/http` 组件中的 `Spiral\Http\Http` 核心类处理接收到的请求。该核心类会把 PSR-7 格式的请求对象（`ServerRequestInterface`）交给一系列基于 PSR-15 规范的中间件进行处理。

在所有的中间件处理完毕之后，Spiral 框架会为请求对象创建一个[IoC 作用域](/zh_CN/framework/scopes.md)。通过这样的方式，你可以像传统的 PHP 程序那样把 PSR-7 请求当作一个全局对象来处理。但从技术上来说，在 Spiral 中这个“全局对象”只在本次用户请求的上下文环境中存在。

请求最终被传递给你选择的 PSR-15 处理器（默认是 `spiral/router`）进行处理。处理器最终生成响应，并沿着上面的路径原路返回，通过所有的中间件之后发送给用户。

> Spiral 路由提供了把定制中间件绑定到单个路由的能力。

## 手动调用 HTTP

除了常规的用户请求方式以外，也可以在应用中直接调用 HTTP 核心对象。这在单元测试、从其它框架中运行 Spiral 等场景下非常有用。要做到这一点，可以先获取 `Spiral\Http\Http` 的实例来实现：

```php
namespace App\Controller;

use Nyholm\Psr7\Uri;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Http\Http;

class HomeController implements SingletonInterface
{
    private $http;

    public function __construct(Http $http)
    {
        $this->http = $http;
    }

    public function index(ServerRequestInterface $request): string
    {
        $response = $this->http->handle(
            $request->withUri(new Uri('/home/other')) // 修改当前请求的 Uri
        );

        return (string)$response->getBody(); // "other"
    }

    public function other(): string
    {
        return 'other';
    }
}
```

> IoC 作用域可以嵌套，所以在类似上面这样的使用方式中，所有的功能都是正常的。但是需要注意，并非所有的组件都支持嵌套（比如创建嵌套的 session 目前就不能实现）。
