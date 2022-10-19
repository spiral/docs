# Validations
Spiral provides a simple way to validate incoming user data by providing an `Spiral\Validation\ValidatorInterface` interface which is binded by default to the `Spiral\Validation\Validator` class.

## ValidatorInterface

```php
interface ValidatorInterface
{
    /**
     * Update validation rules.
     *
     * @param array $rules
     *
     * @return self
     */
    public function setRules(array $rules): ValidatorInterface;

    /**
     * Update validation data (context). Data change must reset validation state and all errors.
     *
     * @param array|\ArrayAccess|\Spiral\Models\EntityInterface $data
     *
     * @return self
     *
     * @throws ValidationException
     */
    public function setData($data): ValidatorInterface;

    /**
     * Get all validation data passed into validator.
     *
     * @return array|\ArrayAccess
     */
    public function getData();

    /**
     * Receive field from context data or return default value.
     *
     * @param string $field
     * @param mixed  $default
     *
     * @return mixed
     */
    public function getValue(string $field, $default = null);

    /**
     * Register outer validation error. Registered error persists until context data are changed
     * or flushRegistered method not called.
     *
     * @param string $field
     * @param string $error
     *
     * @return self
     */
    public function registerError(string $field, string $error): ValidatorInterface;

    /**
     * Reset validation state.
     */
    public function reset();

    /**
     * Flush all registered errors.
     */
    public function flushRegistered();

    /**
     * Check if context data valid accordingly to provided rules.
     *
     * @return bool
     *
     * @throws ValidationException
     */
    public function isValid(): bool;

    /**
     * Evil tween of isValid() method should return true if context data is not valid.
     *
     * @return bool
     *
     * @throws ValidationException
     */
    public function hasErrors(): bool;

    /**
     * List of errors associated with parent field, every field should have only one error assigned.
     *
     * @return array
     */
    public function getErrors(): array;

    /**
     * Get context data (not validated).
     *
     * @return mixed
     */
    public function getContext();

    /**
     * Set context data (not validated).
     *
     * @param $context
     *
     * @return mixed
     */
    public function setContext($context);
}
```

As you can see, validator requires 2 primary sets:
* data is an array of fields to be validated
* rules is set of validation rules to be applied to every data field

You can add some context to validator, it can be used in request filters or in checkers later.

> The default spiral implementation `Validator` requires instances of `ContainerInterface` (Interop, needed to load Checkers and external rule classes) and `ValidatorConfig` (to provide a list of available validations using section "validation"). This means that you have to request the validator instance using the DI or factory to ensure that it's configured correctly. 

Now we can create a sample validator in our controller action and configure validation on a fly:

```php
protected function indexAction(ValidatorInterface $validator)
{
    $validator->setData([
        'name' => $this->input->query('name')
    ])->setRules([
        'name' => ['notEmpty']
    ]);

    dump($validator->getErrors());
}
```

We can also create a validator using factory and define data/rules at moment of construction:

```php
protected function indexAction(FactoryInterface $factory)
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $factory->make(
        ValidatorInterface::class, 
        [
            'data'  => [
                'name' => $this->input->query('name')
            ],
            'rules' => [
                'name' => ['notEmpty']
            ]
        ]
    );

    dump($validator->getErrors());
}
```

> Both declarations are identical.

## Validaton Data
Validator data set can be represented by any associated array where the key is treated as a field name that needs value and data to be validated. In our example, we defined only one field called "name" and associated it with a value fetched from a query.

By default, the validation will fail with the error message "This field is required." associated with the "name" field. You can alter the website url and include query "?name=something" to pass the example validation.

## Validation Errors
Before we move to the format required to define validation rules, let's take a look at the error generated in our case. The error array will look like the following:

```php
[
    'name' => 'This field is required.'
]
```

Such errors can be sent directly to the frontend and be displayed near the input field (see [Spiral Frontend Toolkit](/modules/toolkit.md)) or joined as an array of errors.

> Spiral Validator perform **linear field validation**. This means that only one error can be associated with one field. When there are multiple rules associated with one field - the first failed error message will be used for that rule (other validation methods will be skipped). This approach is used to make sure that every rule will receive a value already passed through previous validations (for example we can have rules like: notEmpty, string, string::regexp which will make sure that field is not empty. Than that value is a string and only after regexp expression will it be executed).

## Validation Rules and Messages
To define multiple validation rules for one field:

```php
protected function indexAction(FactoryInterface $factory)
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $factory->make(
        ValidatorInterface::class, 
        [
            'data'  => [
                'name' => $this->input->query('name')
            ],
            'rules' => [
                'name' => [
                    'notEmpty',
                    'email'     //We want our name field to contain a valid email address
                ]
            ]
        ]
    );

    dump($validator->getErrors());
}
```

### What is a Validation rule?
Generally speaking, a validation rule is a simple function or function alias which must accept a value to be validated first and then return bool or type-casted bool as a result. Let's try to add a few php functions as data validations:

```php
'rules' => [
    'name' => [
        'notEmpty',
        'is_numeric'
    ]
]
```

Now, our validation fails in the name because it contains a non numeric value. The Validator will show a default error message in this case:

```php
[
    'name' => "Condition 'is_numeric' does not meet."
]
```

### Validation Messages and Complex Rule Definition
Based on the given example, validator will create a default error message if some condition fails. We can provide a custom error message by switching the rule definition from a simple to complex form. For this purpose, our rule must be defined as an array and must declare an additional key "message":

```php
'rules' => [
    'name' => [
        'notEmpty',
        ['is_numeric', 'message' => 'Must be numeric.']
    ]
]
```

Every existing rule can be declared in a complex form by putting it's name into an array. Let's rewrite our rules to demonstrate that:

```php
'rules' => [
    'name' => [
        ['notEmpty'],
        ['is_numeric', 'message' => 'Must be numeric.']
    ]
]
```

> You can only use a simple rule definition form when you want to use a default error message without any rule parameters (see below). Validator performs an automatic translation only for default error messages!

### Rule parameters
It was mentioned before that the complex form lets you set custom error message and provide a set of arguments to be passed into your validation method. We can perform that by listing the necessary arguments after the rule name/function. Let's try to use the 'in_array' function for example:

```php
'rules' => [
    'name' => [
        ['notEmpty'],
        ['in_array', ['a', 'b', 'c'], 'message' => 'Invalid value.']
    ]
]
```

> You can use any function as a validation rule, including your own method declared in a form "Class::method". The only requirement is that value needs to be the first argument.

Let's try to create our own validation method in our controller or service.

```php
protected function indexAction(FactoryInterface $factory)
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $factory->make(
        ValidatorInterface::class, 
        [
            'data'  => [
                'name' => $this->input->query('name')
            ],
            'rules' => [
                'name' => [
                    ['notEmpty'],
                    ['Controllers\HomeController::validate', 'abc', 'message' => 'Invalid value.']
                ]
            ]
        ]
    );

    dump($validator->getErrors());
}

public function validate($value, $compare)
{
    return $value == $compare;
}
```

> As you can see, we declared our method as non static. This is because Validator will resolve the rule class using container. This allows us to create specialized services to perform logical validations (for example ensure that outer record exists by it's ID) and much more (technically you can use something like that UserService::findByPK to validate foreign key value).

### Rule aliases
Default Spiral Validation provides the ability to define a set of aliases associated with different validation functions. These aliases are defined in the `application/config/validation.php` file and sent to validator using `ValidationProvider` dependency. Let's check the list of available validation aliases:

```php
'aliases'         => [
    "notEmpty"   => "type::notEmpty",
    "required"   => "type::notEmpty",
    "datetime"   => "type::datetime",
    "timezone"   => "type::timezone",
    "bool"       => "type::boolean",
    "boolean"    => "type::boolean",
    "cardNumber" => "mixed::cardNumber",
    "regexp"     => "string::regexp",
    "email"      => "address::email",
    "url"        => "address::url",
    "file"       => "file::exists",
    "uploaded"   => "file::uploaded",
    "filesize"   => "file::size",
    "image"      => "image::valid",
    "array"      => "is_array",
    "callable"   => "is_callable",
    "double"     => "is_double",
    "float"      => "is_float",
    "int"        => "is_int",
    "integer"    => "is_integer",
    "numeric"    => "is_numeric",
    "long"       => "is_long",
    "null"       => "is_null",
    "object"     => "is_object",
    "real"       => "is_real",
    "resource"   => "is_resource",
    "scalar"     => "is_scalar",
    "string"     => "is_string",
    "match"      => "mixed::match",
]
```

We can now register our own controller method alias and update the validation rules:

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

You can define as many aliases as needed. There is one easy way to create a set of validation methods attached to one name without overloading your validation config - checkers. You can read about checkers below.

## Empty Validation Rules (Stoppers)
Another section you may notice in validation config is "emptyConditions". Let's check it out:

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
    "image:valid",
    "record::notEmpty",
],
```

Empty conditions are a set of rules make sure that a field value is set. Generally speaking, you have to include one of these rules into the field validations if you do not want to leave the field empty. Let's try to demonstrate it below:

```php
protected function indexAction(FactoryInterface $factory)
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $factory->make(
        ValidatorInterface::class,
        [
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
        ]
    );

    dump($validator->getErrors());
}
```

Validation for the field "email" won't happen and fail if field value is empty. Once you set the field value the "email" rule will be applied. Said differently, if the "name" field has one "empty condition", this condition will fail and create an error message if the value is not set.

Spiral Validator provides an additional way to skip the field validations based on the validation rule return response. Make your rule return `Validator::STOP_VALIDATION` and the field validation will be stopped without showing an error message. We will demonstrate such an example in the next section.

## Checkers
In some cases, you may want to create a set of rules with predefined error messages and skip the step where you have to create an alias for every rule. For such purposes, the spiral Validator lets you create your own `Checker` class. Checkers are dedicated to organizing a set of validation rules with their error messages and provide access to such validations using a simple prefix.

If you check the validation configuration file, you will see some checkers already present:

```php
'checkers'        => [
    "type"     => Checkers\TypeChecker::class,
    "required" => Checkers\RequiredChecker::class,
    "number"   => Checkers\NumberChecker::class,
    "mixed"    => Checkers\MixedChecker::class,
    "address"  => Checkers\AddressChecker::class,
    "string"   => Checkers\StringChecker::class,
    "file"     => Checkers\FileChecker::class,
    "image"    => Checkers\ImageChecker::class,
],
```

Every checker must has it's own name. This name can be used as prefix (using :: separator) while defining the validation rules. For example, StringChecker provides a method named "regexp" that has the ability to apply regular expression to a field value. Lets try it:

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

As you may have noticed, the default error message will look something like "Your value does not match the required pattern.". This is possible because Checkers allow us to define their own default messages. Let's try to create our own Checker (every Checker has Translation support by default):

```php
class MyChecker extends Checker
{
    /**
     * Default error messages associated with checker method by name.
     *
     * @var array
     */
    const MESSAGES = [
        'abc' => '[[Invalid value, only a,b,c are allowed.]]'
    ];

    public function abc($value): bool
    {
        return in_array($value, ['a', 'b', 'c']);
    }
}
```

> You may notice that the error message for method check is surrounded by `[[]]`. This technique allows spiral Translator to index and pre-cache checker translations. Read more about it in the section dedicated to Translator.

Now we can register our checker in the validation config under the name "my" and use it our rules:

```php
'rules' => [
    'name' => [
        'notEmpty',
        'my::abc'
    ]
]
```

> You can create aliases for checker methods.

Besides the ability to aggregate many validation methods under one roof, Checkers is more similar with integration with Validator than the usual validation functions. We can demonstrate such ability by accessing the validator instance from the inside of our checker method:

```php
class MyChecker extends Checker
{
    /**
     * Default error messages associated with checker method by name.
     *
     * @var array
     */
    const MESSAGES = [
        'abc'   => '[[Invalid value, only a,b,c are allowed.]]',
        'equal' => '[[Two fields must equal.]]'
    ];

    public function abc($value): bool
    {
        return in_array($value, ['a', 'b', 'c']);
    }

    public function equal($value, string $field): bool
    {
        return $value == $this->getValidator()->getValue($field, null);
    }
}
```

We can now modify our validation code within controller:

```php
protected function indexAction(FactoryInterface $factory)
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $factory->make(
        ValidatorInterface::class,
        [
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
        ]
    );

    dump($validator->getErrors());
}
```

As result, we have a method that will check two validation fields.

## Available validation Checkers
The following validation Checkers can be used.

### Type Checker (prefix "type::")
| Rule          | Parameters         | Description         
| ---           | ---                | ---       
| notEmpty      | [asString:bool - true] | Value should not be empty.                                             
| notNull       | ---                | Value should not be null.                                            
| boolean       | ---                | Value has to be boolean or integer[0,1].                            
| datetime      | ---                | Value has to be valid datetime definition including numeric timestamp. 
| timezone      | ---                | Value has to be valid timezone.                                        

### Required Checker (prefix "required::")
| Rule          | Parameters             | Description    
| ---           | ---                    | ---  
| notEmpty      | [asString:bool - true] | Value should not be empty.                                             
| with          | with:array             | Check if field is not empty but only if any of listed fields presented or not empty.
| withAll       | with:array             | Check if field is not empty but only if all of listed fields presented and not empty.
| without       | without:array          | Check if field is not empty but only if one of listed fields missing or empty.
| withoutAll    | without:array          | Check if field is not empty but only if all of listed fields missing or empty.

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

Both of the listed checker methods have short aliases - "cardNumber" and "match" accordingly.

### Address Checker (prefix "address::")
| Rule          | Parameters                    | Description              
| ---           | ---                           | ---                      
| email         | ---                           | Check if email is valid. 
| url           | [requireScheme:bool - true]   | Check if URL is valid.   

### Number Checker (prefix "number::")
| Rule          | Parameters             | Description           
| ---           | ---                    | ---                   
| range         | begin:float, end:float | Check if the number is in a specified range.
| higher        | limit:float            | Check if the value is bigger or equal to that which is specified.
| lower         | limit:float            | Check if the value is smaller or equal to that which is specified.

### String Checker (prefix "string::")
| Rule          | Parameters            | Description           
| ---           | ---                   | ---                   
| regexp        | expression:string     | Check string using regexp.                  
| shorter       | length:int            | Check if string length is shorter or equal that specified value.              
| longer        | length:int            | Check if the string length is longer or equal to that specified value.                
| length        | length:int            | Check if the string length is equal to specified value.               
| range         | left:int, right:int   | Check if the string length fits within the specified range.                  

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
File checker fully supports the filename provided in a string form or using `UploadedFileInterface`. This makes the checker very useful for uploading files.

| Rule          | Parameters            | Description           
| ---           | ---                   | ---                   
| exists        | ---                   | Check if file exist.
| uploaded      | ---                   | Check if file was uploaded.
| size          | size:int              | Check if file size less that specified value in KB.
| extension     | extensions:array      | Check if file extension in whitelist. Client name of uploaded file will be used!

### Image Checker (prefix "image::")
Image checker extends the file checker and fully supports it's features.

| Rule          | Parameters              | Description           |
| ---           | ---                     | ---                   |
| type          | types:array             | Check if image is within a list of allowed image types.
| valid         | ---                     | Shortcut to check if the image has a valid type (JPEG, PNG and GIF are allowed).
| smaller       | width:int, [height:int] | Check if image is smaller than a specified shape (height check if optional).
| bigger        | width:int, [height:int] | Check if image is bigger than a specified shape (height check is optional).

## Checker Conditions
Sometimes we want to validate field only in certain cases, for example when creating a blog post you want it to have a thumbnail. Thumbnail is required for a new post, but is optional when you update it (unless you upload a new thumnail for it).

Conditions are provided by `\Spiral\Validation\CheckerConditionInterface` interface, to see basic implementation refer to `\Spiral\Validation\Prototypes\AbstractCheckerCondition` class
```php
interface CheckerConditionInterface
{
    /**
     * @param ValidatorInterface $validator
     *
     * @return CheckerConditionInterface
     */
    public function withValidator(ValidatorInterface $validator): CheckerConditionInterface;

    /**
     * @return bool
     */
    public function isMet(): bool;
}
```

### Example usage
Request filter:
```php
class Request extends RequestFilter
{
    const SCHEMA = ['file' => 'file:image'];

    const VALIDATES = [
        'file' => [
            ['file::uploaded', 'condition' => NewPostOrUpdatedFileCondition::class],
            ['image::valid', 'condition' => NewPostOrUpdatedFileCondition::class]
        ],
    ];

    public function getFile(): UploadedFileInterface
    {
        return $this->file;
    }
}
```
Condition file:
```php
class NewPostOrUpdatedFileCondition extends AbstractCheckerCondition
{
    use FileTrait;

    /**
     * @return bool
     */
    public function isMet(): bool
    {
        $context = $this->validator->getContext();

        /** @var \Spiral\ORM\RecordEntity $entity */
        $entity = $context['entity'] ?? null;

        /** @var \Psr\Http\Message\UploadedFileInterface $entity */
        $file = $context['file'] ?? null;

        //Entity is new
        if (empty($entity) || !$entity->isLoaded()) {
            return true;
        }

        //File is provided and uploaded
        if (!empty($file) && $this->isUploaded($file)) {
            return true;
        }

        return false;
}
```
So in this case request filter will require file when there's no entity yet or (entity exists) file is uploaded again.

## Example of Validator usage
Spiral Http component ships with [convenient way](/http/validation.md) to validate incoming request,
use VALIDATES constant to configure request validation rules:
```php
class UploadRequest extends RequestFilter
{
    const SCHEMA = [
        'image' => 'file:image'
    ];
    
    const VALIDATES = [
        'image' => [
            'file::uploaded',
            'image::valid',
            ['image::bigger', 100, 100] //Must be at least 100x100px
        ]
    ];
}
```

```php
protected function indexAction(UploadRequest $request)
{
    if (!$request->isValid()) {
        //errors
    }

    $request->image->moveTo('...');
}
```

> You can extend `ValidatesEntity` to implement your own set of model filters.
