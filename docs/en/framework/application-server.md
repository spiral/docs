# Framework - Application Server

The Spiral Framework uses [RoadRunner](https://roadrunner.dev) as a high-powered application server.

## Downloading

You can download the application server directly 
from [release page](https://github.com/roadrunner-server/roadrunner/releases).

The best way is to use composer package `spiral/roadrunner-cli`. It will help you to download the server
automatically:

```bash
./vendor/bin/rr get
```

> **Note**
> The PHP extensions `php-cli` and `php-zip` should be enabled on your server.

## Installation RoadRunner bridge

RoadRunner integrates with the framework via [`spiral/roadrunner-bridge`](https://github.com/spiral/roadrunner-bridge)
package and can be installed via the Composer package manager:

```bash
composer require spiral/roadrunner-bridge
```

> **Note**
> Read more about `spiral/roadrunner-bridge` package installation and 
> configuration [here](https://github.com/spiral/roadrunner-bridge/blob/master/README.md).

## Running the Server

The RoadRunner server is easy to run on the default `:8080` port:

```bash
./rr serve
```

You can observe the memory consumption of your workers in realtime and other information via

```bash
./rr workers -i
```

> **Note**
> Read more about RoadRunner cli commands [here](https://roadrunner.dev/docs/app-server-cli/2.x/en)

## Building Application Server

A lot of the sections in this documentation will explain how to extend your application capabilities by adding your own
RoadRunner [plugins](https://roadrunner.dev/docs/app-server-build/2.x/en), 
[middleware](https://roadrunner.dev/docs/middleware-writing-a-middleware/2.x/en), or data providers.

> **Note**
> You are not required to learn Golang or build an application server by yourself, the default build will cover all of
> the framework features.

You can read all the info about building an application server on the official 
[site](https://roadrunner.dev/docs/app-server-build/2.x/en).
