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
    Spiral\\Bootloader\Security\FiltersBootloader::class
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
`Spiral\Filter\Filter`. To create custom filter to validate query:

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    public const SCHEMA = [
        'abc' => 'query:abc'
    ];

    public const VALIDATES = [
        'abc' => [
            'notEmpty',
            'string'
        ]
    ];
}
```

> Or use the scaffolding `php app.php create:filter my -f "abc:string(query)"`.

You can request the Filter as method injection (it will be automatically bound to the current http input):

```php
namespace App\Controller;

use App\Filter;

class HomeController
{
    public function index(Filter\MyFilter $f): void
    {     
        dump($f->isValid());
        dump($f->getErrors());
        dump($f->getFields());
    }
}
```

> **Note**
> Try URL with `?abc=1`.

## Extensions

There are two interceptors in Spiral Framework that you can use for automatically pre-validation your request before
delivering to a controller.

### FilterInterceptor

`Spiral\Domain\FilterInterceptor` will automatically validate your filters after resolving. If filter is not valid an
error response will be return.

```php
[
    'status' => 400,
    'errors' => $filter->getErrors(),
]
```

### FilterWithRendererInterceptor

`Spiral\Domain\FilterWithRendererInterceptor` does the same as `Spiral\Domain\FilterInterceptor`, but with this
interceptor you can override the error rendering for a specific filter.

#### Renderer example

```php
<?php
declare(strict_types=1);

namespace App;

use Spiral\Filters\FilterInterface;
use Spiral\Filters\RenderErrors;

/**
 * @template-implements RenderErrors<FilterInterface>
 */
final class SomeErrorsRenderer implements RenderErrors
{
    /**
     * {@inheritdoc}
     */
    public function render(FilterInterface $filter)
    {
        return [
            'success' => false,
            'errors' => $filter->getErrors(),
        ];
    }
}
```

#### Filter example

```php
<?php

declare(strict_types=1);

namespace App;

use Spiral\Filters\Filter;
use Spiral\Filters\RenderWith;

#[RenderWith(SomeErrorsRenderer::class)]
final class FilterWithErrorsRenderer extends Filter
{
    public const SCHEMA = [
        'id' => 'query:id'
    ];

    public const VALIDATES = [
        'id' => [
            ['notEmpty', 'err' => '[[ID is not valid.]]']
        ]
    
```

> **Note**
> Read more about using interceptors in your Domain core [here](/cookbook/domain-core.md)
