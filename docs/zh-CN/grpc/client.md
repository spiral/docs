# GRPC — 客户端

在[上一篇文章](./service.md)中，我们介绍了如何使用 Spiral 和 `spiral/roadrunner-bridge` 包在 PHP 中创建 gRPC 服务 `Pinger`。在本篇中，我们将展示如何创建一个客户端来与 gRPC 服务通信。

## PHP 客户端

`spiral/roadrunner-bridge` 包默认包含 `spiral/grpc-client` 包，它提供了一种简单高效的方式来与 gRPC 服务通信。

> **注意**
> 在[这里](./configuration.md)可以找到 `protobuf` PHP 扩展和代码生成工具 `buf`、`protoc`、`protoc-gen-php-grpc` 的安装说明。

### 1. 从 `.proto` 文件生成 PHP 接口

要创建客户端，我们需要从 `.proto` 文件生成接口。您可以按照[上一篇](./service.md)中的说明生成这些接口。

### 2. 配置 gRPC 客户端

生成接口后，您需要在 `app/config/grpc.php` 文件中配置 gRPC 客户端。配置包括：

```php
return [
    // 文件 'app/config/grpc.php'
    // ... 其他选项 ...

    'client' => new GrpcClientConfig(
        interceptors: [
            SetTimoutInterceptor::createConfig(6_000),
            RetryInterceptor::createConfig(
                maximumAttempts: 1,
            ),
            ExecuteServiceInterceptors::class,
        ],
        services: [
            new \Spiral\Grpc\Client\Config\ServiceConfig(
                connections: ConnectionConfig::createInsecure('localhost:9001'),
                interfaces: [
                    \GRPC\Pinger\PingerInterface::class,
                ],
            ),
        ],
    ),
];
```

在此配置中：
- `interceptors` - 将应用于所有 gRPC 客户端请求的全局拦截器
- `services` - gRPC 服务列表，包含它们的连接详情和接口

`spiral/grpc-client` 包会在运行时自动为接口生成代理类，因此您无需手动实现它们。

### 3. 客户端使用

现在您可以直接将 gRPC 客户端接口注入到您的代码中，并使用它来调用服务方法：

```php app/src/Endpoint/Console/PingServiceCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Question;
use GRPC\Pinger\PingerInterface;
use Spiral\Console\Command;
use GRPC\Pinger\PingRequest;
use Spiral\Grpc\Client\Exception\GrpcClientException;
use Spiral\RoadRunner\GRPC\Context;

#[AsCommand(name: 'ping')]
final class PingCommand extends Command
{
    #[Argument(description: '要 ping 的 URL')]
    #[Question(question: '提供要 ping 的 URL')]
    private string $url;

    public function __invoke(
        PingerInterface $client,
    ): int {
        try {
            $this->writeln(\sprintf('发送 ping 请求 [%s]...', $this->url));

            $response = $client->ping(
                new Context([]),
                new PingRequest(['url' => $this->url]),
            );

            $this->writeln(\sprintf(
                '响应: 状态码 - %d',
                $response->getStatusCode(),
            ));

        } catch (GrpcClientException $e) {
            $this->writeln(\sprintf(
                '错误: 代码 - %d, 消息 - %s',
                $e->getCode(),
                $e->getMessage(),
            ));
        }

        return self::SUCCESS;
    }
}
```

要使用此命令，从命令行运行：

```terminal
php app.php ping https://google.com
```

这将调用 Pinger 服务并打印响应的 HTTP 状态码。

## 高级客户端功能

`spiral/grpc-client` 包提供了几个高级功能，以增强您的 gRPC 客户端实现。

### 拦截器

拦截器允许您修改或扩展 gRPC 请求的行为。该包包含几个内置拦截器：

- `SetTimoutInterceptor` - 为 gRPC 请求设置超时
- `RetryInterceptor` - 自动重试失败的请求
- `ExecuteServiceInterceptors` - 执行特定于服务的拦截器

您还可以通过实现 `InterceptorInterface` 创建自己的拦截器：

```php
use Spiral\Core\InterceptorInterface;
use Spiral\Core\CoreInterceptorInterface;

final class AuthContextInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly AuthContextInterface $authContext,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $token = $this->authContext->getToken();

        if ($token === null) {
            return $handler->handle($context);
        }

        $metadata = \Spiral\Grpc\Client\Interceptor\Helper::withMetadata($context);
        $metadata['auth-token'] = [$token];

        return $handler->handle(\Spiral\Grpc\Client\Interceptor\Helper::withMetadata($context, $metadata));
    }
}
```

### 多连接

您可以为服务配置多个连接，用于负载均衡或故障转移：

```php
new ServiceConfig(
    connections: [
        ConnectionConfig::createInsecure('service-1:9001'),
        ConnectionConfig::createInsecure('service-2:9001'),
    ],
    interfaces: [
        \GRPC\Pinger\PingerInterface::class,
    ],
)
```

### 安全连接

要创建安全连接，请使用 `TlsConfig` 类：

```php
new ConnectionConfig(
    address: 'secure-service:9001',
    tls: new TlsConfig(
        privateKey: '/my-project.key',
        certChain: '/my-project.pem',
    ),
),
```

## Golang 客户端

GRPC 允许您在任何支持的语言中创建客户端 SDK。要在 Golang 中生成客户端，首先安装 GRPC 工具包：

### 1. 安装必要的依赖

要在 Go 中使用 gRPC，您需要安装必要的依赖。您可以使用 Go 包管理器进行安装：

```bash
go get -u google.golang.org/grpc
go get -u github.com/golang/protobuf/protoc-gen-go
```

> **查看更多**
> 在[这里](https://grpc.io/docs/tutorials/basic/go/)阅读更多关于如何创建 Golang GRPC 客户端和服务器的信息。

### 2. 编译 `.proto` 文件

接下来，您需要将 `.proto` 文件编译为 Go 代码。您可以使用 protoc 编译器和 Go 插件进行此操作：

```bash
protoc -I proto/ proto/pinger.proto --go_out=plugins=grpc:pinger
```

这将生成一个 `pinger.pb.go` 文件，其中包含 `.proto` 文件中定义的服务和消息的 Go 类。

> **注意**
> 注意 `pinger.proto` 中的 `package` 名称。

### 3. 创建客户端

现在，您可以为 Pinger 服务创建一个 Go 客户端。以下是一个调用 Pinger 服务的 `ping()` 方法并将 HTTP 状态码打印到控制台的客户端示例：

```golang
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"pinger"
)

func main() {
	// 设置与服务器的连接
	conn, err := grpc.Dial("127.0.0.1:9001", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("未连接: %v", err)
	}
	
	defer conn.Close()

	client := pinger.NewPingerClient(conn)

	// 调用 ping 方法
	response, err := client.Ping(context.Background(), &pinger.PingRequest{
		Url: "https://google.com",
	})

	if err != nil {
		log.Fatalf("调用 ping 出错: %v", err)
	}

	// 打印 HTTP 状态码
	fmt.Println(response.StatusCode)
}
```

您可以使用 Go 命令运行此客户端：

```bash
go run main.go
```

### 传递元数据

在服务器和客户端之间传递元数据：

```golang
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"google.golang.org/grpc/metadata"
	"pinger"
)

func main() {
	// 设置与服务器的连接
	conn, err := grpc.Dial("127.0.0.1:9001", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("未连接: %v", err)
	}
	
	defer conn.Close()

	client := pinger.NewPingerClient(conn)
	
	// 将值附加到服务器
	ctx := metadata.AppendToOutgoingContext(context.Background(), "client-key", "client-value")

	var header metadata.MD
	
	// 调用 ping 方法
	response, err := client.Ping(ctx, &pinger.PingRequest{
		Url: "https://google.com",
	}, grpc.Header(&header))

	if err != nil {
		log.Fatalf("调用 ping 出错: %v", err)
	}

	// 打印 HTTP 状态码
	fmt.Println(response.StatusCode)
}
```

> **查看更多**
> 在[这里](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md)阅读更多关于在 Golang 中处理元数据的信息。