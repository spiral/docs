# Value Accessors

Accessors act as middleware for value processing, allowing you to apply transformations before the actual value
validation. They provide a clean way to standardize, clean, and transform user input without cluttering your filter
logic.

## Understanding Accessors

Accessors follow a decorator pattern where each accessor wraps another `ValueInterface` and transforms the input before
passing it to the next processor in the chain. This allows for powerful, composable data transformations.

**Processing Flow:**

```
User Input → Accessor 1 → Accessor 2 → Base ValueInterface → Validated Output
```

## Built-in Accessors

### ToUpper Accessor

Transforms strings to uppercase before validation:

```php
use Spiral\DataGrid\Specification\Value\Accessor\ToUpper;

// Basic uppercase conversion
$uppercaseString = new ToUpper(new StringValue());
$statusFilter = new Equals('status', $uppercaseString);
$result = $statusFilter->withValue('active'); // Converts to 'ACTIVE'

// Configuration value processing
$configProcessor = new ToUpper(
    new EnumValue(new StringValue(), 'DEBUG', 'INFO', 'WARNING', 'ERROR')
);
$configFilter = new Equals('log_level', $configProcessor);
$result = $configFilter->withValue('debug'); // Converts to 'DEBUG'
```

### ToLower Accessor

Transforms strings to lowercase before validation:

```php
use Spiral\DataGrid\Specification\Value\Accessor\ToLower;

// Basic lowercase conversion
$lowercaseString = new ToLower(new StringValue());
$userFilter = new Equals('username', $lowercaseString);
$result = $userFilter->withValue('JohnDoe'); // Converts to 'johndoe'

// Tag standardization
$tagProcessor = new Split(
    new ArrayValue(
        new ToLower(
            new Trim(new StringValue())
        )
    ),
    ','
);
$tagFilter = new InArray('tags', $tagProcessor);
$result = $tagFilter->withValue('PHP, JAVASCRIPT, Python');
// Results in: ['php', 'javascript', 'python']
```

### Trim Accessor

Removes whitespace from the beginning and end of strings:

```php
use Spiral\DataGrid\Specification\Value\Accessor\Trim;

// Basic whitespace trimming
$trimmedString = new Trim(new StringValue());
$userFilter = new Equals('username', $trimmedString);
$result = $userFilter->withValue('  john_doe  '); // Converts to 'john_doe'

// Form field processing
$nameProcessor = new Trim(new StringValue());
$formFilter = new Map([
    'first_name' => new Equals('first_name', $nameProcessor),
    'last_name' => new Equals('last_name', $nameProcessor),
    'company' => new Equals('company', new Trim(new StringValue(true))) // Optional field
]);
// Cleans all name fields: '  John  ' -> 'John'
```

### Split Accessor

Transforms strings into arrays using a specified delimiter:

```php
use Spiral\DataGrid\Specification\Value\Accessor\Split;

// Basic tag splitting (comma-separated)
$tagSplitter = new Split(new ArrayValue(new StringValue()));
$tagFilter = new InArray('tags', $tagSplitter);
$result = $tagFilter->withValue('php,javascript,python');
// Results in: ['php', 'javascript', 'python']

// Custom delimiter (pipe-separated)
$pipeSplitter = new Split(new ArrayValue(new StringValue()), '|');
$categoryFilter = new InArray('categories', $pipeSplitter);
$result = $categoryFilter->withValue('electronics|computers|mobile');
// Results in: ['electronics', 'computers', 'mobile']

// Combined with trimming for clean results
$cleanTagSplitter = new Split(
    new ArrayValue(new Trim(new StringValue())),
    ','
);
$result = $cleanTagSplitter->convert('  php  ,  javascript  ,  python  ');
// Results in: ['php', 'javascript', 'python'] (trimmed)
```

## Using Accessors in Grid Schemas

Here's how to integrate accessors into your grid schemas for real-world applications:

### E-commerce Product Schema

```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\{Equals, Like, InArray, Between};
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Value\{StringValue, NumericValue, ArrayValue, EnumValue};
use Spiral\DataGrid\Specification\Value\Accessor\{ToLower, Trim, Split};

class ProductSchema extends GridSchema
{
    public function __construct()
    {
        // Clean search query: trim whitespace and convert to lowercase for case-insensitive search
        $searchProcessor = new ToLower(new Trim(new StringValue()));
        $this->addFilter('search', new Like('name', $searchProcessor, '%%%s%%'));
        
        // Category filtering with cleanup
        $categoryProcessor = new ToLower(new Trim(new StringValue()));
        $this->addFilter('category', new Equals('category', $categoryProcessor));
        
        // Multi-category selection (comma-separated)
        $categoriesProcessor = new Split(
            new ArrayValue(
                new ToLower(new Trim(new StringValue())),
            ),
            ',',
        );
        $this->addFilter('categories', new InArray('category', $categoriesProcessor));
        
        // Tag-based filtering with full cleanup chain
        $tagProcessor = new Split(
            new ArrayValue(
                new ToLower(new Trim(new StringValue())),
            ),
            ',',
        );
        $this->addFilter('tags', new InArray('tags', $tagProcessor));
        
        // Price range filtering
        $this->addFilter('price', new Between('price', new NumericValue()));
        
        // Status filtering with standardization
        $statusProcessor = new ToUpper(
            new EnumValue(new StringValue(), 'ACTIVE', 'INACTIVE', 'DISCONTINUED'),
        );
        $this->addFilter('status', new Equals('status', $statusProcessor));
        
        // Sorting options
        $this->addSorter('name', new Sorter('name'));
        $this->addSorter('price', new Sorter('price'));
        $this->addSorter('created_at', new Sorter('created_at'));
        
        // Pagination
        $this->setPaginator(new PagePaginator(20, [10, 20, 50, 100]));
    }
}
```

## Creating Custom Accessors

You can create custom accessors by extending the `Accessor` base class:

### Basic Custom Accessor

```php
use Spiral\DataGrid\Specification\Value\Accessor\Accessor;
use Spiral\DataGrid\Specification\ValueInterface;

class SlugifyAccessor extends Accessor
{
    protected function acceptsCurrent(mixed $value): bool
    {
        return is_string($value);
    }
    
    protected function convertCurrent(mixed $value): string
    {
        // Convert to lowercase, replace spaces with hyphens, remove special chars
        $slug = strtolower($value);
        $slug = preg_replace('/[^\w\s-]/', '', $slug);
        $slug = preg_replace('/[-\s]+/', '-', $slug);
        return trim($slug, '-');
    }
}

// Usage
$slugProcessor = new SlugifyAccessor(new StringValue());
$urlFilter = new Equals('url_slug', $slugProcessor);
$result = $urlFilter->withValue('My Article Title!'); // Converts to 'my-article-title'
```

### Advanced Custom Accessor with Configuration

```php
class NormalizePhoneAccessor extends Accessor
{
    public function __construct(
        ValueInterface $next,
        private readonly string $format = 'E164' // E164, NATIONAL, INTERNATIONAL
    ) {
        parent::__construct($next);
    }
    
    protected function acceptsCurrent(mixed $value): bool
    {
        return is_string($value) || is_numeric($value);
    }
    
    protected function convertCurrent(mixed $value): string
    {
        $phone = preg_replace('/[^\d+]/', '', (string) $value);
        
        return match ($this->format) {
            'E164' => $this->toE164($phone),
            'NATIONAL' => $this->toNational($phone),
            'INTERNATIONAL' => $this->toInternational($phone),
            default => $phone
        };
    }
    
    private function toE164(string $phone): string
    {
        // Add country code if missing, remove formatting
        if (!str_starts_with($phone, '+')) {
            $phone = '+1' . ltrim($phone, '1');
        }
        return $phone;
    }
    
    private function toNational(string $phone): string
    {
        // Convert to (XXX) XXX-XXXX format
        $clean = preg_replace('/[^\d]/', '', $phone);
        if (strlen($clean) === 10) {
            return sprintf('(%s) %s-%s', 
                substr($clean, 0, 3),
                substr($clean, 3, 3),
                substr($clean, 6, 4)
            );
        }
        return $phone;
    }
    
    private function toInternational(string $phone): string
    {
        // Convert to +1 XXX XXX XXXX format
        $clean = preg_replace('/[^\d]/', '', $phone);
        if (strlen($clean) === 10) {
            return sprintf('+1 %s %s %s',
                substr($clean, 0, 3),
                substr($clean, 3, 3),
                substr($clean, 6, 4)
            );
        }
        return $phone;
    }
}

// Usage in schema
class ContactSchema extends GridSchema
{
    public function __construct()
    {
        // Normalize phone numbers to E164 format for searching
        $phoneProcessor = new NormalizePhoneAccessor(new StringValue(), 'E164');
        $this->addFilter('phone', new Equals('phone_number', $phoneProcessor));
        
        // Or use national format for display
        $displayPhoneProcessor = new NormalizePhoneAccessor(new StringValue(), 'NATIONAL');
        $this->addFilter('phone_display', new Like('phone_display', $displayPhoneProcessor));
    }
}
```

### JSON Accessor for Complex Data

```php
class JsonDecodeAccessor extends Accessor
{
    public function __construct(
        ValueInterface $next,
        private readonly bool $associative = true
    ) {
        parent::__construct($next);
    }
    
    protected function acceptsCurrent(mixed $value): bool
    {
        return is_string($value) && $this->isValidJson($value);
    }
    
    protected function convertCurrent(mixed $value): mixed
    {
        return json_decode($value, $this->associative);
    }
    
    private function isValidJson(string $value): bool
    {
        json_decode($value);
        return json_last_error() === JSON_ERROR_NONE;
    }
}

// Usage for filtering by nested JSON properties
class LogSchema extends GridSchema
{
    public function __construct()
    {
        // Filter by JSON metadata
        $metadataProcessor = new JsonDecodeAccessor(new AnyValue());
        $this->addFilter('metadata', new Equals('metadata', $metadataProcessor));
        
        // Other standard filters...
        $this->addFilter('level', new Equals('level', new StringValue()));
        $this->addFilter('message', new Like('message', new StringValue()));
        
        $this->addSorter('timestamp', new Sorter('created_at'));
        $this->setPaginator(new PagePaginator(50, [25, 50, 100, 200]));
    }
}
```

## Chaining Accessors

Accessors can be chained together for complex transformations:

```php
use Spiral\DataGrid\Specification\Value\Accessor\{ToUpper, Trim, Split};

// Complex processing chain: split → trim each → uppercase each
$complexProcessor = new Split(
    new ArrayValue(
        new ToUpper(
            new Trim(new StringValue())
        )
    ),
    '|'
);
$result = $complexProcessor->convert('  admin  |  user  |  guest  ');
// Results in: ['ADMIN', 'USER', 'GUEST']

// Real-world example: Processing form tags
$tagChain = new Split(
    new ArrayValue(
        new SlugifyAccessor(  // Custom slugify
            new ToLower(
                new Trim(new StringValue())
            )
        )
    ),
    ','
);
$result = $tagChain->convert('Web Development, PHP Programming, Database Design');
// Results in: ['web-development', 'php-programming', 'database-design']
```