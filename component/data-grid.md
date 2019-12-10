# Data Grids
Use component `spiral/data-grid` and `spiral/data-grid-bridge` to generate Cycle and DBAL select queries automatically,
based on specifications provided by the end user.

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
To use the data grid you will need 2 base abstractions - grid factory and grid schema.

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

> You can extend the GridSchema and initiate all the specifications in constructor.

### Grid Factory
To use the defined grid schema you will have to obtain an instance of support data source. By default, the
Cycle Select and Database Select Query are supported.

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

You can now invoke this controller, by default first users will be selected from the database. 

To select users from second page open page with POST or QUERY data like: `?paginate[page]=2`.

To activate the Like filter: `?filter[name]=antony`.

To sort by id in ASC or DESC: `?sort[id]=desc`.

## Available Specifications
There are number of specifications available for grids.

> TBD.
