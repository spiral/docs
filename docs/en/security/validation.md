# Validation

Validation in the Spiral Framework consists of three parts: the `Spiral Validation` package with validation interfaces,
a `validator` package with validation implementation and `Request Filters`.

In this article, we will look at the `Spiral Validation` component.

> **Note**
> Read more about other validation parts [Request Filters](../filters/configuration.md)
> or [Spiral Validator](../security/validator.md).

The component available by default in the [application bundle](https://github.com/spiral/app).

## Installation

To enable the component, you just need to add `Spiral\Validation\Bootloader\ValidationBootloader` to the bootloaders 
list, which is located in the class of your application.

```php
namespace App;

use Spiral\Validation\Bootloader\ValidationBootloader;

class App extends Kernel
{
    protected const LOAD = [
        // ...
        ValidationBootloader::class,
        // ...
    ];
}
```
## Usage

After activating the `Validation`, [Filters](../filters/configuration.md) packages and one of the `validator` packages, 
for example, [Spiral Validator](../security/validator.md):

```php
namespace App;

use Spiral\Validation\Bootloader\ValidationBootloader;
use Spiral\Bootloader\Security\FiltersBootloader;
use Spiral\Validator\Bootloader\ValidatorBootloader;

class App extends Kernel
{
    protected const LOAD = [
        // ...
        ValidationBootloader::class,
        FiltersBootloader::class,
        ValidatorBootloader::class,
        // ...
    ];
}
```

The `Request Filters` will be validated automatically, and no further actions with the validator are required.

```php
namespace App\Controller;

use App\Request\MyFilter;

class HomeController
{
    public function index(MyFilter $filter): string
    {
        // already validated data
        \dumprr($filter);
    }
}
```

### ValidationInterface

The `Spiral\Validation\ValidationInterface` has one `validate` method that accepts `validation data`, 
`validation rules`, and `context`. And should return a `Spiral\Validation\ValidatorInterface` instance.

```php
namespace Spiral\Validation;

/**
 * Creates validators with given rules and data.
 */
interface ValidationInterface
{
    /**
     * Create validator for given parameters.
     *
     * @param array|object  $data    Target validation data.
     * @param array         $rules   List of associated validation rules (see Rule).
     * @param mixed         $context Validation context (available for checkers and validation
     *                               methods but is not validated).
     */
    public function validate(array|object $data, array $rules, mixed $context = null): ValidatorInterface;
}
```

Usage example:

```php
namespace App\Controller;

use Spiral\Validation\ValidationInterface;
use Spiral\Validation\ValidatorInterface;

class HomeController
{
    public function index(ValidationInterface $validation): void
    {
        $validator = $validation->validate(
            // data
            [
                'key' => null
            ],
            // rules
            [
                'key' => [
                    'notEmpty'
                ]
            ]
        );

        dump($validator instanceof ValidatorInterface);

        dump($validator->isValid());
        dump($validator->withData(['key' => 'value'])->isValid());
    }
}
```

> **Note**
> You can use the `validator` prototype property.

### ValidatorInterface

The `Spiral\Validation\ValidatorInterface` provides basic API to get result errors and allows them to attach to 
new data or context (immutable).

```php
namespace Spiral\Validation;

use Spiral\Validation\Exception\ValidationException;

/**
 * Singular validation state (with data, context and rules encapsulated).
 */
interface ValidatorInterface
{
    /**
     * Create validator copy with new data set.
     */
    public function withData(array|object $data): ValidatorInterface;

    /**
     * Receive field from context data or return default value.
     */
    public function getValue(string $field, mixed $default = null): mixed;

    /**
     * Check if field is provided in the given data.
     */
    public function hasValue(string $field): bool;

    /**
     * Create new validator instance with new context.
     */
    public function withContext(mixed $context): ValidatorInterface;

    /**
     * Get context data (not validated).
     */
    public function getContext(): mixed;

    /**
     * Check if context data valid accordingly to provided rules.
     *
     * @throws ValidationException
     */
    public function isValid(): bool;

    /**
     * List of errors associated with parent field, every field should have only one error assigned.
     *
     * @return array<string, string> Keys are fields, values are messages
     *
     * @throws ValidationException
     */
    public function getErrors(): array;
}
```

Usage example:

```php
namespace App\Controller;

use Spiral\Validation\ValidationInterface;

class HomeController
{
    public function index(ValidationInterface $validation): void
    {
        $validator = $validation->validate(
            ['key' => null],
            ['key' => ['notEmpty']]
        );

        if (!$validator->isValid()) {
            dump($validator->getErrors());
        }
    }
}
```

### ValidationProviderInterface

The `Spiral\Validation\ValidationProviderInterface` implementation stores registered validators that provide 
validators packages. This interface has one `getValidation` method, which must return the registered 
`ValidationInterface` instance.

## Creating validator

We provide some validators out-of-the-box, such as the [Spiral Validator](../security/validator.md).
This section will be helpful if you want to create your own validator.

### Validation and Validator

To create our own validator, we need to create classes that implement the `ValidationInterface` and `ValidatorInterface` 
interfaces.

```php
namespace App\Validator;

use Spiral\Validation\ValidationInterface;
use Spiral\Validation\ValidatorInterface;

class Validation implements ValidationInterface
{
    public function validate(mixed $data, array $rules, $context = null): ValidatorInterface
    {
        return new Validator(...);
    }
}
```

```php
namespace App\Validator;

use Spiral\Validation\ValidatorInterface;

class Validator implements ValidatorInterface
{
    public function isValid(): bool
    {
        //...
    }
    
    // other required methods
}
```

### FilterDefinition

The validator must provide a `FilterDefinition` which must implement the `Spiral\Filters\Model\FilterDefinitionInterface` 
and `Spiral\Filters\Model\ShouldBeValidated` interfaces. 

```php
namespace App\Validator;

use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\ShouldBeValidated;

class FilterDefinition implements FilterDefinitionInterface, ShouldBeValidated
{
    public function __construct(
        private readonly array $validationRules = [],
        private readonly array $mappingSchema = []
    ) {
    }

    public function mappingSchema(): array
    {
        return $this->mappingSchema;
    }

    public function validationRules(): array
    {
        return $this->validationRules;
    }
}
```

This object must be returned from the filter and must contain validation rules and mapping schema.

### Validator registration

Now we can register the created validator. To do this, use the `register` method in the 
`Spiral\Validation\ValidationProvider` class.

The `$name` parameter must be the fully qualified name of the `FilterDefinition` class. Otherwise, the 
`ValidationProvider` will not be able to determine a validator for the Request Filter.

```php
namespace App\Bootloader;

use App\FilterDefinition;
use App\Validation;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Validation\Bootloader\ValidationBootloader;
use Spiral\Validation\ValidationInterface;
use Spiral\Validation\ValidationProvider;

final class ValidatorBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        ValidationBootloader::class
    ];
    
    public function boot(ValidationProvider $provider): void
    {
        $provider->register(
            FilterDefinition::class,
            static fn(Validation $validation): ValidationInterface => $validation
        );
    }
}
```

> **Note**
> Read more about Bootloaders [here](../framework/bootloaders.md).
