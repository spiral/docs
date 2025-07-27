# Filters

Filters are specifications that carry values and allow users to narrow down data based on specific criteria. They are
the core mechanism for implementing search and filtering functionality in your application.

## How Filters Work

Filters are specifications that carry values. Values can be passed directly via the constructor for fixed filters, or
through `ValueInterface` implementations for dynamic user input.

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value\StringValue;

// Fixed filter - name should always be 'Antony'
$filter = new Filter\Equals('name', 'Antony');

// Dynamic filter - users can specify the value
$filter = new Filter\Equals('name', new StringValue());
$filter = $filter->withValue('John'); // User provides 'John'
```

If you pass the `ValueInterface` to the constructor, you can use the `withValue()` method. The incoming value will be
validated against the `ValueInterface` type and converted appropriately.

## Available Filter Types

### Equals Filter

Filters records where a field exactly matches the given value.

```php
use Spiral\DataGrid\Specification\Filter\Equals;
use Spiral\DataGrid\Specification\Value\StringValue;

// Find users with specific role
$roleFilter = new Equals('role', 'admin');

// Dynamic category filtering
$categoryFilter = new Equals('category', new StringValue());
$result = $categoryFilter->withValue('electronics');

// Order status filtering
$statusFilter = new Equals('status', new EnumValue(
    new StringValue(), 'pending', 'processing', 'shipped', 'delivered'
));
$result = $statusFilter->withValue('shipped');

// Boolean flag filtering
$activeFilter = new Equals('is_active', new BoolValue());
$result = $activeFilter->withValue(true);

// Usage in schema
$schema->addFilter('status', new Equals('status', new StringValue()));
// Usage: ?filter[status]=active
```

### Not Equals Filter

Filters records where a field value does NOT exactly match the given value.

```php
use Spiral\DataGrid\Specification\Filter\NotEquals;

// Exclude cancelled orders
$statusFilter = new NotEquals('status', 'cancelled');

// Find active users (not deleted)
$userFilter = new NotEquals('status', new EnumValue(
    new StringValue(), 'deleted', 'banned', 'suspended'
));
$result = $userFilter->withValue('deleted'); // Users who are NOT deleted

// Exclude draft content
$contentFilter = new NotEquals('publish_status', new StringValue());
$result = $contentFilter->withValue('draft'); // Published content (not draft)

// Usage in schema
$schema->addFilter('exclude_status', new NotEquals('status', new StringValue()));
// Usage: ?filter[exclude_status]=draft
```

### Greater Than Filters

#### Gt (Greater Than) Filter

Filters records where a field value is greater than the specified value.

```php
use Spiral\DataGrid\Specification\Filter\Gt;
use Spiral\DataGrid\Specification\Value\NumericValue;

// Find products more expensive than $50
$priceFilter = new Gt('price', new NumericValue());
$result = $priceFilter->withValue(50);

// Find adult users (age > 18)
$ageFilter = new Gt('age', new IntValue());
$result = $ageFilter->withValue(18);

// Find recent posts (created after specific date)
$dateFilter = new Gt('created_at', new DatetimeValue());
$result = $dateFilter->withValue('2024-01-01');

// Fixed threshold filtering
$performanceFilter = new Gt('cpu_usage', 80); // CPU usage > 80%

// Usage in schema
$schema->addFilter('min_price', new Gt('price', new NumericValue()));
// Usage: ?filter[min_price]=50
```

#### Gte (Greater Than or Equal) Filter

Filters records where a field value is greater than or equal to the specified value.

```php
use Spiral\DataGrid\Specification\Filter\Gte;

// Find products at or above minimum price
$priceFilter = new Gte('price', new NumericValue());
$result = $priceFilter->withValue(50); // Price >= $50

// Age verification (18 or older)
$ageFilter = new Gte('age', new IntValue());
$result = $ageFilter->withValue(18); // Age >= 18

// Minimum rating filter
$ratingFilter = new Gte('rating', new FloatValue());
$result = $ratingFilter->withValue(4.0); // Rating >= 4.0 stars

// Stock availability check
$stockFilter = new Gte('quantity', 1); // In stock (quantity >= 1)

// Usage in schema
$schema->addFilter('min_age', new Gte('age', new IntValue()));
// Usage: ?filter[min_age]=18
```

### Less Than Filters

#### Lt (Less Than) Filter

Filters records where a field value is less than the specified value.

```php
use Spiral\DataGrid\Specification\Filter\Lt;

// Find budget-friendly products
$priceFilter = new Lt('price', new NumericValue());
$result = $priceFilter->withValue(50); // Price < $50

// Find younger users
$ageFilter = new Lt('age', new IntValue());
$result = $ageFilter->withValue(30); // Age < 30

// Low-stock alert
$stockFilter = new Lt('quantity', new IntValue());
$result = $stockFilter->withValue(5); // Quantity < 5 units

// System performance alerts
$cpuFilter = new Lt('cpu_usage', 90); // CPU usage < 90%

// Usage in schema
$schema->addFilter('max_price', new Lt('price', new NumericValue()));
// Usage: ?filter[max_price]=100
```

#### Lte (Less Than or Equal) Filter

Filters records where a field value is less than or equal to the specified value.

```php
use Spiral\DataGrid\Specification\Filter\Lte;

// Maximum price filtering
$priceFilter = new Lte('price', new NumericValue());
$result = $priceFilter->withValue(100); // Price <= $100

// Senior discount eligibility
$ageFilter = new Lte('age', new IntValue());
$result = $ageFilter->withValue(65); // Age <= 65

// Performance monitoring
$cpuFilter = new Lte('cpu_usage', new FloatValue());
$result = $cpuFilter->withValue(85.0); // CPU usage <= 85%

// Budget compliance
$budgetFilter = new Lte('amount', 1000); // Amount <= $1000

// Usage in schema
$schema->addFilter('max_budget', new Lte('amount', new NumericValue()));
// Usage: ?filter[max_budget]=1000
```

### Like Filter

Filters records where a field value matches a pattern (similar to SQL LIKE).

Pattern placeholders:

- `%s` = the search value position
- `%` = wildcard for any characters

Common patterns:

- `'%%%s%%'` (default) = contains (anywhere in text)
- `'%s%%'` = starts with
- `'%%%s'` = ends with

```php
use Spiral\DataGrid\Specification\Filter\Like;

// General search (contains pattern - default)
$searchFilter = new Like('product_name', new StringValue());
$result = $searchFilter->withValue('iPhone'); // Matches "iPhone 15", "Apple iPhone", etc.

// Email domain filtering (ends with pattern)
$emailFilter = new Like('email', new StringValue(), '%%%s');
$result = $emailFilter->withValue('@company.com'); // Matches emails ending with @company.com

// Title prefix filtering (starts with pattern)
$titleFilter = new Like('title', new StringValue(), '%s%%');
$result = $titleFilter->withValue('How to'); // Matches "How to code", "How to cook", etc.

// Fixed pattern search
$codeFilter = new Like('product_code', 'ABC', 'SKU-%s-%%');
// Matches "SKU-ABC-001", "SKU-ABC-premium", etc.

// Usage in schema
$schema->addFilter('search', new Like('name', new StringValue()));
// Usage: ?filter[search]=john
// SQL: WHERE name LIKE '%john%'
```

### InArray Filter

Filters records where a field value exists within a specified array of values.

```php
use Spiral\DataGrid\Specification\Filter\InArray;

// Multi-category product filtering
$categoryFilter = new InArray('category', new StringValue());
$result = $categoryFilter->withValue(['electronics', 'computers', 'mobile']);

// Order status filtering
$statusFilter = new InArray('status', new StringValue());
$result = $statusFilter->withValue(['pending', 'processing']);

// Fixed list of valid IDs
$idFilter = new InArray('product_id', [1, 5, 10, 15, 20]);

// Tag-based content filtering
$tagFilter = new InArray('tags', new ArrayValue(new StringValue()));
$result = $tagFilter->withValue(['php', 'tutorial', 'beginner']);

// User permission filtering
$roleFilter = new InArray('role', new EnumValue(
    new StringValue(), 'admin', 'moderator', 'editor', 'author'
));
$result = $roleFilter->withValue(['admin', 'moderator']);

// Usage in schema
$schema->addFilter('categories', new InArray('category_id', new ArrayValue(new IntValue())));
// Usage: ?filter[categories]=1,3,5
// SQL: WHERE category_id IN (1, 3, 5)
```

### NotInArray Filter

Filters records where a field value does NOT exist within a specified array of values.

```php
use Spiral\DataGrid\Specification\Filter\NotInArray;

// Exclude problematic product categories
$categoryFilter = new NotInArray('category', new StringValue());
$result = $categoryFilter->withValue(['discontinued', 'recalled', 'damaged']);

// Exclude problematic order statuses
$statusFilter = new NotInArray('status', new StringValue());
$result = $statusFilter->withValue(['cancelled', 'refunded', 'failed']);

// Exclude specific user roles
$roleFilter = new NotInArray('role', new StringValue());
$result = $roleFilter->withValue(['banned', 'suspended', 'guest']);

// Fixed blacklist of IDs
$idFilter = new NotInArray('user_id', [1, 5, 10, 15, 20]);

// Usage in schema
$schema->addFilter('exclude_categories', new NotInArray('category', new ArrayValue(new StringValue())));
// Usage: ?filter[exclude_categories]=archived,deleted
```

### Between Filter

Filters values that fall between two boundaries (range filtering).

```php
use Spiral\DataGrid\Specification\Filter\Between;

// Price range filter for e-commerce
$priceFilter = new Between('price', new NumericValue());
$result = $priceFilter->withValue([50, 200]); // Products between $50-$200

// Date range filter with fixed values
$dateFilter = new Between('created_at', ['2024-01-01', '2024-03-31']);

// Age range with boundary control
$ageFilter = new Between('age', new IntValue(), true, false); // 18 <= age < 65

// Rating filter (3 to 5 stars inclusive)
$ratingFilter = new Between('rating', [3, 5], true, true);

// Dynamic salary range
$salaryFilter = new Between('salary', new NumericValue());
$result = $salaryFilter->withValue([60000, 120000]);

// Usage in schema
$schema->addFilter('price_range', new Between('price', new NumericValue()));
// Usage: ?filter[price_range]=50,200
// SQL: WHERE price BETWEEN 50 AND 200
```

### ValueBetween Filter

Filters records where a VALUE falls between two FIELD boundaries.

```php
use Spiral\DataGrid\Specification\Filter\ValueBetween;

// Job age requirement matching
$ageFilter = new ValueBetween(new IntValue(), ['min_age', 'max_age']);
$result = $ageFilter->withValue(25); // Find jobs where 25 is between min_age and max_age

// Budget-friendly product search
$budgetFilter = new ValueBetween(new NumericValue(), ['min_price', 'max_price']);
$result = $budgetFilter->withValue(500); // Products where $500 is within price range

// Event availability checking
$dateFilter = new ValueBetween(new DatetimeValue(), ['start_date', 'end_date']);
$result = $dateFilter->withValue('2024-06-15'); // Events active on this date

// Fixed value between dynamic fields
$currentTimeFilter = new ValueBetween('2024-06-15 14:30:00', ['start_time', 'end_time']);
// Find events active at this specific time

// Usage in schema
$schema->addFilter('event_date', new ValueBetween(new DatetimeValue(), ['start_date', 'end_date']));
// Usage: ?filter[event_date]=2024-06-15
// SQL: WHERE '2024-06-15' BETWEEN start_date AND end_date
```

## Group Filters (Logical Operators)

### All Filter (AND Logic)

Combines multiple filters with AND logic - all filters must match.

```php
use Spiral\DataGrid\Specification\Filter\All;

// Find products that are electronics, under $100, and in stock
$filter = new All(
    new Equals('category', 'electronics'),
    new Lt('price', 100),
    new Gt('stock_quantity', 0)
);

// Find users with specific criteria
$filter = new All(
    new Equals('status', 'active'),
    new Equals('email_verified', true),
    new Gte('created_at', '2024-01-01')
);

// Usage in schema
$schema->addFilter('premium_recent', new All(
    new Equals('status', 'premium'),
    new Gte('created_at', '2023-01-01')
));
// Usage: ?filter[premium_recent]=1 (any truthy value activates the filter)
```

### Any Filter (OR Logic)

Combines multiple filters with OR logic - at least one filter must match.

```php
use Spiral\DataGrid\Specification\Filter\Any;

// Search across multiple fields
$searchFilter = new Any(
    new Like('title', new StringValue()),
    new Like('description', new StringValue()),
    new Like('tags', new StringValue())
);
$result = $searchFilter->withValue('iPhone'); // Matches any field containing "iPhone"

// Multiple status filtering
$statusFilter = new Any(
    new Equals('status', 'pending'),
    new Equals('status', 'processing'),
    new Equals('status', 'shipped')
);

// Flexible user permissions
$accessFilter = new Any(
    new Equals('role', 'admin'),
    new Equals('is_premium', true),
    new Gt('subscription_level', 2)
);

// Usage in schema
$schema->addFilter('global_search', new Any(
    new Equals('id', new IntValue()),
    new Like('name', new StringValue()),
    new Like('email', new StringValue())
));
```

## Advanced Filters

### Map Filter

Complex filter that maps input array values to multiple named sub-filters.

```php
use Spiral\DataGrid\Specification\Filter\Map;

// Price range filter with min/max mapping
$priceRangeFilter = new Map([
    'min' => new Gte('price', new NumericValue()),
    'max' => new Lte('price', new NumericValue())
]);
$result = $priceRangeFilter->withValue(['min' => 50, 'max' => 200]);
// Applies: price >= 50 AND price <= 200

// Date range filtering
$dateRangeFilter = new Map([
    'from' => new Gte('created_at', new DatetimeValue()),
    'to' => new Lte('created_at', new DatetimeValue())
]);
$result = $dateRangeFilter->withValue([
    'from' => '2024-01-01',
    'to' => '2024-12-31'
]);

// Multi-field search
$searchFilter = new Map([
    'title' => new Like('title', new StringValue()),
    'description' => new Like('description', new StringValue()),
    'author' => new Equals('author_id', new IntValue())
]);
$result = $searchFilter->withValue([
    'title' => 'PHP Tutorial',
    'description' => 'beginner',
    'author' => 123
]);

// Usage in schema
$schema->addFilter('advanced_search', $searchFilter);
// Usage: ?filter[advanced_search][title]=PHP&filter[advanced_search][author]=123
```

### Select Filter

Provides filter selection from a predefined set of named filters.

```php
use Spiral\DataGrid\Specification\Filter\Select;

// Predefined content filters
$contentFilter = new Select([
    'popular' => new Gte('view_count', 1000),
    'recent' => new Gte('created_at', '-7 days'),
    'featured' => new Equals('is_featured', true),
    'top_rated' => new Gte('rating', 4.5)
]);

// Select single filter
$result = $contentFilter->withValue('popular'); // Show popular content

// Select multiple filters (AND logic)
$result = $contentFilter->withValue(['popular', 'recent']); // Popular AND recent

// E-commerce product filters
$productFilter = new Select([
    'on_sale' => new Gt('discount_percentage', 0),
    'in_stock' => new Gt('quantity', 0),
    'bestseller' => new Gte('sales_count', 100),
    'new_arrival' => new Gte('created_at', '-30 days'),
    'premium' => new Gte('price', 500)
]);

// Usage in schema
$schema->addFilter('preset', $productFilter);
// Usage: ?filter[preset]=on_sale or ?filter[preset][]=popular&filter[preset][]=recent
```

## Mixed Specifications

### SortedFilter

Combines filtering and sorting operations into a single specification.

```php
use Spiral\DataGrid\Specification\Filter\SortedFilter;
use Spiral\DataGrid\Specification\Sorter\DescSorter;

// Popular posts filter-sort combination
$popularPosts = new SortedFilter(
    'popular_posts',
    new Gte('view_count', 1000),           // Filter: views >= 1000
    new DescSorter('popularity_score')     // Sort: by popularity descending
);

// Recent high-rated products
$topRecentProducts = new SortedFilter(
    'top_recent',
    new All(                               // Filter: recent AND highly rated
        new Gte('created_at', '-30 days'),
        new Gte('rating', 4.0)
    ),
    new DescSorter('rating')               // Sort: by rating descending
);

// Usage in grid schema
$schema->addFilter('trending', new Select([
    'popular' => new SortedFilter(
        'popular',
        new Gte('view_count', 1000),
        new DescSorter('view_count')
    ),
    'recent' => new SortedFilter(
        'recent',
        new Gte('created_at', '-7 days'),
        new DescSorter('created_at')
    )
]));
```

## PostgreSQL-Specific Filters

### ILike Filter

PostgreSQL-specific case-insensitive LIKE filter using ILIKE operator.

```php
use Spiral\DataGrid\Specification\Filter\Postgres\ILike;

// Case-insensitive user search
$nameFilter = new ILike('name', new StringValue());
$result = $nameFilter->withValue('john'); // Matches "John", "JOHN", "john", "JoHn"

// Case-insensitive email search
$emailFilter = new ILike('email', new StringValue(), '%%%s');
$result = $emailFilter->withValue('@GMAIL.COM'); // Matches emails ending with @gmail.com

// Product name search (case-insensitive)
$productFilter = new ILike('product_name', new StringValue());
$result = $productFilter->withValue('iphone'); // Matches "iPhone", "IPHONE", "IPhone"

// Usage in schema (PostgreSQL only)
$schema->addFilter('search', new ILike('name', new StringValue()));
// Usage: ?filter[search]=john
// SQL: WHERE name ILIKE '%john%'
```

## Filter Input Processing

### URL Query Parameters

Filters accept input through URL query parameters:

```
GET /api/products?filter[search]=laptop&filter[price_range]=500,1500&filter[category]=electronics
```

### Array Input

For programmatic use, filters can accept array input:

```php
$filterData = [
    'search' => 'laptop',
    'price_range' => [500, 1500],
    'category' => 'electronics'
];

foreach ($filterData as $key => $value) {
    if ($schema->hasFilter($key)) {
        $filter = $schema->getFilter($key)->withValue($value);
        if ($filter !== null) {
            $grid = $grid->withFilter($key, $filter);
        }
    }
}
```

## Error Handling

The Data Grid component gracefully handles invalid filter input:

```php
// Schema only allows specific categories
$schema->addFilter('category', new Equals('category', new EnumValue(
    new StringValue(), 'electronics', 'books', 'clothing'
)));

// Invalid input is ignored - filter returns null
// ?filter[category]=invalid_category
// Result: Filter is not applied, no error thrown
```

## Best Practices

1. **Use meaningful filter names** - Choose names that make sense to frontend developers
2. **Validate input types** - Always use appropriate ValueInterface implementations
3. **Consider performance** - Add indexes for filtered fields
4. **Limit filter complexity** - Break complex filters into multiple simple ones
5. **Document filter behavior** - Explain non-obvious filter logic

```php
class UserSchema extends GridSchema  
{
    public function __construct()
    {
        // Clear, descriptive filter names
        $this->addFilter('search', new Like('name', new StringValue()));
        $this->addFilter('email_domain', new Like('email', new StringValue(), '%%%s'));
        $this->addFilter('registration_date', new Gte('created_at', new DatetimeValue()));
        $this->addFilter('is_active', new Equals('status', new EnumValue(
            new StringValue(), 'active', 'inactive'
        )));
        
        // Performance-conscious filters
        $this->addFilter('user_id', new Equals('id', new IntValue())); // Primary key lookup
        $this->addFilter('role', new InArray('role_id', new ArrayValue(new IntValue()))); // Indexed foreign key
    }
}
```

## Common Filter Patterns

### E-commerce Product Filtering

```php
class ProductSchema extends GridSchema
{
    public function __construct()
    {
        // Text search
        $this->addFilter('search', new Any(
            new Like('name', new StringValue()),
            new Like('description', new StringValue()),
            new Equals('sku', new StringValue())
        ));
        
        // Price filtering
        $this->addFilter('price_range', new Between('price', new NumericValue()));
        $this->addFilter('max_price', new Lte('price', new NumericValue()));
        
        // Category filtering
        $this->addFilter('category', new InArray('category_id', new ArrayValue(new IntValue())));
        
        // Availability
        $this->addFilter('in_stock', new Gt('stock_quantity', 0));
        $this->addFilter('on_sale', new Gt('discount_percentage', 0));
        
        // Rating
        $this->addFilter('min_rating', new Gte('average_rating', new FloatValue()));
        
        // Date filters
        $this->addFilter('new_arrivals', new Gte('created_at', new DatetimeValue()));
        $this->addFilter('recently_updated', new Gte('updated_at', new DatetimeValue()));
    }
}
```

### User Management Filtering

```php
class UserSchema extends GridSchema
{
    public function __construct()
    {
        // User search
        $this->addFilter('search', new Any(
            new Like('name', new StringValue()),
            new Like('email', new StringValue()),
            new Equals('id', new IntValue())
        ));
        
        // Status filtering
        $this->addFilter('status', new Equals('status', new EnumValue(
            new StringValue(), 'active', 'inactive', 'suspended', 'pending'
        )));
        
        // Role filtering
        $this->addFilter('role', new InArray('role', new ArrayValue(new StringValue())));
        
        // Registration date
        $this->addFilter('registered_after', new Gte('created_at', new DatetimeValue()));
        $this->addFilter('registered_before', new Lte('created_at', new DatetimeValue()));
        
        // Activity
        $this->addFilter('last_login_after', new Gte('last_login_at', new DatetimeValue()));
        $this->addFilter('never_logged_in', new Equals('last_login_at', null));
        
        // Email verification
        $this->addFilter('email_verified', new Equals('email_verified_at', new BoolValue()));
    }
}
```
