# Value Types

Value types are specifications that define how user input should be validated, converted, and processed before being
used in filters and other operations. They ensure data integrity and security by preventing invalid or malicious input
from reaching your data layer.

## How Value Types Work

Value types implement the `ValueInterface` and are used within filter specifications to validate and transform user
input:

```php
use Spiral\\DataGrid\\Specification\\Filter\\Equals;
use Spiral\\DataGrid\\Specification\\Value\\StringValue;

// The StringValue ensures user input is converted to a string
$filter = new Equals('name', new StringValue());

// User input gets validated and converted
$filter = $filter->withValue(123);     // Converts to \"123\"
$filter = $filter->withValue('John');  // Keeps as \"John\"
$filter = $filter->withValue(null);    // Filter returns null (invalid)
```

## Core Value Types

### AnyValue

Accepts any input value without validation or conversion. This is the most permissive value type:

```php
use Spiral\\DataGrid\\Specification\\Value\\AnyValue;

// Accept any search term without validation
$searchFilter = new Like('content', new AnyValue());
$result = $searchFilter->withValue('any string here'); // Passes through unchanged

// Legacy system compatibility
$legacyFilter = new Equals('legacy_field', new AnyValue());
$result = $legacyFilter->withValue(12345); // Number
$result = $legacyFilter->withValue('text'); // String
$result = $legacyFilter->withValue(true); // Boolean - all accepted
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Equals, Like};
use Spiral\\DataGrid\\Specification\\Value\\AnyValue;

class FlexibleSchema extends GridSchema
{
    public function __construct()
    {
        // Accept any type of search input
        $this->addFilter('search', new Like('content', new AnyValue()));
        
        // Legacy metadata field that can contain any value
        $this->addFilter('metadata', new Equals('metadata', new AnyValue()));
        
        // Debug field for development
        $this->addFilter('debug', new Equals('debug_info', new AnyValue()));
    }
}

// Usage: ?filter[search]=anything&filter[metadata]={\"key\":\"value\"}&filter[debug]=123
```

### StringValue

Validates string and numeric input, converting to string format with optional empty string handling:

```php
use Spiral\\DataGrid\\Specification\\Value\\StringValue;

// User name validation
$nameValue = new StringValue();
$userFilter = new Equals('name', $nameValue);
$result = $userFilter->withValue('John Doe');  // Valid string
$result = $userFilter->withValue(123);         // Valid - converts to '123'
$result = $userFilter->withValue('');          // Invalid - empty string (default)

// Allow empty strings (optional fields)
$optionalValue = new StringValue(true);  // Allow empty strings
$requiredValue = new StringValue(false); // Reject empty strings (default)

$optionalValue->accepts('');      // true - empty allowed
$optionalValue->accepts('text');  // true - non-empty string
$requiredValue->accepts('');      // false - empty not allowed
$requiredValue->accepts('text');  // true - non-empty string
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Equals, Like};
use Spiral\\DataGrid\\Specification\\Value\\StringValue;
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class UserSchema extends GridSchema
{
    public function __construct()
    {
        // Required fields (no empty strings)
        $this->addFilter('name', new Like('full_name', new StringValue()));
        $this->addFilter('email', new Equals('email', new StringValue()));
        $this->addFilter('username', new Equals('username', new StringValue()));
        
        // Optional fields (allow empty strings)
        $this->addFilter('bio', new Like('bio', new StringValue(true)));
        $this->addFilter('company', new Equals('company', new StringValue(true)));
        
        // Sortable string fields
        $this->addSorter('name', new Sorter('full_name'));
        $this->addSorter('email', new Sorter('email'));
    }
}

// Usage: ?filter[name]=John&filter[bio]=&sort[name]=asc
```

### IntValue

Validates and converts integer values from numeric input. Includes special handling for PHP 8+ whitespace compatibility:

```php
use Spiral\\DataGrid\\Specification\\Value\\IntValue;

// User ID filtering
$userIdValue = new IntValue();
$userFilter = new Equals('user_id', $userIdValue);
$result = $userFilter->withValue('123'); // Converts to int 123

// Age range filtering
$ageValue = new IntValue();
$ageFilter = new Between('age', $ageValue);
$result = $ageFilter->withValue([18, 65]); // Age between 18-65

// Input validation examples
$intValue = new IntValue();

// Valid inputs
$intValue->accepts(123);           // true - integer
$intValue->accepts('123');         // true - numeric string
$intValue->accepts('-456');        // true - negative integer string
$intValue->accepts('0');           // true - zero string
$intValue->accepts('');            // true - empty string

// Invalid inputs (PHP 8+ whitespace compatibility)
$intValue->accepts('  123  ');     // false - whitespace framed
$intValue->accepts('123abc');      // false - mixed content
$intValue->accepts('12.5');        // false - decimal
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Equals, Gte, Lte, Between};
use Spiral\\DataGrid\\Specification\\Value\\IntValue;
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class ProductSchema extends GridSchema
{
    public function __construct()
    {
        // ID filters
        $this->addFilter('id', new Equals('id', new IntValue()));
        $this->addFilter('category_id', new Equals('category_id', new IntValue()));
        $this->addFilter('user_id', new Equals('created_by', new IntValue()));
        
        // Range filters
        $this->addFilter('min_stock', new Gte('stock_quantity', new IntValue()));
        $this->addFilter('max_stock', new Lte('stock_quantity', new IntValue()));
        $this->addFilter('stock_range', new Between('stock_quantity', new IntValue()));
        
        // Age and quantity filters
        $this->addFilter('min_age', new Gte('age_months', new IntValue()));
        $this->addFilter('rating', new Equals('rating', new IntValue()));
        
        // Sortable integer fields
        $this->addSorter('stock', new Sorter('stock_quantity'));
        $this->addSorter('rating', new Sorter('rating'));
    }
}

// Usage: ?filter[min_stock]=10&filter[stock_range]=5,50&filter[rating]=4&sort[stock]=desc
```

### FloatValue

Validates and converts floating-point number values:

```php
use Spiral\\DataGrid\\Specification\\Value\\FloatValue;

// Product price filtering
$priceValue = new FloatValue();
$priceFilter = new Between('price', $priceValue);
$result = $priceFilter->withValue([10.99, 199.99]); // Price range

// Geographic coordinate filtering
$latitudeValue = new FloatValue();
$locationFilter = new Between('latitude', $latituValue);
$result = $locationFilter->withValue([40.7128, 40.7614]); // NYC area

// Performance metrics
$responseTimeValue = new FloatValue();
$performanceFilter = new Lt('response_time_ms', $responseTimeValue);
$result = $performanceFilter->withValue(0.250); // Under 250ms
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Between, Gte, Lte, Lt};
use Spiral\\DataGrid\\Specification\\Value\\FloatValue;
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class LocationSchema extends GridSchema
{
    public function __construct()
    {
        // Geographic coordinates
        $this->addFilter('latitude', new Between('latitude', new FloatValue()));
        $this->addFilter('longitude', new Between('longitude', new FloatValue()));
        $this->addFilter('min_lat', new Gte('latitude', new FloatValue()));
        $this->addFilter('max_lat', new Lte('latitude', new FloatValue()));
        
        // Performance metrics
        $this->addFilter('response_time', new Lt('response_time_ms', new FloatValue()));
        $this->addFilter('cpu_usage', new Lte('cpu_usage_percent', new FloatValue()));
        
        // Financial data
        $this->addFilter('price_range', new Between('price', new FloatValue()));
        $this->addFilter('discount', new Gte('discount_percentage', new FloatValue()));
        
        // Temperature and measurements
        $this->addFilter('temperature', new Between('temperature_celsius', new FloatValue()));
        $this->addFilter('weight', new Lte('weight_kg', new FloatValue()));
        
        // Sortable float fields
        $this->addSorter('price', new Sorter('price'));
        $this->addSorter('rating', new Sorter('average_rating'));
    }
}

// Usage: ?filter[latitude]=-90,90&filter[price_range]=10.99,199.99&filter[cpu_usage]=85.5
```

### NumericValue

Validates and converts numeric values (integers or floats) with automatic type detection:

```php
use Spiral\\DataGrid\\Specification\\Value\\NumericValue;

// General price filtering (handles $10 and $10.99)
$priceValue = new NumericValue();
$priceFilter = new Between('price', $priceValue);
$result = $priceFilter->withValue([10, 99.99]); // Handles both int and float

// API parameter that can be int or float
$thresholdValue = new NumericValue();
$apiFilter = new Gte('threshold', $thresholdValue);
$result = $apiFilter->withValue('85');    // Returns int 85
$result = $apiFilter->withValue('85.5');  // Returns float 85.5

// Conversion examples (automatic type detection)
$numericValue = new NumericValue();

$numericValue->convert(10);           // Returns int 10
$numericValue->convert('10');         // Returns int 10
$numericValue->convert(10.5);         // Returns float 10.5
$numericValue->convert('10.5');       // Returns float 10.5
$numericValue->convert('10.0');       // Returns int 10 (whole number)
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Between, Gte, Lte, Equals};
use Spiral\\DataGrid\\Specification\\Value\\NumericValue;
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class EcommerceSchema extends GridSchema
{
    public function __construct()
    {
        // Flexible price filtering (accepts both 10 and 10.99)
        $this->addFilter('price', new Between('price', new NumericValue()));
        $this->addFilter('min_price', new Gte('price', new NumericValue()));
        $this->addFilter('max_price', new Lte('price', new NumericValue()));
        
        // Weight and dimensions (can be precise or approximate)
        $this->addFilter('weight', new Between('weight', new NumericValue()));
        $this->addFilter('height', new Lte('height_cm', new NumericValue()));
        $this->addFilter('width', new Lte('width_cm', new NumericValue()));
        
        // Ratings and scores (can be 4 or 4.5)
        $this->addFilter('rating', new Gte('average_rating', new NumericValue()));
        $this->addFilter('score', new Between('score', new NumericValue()));
        
        // Financial amounts
        $this->addFilter('discount', new Gte('discount_amount', new NumericValue()));
        $this->addFilter('tax', new Equals('tax_rate', new NumericValue()));
        
        // Sortable numeric fields
        $this->addSorter('price', new Sorter('price'));
        $this->addSorter('rating', new Sorter('average_rating'));
        $this->addSorter('weight', new Sorter('weight'));
    }
}

// Usage: ?filter[price]=10,199.99&filter[rating]=4.5&filter[weight]=0.5,10&sort[price]=asc
```

### BoolValue

Validates and converts boolean values from various input formats:

```php
use Spiral\\DataGrid\\Specification\\Value\\BoolValue;

// Feature toggle filter
$featureFilter = new Equals('feature_enabled', new BoolValue());
$result = $featureFilter->withValue('true'); // Enables feature
$result = $featureFilter->withValue('0'); // Disables feature

// HTML form checkbox processing
$publishedValue = new BoolValue();
// Checkbox checked: value = '1' or 'on'
// Checkbox unchecked: value not present or '0'
$publishedValue->accepts('1'); // true
$publishedValue->convert('1'); // Returns true
$publishedValue->accepts('0'); // true
$publishedValue->convert('0'); // Returns false

// Accepted input formats:
// - Actual booleans: true, false
// - Numeric strings: '0' (false), '1' (true)
// - Text strings: 'true' (true), 'false' (false) - case insensitive
// - Numeric values: 0 (false), 1 (true)
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\Equals;
use Spiral\\DataGrid\\Specification\\Value\\BoolValue;
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class ContentSchema extends GridSchema
{
    public function __construct()
    {
        // Publication status
        $this->addFilter('published', new Equals('is_published', new BoolValue()));
        $this->addFilter('featured', new Equals('is_featured', new BoolValue()));
        $this->addFilter('archived', new Equals('is_archived', new BoolValue()));
        
        // User preferences
        $this->addFilter('notifications', new Equals('email_notifications', new BoolValue()));
        $this->addFilter('public_profile', new Equals('is_public', new BoolValue()));
        $this->addFilter('verified', new Equals('is_verified', new BoolValue()));
        
        // Feature flags
        $this->addFilter('premium', new Equals('has_premium', new BoolValue()));
        $this->addFilter('beta_access', new Equals('beta_enabled', new BoolValue()));
        
        // Product flags
        $this->addFilter('on_sale', new Equals('on_sale', new BoolValue()));
        $this->addFilter('in_stock', new Equals('in_stock', new BoolValue()));
        
        // Sortable boolean fields (true values first)
        $this->addSorter('featured', new Sorter('is_featured'));
        $this->addSorter('published', new Sorter('is_published'));
    }
}

// Usage: ?filter[published]=1&filter[featured]=true&filter[on_sale]=false&sort[featured]=desc
```

### ScalarValue

Validates scalar values (string, int, float, bool) with optional empty string handling:

```php
use Spiral\\DataGrid\\Specification\\Value\\ScalarValue;

// Mixed form input processing
$formValue = new ScalarValue();
$formFilter = new Equals('user_input', $formValue);
$result = $formFilter->withValue('text input');  // Valid - string
$result = $formFilter->withValue(123);           // Valid - integer
$result = $formFilter->withValue(45.67);         // Valid - float
$result = $formFilter->withValue(true);          // Valid - boolean

// Allow empty strings option
$allowEmptyValue = new ScalarValue(true);  // Allow empty strings
$strictValue = new ScalarValue(false);     // Reject empty strings (default)

$allowEmptyValue->accepts('');     // true - empty allowed
$strictValue->accepts('');         // false - empty not allowed
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Equals, Like};
use Spiral\\DataGrid\\Specification\\Value\\ScalarValue;

class ConfigurationSchema extends GridSchema
{
    public function __construct()
    {
        // Configuration values that can be strings, numbers, or booleans
        $this->addFilter('setting_value', new Equals('setting_value', new ScalarValue()));
        $this->addFilter('config_param', new Equals('config_param', new ScalarValue()));
        
        // API parameters with flexible types
        $this->addFilter('api_param', new Equals('api_parameter', new ScalarValue()));
        $this->addFilter('threshold', new Equals('threshold_value', new ScalarValue()));
        
        // Search fields that accept various input types
        $this->addFilter('search', new Like('searchable_content', new ScalarValue()));
        
        // User-defined fields (allow empty for optional fields)
        $this->addFilter('custom_field', new Equals('custom_field', new ScalarValue(true)));
        $this->addFilter('notes', new Like('notes', new ScalarValue(true)));
        
        // Metadata fields
        $this->addFilter('metadata_key', new Equals('metadata_key', new ScalarValue()));
        $this->addFilter('tag_value', new Equals('tag_value', new ScalarValue()));
    }
}

// Usage: ?filter[setting_value]=debug&filter[threshold]=85.5&filter[api_param]=true&filter[custom_field]=
```

## Array and Collection Values

### ArrayValue

Validates and converts arrays where each element matches a base value type:

```php
use Spiral\\DataGrid\\Specification\\Value\\ArrayValue;

// Multi-select category filter
$categoryFilter = new InArray('category', new ArrayValue(new StringValue()));
$result = $categoryFilter->withValue(['electronics', 'computers', 'mobile']);
// Validates each category name is a valid string

// Batch user ID processing
$userIdsValue = new ArrayValue(new IntValue());
$userIdsValue->accepts([1, 2, 3, 4]); // true - all integers
$userIdsValue->accepts(['1', '2', '3']); // true - converts strings to integers
$userIdsValue->accepts([1, 'invalid', 3]); // false - 'invalid' can't be integer

// Permission management
$permissionValue = new ArrayValue(new EnumValue(
    new StringValue(), 'read', 'write', 'admin', 'moderate'
));
$roleFilter = new InArray('permissions', $permissionValue);
$result = $roleFilter->withValue(['read', 'write']); // Valid permissions
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{InArray, NotInArray};
use Spiral\\DataGrid\\Specification\\Value\\{ArrayValue, StringValue, IntValue};

class MultiSelectSchema extends GridSchema
{
    public function __construct()
    {
        // Multi-select category filtering
        $this->addFilter('categories', new InArray('category_name', new ArrayValue(new StringValue())));
        
        // Multiple user IDs
        $this->addFilter('user_ids', new InArray('user_id', new ArrayValue(new IntValue())));
        
        // Tag filtering
        $this->addFilter('tags', new InArray('tag', new ArrayValue(new StringValue())));
        $this->addFilter('exclude_tags', new NotInArray('tag', new ArrayValue(new StringValue())));
        
        // Department selection
        $this->addFilter('departments', new InArray('department_id', new ArrayValue(new IntValue())));
        
        // Status filtering (multiple allowed)
        $this->addFilter('statuses', new InArray('status', new ArrayValue(new StringValue())));
        
        // Geographic regions
        $this->addFilter('regions', new InArray('region_code', new ArrayValue(new StringValue())));
        
        // Product SKUs
        $this->addFilter('skus', new InArray('sku', new ArrayValue(new StringValue())));
    }
}

// Usage: ?filter[categories]=electronics,computers&filter[user_ids]=1,5,10&filter[tags]=php,tutorial
```

### EnumValue

Validates values against a predefined set of allowed values (enumeration):

```php
use Spiral\\DataGrid\\Specification\\Value\\EnumValue;

// Order status validation
$statusValue = new EnumValue(
    new StringValue(),
    'pending', 'processing', 'shipped', 'delivered', 'cancelled'
);
$orderFilter = new Equals('status', $statusValue);
$result = $orderFilter->withValue('shipped'); // Valid
$result = $orderFilter->withValue('invalid'); // Rejected

// Priority level validation
$priorityValue = new EnumValue(
    new IntValue(),
    1, 2, 3, 4, 5 // 1=urgent, 5=low
);
$taskFilter = new Equals('priority', $priorityValue);
$result = $taskFilter->withValue('3'); // Converts to int 3, valid
$result = $taskFilter->withValue(6); // Invalid priority
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\Equals;
use Spiral\\DataGrid\\Specification\\Value\\{EnumValue, StringValue, IntValue};
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class OrderSchema extends GridSchema
{
    public function __construct()
    {
        // Order status (predefined values only)
        $this->addFilter('status', new Equals('order_status', new EnumValue(
            new StringValue(),
            'pending', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded'
        )));
        
        // Priority levels
        $this->addFilter('priority', new Equals('priority', new EnumValue(
            new IntValue(),
            1, 2, 3, 4, 5  // 1=urgent, 5=low
        )));
        
        // Payment methods
        $this->addFilter('payment_method', new Equals('payment_method', new EnumValue(
            new StringValue(),
            'credit_card', 'paypal', 'bank_transfer', 'cash'
        )));
        
        // Shipping types
        $this->addFilter('shipping', new Equals('shipping_type', new EnumValue(
            new StringValue(),
            'standard', 'express', 'overnight', 'pickup'
        )));
        
        // User roles
        $this->addFilter('role', new Equals('user_role', new EnumValue(
            new StringValue(),
            'customer', 'premium', 'vip', 'admin'
        )));
        
        // Content ratings
        $this->addFilter('rating', new Equals('content_rating', new EnumValue(
            new StringValue(),
            'G', 'PG', 'PG-13', 'R', 'NC-17'
        )));
        
        // Sortable enum fields
        $this->addSorter('status', new Sorter('order_status'));
        $this->addSorter('priority', new Sorter('priority'));
    }
}

// Usage: ?filter[status]=shipped&filter[priority]=1&filter[payment_method]=credit_card
```

### IntersectValue

Validates arrays where at least one element matches any value from a predefined enum set:

```php
use Spiral\\DataGrid\\Specification\\Value\\IntersectValue;

// Tag-based content filtering
$tagIntersectValue = new IntersectValue(
    new StringValue(),
    'programming', 'tutorial', 'beginner', 'advanced'
);
$contentFilter = new InArray('tags', $tagIntersectValue);
$result = $contentFilter->withValue(['cooking', 'programming', 'music']);
// Matches because 'programming' is in the allowed set

// Skill-based job matching
$skillIntersectValue = new IntersectValue(
    new StringValue(),
    'php', 'javascript', 'python', 'java', 'react'
);
$jobFilter = new InArray('required_skills', $skillIntersectValue);
$result = $jobFilter->withValue(['php', 'mysql', 'linux']);
// Matches because candidate has 'php' which is required
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\InArray;
use Spiral\\DataGrid\\Specification\\Value\\{IntersectValue, StringValue};

class JobMatchingSchema extends GridSchema
{
    public function __construct()
    {
        // Skills matching (candidate needs at least one required skill)
        $this->addFilter('required_skills', new InArray('skills', new IntersectValue(
            new StringValue(),
            'php', 'javascript', 'python', 'java', 'react', 'vue', 'angular', 'mysql', 'postgresql'
        )));
        
        // Content tags (show content with any matching tag)
        $this->addFilter('content_tags', new InArray('tags', new IntersectValue(
            new StringValue(),
            'programming', 'tutorial', 'beginner', 'advanced', 'web-development', 'mobile'
        )));
        
        // Product features (has any of the desired features)
        $this->addFilter('features', new InArray('features', new IntersectValue(
            new StringValue(),
            'bluetooth', 'wifi', 'gps', 'camera', 'nfc', '5g'
        )));
        
        // Language support (supports any of the needed languages)
        $this->addFilter('languages', new InArray('supported_languages', new IntersectValue(
            new StringValue(),
            'en', 'es', 'fr', 'de', 'it', 'pt', 'zh', 'ja'
        )));
        
        // Target audience (matches any target demographic)
        $this->addFilter('audience', new InArray('target_audience', new IntersectValue(
            new StringValue(),
            'teenagers', 'adults', 'seniors', 'professionals', 'students'
        )));
    }
}

// Usage: ?filter[required_skills]=php,mysql,linux&filter[content_tags]=cooking,programming,music
// Will match if candidate has php OR mysql OR linux (at least one)
```

### SubsetValue

Validates arrays where ALL elements must exist in the predefined enum set:

```php
use Spiral\\DataGrid\\Specification\\Value\\SubsetValue;

// Permission system (user must have only valid permissions)
$permissionSubsetValue = new SubsetValue(
    new StringValue(),
    'read', 'write', 'admin', 'moderate', 'delete'
);
$userFilter = new InArray('permissions', $permissionSubsetValue);
$result = $userFilter->withValue(['read', 'write']);        // Valid - all permissions valid
$result = $userFilter->withValue(['read', 'invalid']);      // Invalid - contains invalid permission

// Required skills validation (all must be from approved list)
$skillSubsetValue = new SubsetValue(
    new StringValue(),
    'php', 'javascript', 'python', 'react', 'vue', 'mysql'
);
$jobFilter = new InArray('required_skills', $skillSubsetValue);
$result = $jobFilter->withValue(['php', 'mysql']);          // Valid - both skills approved
$result = $jobFilter->withValue(['php', 'cobol']);          // Invalid - 'cobol' not approved
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\InArray;
use Spiral\\DataGrid\\Specification\\Value\\{SubsetValue, StringValue, IntValue};

class SecuritySchema extends GridSchema
{
    public function __construct()
    {
        // User permissions (all must be valid)
        $this->addFilter('permissions', new InArray('user_permissions', new SubsetValue(
            new StringValue(),
            'read', 'write', 'admin', 'moderate', 'delete', 'publish', 'edit'
        )));
        
        // Security clearance levels (all must be from approved list)
        $this->addFilter('clearance', new InArray('clearance_levels', new SubsetValue(
            new IntValue(),
            1, 2, 3, 4, 5  // Valid clearance levels
        )));
        
        // Content categories (all must be from approved taxonomy)
        $this->addFilter('categories', new InArray('content_categories', new SubsetValue(
            new StringValue(),
            'technology', 'business', 'education', 'entertainment', 'sports', 'health'
        )));
        
        // API scopes (all requested scopes must be valid)
        $this->addFilter('scopes', new InArray('api_scopes', new SubsetValue(
            new StringValue(),
            'read:profile', 'write:profile', 'read:posts', 'write:posts', 'delete:posts'
        )));
        
        // Product features (all features must be available)
        $this->addFilter('supported_features', new InArray('features', new SubsetValue(
            new StringValue(),
            'ssl', 'backup', 'monitoring', 'analytics', 'cdn', 'firewall'
        )));
        
        // Geographic regions (all must be served)
        $this->addFilter('regions', new InArray('service_regions', new SubsetValue(
            new StringValue(),
            'us-east', 'us-west', 'eu-central', 'asia-pacific', 'australia'
        )));
    }
}

// Usage: ?filter[permissions]=read,write&filter[scopes]=read:profile,write:profile
// Will only match if ALL specified permissions/scopes are valid
```

## Date and Time Values

### DatetimeValue

Validates and converts datetime values from various flexible input formats:

```php
use Spiral\\DataGrid\\Specification\\Value\\DatetimeValue;

// Flexible date range filtering
$dateValue = new DatetimeValue();
$dateFilter = new Between('created_at', $dateValue);
$result = $dateFilter->withValue(['last week', 'today']); // Relative dates

// Accepted input formats:
// - Unix timestamps: '1640995200', 1640995200
// - ISO dates: '2024-01-15', '2024-01-15T14:30:00Z'
// - Common formats: '01/15/2024', '15-Jan-2024', 'January 15, 2024'
// - Relative expressions: 'yesterday', 'last week', '+1 day', '-30 days'
// - Time expressions: 'now', 'today', 'tomorrow'
// - Empty strings: '' (treated as valid, returns null)
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Between, Gte, Lte};
use Spiral\\DataGrid\\Specification\\Value\\DatetimeValue;
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class EventSchema extends GridSchema
{
    public function __construct()
    {
        // Flexible date filtering
        $this->addFilter('created_after', new Gte('created_at', new DatetimeValue()));
        $this->addFilter('created_before', new Lte('created_at', new DatetimeValue()));
        $this->addFilter('created_range', new Between('created_at', new DatetimeValue()));
        
        // Event date filtering
        $this->addFilter('event_date', new Between('event_date', new DatetimeValue()));
        $this->addFilter('starts_after', new Gte('start_time', new DatetimeValue()));
        $this->addFilter('ends_before', new Lte('end_time', new DatetimeValue()));
        
        // Publication dates
        $this->addFilter('published_after', new Gte('published_at', new DatetimeValue()));
        $this->addFilter('published_range', new Between('published_at', new DatetimeValue()));
        
        // User activity
        $this->addFilter('last_login', new Gte('last_login_at', new DatetimeValue()));
        $this->addFilter('registered_after', new Gte('registered_at', new DatetimeValue()));
        
        // Sortable date fields
        $this->addSorter('created', new Sorter('created_at'));
        $this->addSorter('event_date', new Sorter('event_date'));
        $this->addSorter('published', new Sorter('published_at'));
    }
}

// Usage: ?filter[created_after]=last week&filter[event_date]=2024-01-15,2024-12-31
//        ?filter[published_after]=yesterday&filter[last_login]=2024-01-01
```

### DatetimeFormatValue

Validates and converts datetime strings using a specific format:

```php
use Spiral\\DataGrid\\Specification\\Value\\DatetimeFormatValue;

// HTML date input processing (Y-m-d format)
$dateValue = new DatetimeFormatValue('Y-m-d');
$dateFilter = new Gte('start_date', $dateValue);
$result = $dateFilter->withValue('2024-01-15'); // Valid HTML date input

// US date format processing (m/d/Y)
$usDateValue = new DatetimeFormatValue('m/d/Y');
$eventFilter = new Between('event_date', $usDateValue);
$result = $eventFilter->withValue('01/15/2024'); // January 15, 2024

// API datetime with conversion to different format
$apiDateValue = new DatetimeFormatValue('Y-m-d H:i:s', 'Y-m-d');
$logFilter = new Equals('created_at', $apiDateValue);
$result = $logFilter->withValue('2024-01-15 14:30:00');
// Converts to '2024-01-15' (date only)
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Between, Gte, Lte, Equals};
use Spiral\\DataGrid\\Specification\\Value\\DatetimeFormatValue;
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class StrictDateSchema extends GridSchema
{
    public function __construct()
    {
        // HTML date inputs (Y-m-d format only)
        $this->addFilter('start_date', new Gte('start_date', new DatetimeFormatValue('Y-m-d')));
        $this->addFilter('end_date', new Lte('end_date', new DatetimeFormatValue('Y-m-d')));
        $this->addFilter('birth_date', new Equals('birth_date', new DatetimeFormatValue('Y-m-d')));
        
        // US date format (m/d/Y)
        $this->addFilter('us_date', new Between('event_date', new DatetimeFormatValue('m/d/Y')));
        
        // European date format (d.m.Y)
        $this->addFilter('eu_date', new Between('created_date', new DatetimeFormatValue('d.m.Y')));
        
        // Time-only filters (H:i:s)
        $this->addFilter('start_time', new Gte('start_time', new DatetimeFormatValue('H:i:s')));
        $this->addFilter('end_time', new Lte('end_time', new DatetimeFormatValue('H:i:s')));
        
        // ISO datetime format
        $this->addFilter('timestamp', new Gte('timestamp', new DatetimeFormatValue('Y-m-d\\TH:i:s\\Z')));
        
        // Month/Year reporting
        $this->addFilter('report_month', new Equals('report_period', new DatetimeFormatValue('Y-m')));
        
        // Custom readable format with conversion
        $this->addFilter('display_date', new Equals('display_date', 
            new DatetimeFormatValue('Y-m-d', 'F j, Y')  // Input: 2024-01-15, Output: January 15, 2024
        ));
        
        // Sortable date fields
        $this->addSorter('date', new Sorter('start_date'));
    }
}

// Usage: ?filter[start_date]=2024-01-15&filter[us_date]=01/15/2024&filter[start_time]=14:30:00
```

## Comparison Values

### PositiveValue

Validates values that are strictly greater than zero:

```php
use Spiral\\DataGrid\\Specification\\Value\\PositiveValue;

// Product price validation
$priceValue = new PositiveValue(new NumericValue());
$productFilter = new Gte('price', $priceValue);
$result = $productFilter->withValue(19.99); // Valid - positive price
$result = $productFilter->withValue(0);     // Invalid - zero price
$result = $productFilter->withValue(-5);    // Invalid - negative price

// Order quantity validation
$quantityValue = new PositiveValue(new IntValue());
$orderFilter = new Gte('quantity', $quantityValue);
$result = $orderFilter->withValue(3);  // Valid - ordering 3 items
$result = $orderFilter->withValue(0);  // Invalid - zero quantity
$result = $orderFilter->withValue(-1); // Invalid - negative quantity
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Gte, Lte, Between, Equals};
use Spiral\\DataGrid\\Specification\\Value\\{PositiveValue, NumericValue, IntValue};
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class PositiveValuesSchema extends GridSchema
{
    public function __construct()
    {
        // Product pricing (must be positive)
        $this->addFilter('min_price', new Gte('price', new PositiveValue(new NumericValue())));
        $this->addFilter('max_price', new Lte('price', new PositiveValue(new NumericValue())));
        $this->addFilter('price_range', new Between('price', new PositiveValue(new NumericValue())));
        
        // Order quantities (must be positive)
        $this->addFilter('quantity', new Equals('quantity', new PositiveValue(new IntValue())));
        $this->addFilter('min_quantity', new Gte('order_quantity', new PositiveValue(new IntValue())));
        
        // Performance metrics (positive values only)
        $this->addFilter('response_time', new Lte('response_time_ms', new PositiveValue(new NumericValue())));
        $this->addFilter('throughput', new Gte('requests_per_second', new PositiveValue(new NumericValue())));
        
        // User ratings (1-5 stars, must be positive)
        $this->addFilter('rating', new Gte('rating', new PositiveValue(new IntValue())));
        
        // Financial amounts (must be positive)
        $this->addFilter('budget', new Lte('budget_amount', new PositiveValue(new NumericValue())));
        $this->addFilter('revenue', new Gte('monthly_revenue', new PositiveValue(new NumericValue())));
        
        // Weight and dimensions (must be positive)
        $this->addFilter('weight', new Between('weight_kg', new PositiveValue(new NumericValue())));
        $this->addFilter('height', new Lte('height_cm', new PositiveValue(new NumericValue())));
        
        // Sortable positive fields
        $this->addSorter('price', new Sorter('price'));
        $this->addSorter('rating', new Sorter('rating'));
    }
}

// Usage: ?filter[min_price]=10.99&filter[quantity]=5&filter[rating]=4&filter[weight]=0.1,50
// Note: Zero values will be rejected, only positive values accepted
```

### NegativeValue

Validates values that are strictly less than zero:

```php
use Spiral\\DataGrid\\Specification\\Value\\NegativeValue;

// Account balance deficit tracking
$deficitValue = new NegativeValue(new NumericValue());
$accountFilter = new Lt('account_balance', $deficitValue);
$result = $accountFilter->withValue(-150.50); // Valid deficit
$result = $accountFilter->withValue(100);     // Invalid - positive balance

// Below-freezing temperature monitoring
$belowFreezing = new NegativeValue(new FloatValue());
$weatherFilter = new Lt('temperature_celsius', $belowFreezing);
$result = $weatherFilter->withValue(-5.2);  // Valid - below freezing
$result = $weatherFilter->withValue(0);     // Invalid - not negative
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Lt, Lte, Between, Equals};
use Spiral\\DataGrid\\Specification\\Value\\{NegativeValue, NumericValue, IntValue, FloatValue};
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class NegativeValuesSchema extends GridSchema
{
    public function __construct()
    {
        // Financial deficits and losses
        $this->addFilter('deficit', new Lt('account_balance', new NegativeValue(new NumericValue())));
        $this->addFilter('loss_amount', new Equals('quarterly_loss', new NegativeValue(new NumericValue())));
        $this->addFilter('debt_range', new Between('debt_amount', new NegativeValue(new NumericValue())));
        
        // Temperature monitoring (below zero)
        $this->addFilter('freezing_temp', new Lt('temperature_celsius', new NegativeValue(new FloatValue())));
        $this->addFilter('sub_zero_range', new Between('min_temperature', new NegativeValue(new FloatValue())));
        
        // Performance penalties (negative scores)
        $this->addFilter('penalty', new Equals('penalty_points', new NegativeValue(new IntValue())));
        $this->addFilter('score_decline', new Lt('performance_change', new NegativeValue(new NumericValue())));
        
        // Elevation below sea level
        $this->addFilter('below_sea_level', new Lt('elevation_meters', new NegativeValue(new IntValue())));
        
        // Voltage measurements (negative voltages)
        $this->addFilter('negative_voltage', new Between('voltage', new NegativeValue(new FloatValue())));
        
        // Geographic coordinates (Southern/Western hemispheres)
        $this->addFilter('south_latitude', new Lt('latitude', new NegativeValue(new FloatValue())));
        $this->addFilter('west_longitude', new Lt('longitude', new NegativeValue(new FloatValue())));
        
        // Sortable negative fields
        $this->addSorter('balance', new Sorter('account_balance'));
        $this->addSorter('temperature', new Sorter('temperature_celsius'));
    }
}

// Usage: ?filter[deficit]=-500.00&filter[freezing_temp]=-10.5&filter[penalty]=-25
// Note: Zero and positive values will be rejected, only negative values accepted
```

### NonNegativeValue

Validates values that are greater than or equal to zero:

```php
use Spiral\\DataGrid\\Specification\\Value\\NonNegativeValue;

// Inventory stock validation
$stockValue = new NonNegativeValue(new IntValue());
$inventoryFilter = new Gte('stock_quantity', $stockValue);
$result = $inventoryFilter->withValue(0);   // Valid - out of stock
$result = $inventoryFilter->withValue(100); // Valid - in stock
$result = $inventoryFilter->withValue(-5);  // Invalid - negative stock not allowed

// Account balance validation (basic accounts)
$balanceValue = new NonNegativeValue(new NumericValue());
$accountFilter = new Gte('account_balance', $balanceValue);
$result = $accountFilter->withValue(0.00);     // Valid - zero balance
$result = $accountFilter->withValue(1000.50);  // Valid - positive balance
$result = $accountFilter->withValue(-100.00);  // Invalid - overdraft not allowed
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Gte, Lte, Between, Equals};
use Spiral\\DataGrid\\Specification\\Value\\{NonNegativeValue, NumericValue, IntValue, FloatValue};
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class NonNegativeSchema extends GridSchema
{
    public function __construct()
    {
        // Inventory management (zero or positive stock)
        $this->addFilter('stock', new Gte('stock_quantity', new NonNegativeValue(new IntValue())));
        $this->addFilter('min_stock', new Gte('minimum_stock', new NonNegativeValue(new IntValue())));
        $this->addFilter('available_stock', new Between('available_quantity', new NonNegativeValue(new IntValue())));
        
        // Age validation (0 or positive years)
        $this->addFilter('age', new Between('age_years', new NonNegativeValue(new IntValue())));
        $this->addFilter('min_age', new Gte('age', new NonNegativeValue(new IntValue())));
        
        // Product pricing (free or paid items)
        $this->addFilter('price', new Between('price', new NonNegativeValue(new NumericValue())));
        $this->addFilter('discount', new Equals('discount_amount', new NonNegativeValue(new NumericValue())));
        
        // Performance metrics (zero or positive)
        $this->addFilter('response_time', new Lte('response_time_ms', new NonNegativeValue(new FloatValue())));
        $this->addFilter('error_rate', new Lte('error_percentage', new NonNegativeValue(new FloatValue())));
        
        // Distance measurements (zero or positive)
        $this->addFilter('distance', new Between('distance_km', new NonNegativeValue(new FloatValue())));
        $this->addFilter('radius', new Lte('search_radius', new NonNegativeValue(new FloatValue())));
        
        // Account balances (no overdraft allowed)
        $this->addFilter('balance', new Gte('account_balance', new NonNegativeValue(new NumericValue())));
        $this->addFilter('credit_limit', new Lte('available_credit', new NonNegativeValue(new NumericValue())));
        
        // Sortable non-negative fields
        $this->addSorter('stock', new Sorter('stock_quantity'));
        $this->addSorter('price', new Sorter('price'));
        $this->addSorter('age', new Sorter('age_years'));
    }
}

// Usage: ?filter[stock]=0,1000&filter[price]=0,99.99&filter[age]=18,65&filter[balance]=0
// Note: Negative values will be rejected, zero and positive values accepted
```

### NonPositiveValue

Validates values that are less than or equal to zero:

```php
use Spiral\\DataGrid\\Specification\\Value\\NonPositiveValue;

// Temperature monitoring (freezing and below)
$freezingValue = new NonPositiveValue(new FloatValue());
$weatherFilter = new Lte('temperature_celsius', $freezingValue);
$result = $freezingValue->withValue(0.0);   // Valid - freezing point
$result = $freezingValue->withValue(-5.2);  // Valid - below freezing
$result = $freezingValue->withValue(10.0);  // Invalid - above freezing

// Performance penalty tracking
$penaltyValue = new NonPositiveValue(new IntValue());
$gameFilter = new Lte('score_adjustment', $penaltyValue);
$result = $penaltyValue->withValue(0);   // Valid - no adjustment
$result = $penaltyValue->withValue(-10); // Valid - penalty points
$result = $penaltyValue->withValue(5);   // Invalid - bonus points (positive)
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Lte, Lt, Between, Equals};
use Spiral\\DataGrid\\Specification\\Value\\{NonPositiveValue, NumericValue, IntValue, FloatValue};
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class NonPositiveSchema extends GridSchema
{
    public function __construct()
    {
        // Temperature monitoring (freezing and below)
        $this->addFilter('freezing_temp', new Lte('temperature_celsius', new NonPositiveValue(new FloatValue())));
        $this->addFilter('sub_freezing', new Lt('min_temperature', new NonPositiveValue(new FloatValue())));
        $this->addFilter('temp_range', new Between('temperature_range', new NonPositiveValue(new FloatValue())));
        
        // Account monitoring (zero balances and debts)
        $this->addFilter('balance', new Lte('account_balance', new NonPositiveValue(new NumericValue())));
        $this->addFilter('debt_amount', new Equals('outstanding_debt', new NonPositiveValue(new NumericValue())));
        
        // Performance tracking (zero and declining metrics)
        $this->addFilter('growth_rate', new Lte('quarterly_growth', new NonPositiveValue(new FloatValue())));
        $this->addFilter('performance_change', new Lte('performance_delta', new NonPositiveValue(new NumericValue())));
        
        // Score adjustments (penalties and neutral)
        $this->addFilter('score_penalty', new Lte('score_adjustment', new NonPositiveValue(new IntValue())));
        $this->addFilter('rating_change', new Lte('rating_change', new NonPositiveValue(new IntValue())));
        
        // Elevation (sea level and below)
        $this->addFilter('elevation', new Lte('elevation_meters', new NonPositiveValue(new IntValue())));
        $this->addFilter('depth', new Equals('depth_below_surface', new NonPositiveValue(new IntValue())));
        
        // Voltage measurements (ground and negative)
        $this->addFilter('voltage', new Lte('voltage_level', new NonPositiveValue(new FloatValue())));
        
        // Financial tracking (break-even and losses)
        $this->addFilter('profit', new Lte('quarterly_profit', new NonPositiveValue(new NumericValue())));
        $this->addFilter('roi', new Lte('return_on_investment', new NonPositiveValue(new FloatValue())));
        
        // Sortable non-positive fields
        $this->addSorter('temperature', new Sorter('temperature_celsius'));
        $this->addSorter('balance', new Sorter('account_balance'));
        $this->addSorter('performance', new Sorter('performance_delta'));
    }
}

// Usage: ?filter[freezing_temp]=-10.5&filter[balance]=-500.00&filter[growth_rate]=0
// Note: Positive values will be rejected, zero and negative values accepted
```

## Range and Pattern Values

### RangeValue

Validates values that fall within a specified range between two boundaries:

```php
use Spiral\\DataGrid\\Specification\\Value\\RangeValue;
use Spiral\\DataGrid\\Specification\\Value\\RangeValue\\Boundary;

// Age range validation (18-65 inclusive)
$ageValue = new RangeValue(
    new IntValue(),
    Boundary::including(18),    // Age >= 18
    Boundary::including(65)     // Age <= 65
);
$ageFilter = new Between('age', $ageValue);
$ageValue->accepts(25);    // true - within range
$ageValue->accepts(18);    // true - inclusive boundary
$ageValue->accepts(65);    // true - inclusive boundary
$ageValue->accepts(17);    // false - below minimum
$ageValue->accepts(66);    // false - above maximum

// Price range validation (exclusive upper bound)
$priceValue = new RangeValue(
    new NumericValue(),
    Boundary::including(10.0),  // Price >= $10.00
    Boundary::excluding(100.0)  // Price < $100.00 (not including)
);
$priceFilter = new Between('price', $priceValue);
$priceValue->accepts(10.0);    // true - inclusive minimum
$priceValue->accepts(99.99);   // true - below exclusive maximum
$priceValue->accepts(100.0);   // false - exclusive maximum

// One-sided range (minimum only)
$minimumValue = new RangeValue(
    new IntValue(),
    Boundary::including(1),     // Value >= 1
    Boundary::empty()           // No maximum limit
);
$positiveFilter = new Gte('quantity', $minimumValue);
$minimumValue->accepts(1);      // true - meets minimum
$minimumValue->accepts(1000);   // true - no maximum limit
$minimumValue->accepts(0);      // false - below minimum
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Between, Gte, Lte, Equals};
use Spiral\\DataGrid\\Specification\\Value\\{RangeValue, IntValue, FloatValue, NumericValue};
use Spiral\\DataGrid\\Specification\\Value\\RangeValue\\Boundary;
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class RestrictedRangeSchema extends GridSchema
{
    public function __construct()
    {
        // Age restrictions (18-65 inclusive)
        $this->addFilter('age', new Between('age', new RangeValue(
            new IntValue(),
            Boundary::including(18),
            Boundary::including(65)
        )));
        
        // Product pricing (exclusive upper bound)
        $this->addFilter('price', new Between('price', new RangeValue(
            new NumericValue(),
            Boundary::including(1.0),   // Minimum $1.00
            Boundary::excluding(1000.0) // Less than $1000.00
        )));
        
        // Temperature sensor range (exclusive bounds)
        $this->addFilter('temperature', new Between('sensor_temp', new RangeValue(
            new FloatValue(),
            Boundary::excluding(-40.0), // > -40°C
            Boundary::excluding(85.0)   // < 85°C
        )));
        
        // Rating system (1-5 stars inclusive)
        $this->addFilter('rating', new Between('user_rating', new RangeValue(
            new IntValue(),
            Boundary::including(1),
            Boundary::including(5)
        )));
        
        // Percentage values (0-100 inclusive)
        $this->addFilter('percentage', new Between('completion_percent', new RangeValue(
            new FloatValue(),
            Boundary::including(0.0),
            Boundary::including(100.0)
        )));
        
        // One-sided minimum (positive quantities)
        $this->addFilter('quantity', new Gte('order_quantity', new RangeValue(
            new IntValue(),
            Boundary::including(1),
            Boundary::empty()  // No maximum
        )));
        
        // One-sided maximum (budget limit)
        $this->addFilter('budget', new Lte('project_budget', new RangeValue(
            new NumericValue(),
            Boundary::empty(),      // No minimum
            Boundary::including(50000.0)
        )));
        
        // Working hours (9 AM to 5 PM)
        $this->addFilter('work_hour', new Between('schedule_hour', new RangeValue(
            new IntValue(),
            Boundary::including(9),
            Boundary::including(17)
        )));
        
        // Sortable range fields
        $this->addSorter('age', new Sorter('age'));
        $this->addSorter('price', new Sorter('price'));
        $this->addSorter('rating', new Sorter('user_rating'));
    }
}

// Usage: ?filter[age]=25,45&filter[price]=10.99,999.99&filter[rating]=4,5&filter[quantity]=1
// Values outside the defined ranges will be rejected
```

### RegexValue

Validates string or numeric values against a regular expression pattern:

```php
use Spiral\\DataGrid\\Specification\\Value\\RegexValue;

// Email validation
$emailValue = new RegexValue('/^[^@]+@[^@]+\\.[^@]+$/');
$emailFilter = new Equals('email', $emailValue);
$result = $emailFilter->withValue('user@site.com'); // Valid email
$result = $emailFilter->withValue('invalid-email');    // Invalid - no @ or domain

// Phone number validation (international format)
$phoneValue = new RegexValue('/^\\+?[1-9]\\d{1,14}$/');
$contactFilter = new Equals('phone', $phoneValue);
$result = $contactFilter->withValue('+1234567890');  // Valid international format
$result = $contactFilter->withValue('123-456-7890'); // Invalid - contains dashes

// Password strength validation
$passwordValue = new RegexValue('/^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).{8,}$/');
$userFilter = new Equals('password', $passwordValue);
$result = $userFilter->withValue('StrongPass123'); // Valid - meets all requirements
$result = $userFilter->withValue('weakpass');      // Invalid - no uppercase/digits

// SKU/Product code validation
$skuValue = new RegexValue('/^[A-Z]{2,3}-\\d{4,6}$/');
$productFilter = new Equals('sku', $skuValue);
$result = $productFilter->withValue('ABC-12345');  // Valid SKU format
$result = $productFilter->withValue('abc-123');    // Invalid - lowercase, too short
```

**GridSchema Example:**

```php
use Spiral\\DataGrid\\GridSchema;
use Spiral\\DataGrid\\Specification\\Filter\\{Equals, Like};
use Spiral\\DataGrid\\Specification\\Value\\RegexValue;
use Spiral\\DataGrid\\Specification\\Sorter\\Sorter;

class ValidationSchema extends GridSchema
{
    public function __construct()
    {
        // Email validation
        $this->addFilter('email', new Equals('email_address', new RegexValue('/^[^@]+@[^@]+\\.[^@]+$/')));
        
        // Phone number validation (US format)
        $this->addFilter('phone', new Equals('phone_number', new RegexValue('/^\\+?1?[2-9]\\d{2}[2-9]\\d{2}\\d{4}$/')));
        
        // ZIP code validation (US format)
        $this->addFilter('zip', new Equals('zip_code', new RegexValue('/^\\d{5}(-\\d{4})?$/')));
        
        // Product SKU validation
        $this->addFilter('sku', new Equals('product_sku', new RegexValue('/^[A-Z]{2,3}-\\d{4,6}$/')));
        
        // License plate validation
        $this->addFilter('license', new Equals('license_plate', new RegexValue('/^[A-Z0-9]{6,8}$/')));
        
        // Social Security Number (US format)
        $this->addFilter('ssn', new Equals('ssn', new RegexValue('/^\\d{3}-\\d{2}-\\d{4}$/')));
        
        // Credit card validation (basic format)
        $this->addFilter('card', new Equals('credit_card', new RegexValue('/^\\d{4}-\\d{4}-\\d{4}-\\d{4}$/')));
        
        // Username validation (alphanumeric + underscore)
        $this->addFilter('username', new Equals('username', new RegexValue('/^[a-zA-Z0-9_]{3,20}$/')));
        
        // Password strength (min 8 chars, uppercase, lowercase, digit)
        $this->addFilter('password', new Equals('password_hash', new RegexValue('/^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).{8,}$/')));
        
        // URL validation (basic)
        $this->addFilter('website', new Equals('website_url', new RegexValue('/^https?:\\/\\/[^\\s]+$/')));
        
        // IPv4 address validation
        $this->addFilter('ip', new Equals('ip_address', new RegexValue('/^(?:[0-9]{1,3}\\.){3}[0-9]{1,3}$/')));
        
        // Time format validation (HH:MM)
        $this->addFilter('time', new Equals('appointment_time', new RegexValue('/^([01]?[0-9]|2[0-3]):[0-5][0-9]$/')));
        
        // Sortable validated fields
        $this->addSorter('email', new Sorter('email_address'));
        $this->addSorter('sku', new Sorter('product_sku'));
    }
}

// Usage: ?filter[email]=user@example.com&filter[phone]=+12345678901&filter[sku]=ABC-12345
// Only values matching the exact regex patterns will be accepted
```

# Value Types Documentation - Completion

### UuidValue

Validates UUID strings with optional version-specific validation:

```php
use Spiral\DataGrid\Specification\Value\UuidValue;

// General UUID validation (any valid UUID)
$uuidValue = new UuidValue();
$recordFilter = new Equals('id', $uuidValue);
$result = $recordFilter->withValue('550e8400-e29b-41d4-a716-446655440000'); // Valid UUID
$result = $recordFilter->withValue('invalid-uuid-format');                  // Invalid format

// UUID v4 validation (random UUIDs) - using static factory
$uuidV4Value = UuidValue::v4();
$userFilter = new Equals('user_id', $uuidV4Value);
$result = $userFilter->withValue('550e8400-e29b-41d4-a716-446655440000'); // Valid v4 UUID
$result = $userFilter->withValue('550e8400-e29b-11d4-a716-446655440000'); // Invalid - not v4

// Available UUID masks (via static factories):
$anyUuidValue = UuidValue::valid();          // Any valid UUID
$v1UuidValue = UuidValue::v1();              // Version 1 (timestamp-based)
$v2UuidValue = UuidValue::v2();              // Version 2 (DCE security)
$v3UuidValue = UuidValue::v3();              // Version 3 (name-based MD5)
$v4UuidValue = UuidValue::v4();              // Version 4 (random)
$v5UuidValue = UuidValue::v5();              // Version 5 (name-based SHA-1)
$nilUuidValue = UuidValue::nil();            // NIL UUID (all zeros)

// Version-specific validation examples
$v4UuidValue->accepts('550e8400-e29b-41d4-a716-446655440000'); // true - v4 format
$v4UuidValue->accepts('6ba7b810-9dad-11d1-80b4-00c04fd430c8'); // false - v1 format
$nilUuidValue->accepts('00000000-0000-0000-0000-000000000000'); // true - NIL UUID
```

**GridSchema Example:**

```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\{Equals, InArray};
use Spiral\DataGrid\Specification\Value\{UuidValue, ArrayValue};
use Spiral\DataGrid\Specification\Sorter\Sorter;

class UuidSchema extends GridSchema
{
    public function __construct()
    {
        // Primary key filtering (any valid UUID)
        $this->addFilter('id', new Equals('id', UuidValue::valid()));
        
        // User ID filtering (v4 UUIDs only for security)
        $this->addFilter('user_id', new Equals('user_id', UuidValue::v4()));
        
        // Session ID filtering (v4 UUIDs)
        $this->addFilter('session_id', new Equals('session_id', UuidValue::v4()));
        
        // Tenant/Organization ID (v4 UUIDs)
        $this->addFilter('tenant_id', new Equals('tenant_id', UuidValue::v4()));
        
        // Product UUID filtering
        $this->addFilter('product_id', new Equals('product_id', UuidValue::v4()));
        
        // Parent/Child relationship filtering
        $this->addFilter('parent_id', new Equals('parent_id', UuidValue::valid()));
        
        // Category filtering (multiple UUIDs)
        $this->addFilter('category_ids', new InArray('category_id', new ArrayValue(UuidValue::v4())));
        
        // Tag filtering (multiple UUIDs)
        $this->addFilter('tag_ids', new InArray('tag_id', new ArrayValue(UuidValue::valid())));
        
        // Reference ID filtering (external system UUIDs)
        $this->addFilter('reference_id', new Equals('external_reference_id', UuidValue::valid()));
        
        // Correlation ID filtering (for distributed tracing)
        $this->addFilter('correlation_id', new Equals('correlation_id', UuidValue::v4()));
        
        // Special case: NIL UUID filtering (for null references)
        $this->addFilter('null_reference', new Equals('optional_reference_id', UuidValue::nil()));
        
        // Sortable UUID fields
        $this->addSorter('id', new Sorter('id'));
        $this->addSorter('created', new Sorter('created_at')); // Sort by creation time, not UUID
    }
}

// Usage: ?filter[id]=550e8400-e29b-41d4-a716-446655440000&filter[user_id]=6ba7b810-9dad-41d4-a716-446655440000
```

## Advanced Value Processing

### NotEmpty

Wrapper value that ensures the input is not considered "empty" before processing:

```php
use Spiral\DataGrid\Specification\Value\NotEmpty;

// Ensure search terms are not empty
$searchValue = new NotEmpty(new StringValue());
$searchFilter = new Like('title', $searchValue);

$searchValue->accepts('valid search');  // true - not empty
$searchValue->accepts('');              // false - empty string
$searchValue->accepts('   ');           // false - whitespace only
$searchValue->accepts(0);               // false - zero is considered empty
$searchValue->accepts(false);           // false - false is empty
$searchValue->accepts(null);            // false - null is empty
$searchValue->accepts([]);              // false - empty array

// Required integer field
$requiredInt = new NotEmpty(new IntValue());
$requiredInt->accepts(0);   // false - zero is empty
$requiredInt->accepts(1);   // true - non-zero integer
$requiredInt->accepts('');  // false - empty string
```

**GridSchema Example:**

```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\{Like, Equals, Gte};
use Spiral\DataGrid\Specification\Value\{NotEmpty, StringValue, IntValue, NumericValue};
use Spiral\DataGrid\Specification\Sorter\Sorter;

class RequiredFieldsSchema extends GridSchema
{
    public function __construct()
    {
        // Required search fields (no empty strings)
        $this->addFilter('search', new Like('content', new NotEmpty(new StringValue())));
        $this->addFilter('title', new Like('title', new NotEmpty(new StringValue())));
        
        // Required identification fields
        $this->addFilter('username', new Equals('username', new NotEmpty(new StringValue())));
        $this->addFilter('email', new Equals('email', new NotEmpty(new StringValue())));
        
        // Required numeric fields (zero not allowed)
        $this->addFilter('quantity', new Gte('quantity', new NotEmpty(new IntValue())));
        $this->addFilter('price', new Gte('price', new NotEmpty(new NumericValue())));
        
        // Required category (must have a value)
        $this->addFilter('category', new Equals('category_name', new NotEmpty(new StringValue())));
        
        // Required status (empty status not allowed)
        $this->addFilter('status', new Equals('status', new NotEmpty(new StringValue())));
        
        // Required foreign keys (zero IDs not allowed)
        $this->addFilter('user_id', new Equals('user_id', new NotEmpty(new IntValue())));
        $this->addFilter('department_id', new Equals('department_id', new NotEmpty(new IntValue())));
        
        // Sortable required fields
        $this->addSorter('title', new Sorter('title'));
        $this->addSorter('price', new Sorter('price'));
    }
}

// Usage: ?filter[search]=javascript&filter[quantity]=5&filter[price]=29.99
// Empty values, zeros, and null values will be rejected
```
