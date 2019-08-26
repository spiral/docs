# Accessors and Filters
To automatically typecast your value into specific format use DataEntity getters, setters and accessors.

## Filters and Accessors
ODM will automatically associate appropriate setter for your fields in order to ensure that data is properly type-casted before storing in database, locate default association mapping in `app/config/schemas/documents`:

```php
/*
     * Set of mutators to be applied for specific field types.
     */
    'mutators' => [
        'int'      => ['setter' => 'intval'],
        'float'    => ['setter' => 'floatval'],
        'string'   => ['setter' => 'strval'],
        'bool'     => ['setter' => 'boolval'],

        //Automatic casting of mongoID
        'ObjectID' => ['setter' => [ODM::class, 'mongoID']],

        'array::string'    => ['accessor' => ODMAccessors\StringArray::class],
        'array::objectIDs' => ['accessor' => ODMAccessors\ObjectIDsArray::class],
        'array::integer'   => ['accessor' => ODMAccessors\IntegerArray::class],

        'timestamp' => ['accessor' => Accessors\UTCMongoTimestamp::class],
        /*{{mutators}}*/
    ],
    /*
     * Mutator aliases can be used to declare custom getter and setter filter methods.
     */
    'aliases'  => [
        //Id aliases
        'MongoId'                        => 'ObjectID',
        'objectID'                       => 'ObjectID',
        \MongoDB\BSON\ObjectID::class    => 'ObjectID',

        //Timestamps
        \MongoDB\BSON\UTCDateTime::class => 'timestamp',

        //Scalar typ aliases
        'integer'                        => 'int',
        'long'                           => 'int',
        'text'                           => 'string',

        //Array aliases
        'array::int'                     => 'array::integer',
        'array::MongoId'                 => 'array::objectIDs',
        'array::ObjectID'                => 'array::objectIDs',
        'array::MongoDB\BSON\ObjectID'   => 'array::objectIDs'

        /*{{mutators.aliases}}*/
    ]
```

## Custom Filters
To associate filter with entity value use constants `SETTERS` and `GETTERS`:

```php
const SETTERS = [
    'name' => 'ucfirst'
];
```

## MongoId fields
Any MongoId or ObjectId field will be automatically converted into proper MongoDB object:

```php
$user->group_id = '507f1f77bcf86cd799439011';

dump($user->group_id);
```

## Scalar Arrays
If you wish to store array of values in your object use scalar array definition with support of atomic operations:

```php
class User extends Document
{
    const SCHEMA = [
        //...
        'tags' => array['string']
    ];
}
```

```php
$user->tags->add('tag');
```

> All scalar are solid by default, disable solid state to work with data in atomic way.

```php
$user->tags->solidState(false);
$user->tags->push('tag1');
```

Scalar arrays will typecast your value into proper format, following types are available:

```php
  'tags'    => array['string'],
  'numbers' => array['integer'],
  'ids'     => array[ObjectId::class]
```

## Mongo Datetime
Set your field type as `timestamp` or `UTCMongoTimestamp` to enable embedded MongoTimestamp accessor:

```php
class User extends Document
{
    const SCHEMA = [
        //...
        'timeRegistered' => UTCMongoTimestamp::class
    ];
}
```

```php
$user->timeRegistered = 'now';
```

> All date times are stored in UTC timezone in your database.

## Timestamps Trait
Use `TimestampsTrait` to automatically add and update timeCreated and timeUpdated properties of your model.

```php
class User extends Document
{
    use TimestampsTrait;
}
```