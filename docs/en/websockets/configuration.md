# Websockets — Installation and Configuration

Our Spiral Framework used to have its own [WebSocket plugin](https://roadrunner.dev/docs/plugins-websockets/2.x) for
sending messages to clients, but we found it was too limited. We wanted to create a better solution for handling
real-time data transmission and decided to stop developing our own plugin. Instead, we looked for one of the best
open-source tools out there and found [Centrifugo](https://centrifugal.dev/). We created a RoadRunner plugin
called [Centrifuge](https://roadrunner.dev/docs/plugins-centrifuge) that integrates with Centrifugo server
using [GRPC API](https://centrifugal.dev/docs/server/server_api#grpc-api) and it's a much more powerful and flexible
option.

**Here are some of the benefits of using Centrifugo:**

- **Real-time data transmission:** The integration provides efficient management of real-time data transmission, making
  it an ideal solution for applications that require real-time updates and notifications.
- **Scalability:** The Centrifugo toolset is designed to be highly scalable, allowing for the easy expansion of your
  application as it grows.
- **Flexibility:** Centrifugo provides a powerful and flexible toolset that can be easily customized to meet the
  specific needs of your application.
- **Large community:** Centrifugo has a large and active community that is constantly working to improve the tool, which
  means you'll have access to a wealth of knowledge and resources.
- **Complete Javascript SDK:** Centrifugo provides a
  comprehensive [Javascript SDK](https://github.com/centrifugal/centrifuge-js), making it easy for developers to
  interact with the server and build client-side functionality.

With this integration, you can send events to WebSocket clients as well as handle events from the clients on the server.

**Here are events that can be received on the server:**

- [**Connection requests:**](https://centrifugal.dev/docs/server/proxy#connect-proxy) When a client establishes a
  WebSocket connection to the Centrifugo server, a connection request event is sent to RoadRunner.
- [**Refresh connection events:**](https://centrifugal.dev/docs/server/proxy#refresh-proxy) Centrifugo sends a refresh
  event to RoadRunner when it is time to refresh the connection between the client and the server.
- [**RPC calls:**](https://centrifugal.dev/docs/server/proxy#rpc-proxy) Centrifugo can send RPC (Remote Procedure Call)
  events to RoadRunner over a client connection. These events allow you to invoke server-side functions from the client.
- [**Subscribe requests:**](https://centrifugal.dev/docs/server/proxy#subscribe-proxy) When a client subscribes to a
  channel, Centrifugo sends a subscribe request event to RoadRunner.
- [**Publish calls:**](https://centrifugal.dev/docs/server/proxy#publish-proxy) This request happens **BEFORE** a
  message is published to a channel, so your backend can validate whether a client can publish data to a channel.

## Installation

Register the Bootloader `Spiral\RoadRunnerBridge\Bootloader\CentrifugoBootloader` in your application.

After installing the package, you need to add the bootloader from the package to your application.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\CentrifugoBootloader::class,
    // ...
];
```

> **Note**
> Make sure that you have the [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) package installed.
> This package provides the necessary classes to integrate Cache component with RoadRunner Centrifuge plugin.

## Configuration

### RoadRunner server

To configure the communication between RoadRunner and Centrifugo, you will need to modify the `.rr.yaml` file.

```yaml .rr.yaml
version: '2.7'

rpc:
  listen: tcp://0.0.0.0:6001

server:
  command: "php app.php"
  relay: pipes

centrifuge:
  proxy_address: "tcp://0.0.0.0:10001"
  grpc_api_address: "centrifugo:10000"
```

Where `proxy_address` is the address of the RoadRunner Centrifuge plugin. It will be used by the Centrifugo server to
communicate with RoadRunner.

And `grpc_api_address` is the address of the Centrifugo GRPC server. It will be used by the RoadRunner Centrifuge plugin
to communicate with the Centrifugo server.

> **Note**
> Read more about the configuration of the Centrifuge plugin [here](https://roadrunner.dev/docs/plugins-centrifuge).

### Centrifugo server

To complete the integration, you will need to configure the Centrifugo server by creating a `config.json` file and
specifying the communication settings between Centrifugo and RoadRunner.

**Here is an example `config.json` file that shows how to configure the communication between servers**

```json config.json
{
  // ...
  "allowed_origins": [
    "*"
  ],
  "publish": true,
  "proxy_publish": true,
  "proxy_subscribe": true,
  "proxy_connect": true,
  "allow_subscribe_for_client": true,
  "grpc_api": true,
  "grpc_api_address": "0.0.0.0",
  "grpc_api_port": 10000,
  "proxy_connect_endpoint": "grpc://127.0.0.1:10001",
  "proxy_connect_timeout": "10s",
  "proxy_publish_endpoint": "grpc://127.0.0.1:10001",
  "proxy_publish_timeout": "10s",
  "proxy_subscribe_endpoint": "grpc://127.0.0.1:10001",
  "proxy_subscribe_timeout": "10s",
  "proxy_refresh_endpoint": "grpc://127.0.0.1c:10001",
  "proxy_refresh_timeout": "10s",
  "proxy_rpc_endpoint": "grpc://127.0.0.1:10001",
  "proxy_rpc_timeout": "10s"
}
```

> **Note**
> More information about Centrifugo server configuration you can read on the
> official [documentation](https://centrifugal.dev/docs/server/configuration)

**In this configuration**

- `proxy_connect_endpoint` - RoadRunner server address, which will handle the
  new [connection events](https://centrifugal.dev/docs/server/proxy#connect-proxy)
- `proxy_publish_endpoint` - RoadRunner server address, which will handle
  the [publish events](https://centrifugal.dev/docs/server/proxy#publish-proxy)
- `proxy_subscribe_endpoint` - RoadRunner server address, which will handle
  the [subscribe events](https://centrifugal.dev/docs/server/proxy#subscribe-proxy)
- `proxy_refresh_endpoint` - RoadRunner server address, which will handle
  the [refresh events](https://centrifugal.dev/docs/server/proxy#refresh-proxy)
- `proxy_rpc_endpoint` - RoadRunner server address, which will handle
  the [rpc events](https://centrifugal.dev/docs/server/proxy#rpc-proxy)

Centrifugo allows you to use multiple RoadRunner servers to handle different types of events. This can be useful in
situations where you want to scale up the number of events that your application can handle or where you want to improve
the reliability of the communication between Centrifugo and RoadRunner.

To use multiple, you will need to specify the addresses of the servers in the `config.json` file. For example, you might
configure one RoadRunner server to handle connection events and another to handle RPC calls, as shown in the following
example configuration:

```json config.json
{
  // ...
  "proxy_connect_endpoint": "grpc://127.0.0.1:10001",
  "proxy_connect_timeout": "10s",
  "proxy_rpc_endpoint": "grpc://127.0.0.1:10002",
  "proxy_rpc_timeout": "10s"
}
```

You can use this approach to distribute the workload among multiple RoadRunner servers, depending on your needs.

> **Note**
> Keep in mind that using multiple RoadRunner servers will require additional configuration and setup, and you will need
> to ensure that the servers are properly coordinated to ensure smooth operation. However, the benefits of increased
> scalability and reliability can be well worth the effort.

### Spiral Framework application

To use the Centrifuge in your application, you will need to create a `centrifugo.php` file in
the `app/config` directory. In this file, you can specify **services** that will handle incoming events from
the Centrifugo server, as well as any interceptors that should be applied to the events.

Here is an example of a `centrifugo.php` config file that shows how to specify services and interceptors:

```php app/config/centrifugo.php
use RoadRunner\Centrifugo\Request\RequestType;

return [
    'services' => [
        RequestType::Connect->value => ConnectService::class,
        RequestType::Subscribe->value => SubscribeService::class,
        RequestType::Refresh->value => RefreshService::class,
        RequestType::Publish->value => PublishService::class,
        RequestType::RPC->value => RPCService::class,
    ],
    'interceptors' => [
        RequestType::Connect->value => [
            Interceptor\AuthInterceptor::class,
        ],
        RequestType::Subscribe->value => [
            Interceptor\AuthInterceptor::class,
        ],
        RequestType::RPC->value => [
            Interceptor\AuthInterceptor::class,
        ],
        '*' => [
            Interceptor\TelemetryInterceptor::class,
        ],
    ],
];
```

> **See more**
> For more information on event handlers (**services**) and how to use them in a Spiral application, you can refer to
> the documentation section on [Event Handlers](./services.md). This page provides additional details and examples to
> help you get started with event handling.

### Javascript client

You can use the official [JavaScript SDK](https://github.com/centrifugal/centrifuge-js) for Centrifugo to facilitate
communication between the Centrifugo server and the client browser in your application. It provides a set of APIs that
allow you to connect to the Centrifugo server, subscribe to channels, and receive messages in real-time.

Using the JavaScript SDK, you can establish a WebSocket connection between the client and the Centrifugo server, and all
communication between RoadRunner and the client will be routed through the Centrifugo server. This means that you don't
need to open any additional ports on your server to support real-time communication between the client and the server.

## Example Application

There is a good example [Demo ticket booking system](https://github.com/spiral/ticket-booking) application built on the
Spiral Framework, which is a high-performance PHP framework that follows the principles of microservices and allows
developers to create reusable, independent, and easy-to-maintain components.

It demonstrates how to use RoadRunner's Centrifuge plugin to enable real-time communication between the Centrifugo
server and the client browser.