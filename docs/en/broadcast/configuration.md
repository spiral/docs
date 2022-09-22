# Broadcast - Installation and Configuration

The default Web and GRPC bundles include Broadcast extension pre-enabled on the application server level. To enable
extension in your application run:

```bash
composer require spiral/broadcast
```

Enable `Spiral\Bootloader\Broadcast\BroadcastBootloader` in your application kernel:

```php
protected const LOAD = [
    // ...    
    Spiral\Bootloader\Broadcast\BroadcastBootloader::class,
    // ...    
];
```

## Configuration

By default, the extension will carry event messages inside the application server memory. At the moment, you are able
to use Redis as a distributed pub/sub broker. To enable broadcasting using Redis, modify the `.rr.yaml` file:

```yaml
broadcast:
  # optional, redis broker configuration
  redis:
    addr: "localhost:6379"
    password: ""
    db: 0
```
