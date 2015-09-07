# DataEntity Model
Most of spiral classes like ORM, ODM and HTTP request models extends one common parent - DataEntity model. Such class is one of the basic application cells used to provide set of wrappers and operations related to array dataset.

## Purpose
DataEntity model reponsible for mocking and providing access to array dataset - fields (associated array) using set of getters, setters and accessors. In addition to that model provides way to apply mass assigment to fields (using black/white lists), get all model fields as one array, get list of "public" fields (allowed to be send to client using whitelist) and to be converted into JSON. In addition to that model support [field validation] (validation.md) based on set of defined rules.

## Interfaces
Due DataEntity class used widely across spiral framework you might create your own code to communicate with it. Intead of using DataEntity class itself you can stick to it's primary interface:

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

### Set Model field
To set value of desired model field we can use obviously named method `setField`:

```php
$entity->setField('thing', 199.00);
dump($entity->getFields());
```

## Mass Field assignment

isFillable

## Getters

## Setters

## Accessors

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
Following field names are reserved for model behaviour definition and can not be accessed using magic get **inside** model:

Field      | Description 
---        | ---  
hidden     | List of fields must be hidden from publicFields() method.
fillable   | Set of fields allowed to be filled using setFields() method.
secured    | List of fields not allowed to be filled by setFields() method. By default no fields can be set. Replace with and empty array to allow all fields.
setters    | Field setters.
getters    | Field getters.
accessors  | Accessors used to mock field data and filter every request thought itself.
fields     | Entity data.
errors     | Validation errors.
schema     | Used by ORM, ODM and RequestFilter entities to describe model behaviour.
indexes    | ORM and ODM only, set of indexed to be created in related table/collection.
defaults   | ORM and ODM only, set of default values for model fields.
datatabe   | ORM and ODM only, daatabase name associated with model.
table      | ORM only, table name associated with model.
collection | ODM only, collection name associated with model.
orm        | ORM components, only for Record models.
odm        | ODM component, only for Document models.
parent  | Parent Document/Composition, for ODM models only.

> You are still able to use `getField` and `setField` methods without limitations.

## DataEntity Implementations
ORM, ODM, RequestFilter

## Events

## Traits

## Entity Reflection

