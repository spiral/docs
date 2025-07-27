# Getting Started — Data Grid

**Data Grid** is a PHP component that acts as an intelligent data query abstraction layer. It automatically converts
user interface specifications (filters, sorting, pagination) into database queries or data transformations.

Think of it as a smart translator that sits between your user interface and your data sources. You define once what
operations users are allowed to perform on your data (what fields they can filter, how they can sort, pagination
limits), and the component handles all the complex logic of:

- **Input validation and security**
- **Query optimization**
- **Multi-source compatibility** - Working with databases, arrays, APIs, or custom data sources
- **Result formatting** - Providing consistent pagination and metadata

## The Problem: Manual Input Processing

Let's imagine you're building an **E-commerce website** where customers need to find products among thousands of items.

**Your users want to:**

- **Filter products** by price range ($50-$200), category (Electronics), brand (Apple)
- **Sort results** by popularity, price (low to high), or newest arrivals
- **Navigate pages** - show 20 products per page instead of overwhelming them with 10,000 items at once

```
GET: /api/products?filter[min_price]=50&filter[max_price]=200&filter[category]=Electronics&sort[popularity]=desc&paginate[page]=2&paginate[limit]=20
```

**Without Data Grid**, you'd need to write repetitive code for every single page AND manually process all user input in
your controllers:

```php
// Products Controller - Manual input processing nightmare
public function products(Request $request) 
{
    // 1. Manual input validation and sanitization
    $filters = [];
    $sorts = [];
    $page = 1;
    $limit = 20;
    
    // Process price filter
    if ($request->has('min_price')) {
        $minPrice = $request->get('min_price');
        if (!is_numeric($minPrice) || $minPrice < 0) {
            throw new ValidationException('Invalid minimum price');
        }
        $filters['min_price'] = (float)$minPrice;
    }
    
    if ($request->has('max_price')) {
        $maxPrice = $request->get('max_price');
        if (!is_numeric($maxPrice) || $maxPrice < 0) {
            throw new ValidationException('Invalid maximum price');
        }
        $filters['max_price'] = (float)$maxPrice;
    }
    
    // Process category filter
    if ($request->has('category')) {
        $category = trim($request->get('category'));
        $allowedCategories = ['Electronics', 'Clothing', 'Books', 'Sports'];
        if (!in_array($category, $allowedCategories)) {
            throw new ValidationException('Invalid category');
        }
        $filters['category'] = $category;
    }
    
    // Process name search
    if ($request->has('search')) {
        $search = trim($request->get('search'));
        if (strlen($search) < 2) {
            throw new ValidationException('Search term too short');
        }
        $filters['search'] = $search;
    }
    
    // Process sorting
    if ($request->has('sort_by')) {
        $sortBy = $request->get('sort_by');
        $allowedSorts = ['price', 'name', 'created_at', 'popularity_score'];
        if (!in_array($sortBy, $allowedSorts)) {
            throw new ValidationException('Invalid sort field');
        }
        $sorts['field'] = $sortBy;
        
        $sortDirection = $request->get('sort_direction', 'asc');
        if (!in_array($sortDirection, ['asc', 'desc'])) {
            throw new ValidationException('Invalid sort direction');
        }
        $sorts['direction'] = $sortDirection;
    }
    
    // Process pagination
    if ($request->has('page')) {
        $page = (int)$request->get('page');
        if ($page < 1) {$page = 1;}
    }
    
    if ($request->has('limit')) {
        $limit = (int)$request->get('limit');
        $allowedLimits = [10, 20, 50, 100];
        if (!in_array($limit, $allowedLimits)) {
            $limit = 20; // default
        }
    }
    
    // 2. Manual query building with Laravel Query Builder
    $query = Product::query();
    
    // Apply price filters
    if (isset($filters['min_price'])) {
        $query->where('price', '>=', $filters['min_price']);
    }
    
    if (isset($filters['max_price'])) {
        $query->where('price', '<=', $filters['max_price']);
    }
    
    // Apply category filter
    if (isset($filters['category'])) {
        $query->where('category', $filters['category']);
    }
    
    // Apply search filter
    if (isset($filters['search'])) {
        $query->where('name', 'LIKE', '%' . $filters['search'] . '%');
    }
    
    // Get total count BEFORE applying pagination (separate query)
    $total = $query->count();
    
    // Apply sorting
    if (!empty($sorts)) {
        $query->orderBy($sorts['field'], $sorts['direction']);
    }
    
    // Apply pagination
    $offset = ($page - 1) * $limit;
    $query->skip($offset)->take($limit);
    
    // Execute query
    $products = $query->get();
    
    return [
        'products' => $products,
        'pagination' => [
            'page' => $page,
            'limit' => $limit,
            'total' => $total,
            'pages' => ceil($total / $limit),
        ],
        'applied_filters' => $filters,
        'applied_sorts' => $sorts,
    ];
}
```

This manual approach leads to serious problems:

- **Massive code duplication**
- **Inconsistent validation**
- **Error-prone development**
- **Maintenance nightmare** - Change business rules? Update dozens of controllers

## How Data Grid Solves This

**Spiral Data Grid** acts as a **smart translator** between user requests and your data sources. Instead of writing
repetitive query code, you define **what's allowed once** and the component handles the rest automatically.

### The Magic: Schema-Driven Approach

```php
// Define ONCE what users can do with product data
class ProductSchema extends GridSchema 
{
    public function __construct() 
    {
        // Allow filtering by these fields
        $this->addFilter('price', new Between('price', new NumericValue()));
        //                  ↑                    ↑
        //              Input Key          Database Field
        //          (from user request)    (actual column)
        
        $this->addFilter('category', new Equals('category', new StringValue()));
        //                  ↑                      ↑
        //              Input Key            Database Field
        
        $this->addFilter('search', new Like('name', new StringValue()));
        //                  ↑                   ↑
        //             Input Key         Database Field
        //         (filter[search]=...)   (searches in 'name' column)
        
        // Allow sorting by these fields  
        $this->addSorter('price', new Sorter('price'));
        //                 ↑                   ↑
        //             Input Key         Database Field
        //        (sort[price]=desc)    (sorts by 'price' column)
        
        $this->addSorter('popularity', new Sorter('popularity_score'));
        //                    ↑                        ↑
        //               Input Key              Database Field
        //        (sort[popularity]=desc)    (sorts by 'popularity_score' column)
        
        // Set pagination rules
        $this->setPaginator(new PagePaginator(20, [10, 20, 50, 100]));
    }
}
```

As you can see, in example above, schema defines filters, sorters, and pagination with clear input keys that map to
database fields. This means you can change the database structure or add new filters without changing the interface
that users interact with.

```php
// Input key stays the same, but you can change database structure
$this->addFilter('search', new Like('product_name', new StringValue()));

// Later change to search multiple fields:
$this->addFilter('search', new Any(
    new Like('product_name', new StringValue()),
    new Like('description', new StringValue()),
));
```

Now **any interface** (web page, mobile app, API) can use this schema:

```php
// Controller - same code works for web, mobile, API
public function products(ProductSchema $schema, GridFactoryInterface $factory, ProductRepository $products): array 
{
    // User input: filter[price]=50,200&sort[popularity]=desc&paginate[page]=2
    $grid = $factory->create($products->select(), $schema);
    
    return [
        'products' => iterator_to_array($grid),      // [Product objects]
        'total' => $grid->getOption(GridInterface::COUNT),     // 1,247 total items
        'page' => $grid->getOption(GridInterface::PAGINATOR),  // Current page info
        'filters' => $grid->getOption(GridInterface::FILTERS), // Applied filters
    ];
}
```

## How Grid Schema Receives User Input

The Data Grid component automatically receives user input from various sources - web requests, console commands, APIs,
or arrays. When a user interacts with your application, their filtering, sorting, and pagination preferences are
captured and processed by the Grid Schema.

**User Input Flow:**

```
User Action →    Input Source → Grid Factory   →   Grid Schema   →   Database Query
     ↓                ↓               ↓                 ↓                 ↓
Filter products  HTTP Request   Input Processing   Schema Rules      WHERE clauses
Sort by price    Console Args   Validation         Allowed Fields    ORDER BY
Navigate page    JSON API       Security Check     Type Safety      LIMIT/OFFSET
```

**Input Structure:**

User input is organized into three main categories:

- **`filter`** - What data to show (`?filter[name]=john&filter[status]=active`)
- **`sort`** - How to order results (`?sort[created_at]=desc&sort[name]=asc`)
- **`paginate`** - Which page to display (`?paginate[page]=2&paginate[limit]=25`)

**Example:**

```php
// User searches for products:
// URL: ?filter[search]=laptop&filter[category]=electronics&sort[price]=desc&paginate[page]=2

// Grid Factory processes input and applies Schema rules:
$schema = new ProductSchema(); // Defines what's allowed
$grid = $factory->create($products->select(), $schema);

// Results in optimized database query:
// SELECT * FROM products 
// WHERE name LIKE '%laptop%' AND category = 'electronics'
// ORDER BY price DESC 
// LIMIT 25 OFFSET 25
```

The Grid Factory processes all user input and validates it against your Schema rules, ensuring only allowed operations
reach your database while providing a consistent interface across different input sources.

> **See Also:** [Input Processing Documentation](input.md) - Complete guide to input handling including custom input
> sources, validation, security, and advanced patterns for console, API, and GraphQL implementations.

## Complete Flow Example

Here's how everything works together:

```php
// 1. Define Schema
class ProductSchema extends GridSchema 
{
    public function __construct() 
    {
        $this->addFilter('search', new Like('name', new StringValue()));
        $this->addFilter('category', new Equals('category_id', new IntegerValue()));
        $this->addSorter('price', new Sorter('price'));
        $this->setPaginator(new PagePaginator(25));
    }
}

// 2. Controller receives user request
// GET /products?filter[search]=laptop&filter[category]=2&sort[price]=asc&paginate[page]=2&paginate[limit]=25

public function products(
    ProductSchema $schema, 
    GridFactoryInterface $factory, 
    ProductRepository $products
): array {
    // 3. Create base data source
    $select = $products->select(); // Returns Cycle SelectQuery
    
    // 4. Grid Factory processes everything
    $grid = $factory->create($select, $schema);
    
    // What happens inside create():
    // - Input processor extracts: 
    //   filter[search] = "laptop"
    //   filter[category] = "2" 
    //   sort[price] = "asc"
    //   paginate[page] = "2", paginate[limit] = "25"
    //
    // - Schema application:
    //   search filter: Like('name', 'laptop') ✓ valid
    //   category filter: Equals('category_id', 2) ✓ valid (converted to int)
    //   price sorter: Sorter('price', 'asc') ✓ valid
    //   pagination: Limit(25), Offset(25) ✓ valid
    //
    // - Compiler processes with QueryWriter:
    //   $select->where('name', 'LIKE', '%laptop%')
    //          ->where('category_id', 2)  
    //          ->orderBy('price', 'ASC')
    //          ->limit(25)
    //          ->offset(25)
    
    // 5. Return results with metadata
    return [
        'products' => iterator_to_array($grid),                    // Actual product objects
        'total' => $grid->getOption(GridInterface::COUNT),         // Total count for pagination
        'pagination' => $grid->getOption(GridInterface::PAGINATOR), // Current page info
        'filters' => $grid->getOption(GridInterface::FILTERS),     // Applied filters
    ];
}
```

## Security & Validation

The Grid Factory provides automatic security through the schema validation process:

```php
// Schema only allows specific values
$schema->addFilter('status', new Equals('status', new EnumValue(['active', 'inactive'])));

// User tries malicious input:
// ?filter[status]='; DROP TABLE users;--

// Process:
// 1. Input extracted: '; DROP TABLE users;--
// 2. EnumValue validation: not in ['active', 'inactive'] 
// 3. withValue() returns null (invalid)
// 4. Filter is ignored, no SQL injection possible
```

## Integration

Data Grid is a framework-agnostic package that can be integrated into any PHP framework or standalone application. The
component is designed with framework independence in mind, using standard PHP interfaces and dependency injection
principles.
