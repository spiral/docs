# Redis
Spiral implements access to **Redis** database using the well-known `predis` library.

## Configuring

## Requesting RedisClient
RedisClient is used to represent a single connection to a Redis database, such client implements spiral "controllable injection" pattern and can be requested multiple ways.

EXAMPLE

Redis client fully implement all predis methods, let's check a few simple examples (see predis documentation for more information);

```php
$client->incx('name', 1);

$client->push('name', 'value');

//Perform multiple operations via pipeline
example i don't remember
```

## Simplified Redis calls
In some cases you don't really need RedisClient instance.

EXAMPLE

> Even if you can use this way to work with data, we strongly recommend you use `RedisClient` directly.

