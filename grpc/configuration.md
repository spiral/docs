# GRPC - Installation and Configuration
The [GRPC](https://grpc.io/) protocol provides extremely efficient way of cross-service communication for distributed 
application. The public toolkit include instruments to generate client and server code-bases for many languages
allowing the developer to use the most optimal language for the task.

The default [GRPC build](https://github.com/spiral/app-grpc) includes pre-installed version of [spiral/php-grpc](https://github.com/spiral/php-grpc) 
library.

> You will need to generate application key and certificate in order to make GRPC bundle work, see below how to do that.

You can read more about protobuf [here](https://developers.google.com/protocol-buffers/docs/overview).

## Toolkit Installation
It is possible to run PHP application without any dependencies out of the box. However, in order to develop, debug
and extend GRPC project you will need a number of applications. 

### Install Protoc
In order to compile `.proto` files into target language you will have to install the `protoc` compiler.

You can download the latest `protoc` binaries from [https://github.com/protocolbuffers/protobuf/releases](https://github.com/protocolbuffers/protobuf/releases).

### PHP Server Plugin 
Download and install `protoc-gen-php-grpc` from [spiral/php-grpc releases page](https://github.com/spiral/php-grpc/releases). 
This plugin is require to generate service code for your applications.

> Make sure that plugin is available in your PATH.

### Install Protobuf extension (optional)
The `spiral/php-gprc` extension able to work with proto messages using PHP library. However, in order to achieve higher
performance on larger messages make sure to install `protobuf` extension for PHP.

You can compile the extension manually or install it via [PECL](https://pecl.php.net/package/protobuf).

```bash
$ sudo pecl install protobuf
```

> Note, in case of `Segmentation Fault` error try to install different `protobuf` library. We recommend using `3.10.0` 
> at start. 

```bash
$ sudo pecl install protobuf-3.10.0
```

## Component Installation
To install the component in alternative bundles: 

```bash
$ composer require spiral/php-grpc
```

Activate the component using bootloader `Spiral\Bootloader\GRPC\GRPCBootloader`:

```php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\GRPC\GRPCBootloader::class,
    // ...
];
```

### Application Server
To enable the component in application server add the following configuration section:

```yaml
grpc:
  listen: tcp://0.0.0.0:50051
  workers.command: "php app.php" 
    
  # read how to write proto files in next section
  proto: "proto/service.proto"

  # tls configuration is optional
  tls.key:  "app.key"
  tls.cert: "app.crt"
```

### Watch the Service
You can control the memory consumption of GRPC workers same way as for other services:

```yaml
limit:
  services:
    grpc.maxMemory: 100
```

## Generate Certificate
It is possible to run GRPC without any encryption layer. However in other to secure our application we must issue proper
server key and certificate. You can use any normal SSL certificate (for example issued by [https://letsencrypt.org/](https://letsencrypt.org/)) or
issue it manually via [OpenSSL](https://www.openssl.org/).

To issue server key and certificate:

```bash
$ openssl req -newkey rsa:2048 -nodes -keyout app.key -x509 -days 365 -out app.crt
```

> Make sure to use proper domain name or `localhost`, it will required to make your clients connect properly.

## GRPC UI
Use https://github.com/fullstorydev/grpcui to connect to GRPC from the web browser. You will need to install Golang
to compile the application:

```bash
$ go get github.com/fullstorydev/grpcui
$ go install github.com/fullstorydev/grpcui/cmd/grpcui
```

## Example Application
You can install `app-gprc` skeleton application to play with GRPC services:

```bash
$ composer create-project spiral/app-grpc
$ cd app-grpc
$ openssl req -newkey rsa:2048 -nodes -keyout app.key -x509 -days 365 -out app.crt
```

Start the application:

```bash
$ ./spiral serve -v -d
```

To connect to GRPC endpoints from browser use the `` described above:

```bash
$ grpcui -insecure -import-path ./proto/ -proto service.proto localhost:50051
```
