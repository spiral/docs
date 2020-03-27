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
namespace App\Controller;

use App\Database\UserRepository;
use Spiral\DataGrid\GridFactory;
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\StringValue;

class HomeController
{
    public function index(UserRepository $users, GridFactory $factory)
    {
        $schema = new GridSchema();
        $schema->setPaginator(new PagePaginator(10));
        $schema->addSorter('id', new Sorter('id'));
        $schema->addFilter('name', new Like('name', new StringValue()));

        $result = $factory->create($users->select(), $schema);

        dump(iterator_to_array($result));
    }
}
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
To select users from the second page open page with POST or QUERY data like: `?paginate[page]=2`.<br/>
To activate the Like filter: `?filter[name]=antony`.<br/>
To sort by id in ASC or DESC: `?sort[id]=desc`.<br/>
To get count of total values: `?fetchCount=1`.
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
namespace App\Controller;

use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;

class HomeController
{
    public function index()
    {
        $schema = new GridSchema();
        $schema->setPaginator(new PagePaginator(10, [25, 50, 100, 500]));
        //...
    }
}
```
From the user input, such paginator accepts an array with 2 keys, `limit` and `page`.
If limit is set it should be presented in the `allowedLimits` constructor param. 
```php
use Spiral\DataGrid\Specification\Pagination\PagePagination;

$paginator = new PagePaginator(10, [25, 50, 100, 500]);

$paginator->withValue(['limit' => 123]); //won't apply
$paginator->withValue(['limit' => 10]); //will apply
$paginator->withValue(['limit' => 100]); //will apply

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

* [ordered sorters](#sorter-specifications-ordered-sorters-specification)
* [directional sorter](#sorter-specifications-directional-sorter-specification)
* [sorter](#sorter-specifications-sorter-specification)
* [sorter set](#sorter-specifications-sorter-set-specification)

### Ordered sorters specification
`AscSorter` and `DescSorter` contain the expressions that should be applied with ascending (or descending) sorting order:
```php
use Spiral\DataGrid\Specification\Sorter\AscSorter;
use Spiral\DataGrid\Specification\Sorter\DescSorter;

$ascSorter = new AscSorter('first_name', 'last_name');
$descSorter = new DescSorter('first_name', 'last_name');
```

### Directional sorter specification
This sorter contains 2 independent sorters each for ascending and descending order.
By receiving the order via `withValue` we will get one of the sorters:
```php
use Spiral\DataGrid\Specification\Sorter\AscSorter;
use Spiral\DataGrid\Specification\Sorter\DescSorter;
use Spiral\DataGrid\Specification\Sorter\DirectionalSorter;

$sorter = new DirectionalSorter(new AscSorter('first_name'), new DescSorter('last_name'));

//will sort by first_name asc
$ascSorter = $sorter->withDirection('asc');

//will sort by last_name desc
$descSorter = $sorter->withDirection('desc');
```
> Note that you can sort using different set of fields in both sorters.
> If you have the same set of fields, use [sorter](#sorter-specifications-sorter-specification) instead.

### Sorter specification
This is a sorter wrapper for a directional sorter in case you have the same fields for sorting in both directions:
```php
use Spiral\DataGrid\Specification\Sorter\Sorter;

$sorter = new Sorter('first_name', 'last_name');

//will sort by first_name and last_name asc
$ascSorter = $sorter->withDirection('asc');

//will sort by first_name and last_name desc
$descSorter = $sorter->withDirection('desc');
```

### Sorter set specification
This is just a way of combining sorters into one set, passing direction will apply it to the whole set:
```php
use Spiral\DataGrid\Specification\Sorter\AscSorter;
use Spiral\DataGrid\Specification\Sorter\DescSorter;
use Spiral\DataGrid\Specification\Sorter\SorterSet;

$sorter = new SorterSet(
    new AscSorter('first_name'),
    new DescSorter( 'last_name'),
    new Sorter('email', 'username')
    //...
);

//will sort by first_name, email and username asc, also last_name desc
$ascSorter = $sorter->withDirection('asc');

//will sort by last_name, email and username desc, also first_name asc
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

* [all](#filter-specifications-all-specification)
* [any](#filter-specifications-any-specification)
* [(not) equals](#filter-specifications-not-equals-specification)
* [compare gt/gte lt/lte](#filter-specifications-compare-specification)
* [(not) in array](#filter-specifications-not-in-array-specification)
* [like](#filter-specifications-like-specification)
* [map](#filter-specifications-map-specification)
* [select](#filter-specifications-select-specification)

> There's much more interesting in the [filter values](#filter-values) and [value accessors](#value-accessors) sections below

### All specification
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

### Any specification
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

### (Not) equals specification
These are simple expression filters for logic `=`, `!=` operations.<br/>
Examples with a fixed value:
```php
use Spiral\DataGrid\Specification\Filter;

// the price should be equal to 2
$equals = new Filter\Equals('price', 2);

// the price should not be equal to 2
$notEquals = new Filter\NotEquals('price', 2);
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

### Compare specification
These are simple expression filters for logic `>`, `>=`, `<`, `<=` operations.<br/>
Examples with a fixed value:
```php
use Spiral\DataGrid\Specification\Filter;

// the price should be greater than 2
$gt = new Filter\Gt('price', 2);

// the price should be greater than 2 or equal
$gte = new Filter\Gte('price', 2);

// the price should be less than 2
$lt = new Filter\Lt('price', 2);

// the price should be less than 2 or equal
$lte = new Filter\Lte('price', 2);
```

Examples with `ValueInterface` usage:
```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

//the price should be greater than 2
$gt = new Filter\Gt('price', new Value\NumericValue());
$gt = $gt->withValue('2');

//the price should be greater than 2 or equal
$gte = new Filter\Gte('price', new Value\NumericValue());
$gte = $gte->withValue('2');

//the price should be less than 2
$lt = new Filter\Lt('price', new Value\NumericValue());
$lt = $lt->withValue('2');

//the price should be less than 2 or equal
$lte = new Filter\Lte('price', new Value\NumericValue());
$lte = $lte->withValue('2');
```

### (Not) in array specification
These are simple expression filters for logic `in`, `not in` operations.<br/>
Examples with a fixed value:
```php
use Spiral\DataGrid\Specification\Filter;

// the price should be in array of 2 and 5
$inArray = new Filter\InArray('price', [2, 5]);

// the price should not be in array of 2 and 5
$notInArray = new Filter\NotInArray('price', [2, 5]);
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

### Like specification
This is a simple expression filter for `like` operation.<br/>
Examples with a fixed value:
```php
use Spiral\DataGrid\Specification\Filter;

// the name should be like '%Tony%'
$likeFull = new Filter\Like('name', 'Tony', '%%%s%%');

// the name should be like 'Tony%'
$likeEnding = new Filter\Like('name', 'Tony', '%s%%');
```

Examples with `ValueInterface` usage:
```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// the name should be like '%Tony%'
$like = new Filter\Like('name', new Value\StringValue());
$like = $like->withValue('Tony');
```

### Map specification
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

### Select specification
This specification represents a set of available expressions.
Passing a value from the input will pick a single or several specifications from this set.
> You just need to pass a key or an array of keys. Note that no `ValueInterface` should be declared.

Example with a single value:
```php

use Spiral\DataGrid\Specification\Filter;

//Note, that we have integer keys here
$select = new Filter\Select([
    new Filter\Equals('name', 'value'),
    new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    new Filter\Equals('email', 'email@example.com'),
]);

// the second filter, will be equal to 'any' specification.
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

// the filter will contain both sub-filters as 'all' specification
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

## Filter values
Filter values is the way of converting data types and validation.
> TBD

## Value accessors
> TBD
