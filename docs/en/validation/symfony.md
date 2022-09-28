# Symfony Validator

You can validate your data using the [Symfony Validator](https://github.com/symfony/validator) package.
Spiral provides a [Symfony Validator bridge](https://github.com/spiral-packages/symfony-validator) component
that allows you to quickly and easily integrate `Symfony Validator` into a project based on the `Spiral Framework`.

> **Note**
> Read more about [Validation](factory.md) in the Spiral Framework.

## Installation

To install the component:

```bash
composer require spiral-packages/symfony-validator
```

To enable the component, you just need to add `Spiral\Validation\Symfony\Bootloader\ValidatorBootloader` 
to the bootloaders list, which is located in the class of your application.

```php
namespace App;

use Spiral\Validation\Symfony\Bootloader\ValidatorBootloader;

class App extends Kernel
{
    protected const LOAD = [
        // ...
        ValidatorBootloader::class,
        // ...
    ];
}
```

## Usage

First of all, need to create a filter that will receive incoming data that will be validated by the validator.

### Filter with attributes

Create a filter class and extend it from the base filter class with attributes 
`Spiral\Validation\Symfony\AttributesFilter`. Define the required properties, and add attributes to them indicating 
the data source and validation rules. All available validation rules are described in the official
[Symfony documentation](https://symfony.com/doc/6.0/validation.html#constraints).

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

### Filter with FilterDefinition

If you prefer to configure validation rules in an array, you can use a filter with a `filterDefinition` method 
definition. Create a filter class and extend it from the base filter class `Spiral\Filters\Model\Filter`, add 
`Spiral\Filters\Model\HasFilterDefinition` interface. Implement the `filterDefinition` method, which should return 
a `Spiral\Validation\Symfony\FilterDefinition` object with data mapping rules and validation rules.

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
        return new FilterDefinition(
            [
                'title' => [new Constraints\NotBlank(), new Constraints\Length(min: 5)],
                'slug' => [new Constraints\NotBlank(), new Constraints\Length(min: 5)],
                'sort' => [new Constraints\NotBlank(), new Constraints\Positive()],
                'image' => [new Constraints\Image()]
            ]
        );
    }
}
```

If you prefer to configure fields mapping in an array, you can define fields mapping in a `filterDefinition` method.

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
        return new FilterDefinition(
            [
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
            ]
        );
    }
}
```

> **Note**
> Read more about [Filters](../filters/configuration.md) in the Spiral Framework.

### Request validation

When requesting a filter from a container, the data will be automatically validated.
When validation errors occur, a `Spiral\Filters\Exception\ValidationException` containing validation errors will be thrown.

```php
use App\Filter\CreatePostFilter;
use Spiral\Filters\Exception\ValidationException;

try {
    $filter = $this->container->get(CreatePostFilter::class); 
} catch (ValidationException $e) {
    var_dump($e->errors); // Errors processing
}
```
