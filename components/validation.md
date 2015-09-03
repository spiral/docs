# Validations
Spiral provides simplistic way to validate incoming user data by providing `Spiral\Validation\ValidatorInterface` interface which is bindined by default to `Spiral\Validation\Validator` class.

## ValidatorInterface
Before we will start working with validations let's quickly take a look at `ValidatorInterface` to see what methods are available for us.

```php
interface ValidatorInterface
{
    /**
     * @param array|\ArrayAccess $data  Data to be validated.
     * @param array              $rules Validation rules.
     */
    public function __construct($data, array $rules);

    /**
     * Update validation data (context).
     *
     * @param array|\ArrayAccess $data
     * @return self
     * @throws ValidationException
     */
    public function setData($data);

    /**
     * Update validation rules.
     *
     * @param array $validates
     * @return self
     */
    public function setRules(array $validates);

    /**
     * Check if context data valid accordingly to provided rules.
     *
     * @return bool
     * @throws ValidationException
     */
    public function isValid();

    /**
     * Evil tween of isValid() method should return true if context data is not valid.
     *
     * @return bool
     * @throws ValidationException
     */
    public function hasErrors();

    /**
     * List of errors associated with parent field, every field should have only one error assigned.
     *
     * @return array
     */
    public function getErrors();
}
```

As you can see validator requires only 2 primary sets:
* data is arrays of fields to be validated
* rules is set of validation rules to be applied to every data field

> Default spiral implementation `Validator` requires instances of `ContainerInterface` and `ValidationProvider` instances (to provide list of available validations), this means that you have to request validator instance using container to ensure that it's configured correctly. 

We now can create sample validator in our controller action:

```php
public function index(ValidatorInterface $validator)
{
    $validator->setData([
        'name' => $this->input->query('name')
    ])->setRules([
        'name' => ['notEmpty']
    ]);

    dump($validator->getErrors());
}
```

We can also create validator using container and defined data and rules while class construction:

```php
public function index()
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $this->container->construct(ValidatorInterface::class, [
        'data'  => [
            'name' => $this->input->query('name')
        ],
        'rules' => [
            'name' => ['notEmpty']
        ]
    ]);

    dump($validator->getErrors());
}
```

> Both declarations are identical.

## Validator Data
Validator data set can be represented by any associated array where key is treated as field name, and value and data to be validated. In our example we defined only one field named "name" and associated it with value fetched from query. By default validation will fail with error "This field is required." associated with "name" field, you can alter website url and include query "?name=something" to pass example validation.

## Validation Errors
Before we will jump into format required to defined validation rules let's take a look at error generated in our case, error array will look like:

```php
[
    'name' => 'This field is required.'
]
```

Such errors can be sent directly to frontend to be displayed near input field or joined as array of errors.
> Spiral Validators must perform **linear field validation**, this means that only one error can be raised for one field. When multiple rules associated with one field - first failed rule error message will be used (other validation methods will be skipped), such technique used to ensure that every rule will receive value already passed previous validations (for example we can have rules like: notEmpty, string, string::regexp which will check that field is not empty, than that value is string and only after regexp expression will be executed).

## Validation Rules and Messages
As stated in previous section we can defined multiple validation rules for one field, let's try to modify our example and include email check into it:

```php
public function index()
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $this->container->construct(ValidatorInterface::class, [
        'data'  => [
            'name' => $this->input->query('name')
        ],
        'rules' => [
            'name' => [
                'notEmpty',
                'email'     //We want our name field to contain valid email address
            ]
        ]
    ]);

    dump($validator->getErrors());
}
```

### What is Validation rule?
In general sense validation rule is a simple function or function alias which must accept value to be validated first and return bool or typecasted bool as result. Let's try to add few php functions as data validations:

```php
'rules' => [
    'name' => [
        'notEmpty',
        'is_numeric'
    ]
]
```

Now, our validation will fail in name will contain non numeric value, validator will raise defailt error message in this case:

```php
[
    'name' => "Condition 'is_numeric' does not meet."
]
```

### Validation Messages and Complex Rule Definition
As you can see based on given example validator will create default error message if some condition will fail. We can provide custom error message by switch rule definition from simple to complex form, for this purposes our rule must be defined as array and declare additional key "message":

```php
'rules' => [
    'name' => [
        'notEmpty',
        ['is_numeric', 'message' => 'Must be numeric.']
    ]
]
```

Every existed rule can be declared in complex form by putting it's name into array, let's rewrite our rules to demonstrate that:

```php
'rules' => [
    'name' => [
        ['notEmpty'],
        ['is_numeric', 'message' => 'Must be numeric.']
    ]
]
```

> You can only use simple rule definition form when you want to use default error message without any rule parameters (see below). Validator will perform automatic translation only for default error messages!

### Rule parameters
As mention in a previous paragraph complex form provides you ability not only to set custom error message but provide set of arguments to be passed into validation method, we can do that by listing needed arguments after rule name/function, let's try to use 'in_array' function for example:

```php
'rules' => [
    'name' => [
        ['notEmpty'],
        ['in_array', ['a', 'b', 'c'], 'message' => 'Invalid value.']
    ]
]
```

> You can use any function as validation rule including your own method declared in a form "Class::method", the only requiment that value will come as first argument.

Let's try to create our own validation method in our controller or service.

```php
public function index()
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $this->container->construct(ValidatorInterface::class, [
        'data'  => [
            'name' => $this->input->query('name')
        ],
        'rules' => [
            'name' => [
                ['notEmpty'],
                ['Controllers\HomeController::validate', 'abc', 'message' => 'Invalid value.']
            ]
        ]
    ]);

    dump($validator->getErrors());
}

public function validate($value, $compare)
{
    return $value == $compare;
}
```

> As you can see we declared our method as non static, this is possible due Validator will resolve rule class using container, this provides us ability to create specialized services to perform logical validation (for example ensure that outer record exists by it's ID) and much more.

### Rule aliases
Default spiral Validation provides ability to define set of alises associated with different validation functions, such aliases defined in `application/config/validation.php` file and provided to validator using `ValidationProvider` dependency. Let's check list of available validation aliases:

```php
'aliases'         => [
    "notEmpty"    => "type::notEmpty",
    "required"    => "type::notEmpty",
    "datetime"    => "type::datetime",
    "timezone"    => "type::timezone",
    "bool"        => "type::boolean",
    "boolean"     => "type::boolean",
    "cardNumber"  => "mixed::cardNumber",
    "regexp"      => "string::regexp",
    "email"       => "address::email",
    "url"         => "address::url",
    "file"        => "file::exists",
    "uploaded"    => "file::uploaded",
    "filesize"    => "file::size",
    "image"       => "image::valid",
    "array"       => "is_array",
    "callable"    => "is_callable",
    "double"      => "is_double",
    "float"       => "is_float",
    "int"         => "is_int",
    "integer"     => "is_integer",
    "long"        => "is_long",
    "null"        => "is_null",
    "object"      => "is_object",
    "real"        => "is_real",
    "resource"    => "is_resource",
    "scalar"      => "is_scalar",
    "string"      => "is_string",
    "match"       => "mixed::match"
]
```

We can now register our own alias for our controller method and update validation rules:

```php
    'valueMatcher' => 'Controllers\HomeController::validate'
```

```php
'rules' => [
    'name' => [
        ['notEmpty'],
        ['valueMatcher', 'abc', 'message' => 'Invalid value.']
    ]
]
```

You can defined as many aliases as you needed, hovewer there is one easy way to create set of validation methods joined by one name without overloading validation config - checkers. You can read about checkers below.

## Empty Validation Rules (Stoppers)
Another section you might notice in validation config is "emptyConditions", let's check it:

```php
'emptyConditions' => [
    "notEmpty",
    "required",
    "type::notEmpty",
    "required::with",
    "required::without",
    "required::withAll",
    "required::withoutAll",
    "file::exists",
    "file::uploaded",
    "image::exists",
    "image::uploaded"
],
```

Empty conditions is set of rules dedicated to ensure that field value is set. Generally speaking you have to include one of this rules into field validations if you do not want to leave field empty. Let's try to demonstrate it on example:

```php
public function index()
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $this->container->construct(ValidatorInterface::class, [
        'data'  => [
            'name'  => $this->input->query('name'),
            'email' => $this->input->query('email'),
        ],
        'rules' => [
            'name'  => [
                'notEmpty',
                ['matcher', 'abc', 'message' => 'Invalid value.']
            ],
            'email' => [
                'email'
            ]
        ]
    ]);

    dump($validator->getErrors());
}
```

In given example validation for field "email" will not happen and not fail if field value is empty, only when field value is set "email" rule will be applied. In opposite case, "name" field has one of "empty conditions", such condition will fail and create error message if value not set.

Spiral Validator provides additional way to skip field validations based on return response of validation rule. Simply make your rule return `Validator::STOP_VALIDATION` and field validation will be stopped without raising any error message, we will demonstrate such thing in next section.

## Checkers
In some cases you may want to create set of rules with predefined error messages and skip step when you have to create an alias for every rule. For such purposes spiral Validator provides ability to create your own `Checker` class. Checkers are dedicated to organize set of validation rules with their error messages and provide access to such validations using simple prefix.

If you will check validation configuration file, you will see some checker already presented:

```php
'checkers'        => [
    "type"     => Checkers\TypeChecker::class,
    "required" => Checkers\RequiredChecker::class,
    "number"   => Checkers\NumberChecker::class,
    "mixed"    => Checkers\MixedChecker::class,
    "address"  => Checkers\AddressChecker::class,
    "string"   => Checkers\StringChecker::class,
    "file"     => Checkers\FileChecker::class,
    "image"    => Checkers\ImageChecker::class
],
```

Every checker must has it's own name, such name can be used as prefix (using :: separator) while defining validation rules, for example StringChecker provides method named "regexp" with ability to apply regular expression to field value, lets use it:

```php
/**
 * Check string using regexp.
 *
 * @param string $string
 * @param string $expression
 * @return bool
 */
public function regexp($string, $expression)
{
    return is_string($string) && preg_match($expression, $string);
}
```

```php
'rules' => [
    'name'  => [
        ['notEmpty'],
        ['string::regexp', '/^a+$/i']
    ]               
]
```

As you might notice default error message will look like "Your value does not match required pattern.", this is possible due Checkers are allowed to define their own default messages. Let's try to create our own checker (every Checker has Translation support by default):

```php
class MyChecker extends Checker
{
    /**
     * Default error messages associated with checker method by name.
     *
     * @var array
     */
    protected $messages = [
        'abc' => '[[Invalid value, only a,b,c are allowed.]]'
    ];

    public function abc($value)
    {
        return in_array($value, ['a', 'b', 'c']);
    }
}
```

> You might notice that error message for method check is embraced with `[[]]`, such technique allows spiral Translator to index and pre-cache checker translations. Read more about in guide section dedicated to Translator.

Now we can register our checker in validation config under name "my" and use it our rules:

```php
'rules' => [
    'name' => [
        'notEmpty',
        'my::abc'
    ]
]
```

> You can create aliases for checker methods.

Besides ability to aggregate many validation methods under one roof, Checkers has little bit closer integration with Validator than usual validation functions. We can demonstrate such ability by accessing validator instance from inside of our checker method:

```php
class MyChecker extends Checker
{
    /**
     * Default error messages associated with checker method by name.
     *
     * @var array
     */
    protected $messages = [
        'abc'   => '[[Invalid value, only a,b,c are allowed.]]',
        'equal' => '[[Two fields must equal.]]'
    ];

    public function abc($value)
    {
        return in_array($value, ['a', 'b', 'c']);
    }

    public function equal($value, $field)
    {
        return $value == $this->validator->field($field);
    }
}
```

We can now modify our validation code in controller:

```php
public function index()
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $this->container->construct(ValidatorInterface::class, [
        'data'  => [
            'name'  => $this->input->query('name'),
            'email' => $this->input->query('email'),
        ],
        'rules' => [
            'name'  => [
                'notEmpty',
                'my::abc'
            ],
            'email' => [
                'notEmpty',
                ['my::equal', 'name']
            ]
        ]
    ]);

    dump($validator->getErrors());
}
```

As result we have method which will check two validation fields.

## Available validation Checkers
Following validation Checkers are available to be used.

### Type Checker (prefix "type::")
| Rule          | Parameters         | Description         
| ---           | ---                | ---                   
| notEmpty      | [trim:bool - true] | Value should not be empty.                                             
| boolean       | ---                | Value has to be boolean or integer[0,1].                            
| datetime      | ---                | Value has to be valid datetime definition including numeric timestamp. 
| timezone      | ---                | Value has to be valid timezone.                                        

### Required Checker (prefix "required::")
| Rule          | Parameters            | Description    
| ---           | ---                   | ---  
| with          | with:array            | Check if field not empty but only if any of listed fields presented or not empty.
| withAll       | with:array            | Check if field not empty but only if all of listed fields presented and not empty.
| without       | without:array         | Check if field not empty but only if one of listed fields missing or empty.
| withoutAll    | without:array         | Check if field not empty but only if all of listed fields missing or empty.

Examples:

```php
'rules' => [
    'name'  => [
        'notEmpty',
        'my::abc'
    ],
    'email' => [
        ['required::with', ['name']], //Only required when name is not empty
        ['email']
    ]
]
```

### Mixed Checker (prefix "mixed::")
| Rule          | Parameters                          | Description                                      
| ---           | ---                                 | ---                                              
| cardNumber    | ---                                 | Check credit card passed by Luhn algorithm.      
| match         | field:string, [strict:bool - false] | Check if value matches value from another field. 

Both of listed checker method has short aliases - "cardNumber" and "match" accordingly.

### Address Checker (prefix "address::")
| Rule          | Parameters                    | Description              
| ---           | ---                           | ---                      
| email         | ---                           | Check if email is valid. 
| url           | [requireScheme|bool - true]   | Check if URL is valid.   

### Number Checker (prefix "number::")
| Rule          | Parameters             | Description           
| ---           | ---                    | ---                   
| range         | begin:float, end:float | Check if number in specified range.
| higher        | limit:float            | Check if value is bigger or equal that specified.
| lower         | limit:float            | Check if value smaller of equal that specified.

### String Checker (prefix "string::")
| Rule          | Parameters            | Description           
| ---           | ---                   | ---                   
| regexp        | expression:string     | Check string using regexp.                  
| shorter       | length:int            | Check if string length is shorter or equal that specified value.              
| longer        | length:int            | Check if string length is longer or equal that specified value.                
| length        | length:int            | Check if string length are equal to specified value.               
| range         | left:int, right:int   | Check if string length are fits in specified range.                  

Examples:

```php
'rules' => [
    'name' => [
        'notEmpty',
        ['string::length', 5]
    ]
]
```

### File Checker (prefix "file::")
File checker fully support filename provided in a string form or using `UploadedFileInterface`, this makes this checker very useful for file uploades.

| Rule          | Parameters            | Description           
| ---           | ---                   | ---                   
| exists        | ---                   | Check if file exist.
| uploaded      | ---                   | Check if file was uploaded.
| size          | size:int              | Check if file size less that specified value in KB.
| extension     | extensions|array      | Check if file extension in whitelist. Client name of uploaded file will be used!

### Image Checker (prefix "image::")
Image checker extends file checker and fully support it's features.

| Rule          | Parameters              | Description           |
| ---           | ---                     | ---                   |
| exists        | ---                     | Check if image exist.
| uploaded      | ---                     | Check if image was uploaded.
| size          | size:int                | Check if image size less that specified value in KB.
| extension     | extensions|array        | Check if image extension in whitelist. Client name of uploaded file will be used!
| type          | types:array             | Check if image in a list of allowed image types.
| valid         | ---                     | Shortcut to check if image has valid type (JPEG, PNG and GIF are allowed).
| smaller       | width:int, [height:int] | Check if image smaller that specified rectangle (height check if optional).
| bigger        | width:int, [height:int] | Check if image is bigger that specified rectangle (height check is optional).

## Example of Validator usage
In most of cases you don't need to create validator manually. Validation component provides `ValidatesInterface` and it's related trait `ValidatorTrait` which provides ability to embedd field based validation into your models.

```php
interface ValidatesInterface
{
    /**
     * Check if context data is valid.
     *
     * @return bool
     */
    public function isValid();

    /**
     * Check if context data has errors.
     *
     * @return bool
     */
    public function hasErrors();

    /**
     * List of errors associated with parent field, every field must have only one error assigned.
     *
     * @param bool $reset Clean errors after receiving every message.
     * @return array
     */
    public function getErrors($reset = false);
}
```

Such interface and trait already implemenet by base spiral model - DataEntity. As result, you are recommended to validate incoming user requests using http [RequestFilters] (/http/filters.md). Let's try to write an example to demonstrate usage of image/file checker using RequestFilter:

```php
/**
 * @property \Zend\Diactoros\UploadedFile $image
 */
class UploadRequest extends RequestFilter
{
    /**
     * Input schema.
     *
     * @var array
     */
    protected $schema = [
        'image' => 'file:image'
    ];
    
    /**
     * @var array
     */
    protected $validates = [
        'image' => [
            'image::uploaded',
            'image::valid',
            ['image::bigger', 100, 100] //Must be at least 100x100px
        ]
    ];
}
```

Our controller code will look like:

```php
public function index(UploadRequest $request)
{
    if (!$request->isValid()) {
        //errors
    }

    $request->image->moveTo('...');
}
```
