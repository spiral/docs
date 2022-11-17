# Cycle ORM - Transactions

To persist entity changes, your application services and controllers will require `Cycle\ORM\EntityManagerInterface`.

## Default Configuration

By default, the framework will automatically create a transaction on-demand from the container. Considering that
transactions always clean after the operation `run` , you can request it as a constructor parameter.

```php
use Cycle\ORM\EntityManagerInterface;

class MyService
{
    private EntityManagerInterface $entityManager;
    
    public function __construct(EntityManagerInterface $entityManager)
    {
        $this->entityManager = $entityManager;
    }
}
```

> **Note**
> Make sure that the `persist`/`delete` and `run` methods are always called within the same method scope while using
> service-specific transactions.

## Testing

You can always test the service by mocking `Cycle\ORM\EntityManagerInterface`, consider binding a mocked transaction
object to your application instance to see what is being persisted.


> **Note**
> Read more how to use Create, Update and Delete entities [here](https://cycle-orm.dev/docs/basic-crud/2.x/en).
