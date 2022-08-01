# Composite Filters

The component provides the ability to create nested filters and a nested array of filters. To demonstrate the
composition, we will use a sample filter:

```php
class AddressFilter extends Filter
{
    protected const SCHEMA = [
        'city'    => 'data:city', 
        'address' => 'data:address'
    ];
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

You can create compound filters by nesting other filters inside them. Simply declare field origin as external field
class to do that:

```php
class ProfileFilter extends Filter
{
    protected const SCHEMA = [
        'name'    => 'data:name',
        'address' => AddressFilter::class
    ];
}
```

This Filter will accept the data in a format:

```json
{
  "name": "Antony",
  "address": {
    "city": "San Francisco",
    "address": "Address"
  }
}
```

You can get access to the nested filter using `getField` or magic `__get`:

```php
public function index(ProfileFilter $p)
{
    dump($p->address->city); // San Francisco
}
```

Both filters will be validated together. In case of an error in `address` filter the error will be mounted in sub-array:

```json
{
  "name": "This field is required.",
  "address": {
    "city": "This field is required."
  }
}
```

### Custom Prefix

In some cases, you might need to use data prefix different from the actual key assigned to the nested Filter, use array
notation in which the first element filter class name and second is data prefix:

```php
class ProfileFilter extends Filter
{
    protected const SCHEMA = [
        'name'    => 'data:name',
        'address' => [AddressFilter::class, 'addr']
    ];
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

> You can skip using the `address` key internally, errors will be mounted accordingly.

## Array of Filters

You can populate an array of filters at the same time. Use array with single element pointing to filter class
to declare the array filter:

```php
class MultipleAddressesFilter extends Filter
{
    protected const SCHEMA = [
        'key'       => 'data:key',
        'addresses' => [AddressFilter::class]
    ];
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

You can get access to the array filters via array accessor:

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

You can create an array of filters based on data prefix different from the key name in the Filter, use the second value
or the array in the schema. Unlike a single child, you must specify the `.*` to indicate that value is an array:

```php
class MultipleAddressesFilter extends Filter
{
    protected const SCHEMA = [
        'key'       => 'data:key',
        'addresses' => [AddressFilter::class, 'addr.*']
    ];
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

You can still access the nested array filters using `address` key:

```php
public function index(MultipleAddressesFilter $ma)
{
    dump($ma->getField('addresses')[0]->getField('city')); // San Francisco
    dump($ma->getField('addresses')[1]->getField('city')); // Minsk
}
```

## Composite Filters

You can use nested child filters as part of a larger composite Filter. Use prefix `.` (root) to do that:

```php
class CompositeFilter extends Filter
{
    protected const SCHEMA = [
        'name'    => 'data:name', 
        'address' => [AddressFilter::class, '.'], 
    ];
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
