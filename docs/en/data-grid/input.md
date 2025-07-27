# Input Processing

The Input system is the foundation that makes Data Grid work with different input sources. It provides a standardized
way to extract user parameters regardless of where they come from - HTTP requests, console commands, arrays, or custom
sources.

## Understanding InputInterface

The `InputInterface` is a simple but powerful abstraction that normalizes how Data Grid receives user input:

```php
interface InputInterface
{
    public function hasValue(string $option): bool;
    public function getValue(string $option, mixed $default = null): mixed;
    public function withNamespace(string $namespace): InputInterface;
}
```

**Key Concepts:**

- **Input Keys**: The GridFactory uses specific keys to organize user input (`filter`, `sort`, `paginate`)
- **Array Values**: Always returns arrays of values, not dot notation
- **Namespace Support**: Allows scoping input to specific contexts

## Input Key Structure

The GridFactory organizes user input using three main keys:

### Filter Input (`filter`)

Used for filtering operations. The structure is `filter[field_name]=value`:

```php
// URL: ?filter[name]=john&filter[status]=active&filter[age]=25

$input->getValue('filter'); 
// Returns: ['name' => 'john', 'status' => 'active', 'age' => '25']

$input->getValue('filter')['name'];    // 'john'
$input->getValue('filter')['status'];  // 'active'
$input->getValue('filter')['age'];     // '25'
```

### Sort Input (`sort`)

Used for sorting operations. The structure is `sort[field_name]=direction`:

```php
// URL: ?sort[created_at]=desc&sort[name]=asc

$input->getValue('sort');
// Returns: ['created_at' => 'desc', 'name' => 'asc']

$input->getValue('sort')['created_at']; // 'desc'
$input->getValue('sort')['name'];       // 'asc'
```

### Pagination Input (`paginate`)

Used for pagination operations. Common keys are `page` and `limit`:

```php
// URL: ?paginate[page]=2&paginate[limit]=25

$input->getValue('paginate');
// Returns: ['page' => '2', 'limit' => '25']

$input->getValue('paginate')['page'];   // '2'
$input->getValue('paginate')['limit'];  // '25'
```

## Built-in Input Sources

### ArrayInput

The most basic input source, useful for testing and manual data processing:

```php
use Spiral\DataGrid\Input\ArrayInput;

// Basic array input
$input = new ArrayInput([
    'filter' => ['name' => 'John', 'status' => 'active'],
    'sort' => ['created_at' => 'desc'],
    'paginate' => ['page' => 2, 'limit' => 25],
]);

// Access values
$input->hasValue('filter');                    // true
$input->getValue('filter')['name'];            // 'John'
$input->getValue('sort')['created_at'];        // 'desc'
$input->getValue('paginate')['page'];          // 2
$input->getValue('nonexistent', 'default');   // 'default'
```

**Real-World Usage:**

```php
// Testing scenario
$testInput = new ArrayInput([
    'filter' => [
        'search' => 'laptop',
        'category' => 'electronics',
        'price_range' => [500, 1500],
        'in_stock' => true,
    ],
    'sort' => [
        'popularity' => 'desc',
        'price' => 'asc',
    ],
    'paginate' => [
        'page' => 1,
        'limit' => 20,
    ],
]);

// API endpoint simulation
public function testProductSearch()
{
    $factory = $this->gridFactory->withInput($testInput);
    $grid = $factory->create($products->select(), new ProductSchema());
    
    return iterator_to_array($grid);
}
```

### HTTP Input (Framework Integration)

When using Data Grid with web frameworks, you'll typically have HTTP-specific input implementations:

```php
// Example HTTP Input implementation (framework-specific)
class HttpInput implements InputInterface
{
    public function __construct(private readonly RequestInterface $request) {}
    
    public function hasValue(string $option): bool
    {
        return $this->request->has($option);
    }
    
    public function getValue(string $option, mixed $default = null): mixed
    {
        return $this->request->get($option, $default);
    }
    
    public function withNamespace(string $namespace): InputInterface
    {
        return new self($this->request->withPrefix($namespace));
    }
}
```

**URL Processing Examples:**

```php
// URL: /products?filter[search]=laptop&filter[category]=electronics&sort[price]=desc&paginate[page]=2

$httpInput = new HttpInput($request);

// Filter processing
$filters = $httpInput->getValue('filter');
// ['search' => 'laptop', 'category' => 'electronics']

// Sort processing  
$sorts = $httpInput->getValue('sort');
// ['price' => 'desc']

// Pagination processing
$pagination = $httpInput->getValue('paginate');
// ['page' => '2']
```

## Complex Input Examples

### Multi-Value Filters

Handle complex filtering scenarios with arrays and nested data:

```php
// URL: ?filter[categories]=electronics,computers&filter[tags]=php,tutorial&filter[price_range]=50,200

$input = new ArrayInput([
    'filter' => [
        'categories' => ['electronics', 'computers'],     // Multi-select
        'tags' => ['php', 'tutorial'],                   // Array values
        'price_range' => [50, 200],                      // Range values
        'features' => [                                  // Complex nested
            'bluetooth' => true,
            'wifi' => true,
            'gps' => false,
        ],
    ],
]);

// In your schema
class ProductSchema extends GridSchema
{
    public function __construct()
    {
        // Multi-category filtering
        $this->addFilter('categories', new InArray('category', new ArrayValue(new StringValue())));
        
        // Tag filtering
        $this->addFilter('tags', new InArray('tags', new ArrayValue(new StringValue())));
        
        // Price range
        $this->addFilter('price_range', new Between('price', new NumericValue()));
        
        // Feature map
        $this->addFilter('features', new Map([
            'bluetooth' => new Equals('has_bluetooth', new BoolValue()),
            'wifi' => new Equals('has_wifi', new BoolValue()),
            'gps' => new Equals('has_gps', new BoolValue()),
        ]));
    }
}
```

### Advanced Sorting

Complex sorting scenarios with priorities and directions:

```php
// URL: ?sort[featured]=desc&sort[popularity]=desc&sort[price]=asc&sort[name]=asc

$input = new ArrayInput([
    'sort' => [
        'featured' => 'desc',      // Featured items first
        'popularity' => 'desc',    // Then by popularity  
        'price' => 'asc',          // Then by price (low to high)
        'name' => 'asc',           // Finally alphabetical
    ],
]);

// In controller
public function products(ProductSchema $schema, GridFactoryInterface $factory, ProductRepository $products)
{
    $grid = $factory->create($products->select(), $schema);
    
    // Applied sorting order:
    // ORDER BY featured DESC, popularity_score DESC, price ASC, name ASC
    
    return [
        'products' => iterator_to_array($grid),
        'applied_sorts' => $grid->getOption(GridInterface::SORTERS),
    ];
}
```

### Pagination with Metadata

Advanced pagination scenarios:

```php
// URL: ?paginate[page]=3&paginate[limit]=50&paginate[include_total]=true

$input = new ArrayInput([
    'paginate' => [
        'page' => 3,
        'limit' => 50,
        'include_total' => true,     // Request total count
    ],
]);

// Custom paginator that handles the include_total flag
class MetadataPaginator extends PagePaginator
{
    public function withValue(mixed $value): ?SpecificationInterface
    {
        if (!is_array($value)) {
            return null;
        }
        
        $paginator = parent::withValue($value);
        
        // Handle additional metadata flags
        if (isset($value['include_total']) && $value['include_total']) {
            return $paginator->withTotalCount(true);
        }
        
        return $paginator;
    }
}
```

## Input Namespacing

Use namespaces to scope input to specific contexts or components:

```php
// URL: ?products[filter][name]=laptop&products[sort][price]=desc&users[filter][role]=admin

$input = new ArrayInput([
    'products' => [
        'filter' => ['name' => 'laptop'],
        'sort' => ['price' => 'desc'],
    ],
    'users' => [
        'filter' => ['role' => 'admin'],
        'sort' => ['name' => 'asc'],
    ],
]);

// Create namespaced inputs
$productInput = $input->withNamespace('products');
$userInput = $input->withNamespace('users');

// Use in separate grids
$productGrid = $productFactory->withInput($productInput)->create($products->select(), $productSchema);
$userGrid = $userFactory->withInput($userInput)->create($users->select(), $userSchema);
```

**Multi-Grid Dashboard Example:**

```php
class DashboardController
{
    public function index(
        GridFactoryInterface $factory,
        ProductSchema $productSchema,
        OrderSchema $orderSchema,
        UserSchema $userSchema
    ) {
        $input = $factory->getInput(); // Gets current request input
        
        // Create separate grids with namespaced input
        $productGrid = $factory
            ->withInput($input->withNamespace('products'))
            ->create($this->products->select(), $productSchema);
            
        $orderGrid = $factory
            ->withInput($input->withNamespace('orders'))
            ->create($this->orders->select(), $orderSchema);
            
        $userGrid = $factory
            ->withInput($input->withNamespace('users'))
            ->create($this->users->select(), $userSchema);
        
        return [
            'products' => iterator_to_array($productGrid),
            'orders' => iterator_to_array($orderGrid),
            'users' => iterator_to_array($userGrid),
        ];
    }
}

// URL: ?products[filter][category]=electronics&orders[filter][status]=pending&users[filter][role]=admin
```

## Custom Input Sources

### Console Input

For command-line interfaces:

```php
use Symfony\Component\Console\Input\InputInterface as ConsoleInputInterface;

class ConsoleInput implements InputInterface
{
    public function __construct(private readonly ConsoleInputInterface $input) {}
    
    public function hasValue(string $option): bool
    {
        return $this->input->hasOption($option);
    }
    
    public function getValue(string $option, mixed $default = null): mixed
    {
        return $this->input->getOption($option) ?? $default;
    }
    
    public function withNamespace(string $namespace): InputInterface
    {
        // Implementation depends on your console framework
        return new PrefixedConsoleInput($this->input, $namespace);
    }
}

// Usage in console command
// php app.php products:list --filter='{"name":"laptop"}' --sort='{"price":"desc"}' --paginate='{"page":2}'

class ProductListCommand extends Command
{
    public function execute(InputInterface $input, OutputInterface $output)
    {
        $gridInput = new ConsoleInput($input);
        
        // Parse JSON options
        $filter = json_decode($gridInput->getValue('filter', '{}'), true);
        $sort = json_decode($gridInput->getValue('sort', '{}'), true);
        $paginate = json_decode($gridInput->getValue('paginate', '{}'), true);
        
        $arrayInput = new ArrayInput([
            'filter' => $filter,
            'sort' => $sort,
            'paginate' => $paginate,
        ]);
        
        $factory = $this->gridFactory->withInput($arrayInput);
        $grid = $factory->create($this->products->select(), new ProductSchema());
        
        foreach ($grid as $product) {
            $output->writeln($product->getName());
        }
    }
}
```

### JSON API Input

For APIs that accept JSON request bodies:

```php
class JsonApiInput implements InputInterface
{
    private array $data;
    
    public function __construct(string $jsonBody)
    {
        $this->data = json_decode($jsonBody, true) ?? [];
    }
    
    public function hasValue(string $option): bool
    {
        return isset($this->data[$option]);
    }
    
    public function getValue(string $option, mixed $default = null): mixed
    {
        return $this->data[$option] ?? $default;
    }
    
    public function withNamespace(string $namespace): InputInterface
    {
        return new ArrayInput($this->data[$namespace] ?? []);
    }
}

// Usage with JSON API
// POST /api/products
// Content-Type: application/json
// {
//     "filter": {"name": "laptop", "category": "electronics"},
//     "sort": {"price": "desc"},
//     "paginate": {"page": 1, "limit": 20}
// }

class ApiController
{
    public function search(Request $request)
    {
        $jsonInput = new JsonApiInput($request->getContent());
        
        $factory = $this->gridFactory->withInput($jsonInput);
        $grid = $factory->create($this->products->select(), new ProductSchema());
        
        return [
            'data' => iterator_to_array($grid),
            'pagination' => $grid->getOption(GridInterface::PAGINATOR),
            'filters' => $grid->getOption(GridInterface::FILTERS),
        ];
    }
}
```

### GraphQL Input

For GraphQL resolvers:

```php
class GraphQLInput implements InputInterface
{
    public function __construct(private readonly array $args) {}
    
    public function hasValue(string $option): bool
    {
        return isset($this->args[$option]);
    }
    
    public function getValue(string $option, mixed $default = null): mixed
    {
        return $this->args[$option] ?? $default;
    }
    
    public function withNamespace(string $namespace): InputInterface
    {
        return new ArrayInput($this->args[$namespace] ?? []);
    }
}

// GraphQL Query:
// query {
//   products(
//     filter: {name: "laptop", category: "electronics"}
//     sort: {price: DESC}
//     paginate: {page: 1, limit: 20}
//   ) {
//     id name price category
//     pagination { total pages }
//   }
// }

class ProductResolver
{
    public function products($root, array $args)
    {
        $graphqlInput = new GraphQLInput($args);
        
        $factory = $this->gridFactory->withInput($graphqlInput);
        $grid = $factory->create($this->products->select(), new ProductSchema());
        
        return [
            'data' => iterator_to_array($grid),
            'pagination' => $grid->getOption(GridInterface::PAGINATOR),
        ];
    }
}
```