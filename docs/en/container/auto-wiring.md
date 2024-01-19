# Container — Auto-wiring

Spiral attempts to hide container implementation and configuration from your domain layer by providing rich auto-wiring
functionality. Though auto-wiring rules are straightforward, it's essential to learn them to avoid framework
misbehavior.

The `Spiral\Core\Container\Autowire` class allows you to delegate options to the container and pass specific
configuration values to your classes without hardcoding them. This can be useful for keeping your configuration separate
from your code and for making it easier to modify your application's behavior without changing the code itself.

```php app/config/session.php
use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\FileHandler;

return [
    // ...
    'handler' => new Autowire(
        FileHandler::class,
        [
            'directory' => directory('runtime') . 'session',
            'lifetime' => (int)env('SESSION_LIFETIME', 86400),
        ]
    ),
];
```

When the container tries to resolve the `Autowire`, it will automatically create an instance of the `FileHandler`
class and pass the `directory` and `lifetime` options to the constructor.



This allows you to easily configure your classes and pass in specific options without having to hardcode them in your
code, and can be particularly useful for configuring classes that have many options or that are used in multiple parts
of your codebase.