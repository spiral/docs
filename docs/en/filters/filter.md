# Filter Object

The Filter object used to perform complex data validation and filtration using PSR-7 or any other input.
The Filter object is different, depending on the Validator package used. This article covers the filters that using
with the [Spiral Validator](https://github.com/spiral/validator) package.

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
}
```

Use `Spiral\Filters\Model\FilterProviderInterface`->`createFilter` or simply request filter dependency to create an instance:

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

There are two variants for defining the filter scheme. Using `attributes` with filter properties or using `array
schema` mapping.

#### Attributes

Add properties with the needed type and add an attribute that points to the data source.
For example, we can tell our Filter to point field `login` to the QUERY param `username`:

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

You can combine multiple sources inside the Filter:

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

#### Array based Filters

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

The validation rules can be defined using same approach as in [validation](../security/validation.md) component.

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

### Custom Errors

You can specify the custom error message to any of the rules similar way as in the validation component.

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

To get a filtered data, use filter properties or method `getData` (if you are using a array schema mapping):

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

## Inheritance

You can extend one filter from another, the schema, validation, and setters will be inherited:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Validator\FilterDefinition;

class MyFilter extends BaseFilter
{
    #[Post]
    public string $name;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'name' => ['required']
        ]);
    }
}
```

Where `BaseFilter` is:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class BaseFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $token;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'token' => ['required']
        ]);
    }
}
```

Now filter `MyFilter` will require `token` value as well:

```json
{
  "status": 400,
  "errors": {
    "token": "This value is required.",
    "name": "This value is required."
  }
}
```
