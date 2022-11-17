# GRPC - Installation and Configuration

The [GRPC](https://grpc.io/) protocol provides an extremely efficient way of cross-service communication for distributed
applications. The public toolkit includes instruments to generate client and server code-bases for many languages
allowing developers to use the most optimal language for their task.

The default [GRPC build](https://github.com/spiral/app-grpc) includes a pre-installed version
of the [spiral/php-grpc](https://github.com/spiral/php-grpc) library.

> **Note**
> You will need to generate an application key and certificate to make the GRPC bundle work, see below how to do that.

You can read more about protobuf [here](https://developers.google.com/protocol-buffers/docs/overview).

## Toolkit Installation

It is possible to run a PHP application without any dependencies out of the box. However, to develop, debug, and extend the GRPC project, you will need several instruments.

### Install Protoc

To compile `.proto` files into the target language, you will have to install the `protoc` compiler.

You can download the latest `protoc` binaries
from [https://github.com/protocolbuffers/protobuf/releases](https://github.com/protocolbuffers/protobuf/releases).

### PHP Server Plugin

Download and install `protoc-gen-php-grpc`
from [roadrunner-server/roadrunner releases page](https://github.com/roadrunner-server/roadrunner/releases).

This plugin is required to generate service code for your applications.

> **Note**
> Make sure that the plugin is available in your PATH.

### Install Protobuf extension (optional)

To achieve better performance with larger messages, make sure to install the `protobuf` extension for PHP.

You can compile the extension manually or install it via [PECL](https://pecl.php.net/package/protobuf).

```bash
sudo pecl install protobuf
```

> **Note**
> In case of the `Segmentation Fault` error, try to install another `protobuf` library. We recommend using `3.10.0`
> at the start.

```bash
sudo pecl install protobuf-3.10.0
```

## Component Installation

To install the component in alternative bundles:

```bash
composer require spiral/roadrunner-bridge
```

Activate the component using bootloader `Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader`:

```php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader::class,
    // ...
];
```

## Configuration

Create the config file `app/config/grpc.php` if you want to configure generate service classes:

```php
<?php

declare(strict_types=1);

return [
    /**
     * Path to protoc-gen-php-grpc library.
     * Default: null 
     */
    'binaryPath' => null,
    // 'binaryPath' => __DIR__.'/../../protoc-gen-php-grpc',

    'services' => [
        __DIR__.'/../../proto/echo.proto',
    ],
];
```

## Application Server

To enable the component in the application server, add the following configuration section:

```yaml
grpc:
  listen: tcp://0.0.0.0:50051
  workers.command: "php app.php"

  # read how to write proto files in the next section
  proto: "proto/service.proto"
```

> **Note**
> Full documentation about configuration the RoadRunner `grpc`
> plugin [here](https://roadrunner.dev/docs/app-server-grpc/2.x/en).

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

## Example Application

You can install the `app-grpc` skeleton application to play with the GRPC services:

```bash
composer create-project spiral/app-grpc
cd app-grpc
openssl req -newkey rsa:2048 -nodes -keyout app.key -x509 -days 365 -out app.crt
```

Start the application:

```bash
./rr serve
```
