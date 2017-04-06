# Method Invocations
Tokenizer is able to locate invocations and given parameters for class methods and functions.

Use `Spiral\Tokenizer\InvocationsInterface` to gain access to such functionality.

## Examples
Locate all usages of dump function:

```php
public function indexAction(InvocationsInterface $invocations)
{
    dump($invocations->getInvocations(new \ReflectionFunction('dump')));
}
```

Locate all messages of `say` method of `TranslatorTrait` and dump it's parameters:

```php
use TranslatorTrait;

public function indexAction(InvocationsInterface $invocations)
{
    $invocations = $invocations->getInvocations(new \ReflectionMethod(
        TranslatorTrait::class,
        'say'
    ));

    foreach ($invocations as $invocation) {
        dump($invocation->getArguments());
    }

    $this->say('hello');
}
```

> Note, current implementation is only capable of locating constructions like `self::method()`, `$this->method()` and `static::method()`, type based resolutions are currently not implemented (AST support is required).
