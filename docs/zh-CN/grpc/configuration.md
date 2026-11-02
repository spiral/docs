# GRPC — 快速入门

长期以来，使用 REST APIs 在微服务之间进行通信是一种常见的方法。然而，REST APIs 存在一些限制和挑战，这些限制和挑战会影响系统的性能、可扩展性和可靠性。

从 REST APIs 切换到 gRPC 可以帮助解决这些问题。[gRPC](https://grpc.io/) 协议为分布式应用程序提供了极其高效的跨服务通信方式。公共工具包包括为许多语言生成客户端和服务器代码库的工具，允许开发人员为他们的任务使用最优化语言。

> **了解更多**
> 你可以在[这里](https://developers.google.com/protocol-buffers/docs/overview)阅读更多关于 protobuf 的信息。

## 工具包安装

无需任何依赖即可开箱即用地运行 PHP 应用程序。但是，为了开发、调试和扩展 GRPC 项目，你将需要几个工具。

### 安装代码生成工具

`grpc:generate` 命令行命令使用 [buf](https://buf.build) 将 `.proto` 文件编译成 PHP 代码。它在本地运行，需要三个二进制文件：

- `buf` — 执行代码生成；
- `protoc` — protobuf 编译器：buf 运行它内置的 PHP 生成器来创建消息类；
- `protoc-gen-php-grpc` — RoadRunner 插件，用于创建服务接口。

你可以使用 `roadrunner/cli` 包或 [DLoad](https://github.com/php-internal/dload) 下载它们。两者都会将二进制文件放到当前目录中，因此请在项目的根目录中运行它们，`grpc:generate` 会在那里查找这些二进制文件。

:::: tabs

::: tab RoadRunner CLI

`roadrunner/cli` 包（从 3.1 版本开始）的 `get-grpc-tools` 命令会下载全部三个二进制文件：

```terminal
composer require --dev roadrunner/cli
./vendor/bin/rr get-grpc-tools
```

`protoc` 的版本取决于项目的 protobuf 运行时（`protobuf` 扩展或 `google/protobuf` 包），因为生成的代码只保证能与相同 `protoc` 主版本的运行时一起工作。使用 `--buf`、`--protoc` 和 `--plugin`（用于 `protoc-gen-php-grpc`）选项设置其他版本约束，例如 `--protoc=^33.0`，使用 `--location` 选项将二进制文件下载到其他目录。

> **注意**
> `download-protoc-binary` 命令已弃用：它只下载 `protoc-gen-php-grpc` 插件。请改用 `get-grpc-tools`。
:::

::: tab DLoad

安装 DLoad：

```terminal
composer require --dev internal/dload
```

将二进制文件添加到项目根目录中的 `dload.xml` 文件：

```xml dload.xml
<?xml version="1.0"?>
<dload xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="vendor/internal/dload/dload.xsd">
    <actions>
        <download software="buf" version="^1.50"/>
        <download software="protoc" version="^36.0"/>
        <download software="protoc-gen-php-grpc" version="^3.0"/>
    </actions>
</dload>
```

然后下载它们：

```terminal
./vendor/bin/dload get
```

选择与你的 `google/protobuf` 包相匹配的 `protoc` 主版本：`google/protobuf` 5.36 对应 `protoc` 36.x。
:::

::::

这些二进制文件是为你的操作系统构建的，因此不要提交它们。将它们添加到 `.gitignore` 文件中：

```text .gitignore
/buf
/buf.exe
/protoc
/protoc.exe
/protoc-gen-php-grpc
/protoc-gen-php-grpc.exe
```

### 安装 Protobuf 扩展（可选）

为了使用更大的消息实现更好的性能，请确保安装 PHP 的 `protobuf` 扩展。

你可以手动编译该扩展，或者通过 [PECL](https://pecl.php.net/package/protobuf) 安装它。

```bash
sudo pecl install protobuf
```

**如果出现 `Segmentation Fault` 错误，请尝试安装另一个 `protobuf` 库。我们建议一开始使用 `3.10.0`。**

```bash
sudo pecl install protobuf-3.10.0
```

## 包安装

`spiral/roadrunner-bridge` 包允许你将 RoadRunner 的
[gRPC 插件](https://docs.roadrunner.dev/docs/grpc/grpc) 与 Spiral 结合使用。该包提供了生成 proto 文件、客户端代码和应用程序引导器的工具。

首先，你需要安装 [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) 包。

安装该包后，使用引导器 `Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader` 激活该组件：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader::class,
        \Spiral\RoadRunnerBridge\Bootloader\CommandBootloader::class,
        // ...
    ];
}
```

在[框架 — 引导器](../framework/bootloaders.md) 部分阅读更多关于引导器的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader::class,
    \Spiral\RoadRunnerBridge\Bootloader\CommandBootloader::class,
    // ...
];
```

在[框架 — 引导器](../framework/bootloaders.md) 部分阅读更多关于引导器的信息。
:::

::::

## 配置

创建配置文件 `app/config/grpc.php` 来配置代码生成：

```php app/config/grpc.php
return [
    /**
     * buf、protoc 和 protoc-gen-php-grpc 二进制文件的路径。
     * 为 null 时，先查找 `<root>/<name>`（Windows 上为 `<name>.exe`），然后在 PATH 中查找。
     */
    'bufPath' => null,
    'protocPath' => null,
    'pluginPath' => null,

    /**
     * 你自己的 buf.gen.yaml 模板的路径。为 null 时，使用基于上述二进制文件的模板。
     */
    'bufTemplate' => null,

    /**
     * 所有 proto 文件的根目录：所有导入都从这里解析。
     */
    'servicesBasePath' => directory('root') . 'proto',

    /**
     * 由 grpc:generate 命令行命令编译成 PHP 的 proto 文件。
     */
    'services' => [
        directory('root') . 'proto/calculator.proto',
    ],

    /**
     * 存放生成文件的路径。
     */
    'generatedPath' => directory('root') . 'generated',

    /**
     * 生成类的基础命名空间。它会从生成文件的路径中去掉。
     */
    'namespace' => 'GRPC',

    /**
     * 编译后接收生成文件的生成器。
     */
    'generators' => [],
];
```

此配置中的相对路径从应用程序的根目录解析。

> **注意**
> 同一个配置文件还包含 `interceptors` 和 `client` 部分。在 [GRPC — 拦截器](./interceptors.md) 和
> [GRPC — 客户端](./client.md) 部分阅读更多相关信息。

### 二进制文件

`grpc:generate` 命令按以下顺序查找每个二进制文件：

1. 配置中的路径：`bufPath`、`protocPath` 或 `pluginPath`。
2. 应用程序根目录中的 `buf`、`protoc` 或 `protoc-gen-php-grpc`（Windows 上带 `.exe` 扩展名），`rr get-grpc-tools` 和 `dload get` 会将它们放在那里。
3. `PATH` 环境变量中的目录。

如果找不到二进制文件，该命令会打印错误并以代码 `1` 退出。

### 导入

`servicesBasePath` 是 buf 的输入：所有 `import` 指令都从这里解析，并且 `services` 中的每个文件都必须位于其中。例如，`proto/shop/order.proto` 这样导入 `proto/common/money.proto`：

```proto proto/shop/order.proto
import "common/money.proto";
```

每个服务会与其所在目录及子目录中的其他 proto 文件一起生成。它从 `servicesBasePath` 导入的 proto 文件也会被生成，因为生成的代码会使用它们的类。以 `google/` 开头的导入，例如 Well-Known Types（`google/protobuf/timestamp.proto`）或 Google APIs（`google/api/http.proto`），不会被生成：它们的类由 `google/protobuf` 和 `google/common-protos` 包提供。

如果没有设置 `servicesBasePath`，每个服务所在的目录都是一个单独的输入，其导入从该目录解析。

> **警告**
> 在 `spiral/roadrunner-bridge` 5.0 之前，导入也会从导入文件所在的目录解析。请将 `foo/a.proto` 中的 `import "b.proto"` 改写为 `import "foo/b.proto"`。`binaryPath` 选项已被 `pluginPath` 取代。

### 自定义 buf 模板

设置 `bufTemplate` 以使用你自己的 [buf.gen.yaml](https://buf.build/docs/configuration/v2/buf-gen-yaml/)（`v2` 版本），例如，启用托管模式（managed mode）、添加更多插件，或者配合你自己的 `buf.yaml` 使用 Buf Schema Registry 中的依赖：

```yaml buf.gen.yaml
version: v2
plugins:
  - protoc_builtin: php
    protoc_path: ./protoc
    out: generated
  - local: ./protoc-gen-php-grpc
    out: generated
```

```php app/config/grpc.php
return [
    'bufTemplate' => 'buf.gen.yaml',
    // ...
];
```

模板会按原样使用：由它的插件及其 `out` 目录决定文件的位置，因此 `generatedPath` 和 `namespace` 不会改变文件的位置，`protocPath` 和 `pluginPath` 也不会被使用。导入的 proto 文件也不会被加入生成：模板通过 `include_imports` 选项自行决定。buf 在应用程序的根目录中运行，因此模板中的相对路径从根目录解析。`generators` 会接收该命令在 `generatedPath` 下创建或更新的文件。

## 应用服务器

要在 RoadRunner 应用服务器中启用该组件，请添加以下配置节：

```yaml .rr.yaml
server:
  command: "php app.php"

grpc:
  # 要监听的 GRPC 地址
  listen: "tcp://0.0.0.0:9001"
  proto:
    - "proto/calculator.proto"
```

> **注意**
> 关于配置 RoadRunner gRPC 插件的完整文档[在这里](https://docs.roadrunner.dev/docs/grpc/grpc)。

## 示例应用程序

有一个很好的示例 [**Demo ticket booking system（演示票务预订系统）**](https://github.com/spiral/ticket-booking) 应用程序，它基于 Spiral 构建，遵循微服务原则，并允许开发人员创建可重用、独立且易于维护的组件。

在这个演示应用程序中，你可以找到一个使用 RoadRunner 的 gRPC 插件创建和使用 gRPC 服务的示例。

总的来说，我们的演示票务预订系统是 Spiral 和其他工具如何用于构建现代化和高效应用程序的一个很好的例子。 我们希望你使用它并了解更多关于其功能的知识。

**祝你（假）购票愉快！**

## 生成证书

无需任何加密层即可运行 GRPC。但是，为了保护我们的应用程序，我们必须颁发适当的服务器密钥和证书。 你可以使用任何常规的 SSL 证书（例如，由 [https://letsencrypt.org/](https://letsencrypt.org/) 颁发的证书）或通过 [OpenSSL](https://www.openssl.org/) 手动颁发它。

要颁发服务器密钥和证书：

```bash
openssl req -newkey rsa:2048 -nodes -keyout app.key -x509 -days 365 -out app.crt
```

> **注意**
> 确保使用适当的域名或 `localhost`，这将是你的客户端正确连接所必需的。
