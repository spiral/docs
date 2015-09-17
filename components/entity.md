# DataEntity Model
Most of Spiral classes like ORM, ODM and HTTP request models extend one common parent - DataEntity model. This class is one of the basic application cells used to provide a set of wrappers and operations related to the array dataset.

## Purpose
DataEntity model is responsible for mocking up and providing access to the array dataset - fields (associated with the array) that use a set of getters, setters and accessors. In addition, this model provides a way to apply mass assigment to fields (using black/white lists), get all model fields as one array, get a list of all "public" fields (that can be sent to client using a whitelist) and to be converted into JSON. In addition, the model support [field validation] (validation.md) is based on a set of defined rules.

## Interfaces
Because the DataEntity class is used widely across the Spiral framework, you may want to create your own code to communicate with it. Instead of using DataEntity class by itself, you can stick with it's primary interface:

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
     * Update entity fields using mass assignment. Only allowed fields should be set.
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

Because EntityInterface extends ValidatesInterface, we have to show it's content as well:

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
     * List of errors associated with parent field. Every field can only have one error assigned.
     *
     * @param bool $reset Force re-validation.
     * @return array
     */
    public function getErrors($reset = false);
}
```

## Entity Fields
Every entity must work with a set of mocked up fields. These fields can be represented as an associated array and stored in the property "fields". Knowing this, we can create a simple entity used to mock up some data array:

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

This entity provides the ability to set fields using a contructor method (Pay attention, that the fields will be set without any filtering). Now we can use this entity somewhere in our code:

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

Once we have constructed our entity, we can walk through a set of methods designed to work with our field values.

### Check field existence
If the desired field exists in your model, you can use the method `hasField`.

```php
dump($entity->hasField('name'));
dump($entity->hasField('undefined'));
```

### Get individual field
To get the value of an individual field, let's try the method `getField`. This Method can accept the second parameter to specify the default value if the field doesn't exist.

```php
dump($entity->getField('name'));
dump($entity->getField('undefined', 'DEFAULT VALUE'));
```

### Get all Entity fields
In many cases, we would like to get every model field in an array form. We can use the method `getFields` for these purposes:

```php
dump($entity->getFields());
```

> The resulting array will include the field accessors (see below). Plus every field will be passed through it's associated getter (see below).

### Set Model field
To set the value of a desired model field, we can use `setField`:

```php
$entity->setField('thing', 199.00);
dump($entity->getFields());
```

## Mass Assignment
In almost all cases, you will need to set up multiple model fields at once. Setting fields one by one may look like an option, especially since every field will be filtered using associated setter (see below), but there is much easier way to perform mass assignment - the `setFields` method.
This method lets you use user data as a source (for example, directly from request POST). Hovewer, the entity must be previously configured to specify what fields can and can't be set.

```php
//Get entity data from query
$entity->setFields($this->input->query);
```

> Method can accept an array or Traversable object (meaning you can even pass one entity as a source of items for another entity - this is used in spiral `RequestFilter->populate()` method).

You have to use a set field method or a static method "create" of ORM and ODM entities in order to create an entity based on user input.
To configure, this entity and specify what fields can be set, we will use the specialized behaviour property **fillable**. Let's try to update our entity, to allow to fill in "name": 

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

Now, we can set the entity name by entering it's value in our browser window. 

> There is a second entity property **secured** which specifies what fields are **not allowed** to be set. By default, this is equal to '\*' - meaning that no fields are set unless specified in **fillable** property. 

### isFillable
DataEntity controls the mass assigment access using the protected method `isFillable`. You can ovewrite it to define a custom field access logic.
> Tip: ORM and ODM models can inherit values of fillable and secured properties from it's parents. You can also check what fields are public and fillable in ORM and ODM models via a set of inspect commands in CLI toolkit.

## Setters (filter functions)
You might want to filter the value assigned to some specific field, for example, to perform type casting or some value manipulations. You can either write your own access method like "setName($name)" or use specialized entity behaviour - **setters**. Such behaviour is described in the setters property and applied to field inside `setField` and `setFields` method. Let's try to apply some filter for our "name" and "another" fields.

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

> You can use any valid `call_user_function` callback to describe setters. Please note, setters is a filter function. You should not execute setField inside setter method as it can be used in many places. Use the return filtered value instead. 

Now, no matter how we try to assign value of desired field, it will always be set to desired value:

```php
public function index()
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

> Setters are extremely helpful when you want to store your entity data with preserved field types (for example for MongoDB).

## Getters (filter functions)
Similar to setters you can define a set of filters to be executed inside `getField` and `getFields` methods. 

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

Now our entity will alway store name in uppercase form, but you will see a lowercase value returned when you read such a value.

> Getters are more rare than setters. Hovewer they are very useful with loosely typed databases (MySQL, SQLite etc). For example, boolean value stored in MySQL will be returned as "1" (string one). Using getters can help us to ensure it's always boolean.

## Accessors
The Spiral Entity provides additional way to manage field value - Accessors. Accessor is specified object which is responsible for manipulations with mocked value, the easiest example - timestamp value accessor which can represent numeric value as DataTime (spiral uses [Carbon] (https://github.com/briannesbitt/Carbon)).

Accessors are not really useful outside of ORM and ODM models where we can not only control value, but also tell database how this value must be stored. Hovewer, let's check base interface:

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

> Accessor objects will be returned from getField and getFields methods instead of value.

### Writing an Accessor
To better undestand how accessor works (it will also help us with next guide sections) let's try to write our own accessor.

```php
class NameAccessor implements \Spiral\Models\AccessorInterface
{
    /**
     * @var string
     */
    private $name = '';

    public function __construct($data, $parent)
    {
        $this->name = $data;
    }

    public function embed($parent)
    {
        //We do not need to store parent in this
        //specific accessor
        return clone $this;
    }

    public function setData($data)
    {
        //Always in upper form
        $this->name = strtoupper($data);
    }

    public function serializeData()
    {
        return $this->name;
    }

    /**
     * This is why we need accessors.
     *
     * @return string
     */
    public function niceName()
    {
        return ucfirst(strtolower($this->name));
    }

    public function jsonSerialize()
    {
        return $this->niceName();
    }
}
```

Now we can assign such accessor to our field.

```php
class DemoEntity extends \Spiral\Models\DataEntity
{
    protected $fillable = ['name'];

    protected $accessors = [
        'name' => NameAccessor::class
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

Let's see what value will be returned by editing code in our controller:

```php
public function index()
{
    $entity = new \DemoEntity([
        'name'    => 'value',
        'another' => 123
    ]);

    $entity->setFields($this->input->query);
    dump($entity->getFields());

    dump($entity->getField('name')->niceName());

    //Using magic getter
    dump($entity->name->niceName());
}
```

> Accessors are also involved in packing entity in json form (see below).

## Public Data
Usually you might want to hide some fields from being show to user, most often scenariou when you want to send your object in JSON form. To hide some fields from being published we can use property "hidden", let's try to hide "another" field:

```php
class DemoEntity extends \Spiral\Models\DataEntity
{
    protected $hidden = ['another'];

    protected $fillable = ['name'];

    protected $accessors = [
        'name' => NameAccessor::class
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

Now, we can get list of public fields by executing method `publicFields` of our entity:

```php
public function index()
{
    $entity = new \DemoEntity([
        'name'    => 'value',
        'another' => 123
    ]);

    $entity->setFields($this->input->query);
    dump($entity->publicFields());
}
```

> ORM and ODM models can inherit hidden fields from it's parents. You can check what fields are public and fillable in ORM and ODM models via set of inspect commands in CLI toolkit.

## Converting Entity to JSON
Every data entity object can be freely converted into JSON, you can either pack result of `getFields` or `publicFields` into array, or try to json_encode entity itself.
When entity beign encoded, only it's public fields will be included into resulted JSON. We are going to use ability of HttpDispatcher to convert all JsonSerializable objects into json response:

```php
public function index()
{
    $entity = new \DemoEntity([
        'name'    => 'value',
        'another' => 123
    ]);

    $entity->setFields($this->input->query);
    
    return $entity;
}
```

You might notice that only name were included into json, in addition to that your name will be capitalized as it's value will be packed into json form by related accessor.

## Raw Model Data
If you wish to get entity fields in array form bypassing all getters and accessors you can use method `serializeData`, such method widely used in ORM and ODM to send entity fields into database - be very careful overwriting it.

```php
dump($entity->serializeData());
```

## Validations
Every DataEntity automatically includes ability to validate it's data. Validation rules must be described in **validates** property and might also include error messages embraced with `[[]]`, such messages will be automatically localized into active translator language. Let's try to specify few validation rules for our "name" field.

```php
class DemoEntity extends \Spiral\Models\DataEntity
{
    protected $hidden = ['another'];

    protected $fillable = ['name'];

    protected $accessors = [
        'name' => NameAccessor::class
    ];

    protected $validates = [
        'name' => [
            'notEmpty',
            ['string::longer', 3, 'message' => '[[Name is too short (min 3 symbols).]]']
        ]
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

Now we can check if our entity is valid and get error messages in our controller code (you can play with url query to produce different values):

```php
public function index()
{
    $entity = new \DemoEntity([
        'name'    => 'value',
        'another' => 123
    ]);

    $entity->setFields($this->input->query);

    dump($entity->isValid());
    dump($entity->getErrors());
}
```


You can read more about validation rules [here] (validation.md).

> There is specialized DataEntity used to validate incoming request - [RequestFilter] (/http/filters.md).

### Complex validations
If you want to perform context aware validation you can ovewrite entity method `validate` and add your own custom logic:

```php
protected function validate($reset = false)
{
    parent::validate($reset);

    if (mt_rand(0, 1)) {
        $this->setError('random', 'Some random error');
    }
}
```

## Magic Methods
In addition to getField/getField methods data entity exposes set of magic getters, setters and method to access your fields. For example you can read or write any entity value using `__get` or `__set` methods:

```php
$entity->name = 'new name';
dump($entity->name->niceName());
```

Besides that, entity will use [Doctrine Inflector] (https://github.com/doctrine/inflector) to create set of magic getter and setter methods (attention, this is not the same as getter/setter filters), as result you can access to your fields like that:

```php
$entity->setName('new name');
dump($entity->getName()->niceName());
```

> Attention, DataEntity will convert all field names into camelCase notation, however ORM will use tableize form (field_name) based on set/get function name.

In addtion to that, every data entity uses `getFields` method as source for array iterator, as result we can do something like that:

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
As mentioned in section disclaimer DataEntity is common parent for few important spiral models, such models includes ORM `Record`, ODM `Document` and Http `RequestFilter`. As result you can apply same priciples for filtering, validations and etc for any of this model.

> Attention, ORM and ODM models uses static model cache (you have to run `spiral up` to update it), as result you are able to inherit entity behaviours from it's parents including fillable, hidden fields, validations and their messages and etc. 

## Events
Every data entity model support event dispatching associated with model class, you can find list of available events and their arguments in ORM and ODM sections.
