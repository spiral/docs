# Pagination

Pagination controls how many records are returned and how they're split into pages.

## Page Paginator

The `PagePaginator` is the most common pagination specification that provides simple page-based navigation:

### Basic Usage

```php
use Spiral\DataGrid\Specification\Pagination\PagePaginator;

// Basic pagination with 10 items per page
$schema->setPaginator(new PagePaginator(10));

// Pagination with allowed page sizes
$schema->setPaginator(new PagePaginator(25, [10, 25, 50, 100]));
```

### User Input

The `PagePaginator` accepts user input with two keys:

- `limit` - Number of items per page (must be in allowedLimits array)
- `page` - Current page number (1-based)

```php
$paginator = new PagePaginator(10, [25, 50, 100, 500]);

$paginator->withValue(['limit' => 123]); // Won't apply (not in allowed limits)
$paginator->withValue(['limit' => 50]);  // Will apply
$paginator->withValue(['limit' => 100]); // Will apply

$paginator->withValue(['limit' => 100, 'page' => 2]); // Page 2 with 100 items
```

### URL Query Parameters

```
GET /api/products?paginate[page]=2&paginate[limit]=50
```

## How Pagination Works Internally

Under the hood, the `PagePaginator` converts `limit` and `page` into `Limit` and `Offset` specifications:

```php
// User input: page=3, limit=25
// Converts to:
// - Limit: 25
// - Offset: 50 (calculated as (page - 1) * limit)
```

## Pagination Response Data

When you use pagination, the grid provides metadata about the current page state:

```php
public function products(ProductSchema $schema, GridFactoryInterface $factory, ProductRepository $products)
{
    $grid = $factory->create($products->select(), $schema);
    
    return [
        'products' => iterator_to_array($grid),
        'pagination' => $grid->getOption(GridInterface::PAGINATOR),
        'total' => $grid->getOption(GridInterface::COUNT),
    ];
}
```

### Pagination Metadata

The pagination metadata includes:

```php
[
    'page' => 2,           // Current page number
    'limit' => 25,         // Items per page
    'count' => 25,         // Items on current page
    'countPages' => 8,     // Total number of pages
    'countTotal' => 200    // Total number of items
]
```

## Custom Pagination Examples

### E-commerce Product Listing

```php
class ProductSchema extends GridSchema
{
    public function __construct()
    {
        // Different page sizes for different use cases
        $this->setPaginator(new PagePaginator(
            24,                          // Default: 24 products (4x6 grid)
            [12, 24, 48, 96]            // Options: Small, medium, large, extra large
        ));
    }
}
```

### Admin Data Tables

```php
class UserManagementSchema extends GridSchema
{
    public function __construct()
    {
        // Higher default for admin interfaces
        $this->setPaginator(new PagePaginator(
            50,                          // Default: 50 users
            [25, 50, 100, 200, 500]     // Wide range for bulk operations
        ));
    }
}
```

### Mobile-Optimized Pagination

```php
class MobileProductSchema extends GridSchema
{
    public function __construct()
    {
        // Smaller pages for mobile to improve loading times
        $this->setPaginator(new PagePaginator(
            10,                          // Default: 10 items
            [5, 10, 20]                 // Limited options for mobile
        ));
    }
}
```

## Custom Pagination Implementation

You can create custom pagination logic by implementing your own paginator:

### Cursor-Based Pagination

```php
use Spiral\DataGrid\SpecificationInterface;

class CursorPaginator implements SpecificationInterface
{
    public function __construct(
        private readonly int $limit = 25,
        private readonly ?string $cursor = null
    ) {}
    
    public function withValue(mixed $value): ?SpecificationInterface
    {
        if (!is_array($value)) {
            return null;
        }
        
        return new self(
            $value['limit'] ?? $this->limit,
            $value['cursor'] ?? $this->cursor
        );
    }
    
    // Implementation methods...
}

// Usage
$schema->setPaginator(new CursorPaginator(20));
```

### Infinite Scroll Pagination

```php
class InfiniteScrollPaginator implements SpecificationInterface
{
    public function __construct(
        private readonly int $limit = 20,
        private readonly ?int $lastId = null
    ) {}
    
    public function withValue(mixed $value): ?SpecificationInterface
    {
        if (!is_array($value)) {
            return null;
        }
        
        return new self(
            $value['limit'] ?? $this->limit,
            $value['last_id'] ?? $this->lastId
        );
    }
    
    // Implementation methods...
}

// Usage  
$schema->setPaginator(new InfiniteScrollPaginator(15));
```

## Pagination with Counting

For performance reasons, you might want to control how total counts are calculated:

### Custom Counter Function

```php
/** @var Spiral\DataGrid\GridFactory $factory */
$factory = $factory->withCounter(static function ($select): int {
    // Custom counting logic
    return count($select) * 2;
});
```

### Disable Counting for Performance

```php
// For very large datasets, you might want to disable total counting
$factory = $factory->withCounter(static function ($select): int {
    return -1; // Indicates unknown total
});
```