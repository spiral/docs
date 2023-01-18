# Framework — Interceptors

One of the key features of the Spiral Framework is its support for interceptors, which can be used to add functionality
to the application without modifying the core code of the application. This can help to keep your codebase more modular
and maintainable.

**Here is some benefits of using interceptors:**

- **Separation of Concerns:** Using interceptors allows you to keep the different parts of your application separate and
  organized. For example, you can use an interceptor to handle authentication without having to add that code to every
  single part of your application that requires authentication. This makes it a lot easier to understand and maintain
  your code.
- **Reusability:** With interceptors, you can write code once and use it in multiple parts of your application. This
  means you don't have to keep writing the same code over and over again, saving you time and reducing the likelihood of
  mistakes.
- **Modularity:** The ability to add, remove or replace interceptors without affecting the rest of the application makes
  it more flexible and easy to update.
- **Performance:** Interceptors can be used to optimize the performance of the application by caching responses,
  reducing the number of database queries, and more. This means your application will be faster, and less likely to slow
  down when a lot of users are accessing it at the same time.
- **Ease of use:** Adding interceptors to your application is relatively easy and straightforward, making it accessible
  for developers of all skill levels.

You can use interceptors with various components such as:

- [HTTP](../http/interceptors.md)
- [Events](../advanced/events.md#interceptors)
- [gRPC](../grpc/interceptors.md)
- [Websockets](../websockets/interceptors.md)
- [Queue](../queue/interceptors.md)

> **Note**
> The `spiral/hmvc` component is required for the domain cores. The web bundle includes this package by default.

## Domain Core

In order to use interceptors, you need to have a core class that will be responsible for running the specific logic that
can be intercepted before or after the call.

For example, let's say you have a database layer and you need to log database queries, measure query times, and log slow
queries.

To achieve this, you can create a `DatabaseQueryCore` class that implements the `Spiral\Core\CoreInterface`. This class
would be responsible for running the database query and returning the query result.

```php app/src/Application/Database/DatabaseQueryCore.php
namespace App\Application\Database;

use Spiral\Core\CoreInterface;
use Cycle\Database\DatabaseManager;
use Cycle\Database\StatementInterface;

final class DatabaseQueryCore implements CoreInterface
{
    public function __construct(
        private readonly DatabaseManager $manager
    ) {
    }

    public function callAction(string $database, string $sql, array $parameters = []): StatementInterface
    {
        $sqlParameters = $parameters['sql_parameters'] ?? [];
        \assert(\is_array($sqlParameters));

        $database = $this->manager->database($database);
        return $database->query(
            $sql,
            $sqlParameters
        );
    }
}
```

Then you can use the `Spiral\Core\InterceptableCore` class that allows you to register interceptors and call them when
you handle the core class.

```php
use Spiral\Core\InterceptableCore;
use App\Application\Database\DatabaseQueryCore;
use Cycle\Database\DatabaseManager;

$core = new InterceptableCore(
  new DatabaseQueryCore(new DatabaseManager(...))
);
```

Here is an example of how to call the `DatabaseQueryCore` class to handle database queries:

```php
// Execute a SELECT statement on the 'default' database
$result = $core->callAction(
  'default', 
  'SELECT * FROM users WHERE id = ?', 
  ['sql_parameters' => [1]]
);
```

The `callAction` method will return a `Cycle\Database\StatementInterface` object, which represents the result of the
query.

Now let's talk about interceptors.

## Interceptors

Interceptors work similarly to middleware in HTTP requests, in that they allow developers to add functionality to the
application at various points in the processing flow. However, unlike middleware which is typically specific to HTTP
requests, interceptors can be used to add functionality to a wide range of components.

Interceptors should implement the `Spiral\Core\CoreInterceptorInterface` interface, which requires them to define
a `callAction` method. This method is called by the framework at specific points in the application's execution, such
as before or after a controller action is invoked.

For example, we could create an `SlowQueryDetectorInterceptor` class that implements the `CoreInterceptorInterface` and
logs slow queries.

```php app/src/Interceptor/SlowQueryDetectorInterceptor.php
namespace App\Interceptor;

use Psr\Log\LoggerInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;

class SlowQueryDetectorInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {
    }

    public function process(string $database, string $sql, array $parameters, CoreInterface $core): mixed
    {
        $startTime = \microtime(true);

        $result = $core->callAction($database, $sql, $parameters);
        
        $elapsed = \microtime(true) - $startTime;

        if ($elapsed > 0.1) {
            $this->logger->warning(
                'Slow query detected',
                [
                    'database' => $database,
                    'sql' => $sql,
                    'parameters' => $parameters,
                    'elapsed' => $elapsed
                ]
            );
        }

        return $result;
    }
}
```

Now we can register the `SlowQueryDetectorInterceptor` class with the `InterceptableCore` class and call
the `callAction`
method to handle the database query.

```php
use Spiral\Core\InterceptableCore;
use App\Application\Database\DatabaseQueryCore;
use Cycle\Database\DatabaseManager;

$core = new InterceptableCore(
  new DatabaseQueryCore(new DatabaseManager(...))
);

$core->addInterceptor(new SlowQueryDetectorInterceptor(new Logger(...)));

// Execute a SELECT statement on the 'default' database
$result = $core->callAction(
  'default', 
  'SELECT * FROM users WHERE id = ?', 
  ['sql_parameters' => [1]]
);
```

Now every time the `callAction` method is called, the `SlowQueryDetectorInterceptor` class will be called and will log
slow queries.

We can also add multiple interceptors to the `InterceptableCore` class. For example, we could create another one
interceptor that handles Database exceptions and in some cases treies to reconnect to the database.

```php app/src/Interceptor/DatabaseConnectionInterceptor.php
namespace App\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Cycle\Database\Exception\StatementException\ConnectionException;

final class DatabaseConnectionInterceptor implements CoreInterceptorInterface
{
    // ...

    public function process(string $database, string $sql, array $parameters, CoreInterface $core): mixed
    {
        try {
            return $core->callAction($database, $sql, $parameters);
        } catch (ConnectionException $e) {
            // Try to reconnect...

            // For example, switch to another database...
            return $this->process('slave', $sql, $parameters, $core);
        }
    }
}
```

Don't forget to register the `DatabaseConnectionInterceptor` class with the `InterceptableCore` class.

```php
$core->addInterceptor(new DatabaseConnectionInterceptor(...));
$core->addInterceptor(new SlowQueryDetectorInterceptor(new Logger(...)));
```

You can pass any parameters to the `callAction` method, that can be used by interceptors. For example:

```php
public function process(string $database, string $sql, array $parameters, CoreInterface $core): mixed
{
    // $sql = SELECT * FROM users WHERE id = ?
    $sql .= ' LIMIT ?';
    
    $parameters['sql_parameters'][] = 10;
    
    return $core->callAction($database, $sql, $parameters);
}
```

As you can see interceptors are a convenient way to intercept and modify the behavior of certain parts of the
application, making it more functional and efficient, while keeping the domain specific logic clean and maintainable.

## Events

| Event                                | Description                                               |
|--------------------------------------|-----------------------------------------------------------|
| Spiral\Core\Event\InterceptorCalling | The Event will be fired `before` calling the interceptor. |

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
