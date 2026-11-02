# GRPC — Getting started

Using REST APIs for communication between microservices has been a common approach for many years. However, REST APIs
have some limitations and challenges that can impact the performance, scalability, and reliability of your system.

Switching from REST APIs to gRPC can help address these problems. The [gRPC](https://grpc.io/) protocol provides an
extremely efficient way of cross-service communication for distributed applications. The public toolkit includes
instruments to generate client and server code-bases for many languages allowing developers to use the most optimal
language for their task.

> **See more**
> You can read more about protobuf [here](https://developers.google.com/protocol-buffers/docs/overview).

## Toolkit Installation

It is possible to run a PHP application without any dependencies out of the box. However, to develop, debug, and extend
the GRPC project, you will need several instruments.

### Install Code Generation Tools

The `grpc:generate` console command compiles `.proto` files into PHP code with [buf](https://buf.build). It runs locally
and needs three binaries:

- `buf` — runs the code generation;
- `protoc` — the protobuf compiler: buf runs its built-in PHP generator to create the message classes;
- `protoc-gen-php-grpc` — the RoadRunner plugin that creates the service interfaces.

You can download them with the `roadrunner/cli` package or with [DLoad](https://github.com/php-internal/dload).
Both put the binaries into the current directory, so run them from the root directory of your project, where
`grpc:generate` looks for the binaries.

:::: tabs

::: tab RoadRunner CLI

The `get-grpc-tools` command of the `roadrunner/cli` package (since version 3.1) downloads all three binaries:

```terminal
composer require --dev roadrunner/cli
./vendor/bin/rr get-grpc-tools
```

The `protoc` version follows the protobuf runtime of your project (the `protobuf` extension or the `google/protobuf`
package), because the generated code is only guaranteed to work with the runtime of the same `protoc` major version.
Use the `--buf`, `--protoc` and `--plugin` (for `protoc-gen-php-grpc`) options to set other version constraints,
e.g. `--protoc=^33.0`, and the `--location` option to download the binaries into another directory.

> **Note**
> The `download-protoc-binary` command is deprecated: it downloads only the `protoc-gen-php-grpc` plugin.
> Use `get-grpc-tools` instead.
:::

::: tab DLoad

Install DLoad:

```terminal
composer require --dev internal/dload
```

Add the binaries to the `dload.xml` file in the root directory of your project:

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

And download them:

```terminal
./vendor/bin/dload get
```

Pick the `protoc` major version that matches your `google/protobuf` package: `google/protobuf` 5.36 takes `protoc`
36.x.
:::

::::

The binaries are built for your operating system, so do not commit them. Add them to the `.gitignore` file:

```text .gitignore
/buf
/buf.exe
/protoc
/protoc.exe
/protoc-gen-php-grpc
/protoc-gen-php-grpc.exe
```

### Install Protobuf extension (optional)

To achieve better performance with larger messages, make sure to install the `protobuf` extension for PHP.

You can compile the extension manually or install it via [PECL](https://pecl.php.net/package/protobuf).

```bash
sudo pecl install protobuf
```

**In case of the `Segmentation Fault` error, try to install another `protobuf` library. We recommend using `3.10.0` at
the start.**

```bash
sudo pecl install protobuf-3.10.0
```

## Package Installation

The `spiral/roadrunner-bridge` package allows you to use RoadRunner
[gRPC plugin](https://docs.roadrunner.dev/docs/grpc/grpc) with Spiral. This package provides tools to generate proto 
files, client code, and a bootloader for your application.

At first, you need to install the [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) package.

Once the package is installed, activate the component using 
bootloader `Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader`:

:::: tabs

::: tab Using method

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

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader::class,
    \Spiral\RoadRunnerBridge\Bootloader\CommandBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

## Configuration

Create the config file `app/config/grpc.php` to configure the code generation:

```php app/config/grpc.php
return [
    /**
     * Paths to the buf, protoc and protoc-gen-php-grpc binaries.
     * Null looks for `<root>/<name>` (`<name>.exe` on Windows), then in PATH.
     */
    'bufPath' => null,
    'protocPath' => null,
    'pluginPath' => null,

    /**
     * Path to your own buf.gen.yaml template. Null uses a template with the binaries above.
     */
    'bufTemplate' => null,

    /**
     * The root directory of the proto files: all imports are resolved from it.
     */
    'servicesBasePath' => directory('root') . 'proto',

    /**
     * Proto files that the grpc:generate console command compiles into PHP.
     */
    'services' => [
        directory('root') . 'proto/calculator.proto',
    ],

    /**
     * The path where the generated files are stored.
     */
    'generatedPath' => directory('root') . 'generated',

    /**
     * The base namespace of the generated classes. It is cut from the paths of the generated files.
     */
    'namespace' => 'GRPC',

    /**
     * Generators that receive the generated files after the compilation.
     */
    'generators' => [],
];
```

Relative paths in this config are resolved from the root directory of the application.

> **Note**
> The same config file contains the `interceptors` and `client` sections. Read more about them in the
> [GRPC — Interceptors](./interceptors.md) and [GRPC — Client](./client.md) sections.

### Binaries

The `grpc:generate` command looks for each binary in the following order:

1. The path from the config: `bufPath`, `protocPath` or `pluginPath`.
2. `buf`, `protoc` or `protoc-gen-php-grpc` (with the `.exe` extension on Windows) in the root directory of the
   application, where `rr get-grpc-tools` and `dload get` put them.
3. The directories from the `PATH` environment variable.

If a binary is not found, the command prints an error and exits with code `1`.

### Imports

`servicesBasePath` is the buf input: all `import` directives are resolved from it, and every file in `services` must
be inside it. For example, `proto/shop/order.proto` imports `proto/common/money.proto` as follows:

```proto proto/shop/order.proto
import "common/money.proto";
```

Each service is generated together with the other proto files in its directory and subdirectories. The proto files it
imports from `servicesBasePath` are generated too, because the generated code uses their classes. Imports starting with
`google/`, such as the Well-Known Types (`google/protobuf/timestamp.proto`) or the Google APIs (`google/api/http.proto`),
are not generated: their classes come with the `google/protobuf` and `google/common-protos` packages.

Without `servicesBasePath`, the directory of each service is a separate input that its imports are resolved from.

> **Warning**
> Before `spiral/roadrunner-bridge` 5.0, imports were also resolved from the directory of the importing file.
> Rewrite `import "b.proto"` in `foo/a.proto` as `import "foo/b.proto"`. The `binaryPath` option is replaced with
> `pluginPath`.

### Custom buf Template

Set `bufTemplate` to use your own [buf.gen.yaml](https://buf.build/docs/configuration/v2/buf-gen-yaml/) (version `v2`),
for example, to enable managed mode, add more plugins, or use dependencies from the Buf Schema Registry with your own
`buf.yaml`:

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

The template is used as is: its plugins and their `out` directories decide where the files go, so `generatedPath` and
`namespace` do not relocate them, and `protocPath` and `pluginPath` are not used. The imported proto files are not added
to the generation either: the template decides that itself with the `include_imports` option. buf runs in the root
directory of the application, so relative paths in the template are resolved from it. The `generators` receive the
files that the command created or updated under `generatedPath`.


## Application Server

To enable the component in the RoadRunner application server, add the following configuration section:

```yaml .rr.yaml
server:
  command: "php app.php"

grpc:
  # GRPC address to listen
  listen: "tcp://0.0.0.0:9001"
  proto:
    - "proto/calculator.proto"
```

> **Note**
> Full documentation about configuration the RoadRunner gRPC
> plugin [here](https://docs.roadrunner.dev/docs/grpc/grpc).

## Example Application

There is a good example [**Demo ticket booking system**](https://github.com/spiral/ticket-booking) application built 
on Spiral, that follows the principles of microservices and allows developers to create reusable, independent, and 
easy-to-maintain components.

In this demo application, you can find an example of using RoadRunner's gRPC plugin to create and consume gRPC services.

Overall, our demo ticket booking system is a great example of how Spiral and other tools can be used to build 
a modern and efficient application. We hope you have a blast using it and learning more about the capabilities. 

**Happy (fake) ticket shopping!**

## Generate Certificate

It is possible to run GRPC without any encryption layer. However, to secure our application, we must issue
the proper server key and certificate. You can use any regular SSL certificate (for example, one issued
by [https://letsencrypt.org/](https://letsencrypt.org/)) or issue it manually via [OpenSSL](https://www.openssl.org/).

To issue a server key and certificate:

```bash
openssl req -newkey rsa:2048 -nodes -keyout app.key -x509 -days 365 -out app.crt
```

> **Note**
> Make sure to use a proper domain name or `localhost`, it will be required to make your clients connect properly.
