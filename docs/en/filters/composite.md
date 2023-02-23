# Filters — Composite Filters

Spiral allows for the creation of composite filters, which are filters that are composed of other filters.

Let's imagine that we have a `AddressFilter` that represents a single address that can be used as a part of a profile
with its own validation rules:

> **Note**
> In our examples we will use [Spiral Validator](../validation/spiral.md) for validation, but you can use any other
> validation library.

```php app/src/Endpoint/Web/Filter/AddressFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class AddressFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $city;

    #[Post]
    public string $address;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            'city' => ['required', 'string'],
            'address' => ['required', 'string'],
        ]);
    }
}

```

Spiral provides two types of nested filters:

## Nested Filter

Spiral allows you to create compound filters by nesting other filters inside them. This is done by declaring a property 
in the parent filter class and decorating it with the `Spiral\Filters\attribute\NestedFilter` attribute. The attribute 
takes a class parameter, which is set to the child filter class. This allows the parent filter to accept input data in 
a nested format, where the property decorated with this attribute contains the data for the child filter. This makes it 
easy to validate and filter the data in multiple levels and reuse the child filter in different parent filters.

```php app/src/Endpoint/Web/Filter/ProfileFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\NestedFilter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class ProfileFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;

    #[NestedFilter(class: AddressFilter::class)]
    public AddressFilter $address;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            'name' => ['required', 'string'],
        ]);
    }
}
```

This Filter will accept the data in the format:

```json Post data
{
  "name": "Antony",
  "address": {
    "city": "San Francisco",
    "address": "Address"
  }
}
```

Once the request input data is passed through the filter, you can access the filtered data using the properties of the
filter class.

```php
public function index(ProfileFilter $profile): void
{
    dump($profile->address->city); // San Francisco
}
```

When using nested filters, both the parent filter and the child filter(s) will be validated together. If there are any
validation errors in the child filter, they will be mounted in a sub-array of the parent filter's errors. This allows
you to easily identify which errors belong to which filter, and makes it easier to display the errors to the user.

```json Validation errors
{
  "name": "This field is required.",
  "address": {
    "city": "This field is required."
  }
}
```

### Custom Prefix

The `NestedFilter` attribute allows you to specify a custom prefix for the data that is passed to the child filter. By
default, the prefix is the same as the key assigned to the nested filter property, but in some cases, you may need to
use a different prefix.

```php app/src/Endpoint/Web/Filter/ProfileFilter.php
class ProfileFilter extends Filter implements HasFilterDefinition
{
    #[NestedFilter(class: AddressFilter::class, prefix: 'addr')]
    public AddressFilter $address;
    
    // ...
}
```

The json data format that will work with this filter is:

```json Post data
{
  "name": "Antony",
  "addr": {
    "city": "San Francisco",
    "address": "Address"
  }
}
```

> **Note**
> You can skip the use of the `address` key internally, errors will be mounted accordingly.

### Composite Filters

You can use nested child filters as part of a larger composite Filter. Use the prefix `.` (root) to do that:

```php app/src/Endpoint/Web/Filter/MultipleAddressesFilter.php
class MultipleAddressesFilter extends Filter implements HasFilterDefinition
{
    #[NestedFilter(class: AddressFilter::class, prefix: '.')]
    public AddressFilter $address;
    
    // ...
}
```

The `AddressFilter` will receive data from the top-level, meaning you can send a request like that:

```json Post data
{
  "name": "Antony",
  "city": "San Francisco",
  "address": "Address"
}
```

## Array of Filters

You can use the `Spiral\Filters\attribute\NestedArray` attribute to populate an array of filters at the same time. In
order to use this attribute, you need to declare an array property in your filter class and decorate it with the
`NestedArray` attribute, specifying the class of the filter for each element in the array as the class parameter.

The `input` parameter in the attribute is used to specify the input source that contains the array of filters.

> **Note**
> List of available input sources can be found in
> the [Filters — Filter object](../filters/filter.md#available-attributes) section.

```php app/src/Endpoint/Web/Filter/MultipleAddressesFilter.php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\NestedArray;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class MultipleAddressesFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;

    #[NestedArray(class: AddressFilter::class, input: new Post]
    public array $addresses;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            'name' => ['required', 'string'],
        ]);
    }
}
```

This Filter will accept the data in the format:

```json Post data
{
  "key": "value",
  "addresses": [
    {
      "city": "San Francisco",
      "address": "Address"
    },
    {
      "city": "Minsk",
      "address": "Address #2"
    }
  ]
}
```

Once you have applied the filter, you can access the individual filters in the array using array notation.

```php
public function index(MultipleAddressesFilter $filter)
{
    dump($filter->addresses[0]->city); // San Francisco
    dump($filter->addresses[1]->city); // Minsk
}
```

> **Note**
> If there are any validation errors in the nested filters, they will be mounted according to the structure of the
> nested filters.

### Custom Prefix

Yes, you can pass the custom prefix as a parameter to the constructor of the input source class, when defining the
`NestedArray` attribute. This way, the filter will look for input data using the custom prefix instead of the default 
key name.

```php app/src/Endpoint/Web/Filter/MultipleAddressesFilter.php
class MultipleAddressesFilter extends Filter
{
    #[NestedArray(class: AddressFilter::class, input: new Post('addr'))]
    public array $addresses;
    
    // ...
}
```

This Filter supports the following data format:

```json
{
  "key": "value",
  "addr": [
    {
      "city": "San Francisco",
      "address": "Address"
    },
    {
      "city": "Minsk",
      "address": "Address #2"
    }
  ]
}
```