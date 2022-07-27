# GRPC - Installation and Configuration

> **Note**
> The documentation page contains outdated information relevant to RoadRunner 1.x only.

The [GRPC](https://grpc.io/) protocol provides an extremely efficient way of cross-service communication for distributed applications. The public toolkit includes instruments to generate client and server code-bases for many languages
allowing the developer to use the most optimal language for the task.

The default [GRPC build](https://github.com/spiral/app-grpc) includes pre-installed version of [spiral/php-grpc](https://github.com/spiral/php-grpc)
library.

> You will need to generate an application key and certificate to make GRPC bundle work, see below how to do that.

You can read more about protobuf [here](https://developers.google.com/protocol-buffers/docs/overview).

## Toolkit Installation

It is possible to run a PHP application without any dependencies out of the box. However, to develop, debug, and extend the GRPC project, you will need several instruments.

### Install Protoc

To compile `.proto` files into the target language, you will have to install the `protoc` compiler.

You can download the latest `protoc` binaries from [https://github.com/protocolbuffers/protobuf/releases](https://github.com/protocolbuffers/protobuf/releases).

### PHP Server Plugin

Download and install `protoc-gen-php-grpc` from [spiral/php-grpc releases page](https://github.com/spiral/php-grpc/releases). 
This plugin is required to generate a service code for your applications.

> Make sure that the plugin is available in your PATH.

### Install Protobuf extension (optional)

To achieve higher performance on larger messages, make sure to install a `protobuf` extension for PHP.

You can compile the extension manually or install it via [PECL](https://pecl.php.net/package/protobuf).

```bash
sudo pecl install protobuf
```

> Note, in case of `Segmentation Fault` error, try to install different `protobuf` library. We recommend using `3.10.0`  at the start. 

```bash
sudo pecl install protobuf-3.10.0
```

## Component Installation

To install the component in alternative bundles:

```bash
composer require spiral/php-grpc
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

  # read how to write proto files in the next section
  proto: "proto/service.proto"

  # tls configuration is optional
  tls.key:  "app.key"
  tls.cert: "app.crt"

  # max send limit (MB)
  MaxSendMsgSize: 50
  # max receive limit (MB)
  MaxRecvMsgSize: 50
  # MaxConnectionIdle is a duration for the amount of time after which an
  # idle connection would be closed by sending a GoAway. Idleness duration is
  # defined since the most recent time the number of outstanding RPCs became
  # zero or the connection establishment.
  # default (if set to zero) is infinity.
  MaxConnectionIdle: 0s
  # MaxConnectionAge is a duration for the maximum amount of time a
  # connection may exist before it will be closed by sending a GoAway. A
  # random jitter of +/-10% will be added to MaxConnectionAge to spread out
  # connection storms.
  # default (if set to zero) is infinity.
  MaxConnectionAge: 0s
  # MaxConnectionAgeGrace is an additive period after MaxConnectionAge after
  # which the connection will be forcibly closed.
  # default (if set to zero) is infinity.
  MaxConnectionAgeGrace: 0s
  # MaxConnectionAgeGrace is an additive period after MaxConnectionAge after
  # which the connection will be forcibly closed.
  # default (if set to zero) is 10
  MaxConcurrentStreams: 10
  # After a duration of this time if the server doesn't see any activity it
  # pings the client to see if the transport is still alive.
  # If set below 1s, a minimum value of 1s will be used instead.
  PingTime: 1s
  # After having pinged for keepalive check, the server waits for a duration
  # of Timeout and if no activity is seen even after that the connection is
  # closed.
  # default is 20s
  Timeout: 200s
```

### Watch the Service

You can control the memory consumption of GRPC workers the same way as for other services:

```yaml
limit:
  services:
    grpc.maxMemory: 100
```

## Generate Certificate

It is possible to run GRPC without any encryption layer. However, in other to secure our application, we must issue proper
server key and certificate. You can use any normal SSL certificate (for example issued by [https://letsencrypt.org/](https://letsencrypt.org/)) or
issue it manually via [OpenSSL](https://www.openssl.org/).

To issue server key and certificate:

```bash
openssl req -newkey rsa:2048 -nodes -keyout app.key -x509 -days 365 -out app.crt
```

> Make sure to use a proper domain name or `localhost`, it will be required to make your clients connect properly.

## GRPC UI

Use https://github.com/fullstorydev/grpcui to connect to GRPC from the web browser. You will need to install Golang
to compile the application:

```bash
go get github.com/fullstorydev/grpcui
go install github.com/fullstorydev/grpcui/cmd/grpcui
```

## Example Application

You can install `app-grpc` skeleton application to play with GRPC services:

```bash
composer create-project spiral/app-grpc
cd app-grpc
openssl req -newkey rsa:2048 -nodes -keyout app.key -x509 -days 365 -out app.crt
```

Start the application:

```bash
./spiral serve
```

To connect to GRPC endpoints from browser use the `` described above:

```bash
grpcui -insecure -import-path ./proto/ -proto service.proto localhost:50051
```
