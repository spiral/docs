# HTTP - 错误页面

应用程序会暴露一些错误和异常，这些错误和异常有的需要显示给用户，但有的不应该显示给用户。

HTTP 组件包括默认的错误处理中间件，用于拦截和记录关键错误和用户级异常。

要启用该中间件，请在你的应用程序中添加 `Spiral\Bootloader\Http\ErrorHandlerBootloader` 引导程序

```php
namespace App;

use Spiral\Bootloader as Framework;
use Spiral\DotEnv\Bootloader as DotEnv;
use Spiral\Framework\Kernel;
use Spiral\Prototype\Bootloader as Prototype;
use Spiral\Scaffolder\Bootloader\ScaffolderBootloader;

class App extends Kernel
{
    /*
     * 应用启动时自动注册到系统容器中的组件和扩展。
     */
    protected const LOAD = [
        // ...

        Framework\Http\HttpBootloader::class,
        Framework\Http\RouterBootloader::class,
        Framework\Http\ErrorHandlerBootloader::class,

        // ...
    ];
}
```

## 应用异常

异常处理中间件会处理应用程序抛出的异常，并把它们以对开发者更友好的方式渲染成页面。如果不想把异常发送到浏览器，可以把在 `.env` 文件中把环境变量 `DEBUG` 的值设置为 `false`, 这样的话，系统出错时浏览器始终显示默认的 500 错误页面。

> 应用部署到生产环境时，切勿启用开发模式。

## 客户端异常

但有些异常适合在控制器或者中间件中抛出，通过这些异常来让浏览器显示 HTTP 级别的错误页面。比如我们经常会主动触发 `NotFoundException` 来让页面显示 `404 Not Found`:

```php
namespace App\Controller;

use Spiral\Core\Container\SingletonInterface;
use Spiral\Http\Exception\ClientException\NotFoundException;

class HomeController implements SingletonInterface
{
    public function index()
    {
        throw new NotFoundException();
    }
}
```

其它一些适合抛出的客户端异常有：

状态码 | 异常 
--- | ---
400 | Spiral\Http\Exception\ClientException\BadRequestException
401 | Spiral\Http\Exception\ClientException\UnauthorizedException
403 | Spiral\Http\Exception\ClientException\ForbiddenException
404 | Spiral\Http\Exception\ClientException\NotFoundException
500 | Spiral\Http\Exception\ClientException\ServerErrorException

> 注意：不要在服务（Service）和实体仓库（Repository）中抛出 HTTP 异常，因为这样会导致你的业务实现与 HTTP 分发器（dispatcher）绑定。因此在这些场景下请使用业务相关的异常，然后把它们映射到 HTTP 异常。

## 渲染错误页面

默认情况下，错误处理中间件使用没有附加任何样式的简单错误页面来显示。如果要定制错误页面的样式，可以实现 `Spiral\Http\ErrorHandler\RendererInterface` 接口，并在容器中把该接口与你的实现进行绑定：

```php
namespace App\Errors;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Spiral\Http\ErrorHandler\RendererInterface;
use Spiral\Views\ViewsInterface;

class NiceRenderer implements RendererInterface
{
    private $responseFactory;
    
    private $views;

    public function __construct(
        ResponseFactoryInterface $responseFactory,
        ViewsInterface $views
    ) {
        $this->responseFactory = $responseFactory;
        $this->views = $views;
    }

    public function renderException(Request $request, int $code, string $message): Response
    {
        $response = $this->responseFactory->createResponse($code);

        $response->getBody()->write(
            $this->views->render('errors/' . (string)$code)
        );

        return $response->withStatus($code, $message);
    }
}
```

在引导程序中绑定它：

```php
namespace App\Bootloader;

use App\Errors\NiceRenderer;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\ErrorHandlerBootloader;
use Spiral\Http\ErrorHandler\RendererInterface;

class NiceErrorsBootloader extends Bootloader
{
    public const DEPENDENCIES = [
        ErrorHandlerBootloader::class
    ];

    public const SINGLETONS = [
        RendererInterface::class => NiceRenderer::class
    ];
}
```

## 日志

应用程序默认包含 Monolog 处理程序，该处理程序会订阅由 `ErrorHandlerMiddleware` 中间件发送的消息。HTTP 错误日志位于 `app/runtime/logs/http.log`，默认的日志配置位于 `App\Bootloaders\LoggingBootloader`:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Monolog\Bootloader\MonologBootloader;

class LoggingBootloader extends Bootloader
{
    /**
     * @param MonologBootloader $monolog
     */
    public function boot(MonologBootloader $monolog)
    {
        $monolog->addHandler(
            ErrorHandlerMiddleware::class,
            $monolog->logRotate(directory('runtime') . 'logs/http.log')
        );
    }
}
```
