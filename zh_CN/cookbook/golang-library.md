# 速查手册 - Golang 服务集成

在 Spiral 中，可以通过引入 PHP 或者 Golang 的库或组件来扩展应用的功能。对于 PHP 的库，只需要执行 `composer require` 命令
来引入即可，而对于 Golang 的库，则需要自行构建定制版的[应用服务器](/framework/application-server.md).

> 一个直观的示例：官方文档站的全文检索功能就是通过集成到 Spiral 应用的 [blevesearch](https://github.com/blevesearch/bleve) 来实现的。

In this tutorial, we will show how to integrate https://github.com/russross/blackfriday library to the application server and
write PHP SDK for your application. 
在接下来的文档中，我们就演示一下如何把 [blackfriday](https://github.com/russross/blackfriday) 这个库集成到应用服务器，然后在应用中编写调用该组件的 PHP SDK.

> 温馨提示，本文的内容需要读者熟悉 Golang 编程语言。

## RoadRunner 服务

首先，要引入必要的 go 模块依赖：

```bash
$ go get github.com/russross/blackfriday
```

由于这个服务不需要任何配置项，所以可以把所有的服务代码放到单一的 `markdown/service.go` 文件中：

```go
package markdown

import (
	"github.com/russross/blackfriday"
	"github.com/spiral/roadrunner/service/rpc"
)

const ID = "markdown"

// Service 是将注册到 app server 中的服务
type Service struct{}

func (s *Service) Init(rpc *rpc.Service) (bool, error) {
	if err := rpc.Register(ID, &rpcService{}); err != nil {
		return false, err
	}

	return true, nil
}

// rpcService 是通过 RPC 调用的服务
type rpcService struct{}

func (s *rpcService) Convert(input []byte, output *[]byte) error {
	*output = blackfriday.Run(input)
	return nil
}
```

编写完服务之后，需要在 `main.go` 文件中注册这个新的服务：

```go
rr.Container.Register(markdown.ID, &markdown.Service{})
```

> 需要了解有关 RoadRunner 服务的更多信息，可以点击[这里](https://roadrunner.dev/docs/beep-beep-service)。

构建并启动你的应用服务器，即可激活刚刚创建的服务。

## PHP SDK

在 Spiral 应用中，通过 `Spiral\Goridge\RPC` 立刻就可以调用刚才新创建的服务：

```php
use Spiral\Goridge\RelayInterface;
use Spiral\Goridge\RPC;

// ...

public function index(RPC $rpc)
{
    return $rpc->call(
        'markdown.Convert',
        file_get_contents('README.md'),
        RelayInterface::PAYLOAD_RAW // 使用 []byte 类型的 payload
    );
}
```

不过建议最好是把 RPC 调用封装成合适的服务代码：

```php
use Spiral\Core\Container\SingletonInterface;
use Spiral\Goridge\Exceptions\ServiceException;
use Spiral\Goridge\RelayInterface;
use Spiral\Goridge\RPC;

class Blackfriday implements SingletonInterface
{
    private $rpc;

    public function __construct(RPC $rpc)
    {
        $this->rpc = $rpc;
    }

    public function convert(string $markdown): string
    {
        try {
            return $this->rpc->call('markdown.Convert', $markdown, RelayInterface::PAYLOAD_RAW);
        } catch (ServiceException $e) {
            throw new \RuntimeException("Unable to convert markdown", $e->getCode(), $e);
        }
    }
}
```

然后在业务代码中使用这个封装好的服务：

```php
public function index(Blackfriday $bf)
{
    return $bf->convert(file_get_contents('README.md'));
}
```

## 性能

在这个示例中用来转换 markdown 代码的 blackfriday 库，它的性能是 PHP 中经典的 [parsedown](https://github.com/erusev/parsedown) 组件的两倍。
不仅如此，把 RPC 通讯改为 unix sockets 通讯，还能获得更多的性能提升。具体可以参考 RoadRunner 文档的 **性能优化（Performance Tuning）** 章节.

> 毫无疑问，这样的通讯方式也不是完全没有成本的。在使用时务必做好套接字连接速度与计算复杂性之间的平衡。
