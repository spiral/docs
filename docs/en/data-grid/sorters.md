# Sorters

Sorters are specifications that define how data can be ordered and sorted. They carry a sorting direction and allow
users to control the order in which results are returned.

## How Sorters Work

Sorters define the available sorting options for your data. For sorters that can apply direction, you can pass one of
these values:

- `1`, `'1'`, `'asc'`, `SORT_ASC` for ascending order
- `-1`, `'-1'`, `'desc'`, `SORT_DESC` for descending order

## Available Sorter Types

### Basic Sorter

The most common sorter that can sort by one or more fields in either direction:

```php
use Spiral\DataGrid\Specification\Sorter\Sorter;

// Single field sorter
$schema->addSorter('name', new Sorter('name'));

// Multiple field sorter - sorts by first_name, then last_name
$schema->addSorter('full_name', new Sorter('first_name', 'last_name'));

// Usage: ?sort[name]=asc or ?sort[name]=desc
// Usage: ?sort[full_name]=desc
```

### Ordered Sorters

`AscSorter` and `DescSorter` contain expressions that are applied with a fixed sorting order:

```php
use Spiral\DataGrid\Specification\Sorter\{AscSorter, DescSorter};

// Always sorts in ascending order
$ascSorter = new AscSorter('first_name', 'last_name');

// Always sorts in descending order  
$descSorter = new DescSorter('created_at', 'priority');
```

These are useful when you want to enforce a specific sort order regardless of user input.

### Directional Sorter

This sorter contains 2 independent sorters, each for ascending and descending order. This allows you to use different
fields or logic for each direction:

```php
use Spiral\DataGrid\Specification\Sorter\DirectionalSorter;

$sorter = new DirectionalSorter(
    new AscSorter('first_name'),      // Ascending: sort by first name
    new DescSorter('last_name')       // Descending: sort by last name
);

// User requests ascending sort
$ascSorter = $sorter->withDirection('asc');   // Sorts by first_name ASC

// User requests descending sort  
$descSorter = $sorter->withDirection('desc'); // Sorts by last_name DESC
```

> **Note:**
> You can sort using different fields in both sorters. If you have the same fields for both directions, use a
> regular [Sorter](#basic-sorter) instead.

### Sorter Set

This combines multiple sorters into one set. When a direction is applied, it affects the entire set:

```php
use Spiral\DataGrid\Specification\Sorter\{SorterSet, AscSorter, DescSorter, Sorter};

$sorter = new SorterSet(
    new AscSorter('priority'),           // Always ascending
    new DescSorter('created_at'),        // Always descending  
    new Sorter('name', 'email'),          // Follows set direction
);

// When user sorts ascending:
// ORDER BY priority ASC, created_at DESC, name ASC, email ASC
$ascSorter = $sorter->withDirection('asc');

// When user sorts descending:  
// ORDER BY priority ASC, created_at DESC, name DESC, email DESC
$descSorter = $sorter->withDirection('desc');
```

## Practical Sorter Examples

### E-commerce Product Sorting

```php
class ProductSchema extends GridSchema
{
    public function __construct()
    {
        // Simple field sorting
        $this->addSorter('name', new Sorter('name'));
        $this->addSorter('price', new Sorter('price'));
        $this->addSorter('created_at', new Sorter('created_at'));
        
        // Complex popularity sorting
        $this->addSorter('popularity', new SorterSet(
            new DescSorter('featured'),      // Featured products first
            new DescSorter('sales_count'),   // Then by sales
            new DescSorter('rating'),        // Then by rating
            new AscSorter('price')          // Finally by price
        ));
        
        // Different logic for best/worst sellers
        $this->addSorter('sales', new DirectionalSorter(
            new DescSorter('sales_count', 'rating'),  // Best sellers
            new AscSorter('sales_count', 'created_at') // Worst sellers
        ));
    }
}
```

### User Management Sorting

```php
class UserSchema extends GridSchema  
{
    public function __construct()
    {
        // Basic user sorting
        $this->addSorter('name', new Sorter('first_name', 'last_name'));
        $this->addSorter('email', new Sorter('email'));
        $this->addSorter('joined', new Sorter('created_at'));
        
        // Activity-based sorting
        $this->addSorter('activity', new DirectionalSorter(
            // Most active: recent login, high post count
            new DescSorter('last_login', 'post_count'),
            // Least active: old login, low post count  
            new AscSorter('last_login', 'post_count')
        ));
        
        // Status-based sorting with multiple criteria
        $this->addSorter('status', new SorterSet(
            new DescSorter('is_premium'),    // Premium users first
            new AscSorter('status'),         // Then by status
            new DescSorter('created_at')     // Then by registration date
        ));
    }
}
```

## Input Processing

### URL Query Parameters

Sorters accept input through URL query parameters:

```
GET /api/users?sort[name]=asc&sort[created_at]=desc
```

### Multiple Sorts

You can apply multiple sorts simultaneously:

```
GET /api/products?sort[category]=asc&sort[price]=desc&sort[name]=asc
```

This would produce SQL like:

```sql
ORDER BY category ASC, price DESC, name ASC
```

### Default Sorting

You can set default sorting in your controller:

```php
public function products(ProductSchema $schema, GridFactoryInterface $factory, ProductRepository $products)
{
    $grid = $factory->create($products->select(), $schema);
    
    // Apply default sorting if none provided
    if (!$this->request->has('sort')) {
        $grid = $grid->withSort('popularity', 'desc');
    }
    
    return ['products' => iterator_to_array($grid)];
}
```

## Advanced Sorting Patterns

### Contextual Sorting

Different sorting behavior based on context:

```php
class ProductSchema extends GridSchema
{
    public function __construct(User $user = null)
    {
        // Basic sorting available to everyone
        $this->addSorter('name', new Sorter('name'));
        $this->addSorter('price', new Sorter('price'));
        
        // Premium users get advanced sorting
        if ($user && $user->isPremium()) {
            $this->addSorter('profit_margin', new Sorter('profit_margin'));
            $this->addSorter('stock_level', new Sorter('stock_quantity'));
        }
        
        // Admin-only sorting
        if ($user && $user->isAdmin()) {
            $this->addSorter('internal_priority', new Sorter('admin_priority'));
        }
    }
}
```

### Smart Default Sorting

```php
class SearchResultSchema extends GridSchema
{
    public function __construct(string $searchQuery = '')
    {
        $this->addSorter('name', new Sorter('name'));
        $this->addSorter('created_at', new Sorter('created_at'));
        
        // If there's a search query, relevance sorting is available
        if (!empty($searchQuery)) {
            $this->addSorter('relevance', new DescSorter('search_rank'));
        }
        
        // Default to relevance when searching, created_at otherwise
        $defaultSort = !empty($searchQuery) ? 'relevance' : 'created_at';
        $this->addSorter('default', new Sorter($defaultSort));
    }
}
```

### Custom Sort Logic

For complex sorting requirements, you might need custom database functions:

```php
// In your repository or query builder
class ProductRepository
{
    public function select(): SelectQuery
    {
        return $this->database
            ->select()
            ->from('products')
            ->columns([
                '*',
                // Custom calculated fields for sorting
                '((featured * 10) + (rating * 2) + sales_count) as popularity_score',
                'CASE WHEN stock_quantity > 0 THEN 1 ELSE 0 END as in_stock'
            ]);
    }
}

// Then in your schema
$this->addSorter('popularity', new Sorter('popularity_score'));
$this->addSorter('availability', new DescSorter('in_stock', 'stock_quantity'));
```

## Performance Considerations

### Database Indexes

Ensure sorted fields have appropriate indexes:

```php
// These sorters should have corresponding database indexes
$this->addSorter('created_at', new Sorter('created_at'));     // INDEX(created_at)
$this->addSorter('status', new Sorter('status'));           // INDEX(status)  
$this->addSorter('user_id', new Sorter('user_id'));         // INDEX(user_id)

// Composite sorting needs composite indexes
$this->addSorter('user_date', new Sorter('user_id', 'created_at')); // INDEX(user_id, created_at)
```

### Expensive Sorts

Be careful with expensive sorting operations:

```php
// Efficient: Sorting by indexed columns
$this->addSorter('price', new Sorter('price'));

// Less efficient: Sorting by calculated values
$this->addSorter('total_value', new Sorter('price * quantity'));

// Better: Pre-calculate expensive values
$this->addSorter('total_value', new Sorter('calculated_total_value')); // Pre-computed column
```

## Error Handling

Invalid sort directions are automatically handled:

```php
// Schema defines valid sorter
$schema->addSorter('name', new Sorter('name'));

// Invalid input is ignored
// ?sort[name]=invalid_direction
// Result: No sorting applied, no error thrown

// Valid input is processed
// ?sort[name]=asc → ORDER BY name ASC
// ?sort[name]=desc → ORDER BY name DESC
```

## Best Practices

1. **Use meaningful sorter names** - Choose names that make sense to users
2. **Index sorted fields** - Ensure database performance
3. **Limit complex sorting** - Pre-calculate expensive sort values
4. **Provide sensible defaults** - Most important/relevant sort first
5. **Consider user experience** - Popular sorting options should be easy to access

```php
class BlogPostSchema extends GridSchema
{
    public function __construct()
    {
        // User-friendly sorter names
        $this->addSorter('newest', new DescSorter('published_at'));
        $this->addSorter('oldest', new AscSorter('published_at'));  
        $this->addSorter('popular', new DescSorter('view_count', 'like_count'));
        $this->addSorter('title', new Sorter('title'));
        $this->addSorter('author', new Sorter('author_name'));
        
        // Performance-optimized sorting
        $this->addSorter('trending', new DescSorter('trending_score')); // Pre-calculated
    }
}
```
