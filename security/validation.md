# Validation - WIP
You can validate your data using `spiral/validation` component. The component provides array based DSL to construct
complex validation chains.

The component contains of Checkers, Conditions and Validation object. The Web and GRPC bundle of spiral includes
this component by default.

## Installation and Configuration
The component can be installed:

```bash
$ composer install spiral/validation
```

To install in spiral use bootloader `Spiral\Bootloader\Security\ValidationBootloader`. 

To edit the default component configuration create and edit file `app/config/validation.php`:

```php
<?php

declare(strict_types=1);

use Spiral\Validation;

return [
    // Checkers are resolved using container and provide ability to isolate some validation rules
    // under common name and class. You can register new checkers at any moment without any
    // performance issues.
    'checkers'   => [
        'type'    => Validation\Checker\TypeChecker::class,
        'number'  => Validation\Checker\NumberChecker::class,
        'mixed'   => Validation\Checker\MixedChecker::class,
        'address' => Validation\Checker\AddressChecker::class,
        'string'  => Validation\Checker\StringChecker::class,
        'file'    => Validation\Checker\FileChecker::class,
        'image'   => Validation\Checker\ImageChecker::class,
    ],

    // Enable/disable validation conditions
    'conditions' => [
        'withAny'    => Validation\Condition\WithAnyCondition::class,
        'withoutAny' => Validation\Condition\WithoutAnyCondition::class,
        'withAll'    => Validation\Condition\WithAllCondition::class,
        'withoutAll' => Validation\Condition\WithoutAllCondition::class,
    ],

    // Aliases are only used to simplify developer life.
    'aliases'    => [
        'notEmpty'   => 'type::notEmpty',
        'required'   => 'type::notEmpty',
        'datetime'   => 'type::datetime',
        'timezone'   => 'type::timezone',
        'bool'       => 'type::boolean',
        'boolean'    => 'type::boolean',
        'cardNumber' => 'mixed::cardNumber',
        'regexp'     => 'string::regexp',
        'email'      => 'address::email',
        'url'        => 'address::url',
        'file'       => 'file::exists',
        'uploaded'   => 'file::uploaded',
        'filesize'   => 'file::size',
        'image'      => 'image::valid',
        'array'      => 'is_array',
        'callable'   => 'is_callable',
        'double'     => 'is_double',
        'float'      => 'is_float',
        'int'        => 'is_int',
        'integer'    => 'is_integer',
        'numeric'    => 'is_numeric',
        'long'       => 'is_long',
        'null'       => 'is_null',
        'object'     => 'is_object',
        'real'       => 'is_real',
        'resource'   => 'is_resource',
        'scalar'     => 'is_scalar',
        'string'     => 'is_string',
        'match'      => 'mixed::match',
    ]
];
```

Use the component via provider factory:

```php
namespace App\Controller;

use Spiral\Validation;

class HomeController
{
    public function index(Validation\ValidationInterface $validation)
    {
        $validator = $validation->validate(
        // data
            [
                'key' => null
            ],
            // rules
            [
                'key' => [
                    'notEmpty'
                ]
            ]
        );

        dump($validator instanceof Validation\Validator);

        dump($validator->isValid());
        dump($validator->withData(['key' => 'value'])->isValid());
    }
}
```

> You can use `validator` prototype property.

## The ValidationInterface
The result of `ValidationInterface`->`validate` method is `ValidationInterface`. The interface provides basic 
api to get result errors and allows to attach to new data or context (immutable).

```php
interface ValidatorInterface
{
    public function withData($data): ValidatorInterface;
    public function getValue(string $field, $default = null);
    public function withContext($context): ValidatorInterface;
    public function getContext();
    public function isValid(): bool;
    public function getErrors(): array;
}
```

The proper flow is valid:

```php
public function index(Validation\ValidationInterface $validation)
{
    $validator = $validation->validate(
        ['key' => null],
        ['key' => ['notEmpty']]
    );

    if (!$validator->isValid()) {
        dump($validator->getErrors());
    }
}
```

### Validated Data
Validator can accept any array data source, but internally it will be converted into array form (unless ArrayAccess). 

### Error Format
The validation component will always return one error and first fired error per key.

```php
[
    'name' => 'This field is required.',
    'key'  => 'Another error'
]
```

> Error messages can be localized using `spiral/translator`.

## Validation DSL
The default spiral validator accepts validation rules in a form or nested array. They key is the name of property
to be validated, where the value is array of rules to be applied to the value sequentially:

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            'notEmpty', // key must not be empty
            'string'    // must be string
        ]
    ]
);

if (!$validator->isValid()) {
    dump($validator->getErrors());
}
```

The rule in this case is name of checker method or any available PHP function which can accept `value` as first argument.

For example we can use `is_numeric` directly inside your rule:

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            'notEmpty',  // key must not be empty
            'is_numeric' // must be numeric
        ]
    ]
);
```

### Extended Declaration
In many cases you would need to declare additional rule parameters, conditions or custom error messages. This can 
be done by wrapping rule into `[]`.

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            ['notEmpty'],  // key must not be empty
            ['is_numeric'] // must be numeric
        ]
    ]
);
``` 

> You can omit the `[]` if rule does not need any parameters.

### Checker Rules
You can split your rule name using `::` prefix, where first part is checker name and second is method name, for example:

```php
$validator = $validation->validate(
    ['file' => null],
    [
        'file' => [
            'file::uploaded'
        ]
    ]
);
```

### Parameters
All values listed in rule array will be passed as rule arguments. For example to check value using `in_array`:

```php
$validator = $validation->validate(
    ['name' => 'f'],
    [
        'name' => [
            'notEmpty',
            ['in_array', ['a', 'b', 'c']] // in_array($value, ['a', 'b', 'c'])
        ]
    ]
);
```

To specify regexp pattern:

```php
$validator = $validation->validate(
    ['name' => 'b'],
    [
        'name' => [
            'notEmpty',
            ['regexp', '/^a+$/'] // aaa...
        ]
    ]
);
```

### Error Messages
Validator will render default error message for any custom rule, to set custom error message set the rule
attribute:

```php
$validator = $validation->validate(
    ['file' => 'b'],
    [
        'file' => [
            'notEmpty',
            ['regexp', '/^a+$/', 'error' => 'Invalid pattern, "a+" wanted.'] // aaa...
        ]
    ]
);
```

> You can assign custom error messages to any rule.

### Conditions
In some cases the rule must only be activated based on some external condition, use rule attribute `if` for this
purpose:

```php
$validator = $validation->validate(
    [
        'password'        => '',
        'confirmPassword' => ''
    ],
    [
        'password'        => [
            ['notEmpty']
        ],
        'confirmPassword' => [
            ['notEmpty', 'if' => ['withAll' => ['password']]]
        ]
    ]
);
```

> In the given example the required error on `confirmPassword` will show if `password` is filled.

You can use multiple conditions or combine them with complex rules:

```php
 $validator = $validation->validate(
    [
        'password'        => 'abc',
        'confirmPassword' => 'cde'
    ],
    [
        'password'        => [
            ['notEmpty']
        ],
        'confirmPassword' => [
            ['notEmpty', 'if' => ['withAll' => ['password']]],
            ['match', 'password', 'error' => 'Passwords do not match.']
        ]
    ]
);
```

### Available Conditions
Following conditions available for the usage:

Name | Arguments | Description
--- | --- | ---
withAny | fields:*array* | When at least one field is not empty.
withoutAny | fields:*array* | When at least one field is empty.
withAll | fields:*array* | When all fields are not empty.
withoutAll | fields:*array* | When all fields are empty.

> You can create your own conditions using `Spiral\Validation\ConditionInterface`.

## Validation Rules
The following validation rules are available.

> You can create your own validation rules using `Spiral\Validation\AbstractChecker` or `Spiral\Validation\CheckerInterface`.

### Rules Aliases
The most used rule-set is available thought the set of shortcuts:

Alias | Rule
--- | ---
notEmpty|  type::notEmpty
required|   type::notEmpty
datetime|  type::datetime
timezone|  type::timezone
bool|      type::boolean
boolean|   type::boolean
cardNumber|mixed::cardNumber
regexp|    string::regexp
email|     address::email
url|       address::url
file|      file::exists
uploaded|  file::uploaded
filesize|  file::size
image|     image::valid
array|     is_array
callable|  is_callable
double|    is_double
float|     is_float
int|       is_int
integer|   is_integer
numeric|   is_numeric
long|      is_long
null|      is_null
object|    is_object
real|      is_real
resource|  is_resource
scalar|    is_scalar
string|    is_string
match|     mixed::match

### Type - prefix type::
| Rule          | Parameters             | Description         
| ---           | ---                    | ---       
| notEmpty      | asString:*bool* - true | Value should not be empty (same as `!empty`).                                             
| notNull       | ---                    | Value should not be null.                                            
| boolean       | ---                    | Value has to be boolean or integer[0,1].                            
| datetime      | ---                    | Value has to be valid datetime definition including numeric timestamp. 
| timezone      | ---                    | Value has to be valid timezone.                                        

> All of the rules of this checker are available without prefix.

### Required - prefix required::
| Rule          | Parameters             | Description    
| ---           | ---                    | ---  
| notEmpty      | asString:*bool* - true | Value should not be empty.                                             
| with          | with:*array*           | Check if field is not empty but only if any of listed fields presented or not empty.
| withAll       | with:*array*           | Check if field is not empty but only if all of listed fields presented and not empty.
| without       | without:*array*        | Check if field is not empty but only if one of listed fields missing or empty.
| withoutAll    | without:*array*        | Check if field is not empty but only if all of listed fields missing or empty.

> All of the rules of this checker are available without prefix.

Examples:

```php
'rules' => [
    'name'  => [
        ['notEmpty'],
        ['my::abc']
    ],
    'email' => [
        ['required::with', ['name']], //Only required when name is not empty
        ['email']
    ]
]
```

### Mixed - prefix mixed::
| Rule          | Parameters                            | Description                                      
| ---           | ---                                   | ---                                              
| cardNumber    | ---                                   | Check credit card passed by Luhn algorithm.      
| match         | field:*string*, strict:*bool* - false | Check if value matches value from another field. 

> All of the rules of this checker are available without prefix.

### Address - prefix address::
| Rule          | Parameters                    | Description              
| ---           | ---                           | ---                      
| email         | ---                           | Check if email is valid. 
| url           | requireScheme:*bool* - true   | Check if URL is valid. 

> All of the rules of this checker are available without prefix.

### Number - prefix number::
| Rule          | Parameters                 | Description           
| ---           | ---                        | ---                   
| range         | begin:*float*, end:*float* | Check if the number is in a specified range.
| higher        | limit:*float*              | Check if the value is bigger or equal to that which is specified.
| lower         | limit:*float*              | Check if the value is smaller or equal to that which is specified.


### String - prefix string::
| Rule          | Parameters              | Description           
| ---           | ---                     | ---                   
| regexp        | expression:*string*     | Check string using regexp.                  
| shorter       | length:*int*            | Check if string length is shorter or equal that specified value.              
| longer        | length:*int*            | Check if the string length is longer or equal to that specified value.                
| length        | length:*int*            | Check if the string length is equal to specified value.               
| range         | left:*int*, right:*int* | Check if the string length fits within the specified range.                  

Examples:

```php
'rules' => [
    'name' => [
        ['notEmpty'],
        ['string::length', 5] 
    ]
]
```

### File Checker - prefix file::
File checker fully supports the filename provided in a string form or using `UploadedFileInterface` (PSR-7).
This makes the checker very useful for uploading files.

| Rule          | Parameters            | Description           
| ---           | ---                   | ---                   
| exists        | ---                   | Check if file exist.
| uploaded      | ---                   | Check if file was uploaded.
| size          | size:*int*            | Check if file size less that specified value in KB.
| extension     | extensions:*array*    | Check if file extension in whitelist. Client name of uploaded file will be used!

### Image Checker - prefix image::
Image checker extends the file checker and fully supports it's features.

| Rule          | Parameters                | Description           |
| ---           | ---                       | ---                   |
| type          | types:*array*             | Check if image is within a list of allowed image types.
| valid         | ---                       | Shortcut to check if the image has a valid type (JPEG, PNG and GIF are allowed).
| smaller       | width:*int*, height:*int* | Check if image is smaller than a specified shape (height check if optional).
| bigger        | width:*int*, height:*int* | Check if image is bigger than a specified shape (height check is optional).