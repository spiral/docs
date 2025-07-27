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

## Frontend Integration Examples

### React Pagination Component

```javascript
function ProductList() {
    const [page, setPage] = useState(1);
    const [limit, setLimit] = useState(24);

    const {data, loading} = useQuery(`
        /api/products?paginate[page]=${page}&paginate[limit]=${limit}
    `);

    return (
        <div>
            <ProductGrid products={data.products}/>
            <Pagination
                current={data.pagination.page}
                total={data.pagination.countTotal}
                pageSize={data.pagination.limit}
                onChange={setPage}
            />
            <PageSizeSelector
                value={limit}
                options={[12, 24, 48, 96]}
                onChange={setLimit}
            />
        </div>
    );
}
```

### Vue.js Pagination

```vue

<template>
  <div>
    <product-grid :products="products"/>
    <pagination
        :current="currentPage"
        :total="totalItems"
        :page-size="pageSize"
        @change="handlePageChange"
    />
  </div>
</template>

<script>
  export default {
    data() {
      return {
        currentPage: 1,
        pageSize: 24,
        products: [],
        totalItems: 0
      }
    },

    methods: {
      async loadProducts() {
        const response = await fetch(
            `/api/products?paginate[page]=${this.currentPage}&paginate[limit]=${this.pageSize}`
        );
        const data = await response.json();

        this.products = data.products;
        this.totalItems = data.pagination.countTotal;
      },

      handlePageChange(page) {
        this.currentPage = page;
        this.loadProducts();
      }
    }
  }
</script>
```

## Performance Considerations

### Database Optimization

```php
// Ensure proper indexing for pagination
// For OFFSET/LIMIT queries, you need indexes on sort columns

class ProductRepository
{
    public function select(): SelectQuery
    {
        return $this->database
            ->select()
            ->from('products')
            ->orderBy('created_at', 'DESC'); // INDEX(created_at) needed for efficient pagination
    }
}
```

### Large Dataset Strategies

```php
// For very large datasets, consider:

// 1. Limit maximum page size
$schema->setPaginator(new PagePaginator(25, [10, 25, 50])); // No 1000+ options

// 2. Use cursor-based pagination for infinite scroll
$schema->setPaginator(new CursorPaginator(20));

// 3. Implement search-first approach
if (empty($searchQuery)) {
    // Force users to search/filter before showing results
    throw new ValidationException('Please provide search criteria');
}
```

### Caching Pagination Results

```php
class ProductController
{
    public function index(ProductSchema $schema, GridFactoryInterface $factory, ProductRepository $products)
    {
        $cacheKey = sprintf(
            'products_page_%d_limit_%d_%s',
            $this->request->get('paginate.page', 1),
            $this->request->get('paginate.limit', 25),
            md5(serialize($this->request->get('filter', [])))
        );
        
        return $this->cache->remember($cacheKey, 300, function () use ($schema, $factory, $products) {
            $grid = $factory->create($products->select(), $schema);
            return [
                'products' => iterator_to_array($grid),
                'pagination' => $grid->getOption(GridInterface::PAGINATOR),
            ];
        });
    }
}
```

## Best Practices

1. **Choose appropriate page sizes** - Balance between performance and user experience
2. **Provide size options** - Let users control how much data they see
3. **Index sorted columns** - Essential for efficient OFFSET/LIMIT queries
4. **Consider cursor pagination** - For real-time data or infinite scroll
5. **Cache when possible** - Pagination results are often good candidates for caching
6. **Limit maximum page sizes** - Prevent abuse and performance issues

```php
class ArticleSchema extends GridSchema
{
    public function __construct()
    {
        // Reasonable defaults and limits
        $this->setPaginator(new PagePaginator(
            20,                          // Good default for articles  
            [10, 20, 50, 100]          // Reasonable range, max 100
        ));
        
        // Always provide sorting for consistent pagination
        $this->addSorter('published', new DescSorter('published_at'));
        $this->addSorter('popular', new DescSorter('view_count'));
    }
}
```

## Common Pagination Patterns

### Search Results Pagination

```php
// User searches, then paginates through results
GET /api/articles?filter[search]=technology&paginate[page]=1&paginate[limit]=20
```

### Category Browsing

```php
// User browses category, sorts and paginates
GET /api/products?filter[category]=electronics&sort[price]=asc&paginate[page]=3&paginate[limit]=24
```

### Admin Data Management

```php
// Admin views all users with bulk operations in mind
GET /api/admin/users?sort[created_at]=desc&paginate[page]=1&paginate[limit]=100
```

The Data Grid pagination system provides flexible, efficient ways to handle large datasets while maintaining good
performance and user experience.
