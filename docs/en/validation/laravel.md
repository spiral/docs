# Validation — Laravel Validator

Spiral provides a validation component that allows you to validate data using
the [Laravel Validation bridge](https://github.com/spiral-packages/laravel-validator) package. This validation bridge
provides integration with the Laravel Validator, which is a validation component used in the Laravel framework.

> **See more**
> Read more about validation in the [Validation](factory.md) section.

## Installation

To install the component run the following command:

```terminal
composer require spiral-packages/laravel-validator
```

To enable the component, you just need to add `Spiral\Validation\Laravel\Bootloader\ValidatorBootloader` to the
bootloaders list, which is located in the class of your application.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Validation\Laravel\Bootloader\ValidatorBootloader::class,
    // ...
];
```

## Usage

When the validation component is enabled in your application, it will register itself with
the `\Spiral\Validation\Laravel\FilterDefinition` class as validation name and be available for use with the Spiral
Framework validation component.

You can use the `Spiral\Validator\ValidatorInterface` interface to access the validator and perform validation tasks.
Alternatively, you can use the `Spiral\Validation\ValidationProviderInterface` interface to access the validator by its
class name.

```php
use Spiral\Http\Request\InputManager;
use Spiral\Validation\ValidationProviderInterface;

class UserController
{
    public function create(InputManager $input, ValidationProviderInterface $provider)
    {
        $validator = $provider->getValidation(\Spiral\Validation\Laravel\FilterDefinition::class)
            ->validate(...);
    }
}
```

## Filters

The `spiral/filters` component is a tool for validating HTTP request data in Spiral. It allows you to create
a "Filter" object, which defines the required data that should be extracted from the request object and mapped into the
filter object's properties.

> **See more**
> Read more about filters in the [Filters — Filter object](../filters/filter.md) section.

### Filter with attributes

One way to implement the request fields mapping is through the use of PHP attributes. This allows you to specify which
request field should be mapped to each filter property.

Here is an example of filter object with attributes:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validation\Laravel\FilterDefinition;
use Spiral\Validation\Laravel\Attribute\Input\File;
use Symfony\Component\HttpFoundation\File\UploadedFile;

final class CreatePostFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $title;

    #[Post]
    public string $slug;

    #[Post]
    public int $sort;

    #[File]
    public UploadedFile $image;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'title' => 'string|required|min:5',
            'slug' => 'string|required|min:5',
            'sort' => 'integer|required',
            'image' => 'required|image'
        ]);
    }
}
```

By implementing the `Spiral\Filters\Model\HasFilterDefinition` interface you can specify the validation rules that
should be applied to the data contained in the filter object. The Validation component will then use these rules to
validate the data when the filter object is used.

> **Note**
> The validation rules are described in the official
> [Laravel documentation](https://laravel.com/docs/9.x/validation#available-validation-rules).

### Filter with array mapping

If you prefer to configure fields mapping using arrays, you can define fields mapping in a `filterDefinition` method.

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validation\Laravel\FilterDefinition;

final class CreatePostFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'title' => 'string|required|min:5',
            'slug' => 'string|required|min:5',
            'sort' => 'integer|required',
            'image' => 'required|image'
        ], [
            'title' => 'title',
            'slug' => 'slug',
            'sort' => 'sort',
            'image' => 'symfony-file:image'
        ]);
    }
}
```