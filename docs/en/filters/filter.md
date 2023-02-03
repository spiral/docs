# Filters — Filter object

The Filter class represents a set of request input fields that can be filtered and validated. It provides a set of basic
features for creating filters, such as the ability to bind request input data to filter properties and to access
filtered data.

## Validators

There are three validator bridges available for use with the Spiral Framework filers. You can use any of these
validator bridges in your application, depending on your needs and preferences.

### [Spiral Validator](../validation/spiral.md)

This is the default validator bridge. It is a simple, lightweight validator that can handle basic validation tasks.

```php
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

### [Symfony Validator](../validation/symfony.md)

This validator bridge provides integration with the Symfony Validator component, which is a more powerful and
feature-rich validation library.

```php
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

### [Laravel Validator](../validation/laravel.md)

This validator bridge provides integration with the Laravel Validator, which is a validation component used in the
Laravel framework.

```php
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

## Usage

All the filter objects should implement `Spiral\Filters\Model\FilterInterface`. The interface is injectable, it means
that when you request a filter class from the container, it will be automatically created and request data will be
mapped to each filter property.

> **See more**
> Read more about injectors in the [Advanced — Container injectors](../advanced/injectors.md) section.

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

Now we can use the filter in our application. There are three ways to do it:

:::: tabs

::: tab DI
Simply request the filter as a dependency for example, in some controller method:

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

:::

::: tab Filter Provider
You can also use `Spiral\Filters\Model\FilterProviderInterface->createFilter` to create a filter instance:

```php
$provider = $container->get(\Spiral\Filters\Model\FilterProviderInterface::class);

$provider->createFilter(
    MyFilter::class, 
    $container->get(\Spiral\Filters\InputInterface::class)
);
```

:::

::: tab Container
When you request a filter from the container, it will be automatically created and request data will be mapped.

```php
dump($container->get(MyFilter::class)); 
```

:::

::::

## Filter Schema

There are two ways to define the filter schema:

:::: tabs

::: tab Attributes

```php
use Spiral\Filters\Attribute\Input\Post;

class MyFilter implements FilterInterface
{
    #[Post(key: 'text')]
    public string $text;
}
```

Request data will be mapped to the filter properties, and you will be able to access them directly.

```php
public function index(MyFilter $filter): void
{
    dump($filter->text);
}
```

:::

::: tab Array schema

In this case the filter should implement `Spiral\Filters\Model\HasFilterDefinition` and extend
the`Spiral\Filters\Model\Filter` abstract class to have access to the mapped data from the request.

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
            mappingSchema: ['text' => 'data:text']
        );
    }
}
```

Request data will be mapped according to the `mappingSchema` and you will be able to access them using `getData` method.

```php
public function index(MyFilter $filter): void
{
    dump($filter->getData());
}
```
\
:::

::::

#### Sources

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
> You can use both ways in your filter object. In both cases the filter provider will build a mapping schema for
> properties with attributes and then merge the schema with the schema from the filter definition.

## Attributes

To use request filter attributes, you can use one of the available attributes to specify where the data for the property
should be sourced from. For example, you could use the `Spiral\Filters\Attribute\Input\Query` attribute to map a query 
string parameter to a class property, like this:

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

In this example, the `Query` attribute will map the value of the query string parameter with the key `username` to the
`login` property.

When using an attribute, you can either specify a `key` argument or omit it. If you omit the key argument, the attribute
will use the name of the class property as the key.

For example:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Query]
    public string $login;
}
```

In this case, the `Query` attribute will map the value of the query string parameter with the key `login` to the `login`
property.

You can use multiple attributes in a single filter class to map different parts of the request data to different class
properties.

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

    #[Post(key: 'user')]
    public string $username;

    #[Post]
    public string $password;

    #[Post(key: 'remember')]
    public string $rememberMe;
}
```

By using multiple attributes in this way, you can easily extract and map different pieces of request data to the
appropriate class properties.

### Available attributes

Here is a list of the available request filter attributes:

| Attribute                                     | Description                                                                                                         |
|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Spiral\Filters\Attribute\Input\Post           | a value from the **POST** data.                                                                                     |
| Spiral\Filters\Attribute\Input\Query          | a value from the query string                                                                                       |
| Spiral\Filters\Attribute\Input\Input          | a value from either the POST data or the query string.                                                              |
| Spiral\Filters\Attribute\Input\Data           | a value from any of the request data (POST, query string, etc.).                                                    |
| Spiral\Filters\Attribute\Input\File           | an uploaded file. It will return `Psr\Http\Message\UploadedFileInterface` object or `null`.                         |
| Spiral\Filters\Attribute\Input\Cookie         | a value from a cookie.                                                                                              |
| Spiral\Filters\Attribute\Input\Header         | a value from the request headers.                                                                                   |
| Spiral\Filters\Attribute\Input\IsAjax         | a boolean value indicating whether the request was made with the `X-Requested-With` header set to `xmlhttprequest`. |
| Spiral\Filters\Attribute\Input\IsJsonExpected | a boolean value indicating whether the client expects a `application/json` response.                                |
| Spiral\Filters\Attribute\Input\IsSecure       | a boolean value indicating whether the request was made over HTTPS.                                                 |
| Spiral\Filters\Attribute\Input\Method         | the HTTP method of the request (e.g. GET, POST, etc.).                                                              |
| Spiral\Filters\Attribute\Input\Path           | the current request path.                                                                                           |
| Spiral\Filters\Attribute\Input\RemoteAddress  | the IP address of the client.                                                                                       |
| Spiral\Filters\Attribute\Input\Route          | a value from the current route attributes.                                                                          |
| Spiral\Filters\Attribute\Input\Server         | a value from the request server data.                                                                               |
| Spiral\Filters\Attribute\Input\Uri            | the current page URI in the form of a `Psr\Http\Message\UriInterface` object.                                       |
| Spiral\Filters\Attribute\Input\BearerToken    | the value of the `Authorization` header to a class property.                                                        |

#### Route Parameters

Every route writes the matching parameters into the ServerRequestInterface attribute `matches`, is it possible to access
route values inside your filter.

```php
$router->setRoute(
    'sample',
    new Route('/action/<id>.html', new Controller(HomeController::class))
);
```

:::: tabs

::: tab Attributes
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
:::

::: tab Array schema

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
:::

::::



### Data sanitization

The `Spiral\Filters\Attribute\Setter` attribute allows you to apply a filter function to the incoming value
before it is set on the class property. This can be useful if you want to perform some kind of transformation or
manipulation on the value before it is stored in the class.

Here is an example of how you can use the attribute:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Query]
    #[Setter(filter: 'trim')]
    public string $login;

    #[Query]
    #[Setter(filter: 'intval')]
    public int $age;

    #[Query]
    #[Setter(filter: [self::class, 'sanitizeContent'])]
    public string $description = '';
}
```

> **Note**
> You can use any default PHP functions like `intval`, `strval` etc.

### Creating a custom attribute

You can create your own custom request filter attributes by extending the `Spiral\Filters\Attribute\Input\AbstractInput`
class and implementing the `getValue` and `getSchema` methods. This allows you to define your own data retrieval logic
and specify the type of value that should be returned by the attribute.

```php
namespace App\Validation\Attribute;

use Spiral\Attributes\NamedArgumentConstructor;
use Spiral\Filters\Attribute\Input\AbstractInput;
use Spiral\Filters\InputInterface;
use Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile;

#[\Attribute(\Attribute::TARGET_PROPERTY), NamedArgumentConstructor]
final class Base64DecodedQuery extends AbstractInput
{
    /**
     * @param non-empty-string|null $key
     */
    public function __construct(
        public readonly ?string $key = null,
    ) {
    }

    public function getValue(InputInterface $input, \ReflectionProperty $property): ?string
    {
        $value = $input->getValue('query', $this->getKey($property));
        if ($value === null) {
            return null;
        }
        
        return \base64_decode($value);
    }

    public function getSchema(\ReflectionProperty $property): string
    {
        return 'query:' . $this->getKey($property);
    }
}
```

After that, we can use our attribute in the Filter:

```php
namespace App\Filter;

use App\Validation\Attribute\Base64DecodedQuery;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Base64DecodedQuery]
    public ?string $hash = null;
}
```

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

The validation rules can be defined using the same approach as in [Validator](../validation/spiral.md) component.

> **Note**
> FilterDefinition class should implement `Spiral\Filters\Model\ShouldBeValidated` if a filter object should be
> validated.

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
