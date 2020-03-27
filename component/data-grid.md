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

## Available pagination specifications
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

## Available sorter specifications
Next specifications are available for grids for now:

* [ordered sorters](#available-sorter-specifications-ordered-sorters-specification)
* [directional sorter](#available-sorter-specifications-directional-sorter-specification)
* [sorter](#available-sorter-specifications-sorter-specification)
* [sorter set](#available-sorter-specifications-sorter-set-specification)

### Ordered sorters specification
`AscSorter` and `DescSorter` contain the expressions that should be applied with ascending (or descending) sorting order:
```php
use Spiral\DataGrid\Specification\Sorter\AscSorter;
use Spiral\DataGrid\Specification\Sorter\DescSorter;

$ascSorter = new AscSorter('first_name', 'last_name'); // variadic param inside
$descSorter = new DescSorter('first_name', 'last_name'); // variadic param inside
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
$ascSorter = $sorter->withDirection('asc'); //also 1, '1' and SORT_ASC values allowed
//will sort by last_name desc
$descSorter = $sorter->withDirection('desc'); //also -1, '-1' and SORT_DESC values allowed
```
> Note that you can sort using different set of fields in both sorters.
> If you have the same set of fields, use [sorter](#sorter-specification) instead.

### Sorter specification
This is a sorter wrapper for a directional sorter in case you have the same fields for sorting in both directions:
```php
use Spiral\DataGrid\Specification\Sorter\Sorter;

$sorter = new Sorter('first_name', 'last_name');
//will sort by first_name and last_name asc
$ascSorter = $sorter->withDirection('asc'); //also 1, '1' and SORT_ASC values allowed
//will sort by first_name and last_name desc
$descSorter = $sorter->withDirection('desc'); //also -1, '-1' and SORT_DESC values allowed
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
//will sort by first_name, email and username asc
$ascSorter = $sorter->withDirection('asc'); //also 1, '1' and SORT_ASC values allowed
//will sort by last_name, email and username desc
$descSorter = $sorter->withDirection('desc'); //also -1, '-1' and SORT_DESC values allowed
```

## Available filter specifications
Next specifications are available for grids for now:

* [select](#available-filter-specifications-select-specification)

> There's much more interesting in the [filter values](#filter-values) and [value accessors](#value-accessors) sections below


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

$filter = $select->withValue(1); //the second filter
```
> Filter will be equal to `Filter\Any` specification.

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

$filter = $select->withValue(['one', 'two']);
```
> Filter will contain both embedded filters as `Filter\All` specification.

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

$filter = $select->withValue('four');
```
> Filter will be equal to null


## Filter values
Filter values is the way of converting data types and validation.
> TBD

## Value accessors
> TBD
