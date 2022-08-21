# Filter Objects

The framework component `spiral/filters` provides support for request validation, composite validation, an error message
mapping and locations, etc.

> **Note**
> The component relies on [Validation](/security/validation.md) library, make sure to read it first.

## Installation

The component does not require any configuration and can be activated using the
bootloader `Spiral\Bootloader\Security\FiltersBootloader`:

```php
[
    // ...
    Spiral\Bootloader\Security\FiltersBootloader::class
    // ...
]
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

By default, this interface is bound to [InputManager](/http/request-response.md) and which makes it possible to access
any request's attribute using **source** and **origin** pair with dot-notation support. For example:

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

Input binding is a primary way of delivering data into the filter object.

## Create Filter

The filter object implement might vary from package to package. The default implementation provided via abstract class
`Spiral\Filters\Model\Filter`. To create custom filter to validate query:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Query]
    public string $abc;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'abc' => ['string', 'required']
        ]);
    }
}
```

You can request the Filter as method injection (it will be automatically bound to the current http input):

```php
namespace App\Controller;

use App\Filter\MyFilter;

class HomeController
{
    public function index(MyFilter $filter): void
    {     
        dump($filter->abc);
    }
}
```

> **Note**
> Try URL with `?abc=1`.

The Filter will automatically pre-validate your request before delivering to controller.
