# Filters - Installation and Configuration

The `spiral/filters` allows you to create filters that can be used to filter and validate request input data. A filter
is a PHP class that contains a set of properties, each of which represents a request input field that can be filtered
and validated.

> **Note**
> If you are migrating from Spiral Framework 2.x and want to continue using the old filters you can use
> [spiral/filters-bridge](https://github.com/spiral/filters-bridge) package.
> Read more about using the package [here](../filters/bridge.md).

## Installation

> **Note**
> The component relies on [Validation](../validation/factory.md) component, make sure to read it first.

The component does not require any configuration and can be activated using the
bootloader `Spiral\Bootloader\Security\FiltersBootloader`:

```php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Security\FiltersBootloader::class,
    // ...
];
```

## Input Binding

The filter components operate using the `Spiral\Filter\InputInterface` as a primary data source:

```php
interface InputInterface
{
    public function withPrefix(string $prefix, bool $add = true): InputInterface;

    public function getValue(string $source, string $name = null);
}
```

By default, this interface is bound to [InputManager](../http/request-response.md) and allows to access
any request's attribute using a **source** and **origin** pair with dot-notation support.

For example:

```php
namespace App\Controller;

use Spiral\Filters\InputInterface;

class HomeController
{
    public function index(InputInterface $input): void
    {
        dump($input->getValue('query', 'abc')); // ?abc=1

        // dot notation
        dump($input->getValue('query', 'a.b.c')); // ?a[b][c]=2

        // same as above
        dump($input->withPrefix('a')->getValue('query', 'b.c')); // ?a[b][c]=2
    }
}
```

Input binding is the primary way of delivering data from request into the filter object.

## Create Filter

The implementation of the filter object might vary from package to package. The default implementation is provided via
the abstract class `Spiral\Filters\Model\Filter`. To create a custom filter to validate a simple query value with
key `username`:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

final class UserFilter extends Filter
{
    #[Query]
    public string $username;
}
```

You can request the Filter as a method injection (it will be automatically bound to the current HTTP request input):

```php
namespace App\Controller;

use App\Filter\UserFilter;

class UserController
{
    public function show(UserFilter $filter): void
    {     
        dump($filter->username);
    }
}
```

By default, filters do not perform validation. However, if you want to validate a filter, you can implement the
`HasFilterDefinition` interface and define a set of validation rules for the filter properties using
the `FilterDefinition` class with `Spiral\Filters\Model\ShouldBeValidated` interface implementation:

```php
<?php

declare(strict_types=1);

namespace App\Filter;

use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\ShouldBeValidated;

final class MyFilterDefinition implements FilterDefinitionInterface, ShouldBeValidated
{
    public function __construct(
        private readonly array $validationRules = [],
        private readonly array $mappingSchema = []
    ) {
    }

    public function validationRules(): array
    {
        return $this->validationRules;
    }

    public function mappingSchema(): array
    {
        return $this->mappingSchema;
    }
}
```

Here is an example of registering a filter definition and binding with a validator that will be used to validate filters
with the `MyFilterDefinition` definition:

```php
namespace App\Bootloader;

use App\Validation;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Validation\Bootloader\ValidationBootloader;
use Spiral\Validation\ValidationInterface;
use Spiral\Validation\ValidationProvider;

final class ValidatorBootloader extends Bootloader
{
    public function boot(ValidationProvider $provider): void
    {
        $provider->register(
            \App\Filter\MyFilterDefinition::class,
            static fn(Validation $validation): ValidationInterface => new MyValidation()
        );
    }
}
```

> **Note**
> Red more about Validation component [here](../validation/factory.md) .

And now you can use the filter definition in your filter:


```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use App\Filter\MyFilterDefinition;

final class UserFilter extends Filter implements HasFilterDefinition
{
    #[Query]
    public string $username;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new MyFilterDefinition([
            'username' => ['string', 'required']
        ]);
    }
}
```

> **Note**
> Try URL with `?username=john`. The `UserFilter` will automatically pre-validate your request before delivering it to 
> the controller.
