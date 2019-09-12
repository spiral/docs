# Cycle ORM - Transactions Lifecycle
In order to persist entity changes you application services and controllers will require `Cycle\ORM\TransactionInterface`.

## Default Configuration
By default, framework will automatically create transaction on on demand from container. Considering that transactions always clean
after run operation you can request it as constructor parameter.

```php
use Cycle\ORM;

class MyService
{
    private $tr;
    
    public function __construct(ORM\TransactionInterface $tr)
    {
        $this->tr = $tr;
    }
}
```

> Make sure that `persist`/`delete` and `run` methods are always called within one method scope while using such approach.

## Testing
You can always test your codebase without performing actual by mocking `TransactionInterface`, consider binding singleton test object
to your application instance in test in order to test all invocations.
