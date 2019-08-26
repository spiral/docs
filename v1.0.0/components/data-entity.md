# DataEntity Model
Most of Spiral's classes like ORM, ODM and HTTP request models extend one common parent - the `DataEntity`/`EntityInterface` model. This class is one of the most basic application cells used to provide a set of wrappers and operations related to the array dataset and provide unified way to transfer data between modules.

## Purpose
DataEntity model is responsible for mocking up and providing access to the array hash - fields that use a set of getters, setters and accessors. In addition, this model provides a way to apply a mass assign to fields (using black/white lists), get all model fields as one array, get a list of all "public" fields (that can be sent to client using a whitelist) and can be converted into JSON. In addition, the model support [field validation](/v1.0.0/componentsents/validation.md) is based on a set of defined rules.

## Interfaces
Because the DataEntity class is used widely across the Spiral framework, you may want to create your own code to communicate with it. Instead of using DataEntity class by itself, you can stick with it's primary interface:

```php
interface EntityInterface extends \ArrayAccess
{
    /**
     * Check if field known to entity, field value can be null!
     *
     * @param string $name
     *
     * @return bool
     */
    public function hasField(string $name): bool;

    /**
     * Set entity field value.
     *
     * @param string $name
     * @param mixed  $value
     *
     * @throws EntityExceptionInterface
     */
    public function setField(string $name, $value);

    /**
     * Get value of entity field.
     *
     * @param string $name
     * @param mixed  $default
     *
     * @return mixed
     *
     * @throws EntityExceptionInterface
     */
    public function getField(string $name, $default = null);

    /**
     * Update entity fields using mass assignment. Only allowed fields must be set.
     *
     * @param array|\Traversable $fields
     *
     * @throws EntityExceptionInterface
     */
    public function setFields($fields = []);

    /**
     * Get entity field values.
     *
     * @return array
     *
     * @throws EntityExceptionInterface
     */
    public function getFields(): array;
}
```

## Entity Fields
Every entity must work with a set of mocked up fields. These fields are represented as an associated array and stored in the private property "fields". Knowing this, we can create a simple entity used to mock up some data array:

```php
protected function indexAction()
{
    $entity = new DataEntity([ 
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
To get the value of an individual field, let's use the method `getField`. This method can accept the second parameter to specify the default value if the field doesn't exist.

```php
dump($entity->getField('name'));
dump($entity->getField('undefined', 'DEFAULT VALUE'));
```

### Get all Entity fields
In many cases, we would like to get every model field in an array form. We can use the method `getFields` to do this:

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

## Magic methods
DataEntity class also includes `__get` and `__set` functions aliased with [get/set]Field methods, use it to access properties in a short way:

```php
$entity = new \DemoEntity([
    'name'    => 'value',
    'another' => 123
]);

dump($entity->name);
```

## Mass Assignment
In almost all cases, you will need to set up multiple model fields at once. Setting fields one by one may look like an option, especially since every field will be filtered using associated setter (see below), but there is a much easier way to perform mass assignment - the `setFields` method.

This method uses user data as a source (for example, directly from request POST). However, the entity must be previously configured to specify what fields can and/or can't be set (see below).

```php
//Get entity data from query
$entity->setFields($this->input->query);
```

Now, we can set the entity name by entering it's value in our browser window. 

### isFillable
DataEntity controls the mass assigment access using the protected method `isFillable`. You can overwrite it to define a custom field access logic.

In order to define what fields are fillable or secured for your DataEntity model, extend it and overwrite SECURED or FILLABLE constants:

```php
class MyEntity extends DataEntity
{
    const FILLABLE = ['name', 'email'];
}
```

Or unprotect all fields for mass assignment:

```php
class MyEntity extends DataEntity
{
    const SECURED = []; //By default "*"
}
```

> Tip: ORM and ODM models can inherit the values of fillable and secured properties from it's parents. You can also check what fields are public and fillable in the ORM and ODM models via a set of inspect commands in the CLI toolkit.

## Setters (filter functions)
You might want to filter the value assigned to some specific field. For example, when you perform type casting or some value manipulations. You can either write your own access method like "setName($name)" or use a specialized entity behaviour - **setters**. This behaviour is described in the setters property and applied to the field inside `setField` and `setFields` method. Let's try to apply some filter for our "name" and "another" field.

```php
class DemoEntity extends DataEntity
{
    const FILLABLE = ['name'];

    const SETTERS = [
        'name'    => ['self', 'uppercase'],
        'another' => 'intval'
    ];

    protected function uppercase($name)
    {
        //We can use "self" keyword to call such
        //method in valid object context
        return strtoupper($name);
    }
}
```

Now, no matter how we try to assign the value of a desired field, it will always be set to a desired value:

```php
protected function indexAction()
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

> Setters are extremely helpful when you want to store your entity data with a preserved field types (for example for MongoDB).

## Getters (filter functions)
Similar to setters, you can define a set of filters that can be executed inside the `getField` and `getFields` methods. 

```php
class DemoEntity extends DataEntity
{
    const FILLABLE = ['name'];

    const SETTERS = [
        'name' => 'strtoupper'
    ];

    const GETTERS = [
        'name' => 'strtolower'
    ];
}
```

Now our entity will always store the name in uppercase form. Note that you will see a lowercase value returned when you read such a value.

> Getters are more rare than setters. However, they are very useful with loosely typed PDO responses (MySQL, SQLite etc). For example, any boolean value stored in MySQL will be returned as "1" (string one, exactly). Using getters can help us to ensure it's always boolean.

## Accessors
Separately from setters and 

```php
interface AccessorInterface extends \JsonSerializable
{
    /**
     * Change value of accessor, no keyword "set" used to keep compatibility with model magic
     * methods. Attention, method declaration MUST contain internal validation and filters, MUST NOT
     * affect mocked data directly.
     *
     * @see packValue
     *
     * @param mixed $data
     *
     * @throws AccessException
     */
    public function setValue($data);

    /**
     * Convert object data into serialized value (array or string for example).
     *
     * @return mixed
     *
     * @throws AccessException
     */
    public function packValue();
}
```

> Every entity is accessor as well and `setValue` method mapped to `setFields`.

### Writing an Accessor
Let's create simple accessor with extra method to access stored entity value: 

```php
class NameAccessor implements \Spiral\Models\AccessorInterface
{
    /**
     * @var string
     */
    private $name = '';

    //DataEntity constructor agreement
    public function __construct($data)
    {
        $this->name = $data;
    }
    
    public function setValue($data)
    {
        //Always in upper form
        $this->name = strtoupper($data);
    }

    public function packValue()
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

Now we can assign this accessor to our field.

```php
class DemoEntity extends \Spiral\Models\DataEntity
{
    const FILLABLE = ['name'];

    const ACCESSORS = [
        'name' => NameAccessor::class
    ];
}
```

Let's see what value will be returned by editing the code in our controller:

```php
protected function indexAction()
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

## Converting Entity to JSON
Every data entity object can be freely converted into JSON. You can either a pack result of `getFields` into array, or try to json_encode the entity itself.

```php
protected function indexAction()
{
    $entity = new \DemoEntity([
        'name'    => 'value',
        'another' => 123
    ]);

    $entity->setFields($this->input->query);
    
    return $entity;
}
```

> Feel free to overwrite `jsonSerialize` to implement your own packing method.

## Raw Model Data
If you want to get the entity fields in an array form, bypassing all getters and accessors, you can use the method `packValue`. This method is widely used in ORM and ODM to send the entity fields into database. Be VERY careful overwriting it.

```php
dump($entity->packValue());
```