# DataEntity Model
Most of spiral classes like ORM, ODM and HTTP request models extends one common parent - DataEntity model. Such class is one of the basic application cells used to provide set of wrappers and operations related to array dataset.

## Purpose
DataEntity model reponsible for mocking and providing access to array dataset - fields (associated array) using set of getters, setters and accessors. In addition to that model provides way to apply mass assigment to fields (using black/white lists), get all model fields as one array, get list of "public" fields (allowed to be send to client using whitelist) and to be converted into JSON. In addition to that model support [field validation] (validation.md) based on set of defined rules.

## Interfaces
Due DataEntity class used widely across spiral framework you might create your own code to communicate with it. Instead of using DataEntity class itself you can stick to it's primary interface:

```php
interface EntityInterface extends \ArrayAccess, ValidatesInterface
{
    /**
     * Check if entity has field by it's name.
     *
     * @param string $name
     * @return bool
     */
    public function hasField($name);

    /**
     * Set entity field value.
     *
     * @param string $name
     * @param mixed  $value
     * @throws EntityExceptionInterface
     */
    public function setField($name, $value);

    /**
     * Get value of entity field.
     *
     * @param string $name
     * @param mixed  $default
     * @return mixed|AccessorInterface
     * @throws EntityExceptionInterface
     */
    public function getField($name, $default = null);

    /**
     * Update entity fields using mass assignment. Only allowed fields must be set.
     *
     * @param array|\Traversable $fields
     * @throws EntityExceptionInterface
     */
    public function setFields($fields = []);

    /**
     * Get entity field values.
     *
     * @return array
     * @throws EntityExceptionInterface
     */
    public function getFields();
}
```

Due EntityInterface extends ValidatesInterface we better show it's content too:

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
     * @param bool $reset Force re-validation.
     * @return array
     */
    public function getErrors($reset = false);
}
```

## Entity Fields
Every entity must work with set of mocked fields, such fields can be represented as associated array and stored in property "fields". Knowing that we can create our simple entity used to mock some data array:

```php
class DemoEntity extends \Spiral\Models\DataEntity
{
    /**
     * @param array $fields
     */
    public function __construct(array $fields)
    {
        //No setters/accessors will be called
        $this->fields = $fields;
    }
}
```

Such entity provides ability to set fields using contructor method (attention, fields will be set without any filtering). Now we can use such entity somewhere in our code:

```php
public function index()
{
    $entity = new \DemoEntity([ 
        'name'    => 'value',
        'another' => 123
    ]);

    dump($entity);
}
```

One we have our entity constructed we can walk thought set of methods designed to work with our field values.

### Check field existence
If you want to check if desired field exists in model, you can use method `hasField`.

```php
dump($entity->hasField('name'));
dump($entity->hasField('undefined'));
```

### Get individual field
To get value of individual field let's try method `getField`. Method can accept second parameter to specify default value if field does not exists.

```php
dump($entity->getField('name'));
dump($entity->getField('undefined', 'DEFAULT VALUE'));
```

### Get all Entity fields
In many cases we would like to get every model field in array form, we can use method `getFields` for such purposes:

```php
dump($entity->getFields());
```

> Attention, resulted array will include field accessors (see below), plus every field will be passed thought it's associated getter (see below).

### Set Model field
To set value of desired model field we can use obviously named method `setField`:

```php
$entity->setField('thing', 199.00);
dump($entity->getFields());
```

## Mass Assignment
In many cases (almost in every case, to be honest) you will need to set multiple model fields at once. Setting fields one by one may look like an option, especially since every field will be filtered using associated setter (see below), hovewer there is much more convinient way to perform mass assignment - `setFields` method.
Such method is allowed to use user data as source (for example directly from request POST), hovewer, entity must be configured previously to specify what fields can be set and what can not.

```php
//Get entity data from query
$entity->setFields($this->input->query);
```

> Method can accept array or Traversable object (meaning you can even pass one entity as source of items for another entity - such thing used in spiral `RequestFilter->populate()` method).

You have to use set field method, or static method "create" of ORM and ODM entities to create entity based on user input.
To configure entity and specify what fields can be set we will use specialized behaviour property **fillable**. Let's try to update our entity, to allow filling "name": 

```php
class DemoEntity extends \Spiral\Models\DataEntity
{
    /**
     * @var array
     */
    protected $fillable = ['name'];

    /**
     * @param array $fields
     */
    public function __construct(array $fields)
    {
        //No setters/accessors will be called
        $this->fields = $fields;
    }
}
```

Now, we can set entity name by entering it's value in our browser url query part. 

> There is second entity property **secured** which specifies what fields are **not allowed** to be set, by default it equals to '\*' - meaning no fields can be set unless specified in **fillable** property, set secured property as empty array to make every property fillable (if you really need that).

### isFillable
DataEntity controls mass assigment access using protected method `isFillable`, you can ovewrite it to define custom field access logic.
> Tip: ORM and ODM models can inherit values of fillable and secured properties from it's parents.

## Setters (filter functions)
In many cases you might want to filter value assigned to some specified field, for example to perform type casting or some value manipulations. You can either write your own access method like "setName($name)" or use specialized entity behaviour - **setters**. Such behaviour described in setters property and applied to field inside `setField` and `setFields` method. Let's try to apply some filter for our "name" and "another" fields.

```php
class DemoEntity extends \Spiral\Models\DataEntity
{
    protected $fillable = ['name'];

    protected $setters = [
        'name'    => ['self', 'uppercase'],
        'another' => 'intval'
    ];

    /**
     * @param array $fields
     */
    public function __construct(array $fields)
    {
        //No setters/accessors will be called
        $this->fields = $fields;
    }

    protected function uppercase($name)
    {
        //We can use "self" keyword to call such
        //method in valid object context
        return strtoupper($name);
    }
}
```

> As you can see you can any valid `call_user_function` callback to descrive setters. Please note, setters is filter functions, you should not execute setField inside setter method as it can be used in many places, return filtered value instead. 

Now, no matter how we trying to assign value of desired field, it will always be casted to desired value:

```php
public function index(Database $default)
{
    $entity = new \DemoEntity([
        'name'    => 'value',
        'another' => 123
    ]);

    $entity->setFields($this->input->query);

    //Another can not be filled using mass assignment
    $entity->setField('another', '12345');

    dump($entity);
}
```

> Setters are extremelly halpful when you want to store your entity data with preserved field types (for example for MongoDB).

## Getters (filter functions)
Similar to setters you can define set of filters to be executed inside `getField` and `getFields` methods. 

```php
class DemoEntity extends \Spiral\Models\DataEntity
{
    protected $fillable = ['name'];

    protected $setters = [
        'name' => 'strtoupper'
    ];

    protected $getters = [
        'name' => 'strtolower'
    ];

    /**
     * @param array $fields
     */
    public function __construct(array $fields)
    {
        //No setters/accessors will be called
        $this->fields = $fields;
    }
}
```

Now our entity will store name always in uppercase form, but lowercased value will be returned when you will try to read such value.

> Getters are more rare than setters, hovewer they are useful with loosely typed databases (MySQL, SQLite and etc), for example boolean value stored in MySQL will be returned as "1" (string one), using getters can help us to ensure it's always boolean.

## Accessors

```php
interface AccessorInterface extends ValueInterface, \JsonSerializable
{
    /**
     * Accessors creation flow is unified and must be performed without Container for performance
     * reasons.
     *
     * @param mixed  $data
     * @param object $parent
     * @throws AccessorExceptionInterface
     */
    public function __construct($data, $parent);

    /**
     * Must embed accessor to another parent model. Allowed to clone itself.
     *
     * @param object $parent
     * @return static
     * @throws AccessorExceptionInterface
     */
    public function embed($parent);

    /**
     * Change mocked data.
     *
     * @param mixed $data
     * @throws AccessorExceptionInterface
     */
    public function setData($data);

    /**
     * Serialize mocked data to be stored in database or retrieved by user.
     *
     * @return mixed
     * @throws AccessorExceptionInterface
     */
    public function serializeData();
}
```

> Attention, ORM and ODM declares their own accessor interfaces with additional methods and features. 

## Public Data

## Converting Entity to JSON

## Raw Model Data

## Validations

## Magic Methods

In addtion to that, every data entity uses getFields method as source for array iterator, as result we can do something like that:

```php
foreach ($entity as $field => $value) {
    dump($field);
    dump($value);
}
```

### Reserved Names
Following field names are reserved for model behaviour definition and can not be accessed using magic get/set **inside** model:

Field      | Description 
---        | ---  
hidden     | List of fields must be hidden from publicFields() method.
fillable   | Set of fields allowed to be filled using setFields() method.
secured    | List of fields not allowed to be filled by setFields() method.
setters    | Field setters.
getters    | Field getters.
accessors  | Accessors used to mock field data and filter every request thought itself.
fields     | Entity data.
errors     | Validation errors.
**schema** | Used by ORM, ODM and RequestFilter entities to describe model behaviour.
indexes    | ORM and ODM only, set of indexed to be created in related table/collection.
defaults   | ORM and ODM only, set of default values for model fields.
database   | ORM and ODM only, daatabase name associated with model.
table      | ORM only, table name associated with model.
collection | ODM only, collection name associated with model.
orm        | ORM components, only for Record models.
odm        | ODM component, only for Document models.
parent     | Parent Document/Composition, for ODM models only.

> You are still able to use `getField` and `setField` methods without limitations.

## DataEntity Implementations
ORM, ODM, RequestFilter

## Events
