# Laravel Validator

You can validate your data using the [Laravel Validation](https://github.com/illuminate/validation) package.
Spiral provides a [Laravel validator bridge](https://github.com/spiral-packages/laravel-validator) component 
that allows you to quickly and easily integrate `Laravel Validation` into a project based on the `Spiral Framework`.

> **Note**
> Read more about [Validation](factory.md) in the Spiral Framework.

## Installation

To install the component:

```bash
composer require spiral-packages/laravel-validator
```

To enable the component, you just need to add `Spiral\Validation\Laravel\Bootloader\ValidatorBootloader` 
to the bootloaders list, which is located in the class of your application.

```php
protected const LOAD = [
    // ...
    \Spiral\Validation\Laravel\Bootloader\ValidatorBootloader::class,
    // ...
];
```

## Usage

First of all, need to create a filter that will receive incoming data that will be validated by the validator.

### Filter with attributes

Create a filter class and extend it from the base filter class `Spiral\Filters\Model\Filter`, add 
`Spiral\Filters\Model\HasFilterDefinition` interface. Implement the `filterDefinition` method, which should return 
a `Spiral\Validation\Laravel\FilterDefinition` object with validation rules.

> **Note**
> The validation rules are described in the official 
> [Laravel documentation](https://laravel.com/docs/9.x/validation#available-validation-rules).

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

### Filter with array mapping

If you prefer to configure fields mapping in an array, you can define fields mapping in a `filterDefinition` method.

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
        return new FilterDefinition(
            [
                'title' => 'string|required|min:5',
                'slug' => 'string|required|min:5',
                'sort' => 'integer|required',
                'image' => 'required|image'
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
