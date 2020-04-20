# HTTP - Golang 中间件

你可以在 Golang 端的应用服务器内部实现一些中间件。通过这种方式，可以在高并发的请求到达 PHP 应用之前就对它们进行处理和过滤。应用服务器中的 Golang 中间件是实现请求速率/频率限制、地理位置过滤及其它的此类中间件的最佳场所。

> 在开始之前请务必先了解如何构建[应用服务器](/zh_CN/framework/application/server.md)。

## 从 Golang 向 PHP 传值

为了实现更复杂的业务逻辑，很多时候需要把中间件的某些值安全地传递到 PHP 应用中，但肯定不能用请求头信息之类的手段来实现这个目的，因为它们的值有可能在客户端被挟持或篡改。你应该使用 `github.com/spiral/roadrunner/service/http/attributes` 包来往请求上设置 PSR-7 标准的属性。

## RoadRunner 服务中间件

举个例子，我们可以检测传入请求是否来自爬虫或机器人，并把检测结果通知给 PHP 应用程序。在这个例子中，我们会用到 `https://github.com/avct/uasurfer` 这个包。

首先创建一个服务来注册我们的中间件：

```golang
package bdetect

import (
	"encoding/json"
	"fmt"
	"github.com/avct/uasurfer"
	rhttp "github.com/spiral/roadrunner/service/http"
	"github.com/spiral/roadrunner/service/http/attributes"
	"net/http"
)

const ID = "bdetect"

type Service struct{}

func (s *Service) Init(rhttp *rhttp.Service) (bool, error) {
	rhttp.AddMiddleware(s.middleware)

	return true, nil
}

func (s *Service) middleware(f http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ua := uasurfer.Parse(r.Header.Get("User-Agent"))
		data, _ := json.Marshal(struct {
			Browser        string `json:"browser"`
			BrowserVersion string `json:"browserVersion"`
			IsBot          bool   `json:"isBot"`
		}{
			Browser: ua.Browser.Name.StringTrimPrefix(),
			BrowserVersion: fmt.Sprintf(
				"%v.%v.%v",
				ua.Browser.Version.Major,
				ua.Browser.Version.Minor,
				ua.Browser.Version.Patch,
			),
			IsBot: ua.IsBot(),
		})

		attributes.Set(r, "bdetect", string(data))

		f(w, r)
	}
}
```

然后在 `main.go` 文件中激活该服务：

```golang
func main() {
    // ...

	rr.Container.Register(bdetect.ID, &bdetect.Service{})

    // 你可以通过 cmd.CLI 来注册更多的命令
	rr.Execute()
}
```

## PHP 中间件

在 PHP 中间件中可以消费上面传入的 PSR-7 属性。使用脚手架命令 `php app.php create:middleware botDetect` 快速创建一个中间件：

```php
namespace App\Middleware;

use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

class BotDetect implements MiddlewareInterface
{
    use PrototypeTrait;

    public function process(Request $request, RequestHandlerInterface $handler): Response
    {
        $ua = json_decode($request->getAttribute('bdetect'));

        if ($ua->isBot) {
            return $this->response->html('Not bots allowed', 401);
        }

        return $handler->handle($request);
    }
}
```

再创建和注册一个引导程序来启用这个中间件：

```php
namespace App\Bootloader;

use App\Middleware\BotDetect;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;

class AppBootloader extends Bootloader
{
    public function boot(HttpBootloader $http): void
    {
        // 解析 json 数据
        $http->addMiddleware(BotDetect::class);
    }
}
```
