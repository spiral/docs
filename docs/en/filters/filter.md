# Filter Object

The Filter object is used to perform complex data validation and filtration using PSR-7 or any other input.

You can use filters in two ways:

1. Map the data from the request into the object properties.
2. Map and automatically validate the data from the request.

## Validators

If you only need to populate the data from the request you don't need any validators for it. But if you need to validate
data, at first, you need to choose a validator for it.

There are three validators for Spiral Framework that you can use:

- [Spiral Validator](https://github.com/spiral/validator)

```php
<?php

declare(strict_types=1);

namespace App\Filters;

use Spiral\Filters\Model\FilterInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\File;

class CreatePostFilter implements FilterInterface, HasFilterDefinition
{
    #[Post(key: 'title')]
    public string $title;
    
    #[Post(key: 'text')]
    public string $text;
    
    #[File]
    public UploadedFile $image;
    
    // ...

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(
            validationRules: [
                'title' => [
                    ['notEmpty'],
                    ['string::length', 50]
                ],
                'text' => [['notEmpty']],
                'image' => [['image::valid'], ['file::size', 1024]]
                
                // ...
            ]
        );
    }
}
```

- [Symfony Validator](https://github.com/spiral-packages/symfony-validator)

```php
<?php

declare(strict_types=1);

namespace App\Filters;

use Psr\Http\Message\UploadedFileInterface;
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

- [Laravel Validator](https://github.com/spiral-packages/symfony-validator)

```php
<?php

declare(strict_types=1);

namespace App\Filters;

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

We will use [Spiral Validator](https://github.com/spiral/validator) package in the article examples.

## Usage

All the filter objects should implement `Spiral\Filters\Model\FilterInterface`. The interface will add the ability to
inject
filters with populated data from `Spiral\Filters\InputInterface` from the container.

```php
namespace App\Filter;

use Spiral\Filters\Model\FilterInterface;
use Spiral\Filters\Attribute\Input\Post;

class MyFilter implements FilterInterface
{
    #[Post(key: 'text')]
    public string $text;
}
```

```php
dump($container->get(MyFilter::class)); 
```

You can also use `Spiral\Filters\Model\FilterProviderInterface`->`createFilter` to create an instance:

```php
$provider = $container->get(\Spiral\Filters\Model\FilterProviderInterface::class);
$provider->createFilter(MyFilter::class, $container->get(\Spiral\Filters\InputInterface::class));
```

or simply request the filter as a dependency for example, in some controller

```php
use App\Filter\MyFilter;

class HomeController
{
    public function index(MyFilter $filter): void
    {
        dump($filter);
    }
}
```

## Filter Schema

There are two ways to define the filter schema.

- Using attributes with filter properties.

```php
class MyFilter implements FilterInterface
{
    #[Post(key: 'text')]
    public string $text;
}
```

```php
$filter = $container->get(MyFilter::class);

dump($filter->text); // '...'
dump($filter->getData()); // ['text' => '...'] 
```

- Using an array schema mapping. In this case the filter should implement `Spiral\Filters\Model\HasFilterDefinition` and
  extend the `Spiral\Filters\Model\Filter` class to have access to the mapped data from the request.

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(mappingSchema: 
            [
                'text' => 'data:text'
            ]
        );
    }
}
```

```php
$filter = $container->get(MyFilter::class);
dump($filter->getData()); // ['text' => '...'] 
```

> **Note**
> You can use both ways in your filter object. In this case the filter provider will build a mapping schema for
> properties with attributes and then merge the schema with the schema from the filter definition.

### Attributes

You can add properties with the needed type and add an attribute that points to the data source.
For example, we can tell our Filter to map the field `login` to the QUERY param `username`:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Query(key: 'username')]
    public string $login;
}
```

You can combine multiple sources inside the Filter object:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Cookie;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Query(key: 'redirectURL')]
    public string $redirectTo;

    #[Cookie]
    public string $memberCookie;

    #[Post]
    public string $username;

    #[Post]
    public string $password;

    #[Post]
    public string $rememberMe;
}
```

### Creating a custom attribute

You can create your own attribute, which will contain a custom data retrieval logic.
For example, we need to receive uploaded files as a Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile object.
First of all, we need to create custom input bag for this:

```php
namespace App\Http\Request;

use Spiral\Http\Request\InputBag;
use Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile;

final class FilesBag extends InputBag
{
    public function __construct(array $data, string $prefix = '')
    {
        foreach ($data as $name => $file) {
            $data[$name] = new UploadedFile($file, fn(): string => $this->getTemporaryPath());
        }

        parent::__construct($data, $prefix);
    }

    protected function getTemporaryPath(): string
    {
        return \tempnam(\sys_get_temp_dir(), \uniqid('symfony', true));
    }
}
```

After that, add the created `FilesBag` by the method `addInputBag` in the `HttpBootloader`.

```php
namespace App\Bootloader;

use App\Http\Request\FilesBag;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;

class AppBootloader extends Bootloader
{
    public function init(HttpBootloader $http): void
    {
        $http->addInputBag('symfonyFiles', [
            'class'  => FilesBag::class,
            'source' => 'getUploadedFiles',
            'alias' => 'symfony-file'
        ]);
    }
}
```

Let's create a filter attribute that will get the uploaded file. The attribute class must be extended from 
the `Spiral\Filters\Attribute\Input\AbstractInput` class.

```php
namespace App\Validation\Attribute;

use Spiral\Attributes\NamedArgumentConstructor;
use Spiral\Filters\Attribute\Input\AbstractInput;
use Spiral\Filters\InputInterface;
use Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile;

#[\Attribute(\Attribute::TARGET_PROPERTY), NamedArgumentConstructor]
final class File extends AbstractInput
{
    /**
     * @param non-empty-string|null $key
     */
    public function __construct(
        public readonly ?string $key = null,
    ) {
    }

    public function getValue(InputInterface $input, \ReflectionProperty $property): ?UploadedFile
    {
        return $input->getValue('symfony-file', $this->getKey($property));
    }

    public function getSchema(\ReflectionProperty $property): string
    {
        return 'symfony-file:' . $this->getKey($property);
    }
}
```

After that, we can use our attribute in the Filter:

```php
namespace App\Filter;

use App\Validation\Attribute\File;
use Spiral\Filters\Model\Filter;
use Symfony\Component\HttpFoundation\File\UploadedFile;

class MyFilter extends Filter
{
    #[File]
    public UploadedFile $image;
}
```

### Array based Filters

For example, we can tell our Filter to point the field `login` to the QUERY param `username`:

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(mappingSchema: 
            [
                'login' => 'query:username'
            ]
        );
    }
}
```

You can combine multiple sources inside the Filter:

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(mappingSchema: 
            [
                'redirectTo'   => 'query:redirectURL',
                'memberCookie' => 'cookie:memberCookie',
                'username'     => 'data:username',
                'password'     => 'data:password',
                'rememberMe'   => 'data:rememberMe'
            ]
        );
    }
}
```

> **Note**
> The most common source is `data` (points to PSR-7 - the parsed body), you can use this data to fetch values from the
> incoming JSON payloads.

### Dot Notation

The data **origin** can be specified using the dot notation pointing to some nested structure.

Via attributes:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post(key: 'names.first')]
    public string $firstName;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'firstName' => ['string', 'required']
        ]);
    }
}
```

Via array mapping:

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(
            [
                'firstName' => ['string', 'required']
            ], 
            [
                'firstName' => 'data:names.first'
            ]
        );
    }
}
```

We can accept and validate the following data structure:

```json
{
  "names": {
    "first": "Antony"
  }
}
```

> **Note**
> Error messages will be correctly mounted into the original location. You can also use composite filters for more
> complex use-cases.

### Other Sources

By design, you can use any method of [InputManager](../http/request-response.md) as a source where origin is passed
parameter. The following sources are available:

| Source         | Description                                                       |
|----------------|-------------------------------------------------------------------|
| uri            | The current page Uri in a form of `Psr\Http\Message\UriInterface` |
| path           | The current page path                                             |
| method         | Http method (GET, POST, ...)                                      |
| isSecure       | If https is used                                                  |
| isAjax         | If `X-Requested-With` is set as `xmlhttprequest`                  |
| isJsonExpected | When the client expects `application/json`                        |
| remoteAddress  | User ip address                                                   |

> **Note**
> Read more about the InputManager [here](../http/request-response.md).

For example, to check if a user request is made over https.

Via attributes:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\IsSecure;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[IsSecure]
    public bool $httpsRequest;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'httpsRequest' => [
                ['required', 'error' => 'Connection is not secure.']
            ]
        ]);
    }
}
```

Via array mapping:

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(
            [
                'httpsRequest' => [
                    ['required', 'error' => 'Connection is not secure.']
                ]
            ],
            [
                'httpsRequest' => 'isSecure:httpsRequest'
            ]
        );
    }
}
```

### Route Parameters

Every route writes the matching parameters into the ServerRequestInterface attribute `matches`, is it possible to access
route values inside your filter.

```php
$router->setRoute(
    'sample',
    new Route('/action/<id>.html', new Controller(HomeController::class))
);
```

Via attributes:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Route;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Route(key: 'id')]
    public string $routeId;
}
```

Via array mapping using `attribute:matches.{name}` notation:

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(mappingSchema: 
            [
                'routeId' => 'attribute:matches.id'
            ]
        );
    }
}
```

### Setters

Use setters to typecast the incoming value before passing it to the validator. The Filter will assign null to the value
in case of a typecast error:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Query(key: 'id')]
    #[Setter(filter: 'intval')]
    public int $number;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'number' => ['required', ['number::higher', 5]]
        ]);
    }
}
```

> **Note**
> You can use any default PHP functions like `intval`, `strval` etc.

```php
namespace App\Controller;

use App\Filter\MyFilter;

class HomeController
{
    public function index(MyFilter $filter): void
    {
        dump($filter->number); // always int
    }
}
```

## Validation

> **Note**
> FilterDefinition class should implement `Spiral\Filters\Model\ShouldBeValidated` if a filter object should be
> validated.

The validation rules can be defined using the same approach as in [Validator](../validation/spiral.md) component.

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'name' => ['string', 'required']
        ]);
    }
}
```

You can use all the checkers, conditions, and rules.

### Handle Validation errors

When some of the filter rule has an error, `Spiral\Filters\Exception\ValidationException` exception will be thrown.

Spiral Framework will automatically catch this exception via the `Spiral\Filter\ValidationHandlerMiddleware` middleware
and return a response with the error message via `Spiral\Filters\ErrorsRendererInterface`.

You just need to register middleware:

```php
namespace App\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Filter\ValidationHandlerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            // ...
            ValidationHandlerMiddleware::class,
        ];
    }
}
```

By default, the middleware uses `Spiral\Filter\JsonErrorsRenderer` for rendering filter errors. You can change the
renderer by binding your own implementation of `Spiral\Filters\ErrorsRendererInterface` to the container.

```php
namespace App\Filter;
use Psr\Http\Message\ResponseInterface;
use Spiral\Filters\ErrorsRendererInterface;
use Spiral\Http\ResponseWrapper;

final class CustomJsonErrorsRenderer implements ErrorsRendererInterface
{
    public function __construct(
        private readonly ResponseWrapper $wrapper
    ) {
    }

    public function render(array $errors, mixed $context = null): ResponseInterface
    {
        return $this->wrapper->json([
            'errors' => $errors,
            'context' => (string) $context
        ])
            ->withStatus(422, 'The given data was invalid.');
    }
}
```

And then you need to register middleware for specific route group.

```php
namespace App\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Filters\ErrorsRendererInterface;
use App\Filter\CustomJsonErrorsRenderer;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // Custom renderer to the container binding
    protected const SINGLETONS = [
        ErrorsRendererInterface::class => CustomJsonErrorsRenderer::class,
    ];

    // ...
}
```

### Custom Errors

You can specify a custom error message to any of the rules in the same way as in the validator component.

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'name' => [
                ['required', 'error' => 'Name must not be empty']
            ]
        ]);
    }
}
```

If you plan to localize the error message later, wrap the text in `[[]]` to automatically index and replace the
translation:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'name' => [
                ['required', 'error' => '[[Name must not be empty]]']
            ]
        ]);
    }
}
```

## Usage

Once the Filter is configured you can access its fields (filtered data).
The `Spiral\Filters\Model\Interceptor\ValidateFilterInterceptor` will automatically validate the data when the filter
is requested and throw a `Spiral\Filters\Exception\ValidationException` if the data is not valid.

### Get Fields

To get a filtered data, use filter properties or the method `getData` (if it extends `Spiral\Filters\Model\Filter`):

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;
    
    #[Post]
    public string $email;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'name' => ['required']
        ]);
    }
}
```

The following fields are available:

```php
public function index(MyFilter $filter): void
{
    dump($filter->getData()); // {name: ..., email: ...}

    // or
    dump($filter->email);
    dump($filter->name);
}
```
