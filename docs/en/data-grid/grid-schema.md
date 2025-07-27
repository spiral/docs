# Grid Schema

The Grid Schema is the heart of the Data Grid component. Think of the Schema as a set of rules for how to get data based
on what the user asks for. It defines what filters, sorters, and pagination options are available to users.

## Basic Schema Structure

The component provides two base abstractions - **Grid Factory** and **Grid Schema**.

```php
use Spiral\DataGrid\GridSchema; 
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\StringValue;

$schema = new GridSchema();

// User pagination: limit results to 10 per page
$schema->setPaginator(new PagePaginator(10));

// Sorting option: by id
$schema->addSorter('id', new Sorter('id'));

// Filter option: find by name matching user input
$schema->addFilter('name', new Like('name', new StringValue()));
```

In short, with Schema, developers can customize filters and sorting options when fetching data.

## Creating Custom Schemas

You can extend the base `GridSchema` class to create reusable schemas for your entities:

```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Sorter;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Value;

class UserSchema extends GridSchema
{
    public function __construct()
    {
        // Define available filters
        $this->addFilter('name', new Filter\Like('name', new Value\StringValue()));
        $this->addFilter('email', new Filter\Like('email', new Value\StringValue()));
        $this->addFilter('status', new Filter\Equals('status', new Value\EnumValue(['active', 'inactive', 'banned'])));
        $this->addFilter('created_at', new Filter\Gte('created_at', new Value\DatetimeValue()));
        
        // Define available sorters
        $this->addSorter('name', new Sorter\Sorter('name'));
        $this->addSorter('email', new Sorter\Sorter('email'));
        $this->addSorter('created_at', new Sorter\Sorter('created_at'));
        
        // Set pagination
        $this->setPaginator(new PagePaginator(25, [10, 25, 50, 100]));
    }
}
```

## Grid Factory

The Grid Factory is responsible for creating grid instances from your data source and schema:

```php
use Spiral\DataGrid\GridFactoryInterface;

class UserController 
{
    public function index(
        UserSchema $schema, 
        GridFactoryInterface $factory, 
        UserRepository $users
    ): array {
        $grid = $factory->create($users->select(), $schema);
        
        return [
            'users' => iterator_to_array($grid),
            'pagination' => $grid->getOption(GridInterface::PAGINATOR),
            'filters' => $grid->getOption(GridInterface::FILTERS),
        ];
    }
}
```

## Schema Components

### Adding Filters

Filters allow users to narrow down the data based on specific criteria. The Data Grid component provides various filter
types for different use cases.

```php
// Simple equality filter
$schema->addFilter('status', new Filter\Equals('status', new Value\StringValue()));

// Like filter for text search
$schema->addFilter('search', new Filter\Like('name', new Value\StringValue()));

// Range filter for numeric values
$schema->addFilter('price', new Filter\Between('price', new Value\NumericValue()));

// Multiple choice filter
$schema->addFilter('category', new Filter\Equals('category_id', new Value\EnumValue([1, 2, 3, 4])));
```

> **See Also:** [Filters Documentation](filters.md) - Complete guide to all available filter types including Like,
> Equals, Between, Greater Than/Less Than, Any/All combinations, and custom filter implementations.

### Adding Sorters

Sorters define how the data can be ordered. You can create simple sorters, complex multi-field sorters, or directional
sorters with different behavior for ascending and descending order.

```php
// Simple sorter
$schema->addSorter('name', new Sorter\Sorter('name'));

// Multiple field sorter
$schema->addSorter('full_name', new Sorter\Sorter('first_name', 'last_name'));

// Directional sorter with different fields for asc/desc
$schema->addSorter('popularity', new Sorter\DirectionalSorter(
    new Sorter\AscSorter('views_count'),
    new Sorter\DescSorter('created_at', 'views_count')
));
```

> **See Also:** [Sorters Documentation](sorters.md) - Detailed guide to all sorter types including basic sorters,
> directional sorters, sorter sets, and advanced sorting patterns for complex use cases.

### Setting Pagination

Pagination controls how many records are returned and how they're split into pages. The Data Grid provides flexible
pagination options including page-based and custom pagination strategies.

```php
// Basic pagination with default page size
$schema->setPaginator(new PagePaginator(25));

// Pagination with allowed page sizes
$schema->setPaginator(new PagePaginator(25, [10, 25, 50, 100]));
```

> **See Also:** [Pagination Documentation](pagination.md) - Comprehensive guide to pagination including page-based
> pagination, cursor-based pagination, infinite scroll, and performance considerations for large datasets.

### Value Types and Validation

All filters use value types to validate and convert user input. Value types ensure data integrity and security by
preventing invalid input from reaching your data layer.

```php
// String validation and conversion
$schema->addFilter('name', new Filter\Like('name', new Value\StringValue()));

// Numeric validation with automatic type conversion
$schema->addFilter('age', new Filter\Gte('age', new Value\IntegerValue()));

// Date validation with flexible input formats
$schema->addFilter('created_after', new Filter\Gte('created_at', new Value\DatetimeValue()));

// Enum validation for controlled choices
$schema->addFilter('status', new Filter\Equals('status', new Value\EnumValue(['active', 'inactive'])));

// Array validation for multiple selections
$schema->addFilter('categories', new Filter\In('category_id', new Value\ArrayValue(new Value\IntegerValue())));
```

> **See Also:** [Value Types Documentation](values.md) - Complete reference for all value types including StringValue,
> IntegerValue, DatetimeValue, EnumValue, ArrayValue, and custom value type creation with validation and security
> features.

## Input Key Mapping

One of the powerful features of Grid Schema is the ability to map user-friendly input keys to actual database fields:

```php
// User searches by 'search' but it queries multiple fields
$schema->addFilter('search', new Filter\Any([
    new Filter\Like('name', new Value\StringValue()),
    new Filter\Like('email', new Value\StringValue()),
    new Filter\Like('description', new Value\StringValue())
]));

// User sorts by 'popularity' but it uses complex sorting logic
$schema->addSorter('popularity', new Sorter\SorterSet([
    new Sorter\DescSorter('featured'),
    new Sorter\DescSorter('views_count'),
    new Sorter\AscSorter('created_at')
]));
```

This approach provides several benefits:

1. **Stable API** - Input keys remain consistent even if database schema changes
2. **User-friendly** - Intuitive field names for frontend developers
3. **Flexible** - One input key can map to complex database operations
4. **Maintainable** - Changes are centralized in the schema

## Schema Validation

The schema automatically validates user input against the defined specifications:

```php
// This schema only allows specific status values
$schema->addFilter('status', new Filter\Equals('status', new Value\EnumValue(['draft', 'published', 'archived'])));

// User input: ?filter[status]=invalid_status
// Result: Filter is ignored, no error thrown, invalid input is safely discarded
```

## Advanced Schema Patterns

### Multi-Source Schema

Create schemas that work with different data sources:

```php
class UniversalProductSchema extends GridSchema
{
    public function __construct()
    {
        // These filters work with SQL, APIs, and collections
        $this->addFilter('name', new Filter\Like('name', new Value\StringValue()));
        $this->addFilter('price_range', new Filter\Between('price', new Value\NumericValue()));
        $this->addSorter('popularity', new Sorter\DescSorter('view_count'));
        $this->setPaginator(new PagePaginator(20));
    }
}

// Same schema, different data sources
$sqlGrid = $factory->create($sqlRepository->select(), $schema);
$apiGrid = $factory->create($apiClient, $schema);
$arrayGrid = $factory->create($productArray, $schema);
```

> **See Also:** [Custom Writers Documentation](writers.md) - Learn how to create custom writers for different data
> sources including APIs, Elasticsearch, and custom data structures.

Grid Schemas provide the foundation for building powerful, flexible, and secure data filtering and sorting systems. By
defining your schemas thoughtfully and following best practices, you can create maintainable and performant data access
layers that work across multiple interfaces and data sources.
