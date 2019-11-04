# Composite Filters
The component provide the ability to create nested filters and nested array of filters.

## Child Filter

### Custom Prefix

## Array of Filters

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