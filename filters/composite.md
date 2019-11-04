# Composite Filters
The component provide the ability to create nested filters and nested array of filters. To demonstrate the composition
we will use sample filter:

```php
class AddressFilter extends Filter
{
    protected const SCHEMA = [
        'city'    => 'data:city', 
        'address' => 'data:address'
    ];
}
```

This filter can accept the following data format:

```json
{
  "city": "San Francisco", 
  "address": "Address"
}
```

## Child Filter

### Custom Prefix

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

Such filter can accept following data format:

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

> The errors will be mounted accordingly.

### Custom Prefix

### Iterate Source

## Composite Filters
You are use nested child filters as part of larger composite filter. Use prefix `.` (root) in order to do that:

```php
class CompositeFilter extends Filter
{
    protected const SCHEMA = [
        'name'    => 'data:name', 
        'address' => [AddressFilter::class, '.'], 
    ];
}
```

The `AddressFilter` will receive data from top level, meaning you can send request like that:

```json
{
  "name": "Antony",
  "city": "San Francisco", 
  "address": "Address"
}
```