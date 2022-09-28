# Filter Object

A Filter object uses to perform complex data validation and filtration using PSR-7 or any other input.

You can use filters in two ways:

1. Map data from the request into object properties.
2. Map and automatically validate data from the request.

## Validators

If you need only populate data from the request you don't need any validators for it. But if you need to validate data,
at first, you need to choose a validator for it.

There are three validators for Spiral Framework you can use:

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

All filter objects should implement `Spiral\Filters\Model\FilterInterface`. The interface will add the ability inject
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

You can use `Spiral\Filters\Model\FilterProviderInterface`->`createFilter` also to create an instance:

```php
$provider = $container->get(\Spiral\Filters\Model\FilterProviderInterface::class);
$provider->createFilter(MyFilter::class, $container->get(\Spiral\Filters\InputInterface::class));
```

or simply request filter as a dependency for example, in some controller

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

- Using array schema mapping. In this case a filter should implement `Spiral\Filters\Model\HasFilterDefinition` and
  extend `Spiral\Filters\Model\Filter` class to have access to the mapped data from the request.

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
> You can use both ways in your filter object. In this case filter provider will build mapping schema for properties
> with attributes and then will merge the schema with schema from filter definition.

### Attributes

Add properties with the needed type and add an attribute that points to the data source.
For example, we can tell our Filter to map field `login` to the QUERY param `username`:

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

### Array based Filters

For example, we can tell our Filter to point field `login` to the QUERY param `username`:

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
> The most common source is `data` (points to PSR-7 - parsed body), you can use this data to fetch values from incoming
> JSON payloads.

### Dot Notation

The data **origin** can be specified using dot notation pointing to some nested structure.

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
> The error messages will be correctly mounted into the original location. You can also use composite filters for more
> complex use-cases.

### Other Sources

By design, you can use any method of [InputManager](../http/request-response.md) as source where origin is passed
parameter. Following sources are available:

| Source         | Description                                                   |
|----------------|---------------------------------------------------------------|
| uri            | Current page Uri in a form of `Psr\Http\Message\UriInterface` |
| path           | Current page path                                             |
| method         | Http method (GET, POST, ...)                                  |
| isSecure       | If https used.                                                |
| isAjax         | If `X-Requested-With` set as `xmlhttprequest`                 |
| isJsonExpected | When client expects `application/json`                        |
| remoteAddress  | User ip address                                               |

> **Note**
> Read more about InputManager [here](../http/request-response.md).

For example to check if a user request made over https.

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

Every route writes matched parameters into ServerRequestInterface attribute `matches`, is it possible to access route
values inside your filter.

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
in case of typecast error:

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
> You can use any of the default PHP functions like `intval`, `strval` etc.

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

The validation rules can be defined using same approach as in [Validator](../validation/spiral.md) component.

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

By default, the Spiral Framework doesn't handle filter validation errors. When some of the filter rule has an error,
`Spiral\Filters\Exception\ValidationException` exception will be thrown.

There are two ways to handle a validation exception:
 - Middleware
 - Interceptions

Both ways are similar. You can see an example of validation exception handler below:

```php
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Spiral\Filters\Exception\ValidationException;

class JsonValidationMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ResponseFactoryInterface $responseFactory
    ) 
    {}

    public function process(Request $request, RequestHandlerInterface $handler): Response
    {
        try {
            return $handler->handle($request);
        } catch (ValidationException $e) {
            return $this->responseFactory->createResponse($e->getCode(), $e->getMessage())
              ->getBody()
              ->write(\json_encode([
                  'errors' => $e->errors,
              ]));
        }
    }
}
```

And then you need to register middleware for specific route group.

```php
namespace App\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Csrf\Middleware\CsrfMiddleware;
use Spiral\Session\Middleware\SessionMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                CookiesMiddleware::class,
                SessionMiddleware::class,
                CsrfMiddleware::class,
            ],
            'api' => [
                JsonValidationMiddleware::class  // <===== Our new middleware
            ],
        ];
    }
}
```

### Custom Errors

You can specify the custom error message to any of the rules similar way as in the validator component.

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

If you plan to localize error message later, wrap the text in `[[]]` to automatically index and replace the translation:

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

Once the Filter configured you can access its fields (filtered data).
The `Spiral\Filters\Model\Interceptor\ValidateFilterInterceptor` will automatically validate the data when the filter
is requested and throw a `Spiral\Filters\Exception\ValidationException` if the data is not valid.

### Get Fields

To get a filtered data, use filter properties or method `getData` (if it extends `Spiral\Filters\Model\Filter`):

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

Following fields are available:

```php
public function index(MyFilter $filter): void
{
    dump($filter->getData()); // {name: ..., email: ...}

    // or
    dump($filter->email);
    dump($filter->name);
}
```
