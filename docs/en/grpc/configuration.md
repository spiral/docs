# GRPC — Installation and Configuration

Using REST APIs for communication between microservices has been a common approach for many years. However, REST APIs
have some limitations and challenges that can impact the performance, scalability, and reliability of your system.

Switching from REST APIs to gRPC can help address these problems. The [gRPC](https://grpc.io/) protocol provides an
extremely efficient way of cross-service communication for distributed applications. The public toolkit includes
instruments to generate client and server code-bases for many languages allowing developers to use the most optimal
language for their task.

> **Note**
> You can read more about protobuf [here](https://developers.google.com/protocol-buffers/docs/overview).

## Toolkit Installation

It is possible to run a PHP application without any dependencies out of the box. However, to develop, debug, and extend
the GRPC project, you will need several instruments.

### Install Protoc

To compile `.proto` files into the target language, you will have to install the `protoc` compiler.

> **Note**
> You can download the latest `protoc` binaries from[https://github.com/protocolbuffers/protobuf/releases.

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
[gRPC plugin](https://roadrunner.dev/docs/app-server-grpc/2.x/en) with the Spiral Framework. This package provides
tools to generate proto files, client code, and a bootloader for your application.

To install the `spiral/roadrunner-bridge` package, use the following command:

```bash
composer require spiral/roadrunner-bridge
```

Activate the component using bootloader `Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader`:

```php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader::class,
    \Spiral\RoadRunnerBridge\Bootloader\CommandBootloader::class,
    // ...
];
```

After the package installed you will need to download the `protoc-gen-php-grpc`. It is a plugin for the `protoc` tool
that generates PHP code for gRPC services.

> **Note**
> To download the binary, you can use `download-protoc-binary` console command provided by the
> `spiral/roadrunner-cli` package. This command will download the latest version of the `protoc-gen-php-grpc` binary and
> save it to the root directory of your project.

```bash
./vendor/bin/rr download-protoc-binary
```

## Configuration

Create the config file `app/config/grpc.php` if you want to configure generate service classes:

```php
<?php

declare(strict_types=1);

return [
    /**
     * The path where generated DTO (Data Transfer Object) files will be stored.
     */
    'generatedPath' => directory('app') . '/GRPC',

    /**
     * The base namespace for the generated proto files.
     */
    'namespace' => '\App\GRPC',

    /**
     * The root dir for all proto files, where imports will be searched.
     */
    'servicesBasePath' => directory('root') . '/proto',

    /**
     * The path to the protoc-gen-php-grpc library.
     */
    'binaryPath' => directory('root').'/bin/protoc-gen-php-grpc',

    /**
     * An array of paths to proto files that should be compiled into PHP by the grpc:generate console command.
     */
    'services' => [
        // directory('root').'proto/calculator.proto',
    ],
];
```

## Application Server

To enable the component in the RoadRunner application server, add the following configuration section:

```yaml
version: "2.7"

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
> plugin [here](https://roadrunner.dev/docs/app-server-grpc/2.x/en).

## Example Application

There is a good example [**Demo ticket booking system**](https://github.com/spiral/ticket-booking) application built 
on the Spiral Framework, which is a high-performance PHP framework that follows the principles of microservices and 
allows developers to create reusable, independent, and easy-to-maintain components.

In this demo application, you can find an example of using RoadRunner's gRPC plugin to create and consume gRPC services.

Overall, our demo ticket booking system is a great example of how Spiral Framework and other tools can be used to build 
a modern and efficient application. We hope you have a blast using it and learning more about the capabilities of 
Spiral Framework and the other tools we've used. 

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
