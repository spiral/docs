# Data Grids
Use component `spiral/data-grid` and `spiral/data-grid-bridge` to generate Cycle and DBAL select queries automatically,
based on specifications provided by the end-user.

## Installation
To install the component:

```bash
$ composer require spiral/data-grid-bridge
```

Activate the bootloader `Spiral\DataGrid\Bootloader\GridBootloader` in your application after the Database and Cycle
bootloaders:

```php
protected const LOAD = [
    // ...
    Spiral\DataGrid\Bootloader\GridBootloader::class
];
```

## Usage
To use the data grid, you will need two base abstractions - grid factory and grid schema.

### Grid Schema
Grid Schema is the object which describes how the data selector should be configured based on user input. Use
`Spiral\DataGrid\GridSchema`:

```php
use Spiral\DataGrid; 
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\StringValue;

$schema = new DataGrid\GridSchema();

// allow user to paginate the result set with 10 results per page
$schema->setPaginator(new PagePaginator(10));

// allow use to sort by name
$schema->addSorter('id', new Sorter('id'));

// find by matching name, the value supplied by user
$schema->addFilter('name', new Like('name', new StringValue()));
```

> You can extend the GridSchema and initiate all the specifications in the constructor.

### Grid Factory
To use the defined grid schema, you will have to obtain an instance of a supported data source.
By default, the Cycle Select and Database Select Query are supported.

```php
use Spiral\DataGrid\GridFactory;
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\StringValue;

$schema = new GridSchema();
$schema->setPaginator(new PagePaginator(10));
$schema->addSorter('id', new Sorter('id'));
$schema->addFilter('name', new Like('name', new StringValue()));

/**
 * @var App\Database\UserRepository $users 
 * @var GridFactory $factory
 */
$result = $factory->create($users->select(), $schema);

print_r(iterator_to_array($result));
```

If you want any of the specifications be applied by default, you can pass them next way:
```php
/** @var Spiral\DataGrid\GridFactory $factory */
$factory = $factory->withDefaults([
    GridFactory::KEY_SORT     => ['id' => 'desc'],
    GridFactory::KEY_FILTER   => ['name' => 'Antony'],
    GridFactory::KEY_PAGINATE => ['page' => 3, 'limit' => 100]
]);
```
How to apply the specifications:
- to select users from the second page open page with POST or QUERY data like: `?paginate[page]=2`
- to activate the `like` filter: `?filter[name]=antony`
- to sort by id in ASC or DESC: `?sort[id]=desc`
- to get count of total values: `?fetchCount=1`
> These params are defined in the `GridFactory`, you can overwrite them.

If you need to count items using a complex function, you can pass a callable function via `withCounter` method:
```php
/** @var Spiral\DataGrid\GridFactory $factory */
$factory = $factory->withCounter(static function ($select): int {
    return count($select) * 2;
});
```
> This is a simple example, but this function might be very helpful in case of complex SQL requests with joins.

## Pagination specifications
### Page Paginator specification
This is a simple page+limit pagination:
```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;

$schema = new GridSchema();
$schema->setPaginator(new PagePaginator(10, [25, 50, 100, 500]));
// ...
```
From the user input, such paginator accepts an array with 2 keys, `limit` and `page`.
If limit is set it should be presented in the `allowedLimits` constructor param. 
```php
use Spiral\DataGrid\Specification\Pagination\PagePagination;

$paginator = new PagePaginator(10, [25, 50, 100, 500]);

$paginator->withValue(['limit' => 123]); // won't apply
$paginator->withValue(['limit' => 50]);  // will apply
$paginator->withValue(['limit' => 100]); // will apply

$paginator->withValue(['limit' => 100, 'page' => 2]);
```
Under the hood, this paginator converts `limit` and `page` into the `Limit` and `Offset` specification.
You are free to write your own paginator, like cursor-based one (for example: `lastID`+`limit`). 

## Sorter specifications
Sorters are specifications that carry sorting direction.
For sorters that can apply direction, you can pass one of the next values:
- `1`, `'1'`, `'asc'`, `SORT_ASC` for ascending order
- `-1`, `'-1'`, `'desc'`, `SORT_DESC` for descending order

Next specifications are available for grids for now:

* [ordered sorters](#sorter-specifications-ordered-sorters)
* [directional sorter](#sorter-specifications-directional-sorter)
* [sorter](#sorter-specifications-sorter)
* [sorter set](#sorter-specifications-sorter-set)

### Ordered sorters
`AscSorter` and `DescSorter` contain the expressions that should be applied with ascending (or descending) sorting order:
```php
use Spiral\DataGrid\Specification\Sorter;

$ascSorter = new Sorter\AscSorter('first_name', 'last_name');
$descSorter = new Sorter\DescSorter('first_name', 'last_name');
```

### Directional sorter
This sorter contains 2 independent sorters each for ascending and descending order.
By receiving the order via `withValue` we will get one of the sorters:
```php
use Spiral\DataGrid\Specification\Sorter;

$sorter = new Sorter\DirectionalSorter(
    new Sorter\AscSorter('first_name'),
    new Sorter\DescSorter('last_name')
);

// will sort by first_name asc
$ascSorter = $sorter->withDirection('asc');

// will sort by last_name desc
$descSorter = $sorter->withDirection('desc');
```
> Note that you can sort using different set of fields in both sorters.
> If you have the same set of fields, use [sorter](#sorter-specifications-sorter-specification) instead.

### Sorter
This is a sorter wrapper for a directional sorter in case you have the same fields for sorting in both directions:
```php
use Spiral\DataGrid\Specification\Sorter;

$sorter = new Sorter\Sorter('first_name', 'last_name');

// will sort by first_name and last_name asc
$ascSorter = $sorter->withDirection('asc');

// will sort by first_name and last_name desc
$descSorter = $sorter->withDirection('desc');
```

### Sorter set
This is just a way of combining sorters into one set, passing direction will apply it to the whole set:
```php
use Spiral\DataGrid\Specification\Sorter;

$sorter = new Sorter\SorterSet(
    new Sorter\AscSorter('first_name'),
    new Sorter\DescSorter('last_name'),
    new Sorter\Sorter('email', 'username')
    // ...
);

// will sort by first_name, email and username asc, also last_name desc
$ascSorter = $sorter->withDirection('asc');

// will sort by last_name, email and username desc, also first_name asc
$descSorter = $sorter->withDirection('desc');
```

## Filter specifications
Filters are specifications that carry values.
Values can be passed via the constructor directly. In this case the filter value is fixed and will be applied as is.
```php
use Spiral\DataGrid\Specification\Filter;

// name should be 'Antony'
$filter = new Filter\Equals('name', 'Antony');

// name is still 'Antony' 
$filter = $filter->withValue('John');   
```
If you pass the `ValueInterface` to the constructor then you can use `withValue()` method.
Then the incoming value will be checked if it matches the `ValueInterface` type and be converted.

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price is not defined yet
$filter = new Filter\Equals('price', new Value\NumericValue());

// the value will be converted to int and the price should be equal to 7  
$filter = $filter->withValue('7'); 

// this value is not applicable due to it is not numeric  
$filter = $filter->withValue([123]);
```

Next specifications are available for grids for now:

* [all](#filter-specifications-all)
* [any](#filter-specifications-any)
* [(not) equals](#filter-specifications-not-equals)
* [compare gt/gte lt/lte](#filter-specifications-compare)
* [(not) in array](#filter-specifications-not-in-array)
* [like](#filter-specifications-like)
* [map](#filter-specifications-map)
* [select](#filter-specifications-select)
* [between](#filter-specifications-between)

> There's much more interesting in the [filter values](#filter-values) and [value accessors](#value-accessors) sections below

### All
This is a union filter for logic `and` operation.<br/>
Examples with fixed values:
```php
use Spiral\DataGrid\Specification\Filter;

// the price should be equal to 2 and the quantity be greater than 5
$all = new Filter\All(
    new Filter\Equals('price', 2),
    new Filter\Gt('quantity', 5)
);
```

Passed value will be applied to all sub-filters:<br/>
Examples with `ValueInterface` usage:
```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

$all = new Filter\All(
    new Filter\Equals('price', new Value\NumericValue()),
    new Filter\Gt('quantity', new Value\IntValue()),
    new Filter\Lt('option_id', 4)
);

// the price should be equal to 5, the quantity be greater than 5 and the option_id less than 4
$all = $all->withValue(5);
```

### Any
This is a union filter for logic `or` operation.<br/>
Examples with fixed values:
```php
use Spiral\DataGrid\Specification\Filter;

// the price should be equal to 2 or the quantity be greater than 5
$any = new Filter\Any(
    new Filter\Equals('price', 2),
    new Filter\Gt('quantity', 5)
);
```

Passed value will be applied to all sub-filters.<br/>
Examples with `ValueInterface` usage:
```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

$any = new Filter\Any(
    new Filter\Equals('price', new Value\NumericValue()),
    new Filter\Gt('quantity', new Value\IntValue()),
    new Filter\Lt('option_id', 4)
);

// the price should be equal to 5 or the quantity be greater than 5 or the option_id less than 4
$any = $any->withValue(5);
```

### (Not) equals
These are simple expression filters for logic `=`, `!=` operations.<br/>
Examples with a fixed value:
```php
use Spiral\DataGrid\Specification\Filter;

$equals = new Filter\Equals('price', 2);       // the price should be equal to 2
$notEquals = new Filter\NotEquals('price', 2); // the price should not be equal to 2
```

Examples with `ValueInterface` usage:
```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// the price should be equal to 2
$equals = new Filter\Equals('price', new Value\NumericValue());
$equals = $equals->withValue('2');

// the price should not be equal to 2
$notEquals = new Filter\NotEquals('price', new Value\NumericValue());
$notEquals = $notEquals->withValue('2');
```

### Compare
These are simple expression filters for logic `>`, `>=`, `<`, `<=` operations.<br/>
Examples with a fixed value:
```php
use Spiral\DataGrid\Specification\Filter;

$gt = new Filter\Gt('price', 2);   // the price should be greater than 2
$gte = new Filter\Gte('price', 2); // the price should be greater than 2 or equal
$lt = new Filter\Lt('price', 2);   // the price should be less than 2
$lte = new Filter\Lte('price', 2); // the price should be less than 2 or equal
```

Examples with `ValueInterface` usage:
```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// the price should be greater than 2
$gt = new Filter\Gt('price', new Value\NumericValue());
$gt = $gt->withValue('2');

// the price should be greater than 2 or equal
$gte = new Filter\Gte('price', new Value\NumericValue());
$gte = $gte->withValue('2');

// the price should be less than 2
$lt = new Filter\Lt('price', new Value\NumericValue());
$lt = $lt->withValue('2');

// the price should be less than 2 or equal
$lte = new Filter\Lte('price', new Value\NumericValue());
$lte = $lte->withValue('2');
```

### (Not) in array
These are simple expression filters for logic `in`, `not in` operations.<br/>
Examples with a fixed value:
```php
use Spiral\DataGrid\Specification\Filter;

$inArray = new Filter\InArray('price', [2, 5]);       // the price should be in array of 2 and 5
$notInArray = new Filter\NotInArray('price', [2, 5]); // the price should not be in array of 2 and 5
```

Examples with `ValueInterface` usage:
```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// the price should be in array of 2 and 5
$inArray = new Filter\InArray('price', new Value\NumericValue());
$inArray = $inArray->withValue(['2', '5']);

// the price should not be in array of 2 and 5
$notInArray = new Filter\NotInArray('price', new Value\NumericValue());
$notInArray = $notInArray->withValue(['2', '5']);
```

### Like
This is a simple expression filter for `like` operation.<br/>
Examples with a fixed value:
```php
use Spiral\DataGrid\Specification\Filter;

$likeFull = new Filter\Like('name', 'Tony', '%%%s%%'); // the name should be like '%Tony%'
$likeEnding = new Filter\Like('name', 'Tony', '%s%%'); // the name should be like 'Tony%'
```

Examples with `ValueInterface` usage:
```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// the name should be like '%Tony%'
$like = new Filter\Like('name', new Value\StringValue());
$like = $like->withValue('Tony');
```

### Map
Map is a complex filter representing a map of filters with their own values.
```php
use Spiral\DataGrid\Specification\Filter;

// the price should be greater than 2 and the quantity be less than 5
$map = new Filter\Map([
    'from' => new Filter\Gt('price', 2),
    'to'   => new Filter\Lt('quantity', 5)
]);
```

Passed values will be applied to all sub-filters, all values are required:<br/>
Examples with `ValueInterface` usage:
```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

$map = new Filter\Map([
    'from' => new Filter\Gt('price', new Value\NumericValue()),
    'to'   => new Filter\Lt('quantity', new Value\NumericValue())
]);

// the price should be greater than 2 and the quantity be less than 5
$map = $map->withValue(['from' => 2, 'to' => 5]);

// invalid input, map will be set to null
$map = $map->withValue(['to' => 5]);
```

### Select
This specification represents a set of available expressions.
Passing a value from the input will pick a single or several specifications from this set.
> You just need to pass a key or an array of keys. Note that no `ValueInterface` should be declared.

Example with a single value:
```php
use Spiral\DataGrid\Specification\Filter;

// note, that we have integer keys here
$select = new Filter\Select([
    new Filter\Equals('name', 'value'),
    new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    new Filter\Equals('email', 'email@example.com'),
]);

// select the second filter, will be equal to 'any' specification.
$filter = $select->withValue(1);
```

Example with multiple values:
```php
use Spiral\DataGrid\Specification\Filter;

$select = new Filter\Select([
    'one'  => new Filter\Equals('name', 'value'),
    'two'  => new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    'three' => new Filter\Equals('email', 'email@example.com'),
]);

// the filter will contain both sub-filters wrapped in 'all' specification
$filter = $select->withValue(['one', 'two']);
```

Example with an unknown value:
```php
use Spiral\DataGrid\Specification\Filter;

$select = new Filter\Select([
    'one'  => new Filter\Equals('name', 'value'),
    'two'  => new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    'three' => new Filter\Equals('email', 'email@example.com'),
]);

// filter will be equal to null
$filter = $select->withValue('four');
```

### Between
This filter represents the SQL `between` operation, but can be presented as two `gt/gte` and `lt/lte` filters.
You have an ability to define whether the boundary values should be included or not.
If the boundary values aren't included, this filter will be converted into `gt`+`lt` filters, otherwise when getting
filters via `getFilters()` method you can specify either use the original `between` operator or `gte`+`lte` filters.
> Not all databases support `between` operation, that's why converion to `gt/gte`+`lt/lte` is by default.

Between filter has two modifications: field-based and value-based:
```php
use Spiral\DataGrid\Specification\Filter;

$fieldBetween  = new Filter\Between('field', [10, 20]);
$valueBetween  = new Filter\ValueBetween('2020 Apr, 10th', ['start_date', 'end_date']);
```
Examples above are similar to the next SQL queries:
```sql
# field-based
select * from table_name where field between 10 and 20;
# or using gte/lte conversion
select * from table_name where field >= 10 and field <= 20;

# value-based
select * from table_name where '2020 Apr, 10th' between start_date and end_date;
# or using gte/lte conversion
select * from table_name where start_date <= '2020 Apr, 10th' and end_date >= '2020 Apr, 10th';
```

Example using `ValueInterface`:
```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// the price should be between 10 and 20
$fieldBetween  = new Filter\Between('price', new Value\NumericValue());
$fieldBetween = $fieldBetween->withValue([10, '20']);

// the '2020 Apr, 10th' should be between start_date and end_date
$valueBetween  = new Filter\ValueBetween(new Value\DatetimeValue(), ['start_date', 'end_date']);
$valueBetween = $valueBetween->withValue('2020 Apr, 10th');
```

Select render type:
```php
use Spiral\DataGrid\Specification\Filter;

$between  = new Filter\Between('price', [10, 20]);

$between->getFilters();     // will be converted to gte+lte
$between->getFilters(true); // will be presented as is

$notIncludingBetween  = new Filter\Between('price', [10, 20], false, false);

// will be converted to gte+lte anyway
$notIncludingBetween->getFilters();
$notIncludingBetween->getFilters(true);
```
> The same is for `ValueBetween` filter

## Filter values
Filter values is the way of converting input type and its validation.
Please don't use `convert()` method without validating the input via `accepts()` method.
They can tell you is the input acceptable and converts it to a desired type if possible.
Next values are available for grids for now:

* [any](#filter-values-any)
* [array](#filter-values-array)
* [bool](#filter-values-bool)
* [zero compare](#filter-values-zero-compare)
* [numbers](#filter-values-numbers)
* [datetime](#filter-values-datetime)
* [enum](#filter-values-enum)
* [intersect](#filter-values-intersect)
* [subset](#filter-values-subset)
* [string](#filter-values-string)
* [scalar](#filter-values-scalar)
* [regex](#filter-values-regex)
* [uuid](#filter-values-uuid)
* [range](#filter-values-range)

### Any
This value accepts any input and doesn't convert them:
```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\AnyValue();
 
print_r($value->accepts('123')); // always true
print_r($value->convert('123')); // always equal to the input
```

### Array
This value expects an array and converts all of them according to the base value type. The input should not be empty:
```php
use Spiral\DataGrid\Specification\Value;

// expects an array of int values
$value = new Value\ArrayValue(new Value\IntValue());
 
print_r($value->accepts('123'));   // false
print_r($value->accepts([]));      // false
print_r($value->accepts(['123'])); // true
print_r($value->convert(['123'])); // [123]
```

### Bool
This value expects a bool input, 1/0 (as int or strings), and `true`/`false` strings:
```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\BoolValue();
 
print_r($value->accepts('123'));   // false
print_r($value->accepts('0'));     // true
print_r($value->accepts(['123'])); // false
print_r($value->convert('1'));     // true
print_r($value->convert('false')); // false
```

### Zero-compare
These values are supposed to check your input if it is positive/negative/non-positive/non-negative according to the base value type:
```php
use Spiral\DataGrid\Specification\Value;

$positive = new Value\PositiveValue(new Value\IntValue());       // as int should be > 0
$negative = new Value\NegativeValue(new Value\IntValue());       // as int should be < 0
$nonPositive = new Value\NonPositiveValue(new Value\IntValue()); // as int should be >= 0
$nonNegative = new Value\NonNegativeValue(new Value\IntValue()); // as int should be <= 0
```

### Numbers
Applies numeric values, also empty strings (zero is also a value):
```php
use Spiral\DataGrid\Specification\Value;

$int = new Value\IntValue();         // converts to int
$float = new Value\FloatValue();     // converts to float
$numeric = new Value\NumericValue(); // converts to int/float
```

>To be continued

### Datetime
This value expects a string representing a timestamp or a datetime and converts it into a `\DateTimeImmutable`:
```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\DatetimeValue();
 
print_r($value->accepts('abc'));     // false
print_r($value->accepts('123'));     // true
print_r($value->accepts('-1 year')); // true
print_r($value->convert('-1 year')); // DateTimeImmutable object
```

### Enum
This value expects an input to be a part of a given enum array and converts it according to the base value type.
All enum values are converted also:
```php
use Spiral\DataGrid\Specification\Value;

// expects an array of int values
$value = new Value\EnumValue(new Value\IntValue(), 1, '2', 3);
 
print_r($value->accepts('3')); // true
print_r($value->accepts(4));   // false
print_r($value->convert('3')); // 3
```

### Intersect
This value is based on an enum value, the difference is that at least one of the array input elements should match the given enum array:
```php
use Spiral\DataGrid\Specification\Value;

// expects an array of int values
$value = new Value\IntersectValue(new Value\IntValue(), 1, '2', 3);
 
print_r($value->accepts('3'));    // true
print_r($value->accepts(4));      // false
print_r($value->accepts([3, 4])); // true
print_r($value->convert('3'));    // [3]
```

### Subset
This value is based on an enum value, the difference is that all of the array input elements should match the given enum array:
```php
use Spiral\DataGrid\Specification\Value;

// expects an array of int values
$value = new Value\SubsetValue(new Value\IntValue(), 1, '2', 3);
 
print_r($value->accepts('3'));    // true
print_r($value->accepts(4));      // false
print_r($value->accepts([3, 4])); // false
print_r($value->accepts([2, 3])); // true
print_r($value->convert('3'));    // [3]
```

### String
Applies string-like input, also empty strings (if a corresponding constructor param passed):
```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\StringValue();
$allowEmpty = new Value\StringValue(true);

print_r($value->accepts(''));      // false
print_r($value->accepts(false));   // false
print_r($value->accepts('3'));     // true
print_r($value->accepts(4));       // true
print_r($value->convert(3));       // '3'
print_r($allowEmpty->accepts('')); // true
```

### Scalar
Applies scalar values, also empty strings (if a corresponding constructor param passed):
```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\ScalarValue();
$allowEmpty = new Value\ScalarValue(true);

print_r($value->accepts(''));       // false
print_r($value->accepts(false));    // true
print_r($value->accepts('3'));      // true
print_r($value->accepts(4));        // true
print_r($value->convert(3));        // '3'
print_r($allowEmpty->accepts('')); // true
```

### Regex
Applies string-like input and check if it matches the given regex pattern, converts to string:
```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\RegexValue('/\d+/');

print_r($value->accepts(''));  // false
print_r($value->accepts(3));   // true
print_r($value->accepts('4')); // true
print_r($value->convert(3));   // '3'
```

### Uuid
Applies UUID-formatted strings, a user can choose which validation pattern to use:
- any (just check the string format)
- nil (special uuid null value)
- one of [1-5] versions
The output is converted to string.
```php
use Spiral\DataGrid\Specification\Value;

$v4 = new Value\UuidValue('v4');
$valid = new Value\UuidValue();

print_r($v4->accepts(''));                                     // false
print_r($v4->accepts('00000000-0000-0000-0000-000000000000')); // false
print_r($valid->accepts(''));                                     // false
print_r($valid->accepts('00000000-0000-0000-0000-000000000000')); // true
```

### Range
This value expects an input to be a inside of a given range and converts it according to the base value type.
Range boundary values are converted also. You can specify either the input can be also equals to the boundary values or not:
```php
use Spiral\DataGrid\Specification\Value;

// as it, expects the value be >=1 and <3
$value = new Value\RangeValue(
    new Value\IntValue(),
    Value\RangeValue\Boundary::including(1),
    Value\RangeValue\Boundary::excluding(3)
);
 
print_r($value->accepts('3')); // false
print_r($value->accepts(1));   // false
```

## Value accessors
Accessors act like values from the section above but have another purpose - you can use them to perform not-type
transformations, for example using strings, you may want to trim the value or convert it to uppercase. 
They can be applied only if the value applicable by a given `ValueInterface`. Examples Below:
```php
use Spiral\DataGrid\Specification\Value;
use Spiral\DataGrid\Specification\Value\Accessor;

print_r((new Accessor\ToUpper(new Value\StringValue()))->convert('abc')); // 'ABC'
print_r((new Accessor\ToUpper(new Value\StringValue()))->convert('ABC')); // 'ABC'
print_r((new Accessor\ToUpper(new Value\StringValue()))->convert(123));   // '123'
print_r((new Accessor\ToUpper(new Value\ScalarValue()))->convert(123));   // 123
```

All supported accessors have the next handling order: perform own operations first, then pass them to a lower level.
For example, we have `add` and `multiply` accessors:
```php
use Spiral\DataGrid\Specification\Value;
use Spiral\DataGrid\Specification\Value\Accessor;

$multiply = new Accessor\Multiply(new Accessor\Add(new Value\IntValue(), 2), 2);
$add = new Accessor\Add(new Accessor\Multiply(new Value\IntValue(), 2), 2);

print_r($multiply->convert(2)); // 2*2+2=6
print_r($add->convert(2));      // (2+2)*2=8
```

Next accessors are available for grids for now:
- `trim`
- `toUpper`
- `toLower`
