# Getting started — Application Server

RoadRunner is a high-performance application server designed to handle a wide range of request types, including HTTP,
gRPC, TCP, Queue Job consuming, and Temporal. It operates by running workers only once when it is initiated and then
directing requests to a [dispatcher](../framework/dispatcher.md) based on their type. This means that each worker is
isolated and works independently, following a "share nothing" approach where resources are not shared between workers.

Using RoadRunner can significantly improve the speed and efficiency by eliminating the need for
the application to go through the bootstrapping process repeatedly. This can save on CPU and memory resources and reduce
response time.

> **Note**
> Read more about Framework and application server symbiosis
> in the [Framework — Application Lifecycle](../framework/lifecycle.md) section.

## Installation

There are several ways to download RoadRunner:

:::: tabs

::: tab Composer

The best way is to use composer package `spiral/roadrunner-cli`. It will help you to download the server
automatically.

Just install the package in your project and run the following command:

```bash
./vendor/bin/rr get
```

> **Note**
> PHP's extensions `php-curl` and `php-zip` are required to download RoadRunner automatically.
:::

::: tab Docker

To get the roadrunner binary file you can use our docker image: `spiralscout/roadrunner:2.X.X` or use the GitHub
package: `ghcr.io/roadrunner-server/roadrunner:2.X.X`

```bash
FROM ghcr.io/roadrunner-server/roadrunner:2.X.X AS roadrunner
FROM php:8.1-cli

COPY --from=roadrunner /usr/bin/rr /usr/local/bin/rr

# USE THE RR
```

:::

::: tab Github

The easiest way to get the latest RoadRunner version is to use one of the pre-built release binaries which are available
for OSX, Linux, FreeBSD, and Windows on GitHub [release page](https://github.com/roadrunner-server/roadrunner/releases).

:::

::: tab Linux
Installation option for the Debian-derivatives (Ubuntu, Mint, MX, etc)

```bash
wget https://github.com/roadrunner-server/roadrunner/releases/download/v2.X.X/roadrunner-2.X.X-linux-amd64.deb
sudo dpkg -i roadrunner-2.X.X-linux-amd64.deb
```

:::

::::

## Configuration

You can configure the number of workers, memory limits, and other plugins using `.rr.yaml` file:

```yaml
version: '2.7'

rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: "php app.php"
  relay: pipes

# HTTP plugin settings
http:
  address: 0.0.0.0:8080
  middleware: [ "gzip", "static" ]
  static:
    dir: "public"
    forbid: [ ".php", ".htaccess" ]
  pool:
    num_workers: 2
    supervisor:
      max_worker_memory: 100
```

To set the number of workers for HTTP:

```yaml
http:
  pool:
    num_workers: 4
```

### Developer Mode

To force worker reload after every request (full debug mode) and limit processing to a single worker, add a `debug`
option:

```yaml
http:
  pool:
    debug: true
```

> **Note**
> Read more about application server configuration in the official [documentation](https://roadrunner.dev/docs).

## Running the Server

:::: tabs

::: tab Linux

Use the following command to start application server on **Linux**

```bash
./rr serve
```

> **Warning**
> Make sure that `rr` binary is executable.

:::

::: tab Windows
Use the following command to start application server on **Windows**

```bash
./rr.exe serve
```
:::

::::

> **Note**
> Read more about Server Commands in the [RoadRunner documentation](https://roadrunner.dev/docs/app-server-cli).

## RoadRunner bridge

The [spiral/roadrunner-bridge](https://github.com/spiral/roadrunner-bridge) package is a composer package that provides
integration between the Spiral framework and the RoadRunner application server.

> **Note**
> The component is available by default in the [application bundle](https://github.com/spiral/app).

This package allows developers to use RoadRunner's various plugins, including `http`, `grpc`, `jobs`, `tcp`, `kv`,
`centrifugo`, `logger`, and `metrics`, with the Spiral framework.

## Gotchas

There are a couple of limitations to be aware of.

#### Memory Leaks

Since the application stays in memory for a long time, even a small memory leak might lead to process restart.
RoadRunner will monitor memory consumption and perform a soft reset, but it is best to avoid memory leaks in your
application source code.

Though Framework and all of its components are written with memory management in mind, you still have to make sure that
your domain code is not leaking.

#### Application State

> **Note**
> Framework includes a set of instruments to simplify the development process and avoid memory/state leaks such as
> IoC Scopes, Cycle ORM, Immutable Configs, Domain Cores, Routes, and Middleware.

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Application lifecycle](../framework/lifecycle.md)
* [Dispatchers](../framework/dispatcher.md)
* [Finalizers](../framework/finalizers.md)
* [Static memory](../advanced/memory.md)
* [Custom Dispatcher](../cookbook/custom-dispatcher.md)