# Filters — Getting started

The `spiral/filters` is a powerful component for filtering and validating input data. It allows you to define a set of
rules for each input field, and then use those rules to ensure that the input data is in the correct format and meets
any other requirements you have set. You can use filters to validate data from HTTP requests, gRPC requests, console
commands, and other sources.

One of the benefits of using filters is that it helps to centralize your input validation logic in a single place. This
can make it easier to maintain your code, as you don't need to duplicate validation logic in multiple places throughout
your application.

Additionally, filters can be reused across different parts of your application, which can help to reduce code
duplication and make it easier to manage your validation logic.

![Filters](https://user-images.githubusercontent.com/773481/211005150-ba8803ed-42c1-40eb-9cf0-7e45e15b0b73.png)
*Illustration of the process of filtering and validating input data in an HTTP layer*

> **See more**
> Read more about how to use filters for console commands in
> the [Cookbook — Console command input validation](../cookbook/console-validation.md) section.

<hr>

## Installation

> **Note**
> The component relies on [Validation](../validation/factory.md) component, make sure to read it first.

The component does not require any configuration and can be activated using the
bootloader `Spiral\Bootloader\Security\FiltersBootloader`:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Security\FiltersBootloader::class,
    // ...
];
```

## Input sources

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

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

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
the abstract class `Spiral\Filters\Model\Filter`. 

To create a custom filter to validate a simple query value with key `username`, use the scaffolding command:

```terminal
php app.php create:filter UserFilter -p username:query
```

> **Note**
> Read more about scaffolding in the [Basics — Scaffolding](../basics/scaffolding.md#request-filter) section.

After executing this command, the following output will confirm the successful creation:

```output
Declaration of '[32mUserFilter[39m' has been successfully written into '[33mapp/src/Endpoint/Web/Filter/UserFilter.php[39m'.
```

```php app/src/Endpoint/Web/Filter/UserFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

final class UserFilter extends Filter
{
    #[Query(key: 'username')]
    public string $username;
}
```

You can request the Filter as a method injection (it will be automatically bound to the current HTTP request input):

```php app/src/Endpoint/Web/UserController.php
namespace App\Endpoint\Web;

class UserController
{
    public function show(Filter\UserFilter $filter): void
    {     
        dump($filter->username);
    }
}
```

By default, filters do not perform validation. However, if you want to validate a filter, you can implement the
`HasFilterDefinition` interface and define a set of validation rules for the filter properties using
the `FilterDefinition` class with `Spiral\Filters\Model\ShouldBeValidated` interface implementation:

```php app/src/Filter/MyFilterDefinition.php
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

```php app/src/Application/Bootloader/ValidatorBootloader.php
namespace App\Application\Bootloader;

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

```php app/src/Endpoint/Web/Filter/UserFilter.php
namespace App\Endpoint\Web\Filter;

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
