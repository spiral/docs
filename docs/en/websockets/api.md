# Centrifugo API

Spiral offers a user-friendly approach to sending diverse commands to Centrifugo via its GRPC API. Centrifugo has
inherent support for the GRPC API, enabling communication with the server using a compressed binary format for commands.

This approach leverages the power of HTTP/2, which is the underlying transport protocol for GRPC.

> **Note**
> To learn more about the different available API methods, please refer to the Centrifugo documentation on server API,
> which can be found on the [Centrifugo website](https://centrifugal.dev/docs/server/server_api).

## Usage

The `RoadRunner\Centrifugo\CentrifugoApiInterface` provides a set of methods that allow for easy communication with the
Centrifugo server.

You can request the `CentrifugoApiInterface` instance from the container.

**Here is an example of how to do this:**

```php
final class SubscribeUserToChannelHandler
{
    public function __construct(
        private \RoadRunner\Centrifugo\CentrifugoApiInterface $api
    ) {
    }

    public function handle(User $user, Channel $channel): void
    {
        $this->api->subscribe($channel->getName(), $user->getId());
    }
}
```

## Available API Methods


Let's go through the available methods:

### Publish

This method is used to publish data into a specific channel.

**It takes the following parameters:**

- `$channel`: a non-empty string representing the name of the channel to publish the message to.
- `$message`: a string representing the JSON encoded message to publish.
- `$skipHistory`: a boolean indicating whether or not to skip adding the message to the channel's history. Defaults to
  true.
- `$tags`: an array of key-value pairs representing additional metadata to attach to the message. Defaults to an empty
  array.

An example of how to use this method:

```php
public function handle(User $user, Channel $channel): void
{
    $this->api->publish($channel->getName(), 'Hello world!');
}
```

> **See more**
> Refer to the [Centrifugo documentation](https://centrifugal.dev/docs/server/server_api#publish) for more information.

### Broadcast

This method is similar to the publish method, but allows sending the same data to multiple channels.

**It takes the following parameters:**

- `$channels`: an array of non-empty strings representing the names of the channels to broadcast the message to.
- `$message`: a string representing the JSON encoded message to broadcast.
- `$skipHistory`: a boolean indicating whether or not to skip adding the message to the channels' histories. Defaults to
  true.
- `$tags`: an array of key-value pairs representing additional metadata to attach to the message. Defaults to an empty
  array.

An example of how to use this method:

```php
public function handle(User $user, Channel ...$channels): void
{
    $this->api->broadcast(
      \array_map(fn (Channel $channel) => $channel->getName(), $channels),
      'Hello world!'
    );
}
```

> **See more**
> Refer to the [Centrifugo documentation](https://centrifugal.dev/docs/server/server_api#broadcast) for more
> information.

### Subscribe

This method allows subscribing a user to a specified channel.

**It takes the following parameters:**

- `$channel`: The name of the channel to subscribe to. Must be a non-empty string.
- `$user`: The ID of the user who is subscribing. Must be a string.
- `$expireAt` (optional): An optional expiration date/time for the subscription, represented as a `\DateTimeInterface`
  object. If not provided, the subscription will not expire.
- `$info` (optional): An array of additional information to include with the subscription.
- `$client` (optional): An optional ID to identify the client making the subscription.
- `$data` (optional): An array of additional data to include with the subscription.
- `$session` (optional): An optional ID to identify the session for the subscription.

An example of how to use this method:

```php
public function handle(User $user, Channel $channel): void
{
    $this->api->subscribe($channel->getName(), $user->getId(), ...);
}
```

> **See more**
> Refer to the [Centrifugo documentation](https://centrifugal.dev/docs/server/server_api#subscribe) for more
> information.

### Unsubscribe

This method allows unsubscribing a user from a channel.

It takes in the following parameters:

- `$channel`: The name of the channel to unsubscribe from.
- `$user`: The user to unsubscribe from the channel.
- `$client` (optional): The client ID of the user. This parameter is optional and can be set to null if not
  needed.
- `$session` (optional): The session ID of the user. This parameter is optional and can be set to null if
  not needed.

An example of how to use this method:

```php
public function handle(User $user, Channel $channel): void
{
    $this->api->unsubscribe($channel->getName(), $user->getId(), ...);
}
```

> **See more**
> Refer to the [Centrifugo documentation](https://centrifugal.dev/docs/server/server_api#unsubscribe) for more
> information.

### Disconnect

This method disconnects a user by their ID.

It takes in the following parameters:

- `$user`: The ID of the user to disconnect.
- `$client` (Optional): The ID of the client to disconnect. If not provided, all clients associated with the user will
  be disconnected.
- `$whitelist` (Optional): An array of client IDs that should not be disconnected.
- `$session` (Optional): The ID of the session to disconnect. If not provided, all sessions associated with the user
  will be disconnected.
- `$disconnect` (Optional): A Disconnect object containing information about the disconnect event.

An example of how to use this method:

```php
public function handle(User $user): void
{
    $this->api->disconnect($user->getId(), ...);
}
```

> **See more**
> Refer to the [Centrifugo documentation](https://centrifugal.dev/docs/server/server_api#disconnect) for more
> information.

### Presence

This method is used to retrieve the list of active clients connected to a channel.

It takes the following parameter:

- `$channel`: A non-empty string representing the name of the channel to retrieve the list of clients from.

An example of how to use this method:

```php
public function handle(Channel $channel): void
{
   $result = $this->api->presence($channel->getName());
   // ...
}
```

> **See more**
> Refer to the [Centrifugo documentation](https://centrifugal.dev/docs/server/server_api#presence) for more
> information.

### Presence Stats

This method is used to retrieve the short channel presence information - number of clients and number of unique users 
(based on user ID).

It takes the following parameter:

- `$channel`: A non-empty string representing the name of the channel.

An example of how to use this method:

```php
public function handle(Channel $channel): void
{
   $stats = $this->api->presenceStats($channel->getName());
   // ...
}
```

> **See more**
> Refer to the [Centrifugo documentation](https://centrifugal.dev/docs/server/server_api#presence_stats) for more
> information.

### Channels

This method is used to retrieve a list of active channels, which are channels that have one or more active subscribers.

It takes the following parameter:

- `$pattern` (Optional): a non-empty string is provided, only channels matching the pattern are returned.

An example of how to use this method:

```php
public function handle(Channel $channel): void
{
   $channels = $this->api->channels();
   // ...
}
```