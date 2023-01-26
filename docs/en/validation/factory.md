# Validation

The validation component of the Spiral Framework allows you to validate data that is submitted by a user or received
from an external source.

The component does not contain any validator implementation out of the box. Instead, it provides a set of interfaces and
abstract classes that define the expected behavior of a validator.

## Installation

To enable the component, you just need to add `Spiral\Validation\Bootloader\ValidationBootloader` to the bootloaders
list.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Validation\Bootloader\ValidationBootloader::class,
    // ...
];
```

## Configuration

There are three validator bridges available for use with the Spiral Framework validation component:

- [Spiral Validator](./spiral.md) - This is the default validator bridge. It is a simple,
  lightweight validator that can handle basic validation tasks.
- [Symfony Validator](./symfony.md) - This validator bridge provides integration
  with the Symfony Validator component, which is a more powerful and feature-rich validation library.
- [Laravel Validator](./laravel.md) - This validator bridge provides integration
  with the Laravel Validator, which is a validation component used in the Laravel framework.

You can use any of these validator bridges in your application, depending on your needs and preferences.

Most applications use a single validator implementation, but Spiral Framework allows you to use multiple
validators in your application if needed. In this case, you can define a default validator in the
`app/config/validation.php` configuration file.

```php app/config/validation.php
return [
    'defaultValidator' => 'my-validator',
    // ...
];
```

In addition to setting the default validator in the configuration file, you can also set the default validator using the
`Spiral\Validation\Bootloader\ValidationBootloader`.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Validation\Bootloader\ValidationBootloader;

final class AppBootloader extends Bootloader
{
    public function boot(ValidationBootloader $validation): void
    {
        $validation->setDefaultValidator('my-validator');
    }
}
```

## Usage

In this section, we will show you how to use the validation component to validate data.

Here is an example of how to validate data using the default validator:

```php app/src/Interface/Controller/UserController.php
use Spiral\Http\Request\InputManager;
use Spiral\Validation\ValidatorInterface;

class UserController
{
    public function create(InputManager $input, ValidatorInterface $validator)
    {
        $validator = $validator->validate([
            'username' => $input->post('username'),
            'email' => $input->post('email'),
        ], [
            'username' => 'required',
            'email' => 'required|email',
        ]);
        
        if (!$validator->isValid()) {
            $errors = $validator->getErrors();
            // ...
        }

        // Store the user in the database...
    }
}
```

`Spiral\Validation\ValidationInterface` has one `validate` method that accepts validation data, validation rules, and
context. The method returns a `Spiral\Validation\ValidatorInterface` instance.

When the application resolves the `Spiral\Validation\ValidatorInterface` from the container, it will request it from the
`Spiral\Validation\ValidationProviderInterface` interface, which is responsible for providing validator instances. The
`ValidationProviderInterface` determines which validator to use based on the default validator that has been set, either
in the configuration file or using the `ValidationBootloader`.

If you want to use a different validator, you can request it from the `ValidationProviderInterface`.

```php app/src/Interface/Controller/UserController.php
use Spiral\Http\Request\InputManager;
use Spiral\Validation\ValidationProviderInterface;

class UserController
{
    public function create(InputManager $input, ValidationProviderInterface $provider)
    {
        $validator = $provider->getValidation('my-validator')->validate([
            'username' => $input->post('username'),
            'email' => $input->post('email'),
        ], [
            'username' => 'required',
            'email' => 'required|email',
        ]);

        // Validate the data...
        // Store the user in the database...
    }
}
```

## Custom validators

To create a custom validator, you will need to create a class that implements the `ValidationInterface` interface. This
interface defines a single method, validate, which takes an array of data to validate and an array of validation rules
as arguments and returns a validator object.

```php
namespace App\Validator;

use Spiral\Validation\ValidationInterface;
use Spiral\Validation\ValidatorInterface;

final class MyValidation implements ValidationInterface
{
    public function validate(mixed $data, array $rules, $context = null): ValidatorInterface
    {
        return (new MyValidator(
            new MyValidationService($rules)
        ))
          ->withData($data)
          ->withContext($context);
    }
}
```

You will also need to create a validator object that implements `ValidatorInterface` and is responsible for performing
the actual validation. This can be a standalone class or a class that extends one of the validator classes provided by
the Spiral Framework or another library. The validator should implement the logic for checking the data against the
validation rules and returning a boolean value indicating whether the data is valid or not.

```php
namespace App\Validator;

use Spiral\Validation\ValidatorInterface;

final class MyValidator implements ValidatorInterface
{
    protected array|object $data = [];
    protected mixed $context = null;
        
    public function __construct(
        private readonly MyValidationService $validationService
    ) {}
    
    public function isValid(): bool
    {
        return $this->validationService->validate($this->data, $this->context);
    }
    
    public function getErrors(): array
    {
        return $this->validationService->getErrors();
    }
    
    // other required methods
}
```

Now we can register the created validator. To do this, use the `register` method using  
`Spiral\Validation\ValidationProvider` class.

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
            'my-validator',
            static fn(Validation $validation): ValidationInterface => new MyValidation()
        );
    }
}
```

> **See more**
> Read more about Bootloaders [here](../framework/bootloaders.md).

It's worth noting that this is just one example of how you might create a custom validator in the Spiral Framework.
There are many other approaches and techniques you can use to customize and extend the validation process, depending on
your specific needs and requirements.