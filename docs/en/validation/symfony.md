# Validation — Symfony Validator

Spiral provides a validation component that allows you to validate data using
the [Symfony Validation bridge](https://github.com/spiral-packages/symfony-validator) package. This validation bridge
provides integration with the [Symfony Validator](https://github.com/symfony/validator) component, which is a more
powerful and feature-rich validation library.

> **See more**
> Read more about validation in the [Validation](factory.md) section.

## Installation

To install the component run the following command:

```terminal
composer require spiral-packages/symfony-validator
```

To enable the component, you just need to add `Spiral\Validation\Symfony\Bootloader\ValidatorBootloader`
to the bootloaders list, which is located in the class of your application.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Validation\Symfony\Bootloader\ValidatorBootloader::class,
    // ...
];
```

## Usage

When the validation component is enabled in your application, it will register itself with
the `\Spiral\Validation\Symfony\FilterDefinition` class as validation name and be available for use with the Spiral
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
        $validator = $provider->getValidation(\Spiral\Validation\Symfony\FilterDefinition::class)
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
use Spiral\Validation\Symfony\Attribute\Input\File;
use Spiral\Validation\Symfony\AttributesFilter;
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\Component\Validator\Constraints;

final class CreatePostFilter extends AttributesFilter
{
    #[Post]
    #[Constraints\NotBlank]
    #[Constraints\Length(min: 5)]
    public string $title;

    #[Post]
    #[Constraints\NotBlank]
    #[Constraints\Length(min: 5)]
    public string $slug;

    #[Post]
    #[Constraints\NotBlank]
    #[Constraints\Positive]
    public int $sort;
    
    #[File]
    #[Constraints\Image]
    public UploadedFile $image;
}
```

> **Note**
> All available validation rules are described in the official
> [Symfony documentation](https://symfony.com/doc/6.0/validation.html#constraints).

### Filter with FilterDefinition

If you prefer to configure validation rules using arrays, you can define fields mapping in a `filterDefinition` method.

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validation\Symfony\FilterDefinition;
use Symfony\Component\Validator\Constraints;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Validation\Symfony\Attribute\Input\File;

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
            'title' => [new Constraints\NotBlank(), new Constraints\Length(min: 5)],
            'slug' => [new Constraints\NotBlank(), new Constraints\Length(min: 5)],
            'sort' => [new Constraints\NotBlank(), new Constraints\Positive()],
            'image' => [new Constraints\Image()],
        ]);
    }
}
```

### Filter with array mapping

If you prefer to configure fields mapping using arrays, you can define fields mapping in a `filterDefinition` method.

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validation\Symfony\FilterDefinition;
use Symfony\Component\Validator\Constraints;

final class CreatePostFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'title' => [new Constraints\NotBlank(), new Constraints\Length(min: 5)],
            'slug' => [new Constraints\NotBlank(), new Constraints\Length(min: 5)],
            'sort' => [new Constraints\NotBlank(), new Constraints\Positive()],
            'image' => [new Constraints\Image()]
        ],
        [
            'title' => 'title',
            'slug' => 'slug',
            'sort' => 'sort',
            'image' => 'symfony-file:image'
        ]);
    }
}
```