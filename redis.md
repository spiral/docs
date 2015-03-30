# Redis
Spiral implements access to **Redis** database using well known `predis` library.

## Configuring

## Requesting RedisClient
RedisClient used to represent single connection to Redis database, such client implements spiral "controllable injection" pattern and can be requested multiple way.

EXAMPLE

Redis client fully implement all predis methods, let's check few simple example (see predis documentation for more information);

```php
$client->incx('name', 1);

$client->push('name', 'value');

//Perform multiple operations via pipeline
example i don't remember
```

## Simplified Redis calls
In some cases you don't really need RedisClient instance.

EXAMPLE

> Even if you can use this way to work with data, we strongly recommend to work with `RedisClient` directly.

