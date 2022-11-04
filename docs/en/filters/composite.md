# Composite Filters

The component provides an  ability to create nested filters and a nested array of filters. To demonstrate the
composition, we will use a sample filter:

```php
namespace App\Request;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;

class AddressFilter extends Filter
{
    #[Post]
    public string $city;

    #[Post]
    public string $address;
}
```

This Filter can accept the following data format:

```json
{
  "city": "San Francisco",
  "address": "Address"
}
```

## Child Filter

You can create compound filters by nesting other filters inside them. Simply declare the field with a child filter 
and add the attribute `Spiral\Filters\Attribute\NestedFilter`:

```php
namespace App\Request;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\NestedFilter;
use Spiral\Filters\Model\Filter;

class ProfileFilter extends Filter
{
    #[Post]
    public string $name;

    #[NestedFilter(class: AddressFilter::class)]
    public AddressFilter $address;
}
```

This Filter will accept the data in the format:

```json
{
  "name": "Antony",
  "address": {
    "city": "San Francisco",
    "address": "Address"
  }
}
```

You can get access to the nested Filter using class properties:

```php
public function index(ProfileFilter $profile): void
{
    dump($profile->address->city); // San Francisco
}
```

Both the filters will be validated together. In case of an error in the `address` filter, the error will be mounted in a sub-array:

```json
{
  "name": "This field is required.",
  "address": {
    "city": "This field is required."
  }
}
```

### Custom Prefix

In some cases, you might need to use a data prefix different from the actual key assigned to the nested Filter, use the
parameter `prefix` in the `NestedFilter`:

```php
namespace App\Request;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\NestedFilter;
use Spiral\Filters\Model\Filter;

class ProfileFilter extends Filter
{
    #[Post]
    public string $name;

    #[NestedFilter(class: AddressFilter::class, prefix: 'addr')]
    public AddressFilter $address;
}
```

This Filter can accept the following data format:

```json
{
  "name": "This field is required.",
  "addr": {
    "city": "This field is required."
  }
}
```

> **Note**
> You can skip the use of the `address` key internally, errors will be mounted accordingly.

## Array of Filters

You can populate an array of filters at the same time. Use an array property type and add  theattribute 
`Spiral\Filters\Attribute\NestedArray` with a Filter class for each element as the parameter `class` and data input:

```php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\NestedArray;
use Spiral\Filters\Model\Filter;

class MultipleAddressesFilter extends Filter
{
    #[Post]
    public string $name;

    #[NestedArray(class: AddressFilter::class, input: new Post('addresses'))]
    public array $addresses;
}
```

Such Filter can accept the following data format:

```json
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

You can access array filters via the array accessor:

```php
public function index(MultipleAddressesFilter $ma)
{
    dump($ma->addresses[0]->city); // San Francisco
    dump($ma->addresses[1]->city); // Minsk
}
```

> **Note**
> The errors will be mounted accordingly.

### Custom Prefix

You can create an array of filters based on a data prefix different from the key name in the Filter, use the
parameter `prefix` in the `NestedArray`:

```php
namespace App\Request;

use Spiral\Filters\Attribute\Input\Input;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\NestedArray;
use Spiral\Filters\Model\Filter;

class MultipleAddressesFilter extends Filter
{
    #[Post]
    public string $name;

    #[NestedArray(class: AddressFilter::class, input: new Input('addresses'), prefix: 'addr')]
    public array $addresses;
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

You can still access the nested array filters using the `addresses` property:

```php
public function index(MultipleAddressesFilter $ma): void
{
    dump($ma->addresses[0]->city); // San Francisco
    dump($ma->addresses[1]->city); // Minsk
}
```

## Composite Filters

You can use nested child filters as part of a larger composite Filter. Use the prefix `.` (root) to do that:

```php
namespace App\Request;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\NestedFilter;
use Spiral\Filters\Model\Filter;

class ProfileFilter extends Filter
{
    #[Post]
    public string $name;

    #[NestedFilter(class: AddressFilter::class, prefix: '.')]
    public AddressFilter $address;
}
```

The `AddressFilter` will receive data from the top-level, meaning you can send a request like that:

```json
{
  "name": "Antony",
  "city": "San Francisco",
  "address": "Address"
}
```
