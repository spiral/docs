# Filters — Interceptors

Spiral Framework provides a way for developers to customize the behavior of their filters through the use of
interceptors. An interceptor is a piece of code that is executed after a filter created, and
which allows developers to hook into the filter handling pipeline to perform some action.

Interceptors can be useful for a variety of purposes, such as handling errors or adding additional context to the
filter.

To use interceptors you will need to implement the `Spiral\Core\CoreInterceptorInterface` interface.

## Interceptor for filter

Let's imagine that we need to check user authorization to access some filter. We can do it with interceptor:

```php
use Spiral\Domain\Annotation\Guarded;

#[Guarded(permission: 'user.store')]
final class StoreUser extends Filter
{
    #[Post]
    public string $username;
    
    #[Post]
    public string $email;
    
    //...
}
```

> **See more**
> Read more about user Role-Based Access Control in the [Security — Role-Based Access Control](../security/rbac.md)
> section.

Here is an example of a simple interceptor that checks authorization:

```php
namespace App\Filter;

use Spiral\Attributes\ReaderInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Domain\Annotation\Guarded;
use Spiral\Http\Exception\ClientException\UnauthorizedException;
use Spiral\Security\GuardInterface;

class CheckGuardInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly GuardInterface $guard,
        private readonly ReaderInterface $reader,
    ) {
    }

    public function process(string $filterClass, string $action, array $parameters, CoreInterface $core): mixed
    {
        $refClass = new \ReflectionClass($filterClass);
        // Read attributes from the filter class and check if it has Guarded annotation
        $guarded = $this->reader->firstClassMetadata($refClass, Guarded::class);

        // If the filter has Guarded attributes, check if the user has the permission
        if ($guarded && !$this->guard->allows($guarded->permission ?: $filterClass)) {
            throw new UnauthorizedException($guarded->errorMessage ?: 'Access denied');
        }

        // If the filter has no Guarded attributes, or the user has the permission, continue
        return $core->callAction($filterClass, $action, $parameters);
    }
}
```

To use this interceptor, you will need to register it in the configuration file `app/config/filters.php`.

```php

use App\Filter\CheckGuardInterceptor;
use Spiral\Filters\Model\Interceptor\PopulateDataFromEntityInterceptor;
use Spiral\Filters\Model\Interceptor\ValidateFilterInterceptor;

return [    
     'interceptors' => [
        CheckGuardInterceptor::class,
        PopulateDataFromEntityInterceptor::class,
        ValidateFilterInterceptor::class,
     ],
     // ...
];
```

> **Note**
> Pay attention that we have `PopulateDataFromEntityInterceptor` and `ValidateFilterInterceptor` in the list, they are
> required for the filter to work properly.