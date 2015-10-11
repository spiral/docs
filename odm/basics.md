## ODM Component, Basics
The Spiral ODM component has some similaries to the [ORM engine] (/orm/basics.md) as they are both based on the [**DataEntity**] (/components/entity.md) model and [Behaviour Schemas] (/schemas.md). However, the ODM removes the  term "relation" and replaces it with the classical [compositions and aggregations] (https://en.wikipedia.org/wiki/Object_composition) definition. In addition, the ODM classes support inheritance and can also can be embedded into ORM as JSON objects. 

The ODM component was designed mainly to work with a MongoDB database and include support for atomic operations. However, the `DocumentEntity` class and it's compositions can be used outside of Mongo to create the OOP data representation for various structures (XML files, API responses, etc).

The Spiral ODM does not use entity cache like in ORM. Instead it uses "streaming like" functionality which can be used to process huge amounts of data using Document models.

Check extended usage for DocumentEntity, Compompositions, Aggreagations and Inheritance [here] (oop.md).

## Mongo Connection and Databases
With ORM, you are not required to spend a ton of time trying to configure your ODM component. The only thing you have to make sure of (if you want to store your data in MongoDB) is that the Mongo connection is properly set. To ensure that, simply check the configuration file "odm.php" located inside your "applicatio/config" directory:

```php
'default'   => 'default',
'aliases'   => [
    'database' => 'default',
    'db'       => 'default',
    'mongo'    => 'default'
],
'databases' => [ 
    'default' => [
        'server'    => 'mongodb://localhost:27017',
        'profiling' => MongoDatabase::PROFILE_SIMPLE,
        'database'  => 'spiral-empty',
        'options'   => [
            'connect' => true
        ]
    ]
],
```

Configration is very similar to the [DBAL] (/database/overview.md) and fully supports database aliases and controllable/contextual injections. If you want to get direct access to the `MongoDatabase` in your controller, simply request it using `ODM->db()` factory method or via `MongoDatabase` dependency using a database alias:

```php
public function index(MongoDatabase $database, ODM $odm)
{
    dump($database->getCollectionNames());
    dump($odm->db('default'));
}
```

> `MongoDatabase` class extends original `MongoDB` class so you can use it as regular mongo database.

## Document
To define your first model related to MongoDB collection, we only have to create/generate a declaration of `Document` class with the desired set of fields listed in the model **schema**. We can either create this model manually or generate it using the console command "create:document". Let's create our first model User. Using  ORM we are able to pre-define a set of desired fields: "create:document user -f id:MongoId -f name:string -f email:string -f balance:float". The resulting entity will be located in the application/classes/Database/User.php:

```php
class User extends Document 
{
    /**
     * @var array
     */
    protected $fillable = [
    ];

    /**
     * Entity schema.
     * 
     * @var array
     */
    protected $schema = [
        'id'      => 'MongoId',
        'name'    => 'string',
        'email'   => 'string',
        'balance' => 'float'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'    => ['notEmpty'],
        'email'   => ['notEmpty'],
        'balance' => ['notEmpty']
    ];
}
```

You might notice that the declaration is very similar to ORM Record. However, the ODM component does not affect the MongoDB schema. Its just stores it in your application. 

> You can read more about entity configuration (fillable, hidden, secured and validates properties) in the section declared to [DataEntity] (components/entity.md).

#### Schema
The ODM classes Document (and `DocumentEntity`, see in extended usage) only lets you manipulate a set of fields described in the model schema. The declaration of fields can be performed by simply creating an associated array between the field name and it's type. 

```php
protected $schema = [
    'id'      => 'MongoId',
    'name'    => 'string',
    'email'   => 'string',
    'balance' => 'float'
];
```

You can associate your field with any desired type. However, since MongoDB does not allow dynamic typing we have to make sure that every value listed in the schema is always typecased into a specified type. You can solve this problem by utilizing the model getters and setters or simply using a set of pre-declared types. Check the ODM config:

```php
'schemas'   => [
    'mutators'       => [
        'int'       => ['setter' => 'intval'],
        'float'     => ['setter' => 'floatval'],
        'string'    => ['setter' => 'strval'],
        'bool'      => ['setter' => 'boolval'],
        'MongoId'   => ['setter' => [ODM::class, 'mongoID']],
        'array'     => ['accessor' => ScalarArray::class],
        'MongoDate' => ['accessor' => Accessors\ODMTimestamp::class],
        'timestamp' => ['accessor' => Accessors\ODMTimestamp::class],
        'storage'   => ['accessor' => Accessors\StorageAccessor::class]
    ],
    'mutatorAliases' => [
    ]
]
```

As you can see, every string field will automatically get the associated setter `setval` while the schema updates.

> Schema can also contain aggregations and compositions. You can read about it below.

#### Default Values
ODM will automatically force the default values based on the typecasted null value (string => "", int => 0). You can set your own default values using the model property "defaults":

```php
protected $defaults = [
    'balance' => 100.00    
];
```

#### Associated Collection and Database
By default, Spiral will associate your `Document` model with the default ODM database and collection. The name is generated based on the class name (in our case User => users). You can alter both of these values by declaring non empty properties like `database` and `collection`.

> If you're unsure what collection will be used based on the model name - simply force `collection` for every Document. It's not going to harm anything and will make your code more readable.

#### Indexes
To declare which indexes are created in the associated collection, simply declare them in the `indexes` property of your Document. The Declaration form is similar to the MongoCollection
ensureIndexes method. Indexes will be created at the moment when the schema updates (see next).

```php
protected $indexes = [
    ['email' => 1, '@options' => ['unique' => true]],
    ['name' => 1, 'balance' => -1]
];
```

> Use "@options" key to specify the index options.

## Schema Update
Once you've created your first Document or Documents, you then have to register them inside your ODM component schema. This operation is called "schema update" and does not require any operation other then putting the console command "spiral update" or "spiral up". The schema update will automatically locate all your Document and DocumentEntity models, analyze them, create an automatic set of mutators and store the information about the compiled behaviours inside the application memory.

Only related database operation are performed while the schema is updated - ensuring that the requested indexes exist in the associated collection.

> The Spiral Framework will generate virtual documentation for your models, so your IDE will highlight every available field and composition.

#### Modifying Documents schema
Since ODM schema is not stored in the database, it can be efficiently updated without any storage change. As a result you are able to add new columns, compositions etc into your Documents at any moment. Simply declare them in your model schema and update the ODM after.

```php
protected $schema = [
    'id'      => 'MongoId',
    'name'    => 'string',
    'email'   => 'string',
    'balance' => 'float',
    'data'    => 'string'
];
```

## Create Document
Once your ODM schema has been updated, we are ready to work with our models. To save some data into our users collection let's create our first entity in our controller method:

```php
public function index()
{
    $u = new User();
    $u->name = 'Anton';
    $u->email = 'test@email.com';
    $u->balance = 99;
    $u->save();

    dump($u);
}
```

> If you are working under PHPStorm, IDE will highlight all the possible record fields for you.

![Tooltips](https://raw.githubusercontent.com/spiral/guide/master/resources/tooltips.png)

You can also create your entity using the static method which will respect all fillable and secured fields.

```php
$u = User::create($request);
```

>Attention, the create method will only return a populated entity! You have to save it by it yourself!

#### Validations
The document model follows the principles of validations described in the `DataEntity` model. If you want to create more complex validations based on some external state, you can redefine the `validate` method:

```php
/**
 * @param bool|false $reset
 * @return bool
 */
protected function validate($reset = false)
{
    parent::validate($reset);

    if ($this->hasUpdates('email') && !$this->hasError('email')) {
        //We are using array based where statement
        $selection = $this->odmCollection()->where([
            'email' => $this->email,
            '_id'    => ['$ne' => $this->_id]
        ]);

        if ($selection->count() != 0) {
            $this->setError('email', self::translate("Email must be unique."));
        }
    }

    return false;
}
```

Now our controller code will return an error related to the email field:

```php
$user = new User();
$user->name = 'Anton';
$user->email = 'test@email.com';
$user->balance = 99;
if (!$user->save()) {
    dump($user->getErrors());
}
```

This validation method will check if the email is unique. It will only do this when the email field is updated (created or changed). Now, we will get an error message associated with the model field.

Before we jump to the next step, lets try to create a few more users. In this case, we can use the static method create which accepts a set of initial model fields.

Since we have no fillable fields, this method should not work in our case (we will use the setFields method with disabled access policy - by setting the second argument as true). In order to fill our demo entities let's connect faker package.

```php
for ($i = 0; $i < 100; $i++) {
    $user = User::create()->setFields([
        'name'    => $faker->name,
        'email'   => $faker->email,
        'balance' => $faker->randomFloat(2, 0, 999)
    ], true);

    //Saving user to database
    $user->save();
}
```

> Create method will only return the populated Document entity. You have to save this entity to the database manually (simply call save() method).

## Selections
Once your collection is populated with data, it's time to select some of the Users to perform something. The Spiral ODM exposes 3 ActiveRecord like methods which you can use for these purposes: `find`, `findOne` and `findByPK`.

There is 4th method which is used inside every find method - `odmCollection`. You can use it to get access to the  `Collection` class associated with our document. You can also get the associated collecion by calling the method `odmCollection` of the ODM component.

#### FindOne
To find a document based on some conditions and sorting, we can use the static method `findOne`. This method will return null if it's unabled to locate the required document. 
The ODM component does not provide any query wrapper at the top of the MongoDB so you are able to use the pure mongo queries for your requests:

```php
//Don't worry about second argument yet, it's about pre-loading
$user = User::findOne(['email' => 'test@email.com'];
dump($user);
```

> The ODM provides only one feature of automatic conversion of the `DateTime` classes in your queries into instance of `MongoDate`.

#### FindByPK
When you want to find your document based on it's primary key value (MongoId in "_id" field), you can use the method `findByPK`. This method will behave the same way as `findOne` if no entity is found.

```php
$user = User::findByPK($mongoID);
dump($user);
```

> Method can accept both `MongoId` and string which will be automatically combined to valid id instance.

The most common option is to find the model based on the provided primary key or return client exception on failure. Spiral does not embed any application specific logic into the ODM component so you have to raise an exception on your own. This is easy to do:

```php
public function index($id)
{
    if (empty($user = User::findByPK((int)$id))) {
        throw new NotFoundException('No such user');
    }

    dump($user);
}
```

Or using entity service:

```
public function index(UserService $users, $id)
{
    if (empty($user = $users->findByPK((int)$id))) {
        throw new NotFoundException('No such user');
    }

    dump($user);
}
```

#### Find and Working with Collection
To find one or multiple document instances from desired collection, we have to first get the associated instance of `Collecion`. Then we can do this using the ODM component method `odmCollection` or via the static function `find` of our User model.

```php
$collection = User::find();
```

Class is built at top of the MongoCollection and dictates some of it's functions. You are able to set or clarify the selection query using the methods `query` or `where`:

```php
$collection = User::find()->where([
    'email' => new \MongoRegex('/test@email.com/')
]);

dump($collection->count());
```

In addition, you are able to paginate your result using the techniques described in [Pagination] (/components/pagination.md) component:

```php
$collection = User::find()->paginate(10);
```

> Check other Collection methods, such as limit, offset (skip) and sortBy.

##### Fetching Documents
Once you've configured your Collection selection with desired query, you can fetch the found document by simply iterating over collection or explicitly calling the method `getIterator`/`getCursor`, which will return an instance of `DocumentCursor`:

```php
foreach (User::find()->paginate(10) as $user) {
    dump($user);
}
```

```php
$cursor = User::find()->sortBy(['name' => 1])->paginate(10)->getCursor();

/**
 * @var DocumentCursor $cursor
 */
foreach ($cursor as $user) {
    dump($user);
}
```

##### DocumentCursor
The provided instance of `DocumentCursor` wraps the top of the `MongoCursor` and provides support for lazy creation of the collection documents (no entity cache is used). As a result, you can iterate over a huge amount of Documents without any significant memory usage. Also you can access the `MongoCursor` functionality:

```php
$cursor = User::find()->sortBy(['name' => 1])->paginate(10)->getCursor();

/**
 * @var DocumentCursor $cursor
 * @var User           $user
 */
while ($user = $cursor->getNext()) {
    dump($user);
    dump($cursor->dead());
}

dump($cursor->info());
```

##### Fetching Fields
If you wish to fetch only the specified set of fields from your collection, simply specify their names in the form of an array into the `fetchFields` method:

```php
foreach (User::find()->fetchFields(['name']) as $data) {
    dump($data);
}
```

This methods will read all the desired fields and return them in the form of an array. You can also read them in a streaming mode by utilizing the `fields` method of your `DocumentCursor`:

```php
$cursor = User::find()->getCursor()->fields([
    'email' => true
]);

while ($data = $cursor->getNext()) {
    dumP($data);
}

dumP($cursor->info());
```

> Attention, format in this case follows the [offical documentation] (http://php.net/manual/ru/mongocursor.fields.php)!

## Modify Document
Once you've selected the neccesary document (for example using `findByPK`), you can modify it's fields the same way as any other `DataEntity`. To save your updates into the database, you have to run the method save(). This method returns false if the model validations fail.

```php
public function index($id)
{
    $user = User::findByPK($id);
    $user->name = 'New Name';

    if (!$user->save()) {
        dump($user->getErrors());
    }
}
```

#### Dirty Fields and Solid State
If you dump the User models before saving them into the database, you will see the property "atomics" which demonstrates what query will be sent to the MongoDB to perform the requested updated. 

```
atomics:dynamic = array(1)
(
·    ['$set'] = array(1)
·    (
·    ·    ['name'] = string(8) New Name
·    )
)
```

As you might notice, Document sends only updated values for the name field using the atomic operation set. This approach is useful in many cases when the data is updated from many places. However if you want to save your entity data into the database as a solid dataset, you can force the entity "solid state". When the model (Document) is in a solid state, every small change will create a full dataset update.

```php
$user = User::findByPK($id);
$user->name = 'New Name';

$user->solidState(true);
dump($user);
```

Now our atomic operation will look like this:

```
atomics:dynamic = array(1)
(
·    ['$set'] = array(4)
·    (
·    ·    ['balance'] = double(5) 27.920000000000002
·    ·    ['name'] = string(8) New Name
·    ·    ['email'] = string(22) klangworth@hotmail.com
·    )
)
```

> If you want to keep our model in solid state by default, you can call this method in the overwritten model constructor. SolidState automatically applies to every model that is created.

#### Filters and Accessors
Using the ORM or regular DataEntity class, you are able to assign setters, getters and accessors to your fields. ODM component has two notable accessor we might need to check: `MongoTimestamp` and `ScalarArrar`.

##### MongoTimestamp Accessor
Based on the provided configuration, you might notice that ODM will assign MongoTimestamp accessor to every timestamp or "MongoDate" field. Let's try to add these fields into our model:

```php
protected $schema = [
    'id'             => 'MongoId',
    'name'           => 'string',
    'email'          => 'string',
    'balance'        => 'float',
    'timeRegistered' => 'MongoDate'
];
```

After the schema is updated, we are able to use that field as a Carbon instance.

```php
public function index()
{
    $user = User::findOne();
    $user->name = 'New Name';

    $user->timeRegistered->setDateTime(2015, 1, 1, 12, 0, 0);
    dump($user);

    if (!$user->solidState(true)->save()) {
        dump($user->getErrors());
    }
}
```

You can also assign time to the "timeRegistered" field. Accessor must automatically handle this using starttime function:

```php
$user->timeRegistered = 'next friday 10am';
```

##### Scalar Array Accessor
The ODM component provides one accessor which simplifies operations with hard types of scalar array and adds atomics support for these fields. To declare this field as a scalar array, simply declare it in a form of "[type]":

```php
protected $schema = [
    'id'             => 'MongoId',
    'name'           => 'string',
    'email'          => 'string',
    'balance'        => 'float',
    'timeRegistered' => 'MongoDate',
    'tags'           => ['string']
];
```

The ODM component will automatically assign accessor `Spiral\ODM\Accessors\ScalarArray` to this field. Accessor will automatically filter the array values and ensure that they are all the same type. Accessor supports the following types:
* int
* float
* string
* MongoId

Once your schema is updated, you can work with this field just like with regular array:

```php
$user = User::findOne();
$user->tags[] = 'new';

echo $user->tags->count();

foreach ($user->tags as $tag) {
    dump($tag);
}
```

This is the most important part of ScalarArray functionality and its ability to use mongo atomic opeations for it's values. However, by default the ScalarArray is always in **solidState**. If you want to apply the atomic operations to it's fields, you have to reset this state first. For example we can push the new value into our array:

```php
$user = User::findOne();
$user->tags->solidState(false)->push('new tag');
dump($user);
```

Model atomic operations look like this:

```
atomics:dynamic = array(2)
(
·    ['$push'] = array(1)
·    (
·    ·    ['tags'] = array(1)
·    ·    (
·    ·    ·    ['$each'] = array(1)
·    ·    ·    (
·    ·    ·    ·    [0] = string(7) new tag
·    ·    ·    )
·    ·    )
·    )
)
```

> There is also `pull`, `addToSet` operations.

## Delete Documents
You can also delete any existing Document by executing it's method "delete". 

## Events and Traits
In the DataEntity model, Document declares a few [events] (/components/events.md) that you can track or change the models behaviour:

Event                   | Context                                   | Return | Description
---                     | ---                                       | ---    | ---
setFields               | $fields                                   | *      | Called inside `setFields` method.
publicFields            | $fields                                   | *      | Called before return in `publicFields` method.
jsonSerialize           | $publicFields                             | *      | Called while packing model into json.
validation              | -                                         | -      | Before validation.
validated               | $errors                                   | *      | After validation.
**describe** (static)   | [$property, $value, EntitySchema $schema] | value  | Called while model analysis. Such evet can be used to redefine or alter schema, validations, mutators etc.
created                 | -                                         | -      | Called inside static method "create" after assigning model fields.
collection              | Collection $selector                      | *      | Called by "find" and other selection methods.
saving                  | -                                         | -      | Before model data saved into database (model is only created). 
saved                   | -                                         | -      | After model data saved into database.
updating                | -                                         | -      | Before model data updated in database (model already exist).
updated                 | -                                         | -      | After model data updated in database.
deleting                | -                                         | -      | Before model deleted from database.
deleted                 | -                                         | -      | After model deleted from database.

In most cases, you don't need to handle any of event since `save` and `delete` methods can be easily overwriten. However, events can be very useful when creating a set of traits used to modify Document schema and update/save behaviour (see next).

#### Timestamps Trait
One of the most common Document traits you might want to use is the `TimestampsTrait`. This trait handles the "saving", "updating" and "describe" of events to alter Document schema and updates two additional fields "timeCreated" and "timeUpdated" (both type MongoDate). Our final demo Document `User` might look like this:

```php
class User extends Document
{
    use TimestampsTrait;
    
    /**
     * @var array
     */
    protected $fillable = [

    ];

    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'id'             => 'MongoId',
        'name'           => 'string',
        'email'          => 'string',
        'balance'        => 'float',
        'timeRegistered' => 'MongoDate'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'    => [
            'notEmpty'
        ],
        'email'   => [
            'notEmpty'
        ],
        'balance' => [
            'notEmpty'
        ]
    ];

    protected $defaults = [
        'balance' => 100.00
    ];

    /**
     * @param bool|false $reset
     * @return bool
     */
    protected function validate($reset = false)
    {
        parent::validate($reset);

        if ($this->hasUpdates('email') && !$this->hasError('email')) {
            //We are using array based where statement
            $selection = $this->odmCollection()->where([
                'email' => $this->email,
                '_id'   => ['$ne' => $this->_id]
            ]);

            if ($selection->count() != 0) {
                $this->setError('email', self::translate("Email must be unique."));
            }
        }

        return false;
    }
}
```

## Services and Controllers
While working with the ODM models you are able to call the `find`, `findByPK` and `save` methods inside your controllers directly. However, spiral provides the ability to pre-generate specific [Service] (/application/services.md) which can help you abstract any database specific operations from your controllers code. 

You can scaffold this class using the console command "create:service user -e user" and your code may look like this:

```php
class UserService extends Service  implements SingletonInterface
{
    /**
     * Declares to IoC container that class must be treated as Singleton.
     */
    const SINGLETON = self::class;

    /**
     * Create new blank User. You must save the entity using the save method.
     * 
     * @param array|\Traversable $fields Initial set of fields.
     * @return User
     */
    public function create($fields = [])
    {
        return User::create($fields);
    }

    /**
     * Save User instance.
     * 
     * @param User $user
     * @param bool  $validate
     * @param array $errors Will be populated if saving fails.
     * @return bool
     */
    public function save(User $user, $validate = true, &$errors = NULL)
    {
        if ($user->save($validate)) {
            return true;
        }
        
        $errors = $user->getErrors();
        
        return false;
    }

    /**
     * Delete User.
     * 
     * @param User $user
     */
    public function delete(User $user)
    {
        $user->delete();
    }

    /**
     * Find User it's primary key.
     * 
     * @param mixed $primaryKey
     * @return User|null
     */
    public function findByPK($primaryKey)
    {
        return User::findByPK($primaryKey);
    }

    /**
     * Find User using set of where conditions.
     * 
     * @param array $where
     * @return User[]|Selector
     */
    public function find(array $where = [])
    {
        return User::find($where);
    }
}
```

As a result, you can use this service inside your controllers to avoid the static methods of Document model (they can always be removed from the service too):

```php
public function index(UserService $users)
{
    $user = $users->findByPK(1);
    $user->inc('balance', 1.6);

    if (!$users->save($user)) {
        dump($user->getErrors());
    }
}
```

Services like this are the best place to locate custom selection methods (like `findActive`) or even alter the delete method to implement "soft deletes".

> You can also generate the CRUD controller to work with any existing entity service using command 'create:controller name -s service'.

## Inheritance and Abstract Documents
Since Spiral ODM uses static analysis, there are no real limitations on how you can create your models. As a result you can declare the **abstract** Document with it's schema, validations etc and later extend this class in your application models. This technique can be very useful when writing [modules] (/components/modules.md).

> When extending, ODM will merge schemas and other properties of the Document and it's parent.

**You can also use the data driven model inheritance, check [this article] (oop.md).**

## Inspections
While running the schema update (spiral up) command, you might notice text which contains a list of the inspected entities and resulting rating. This information provided by spiral **Entity Inspector** analyses ORM and ODM schemas to find unprotected or blacklisted fields. We can get more details by running inspection on selected entity:

```
> spiral inspect:entity Database/User
+-----------------+-----------+----------+----------+-----------+----------------+
| Field           | Rank      | Fillable | Filtered | Validated | Hidden         |
+-----------------+-----------+----------+----------+-----------+----------------+
| id              | Very Good | no       | yes      | no        | no             |
| name            | Very Good | no       | yes      | yes       | no             |
| email           | Good      | no       | yes      | yes       | no (blacklist) |
| balance         | Very Good | no       | yes      | no        | no             |
| timeRegistered  | Very Good | no       | yes      | no        | no             |
| timeCreated     | Very Good | no       | yes      | no        | no             |
| timeCpdated     | Very Good | no       | yes      | no        | no             |
+-----------------+-----------+----------+----------+-----------+----------------+
```

> You can also check other inspector commands to receive a list of every fillable or non hidden field in your models.
