# Filter Objects
The framework component `spiral/filters` provides support for request validation, composite validation, error message
mapping and locations and etc.

> The component relies on [Validation](/security/validation.md) library, make sure to read it first.

## Installation
The component does not require any configuration and can be activated using the bootloader `Spiral\Bootloader\Security\FiltersBootloader`:

```php
[
    // ...
    Spiral\\Bootloader\Security\FiltersBootloader::class
    // ...
]
```

## Input Binding
The filter components operate using the `Spiral\Filter\InputInterface` as primary data source:

```php
interface InputInterface
{
    public function withPrefix(string $prefix, bool $add = true): InputInterface;

    public function getValue(string $source, string $name = null);
}
```

By default, this interface is binded to [InputManager](/http/request-response.md) and which makes possible to access
any request attribute using **source** and **origin** pair with dot-notation support. For example:

```php
namespace App\Controller;

use Spiral\Filters\InputInterface;

class HomeController
{
    public function index(InputInterface $input)
    {
        dump($input->getValue('query', 'abc')); // ?abc=1

        // dot notation
        dump($input->getValue('query', 'a.b.c')); // ?a[b][c]=2

        // same as above
        dump($input->withPrefix('a')->getValue('query', 'b.c')); // ?a[b][c]=2
    }
}
```

Input binding is primary way of delivering data into filter object.

## Extensions
Activate `Spiral\Domain\FilterInterceptor` in your [domain core](/cookbook/domain-core.md) to automatically pre-validate
your request before delivering to controller.