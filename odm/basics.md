## ODM Component, Basics
Spiral ODM component has some similaries to [ORM engine] (/orm/basics.md) as they both based on [**DataEntity**] (/components/entity.md) model and [Behaviour Schemas] (/schemas.md). However ODM removes termin "relation" and replaces it with classical [compositions and aggregations] (https://en.wikipedia.org/wiki/Object_composition) temninalogy. In addition, ODM classes support inheritance and also can be embedded into ORM as JSON objects. 

ODM component was mainly designed to work with MongoDB database including support of atomic operations, however `DocumentEntity` class and it's compositions can be used outside of Mongo scope to create OOP data representation of various structures (XML files, API responses, etc).

Spiral ODM does not use entity cache like in ORM, instead you are given with "streaming like" functionality which can be used to process huge massives of data using Document models.

Check extened usage about DocumentEntity, Compompositions, Aggreagations and Inheritance [here] (oop.md).

## Mongo Connection and Databases
As in case with ORM you are not required to spend fortune of time to configuring ODM component, the only thing you have to make sure (in case if you want to store your data in MongoDB) is that mongo connection is properly set, to do that simply check configuration file "odm.php" located inside your "applicatio/config" directory:

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

Configration is very similar to [DBAL] (/databases/overview.md) and fully support database aliases and controllable/contextual injections. If you wish to get direct access to `MongoDatabase` in your controller simply request it using `ODM->db()` factory method or via `MongoDatabase` dependency using database alias:

```php
public function index(MongoDatabase $database, ODM $odm)
{
    dump($database->getCollectionNames());
    dump($odm->db('default'));
}
```

> `MongoDatabase` class extends original `MongoDB` class so you can use it as regular mongo database.

## Document
To define your first model related to MongoDB collection we only have to create/generate declaration of `Document` class whith desired set of fields listed in model **schema**. We can either create such model manually or generate it using console command "create:document", let's create our first model User, as in case with ORM we are able to pre-define set of desired fields: "create:document user -f id:MongoId -f name:string -f email:string -f balance:float". Resulted entity will be located in application/classes/Database/User.php:

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

You might notice that declaration is very similar to ORM Record, hovewer ODM component does not affect MongoDB schema, but rather stores it in your application. 

> You can read more about entity configuration (fillable, hidden, secured and validates properties) in a section declared to [DataEntity] (components/entity.md).

#### Schema
ODM classes Document (and `DocumentEntity`, see in extended usage) will only allow you to manipulate with set of fields described in model schema. Declaration of field can be performed by simply creating associated array between field name and it's type. 

```php
protected $schema = [
    'id'      => 'MongoId',
    'name'    => 'string',
    'email'   => 'string',
    'balance' => 'float'
];
```

You can associate your field with any desired type, however, since MongoDB does not allow dynamic typing we have to make sure that every value listed in schema is always typecased into specified type. You can solve such problem by utilizing model getters and setters or simply using set of pre-declared types, check ODM config:

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

As you can see, every string field will automatically get associated setter `setval` while schema update.

> Schema can also contain aggregations and compositions, you can read about it below.

#### Default Values
ODM will automatically force default values based on typecasted null value (string => "", int => 0), you can set your own default values using model property "defaults":

```php
protected $defaults = [
    'balance' => 100.00    
];
```

#### Associated Collection and Database
By default spiral will associate your `Document` model with the default ODM database and collection which name are generated based on class name (in our case User => users). You can alter both of this values by declaring non empty properties `database` and `collection`.

> If you unsure what collection will be used based on model name - simply force `collection` for every Document, it's not going to harm anyone but will make your code more readable.

#### Indexes
To declare indexes to be created in associated collection simply declare them in `indexes` property of your Document. Declaration form is similar to MongoCollection
ensureIndexes method. Indexes will be created as moment of schema update (see next).

```php
protected $indexes = [
    ['email' => 1, '@options' => ['unique' => true]],
    ['name' => 1, 'balance' => -1]
];
```

> Use "@options" key to specify index options.

## Schema Update
Once you created your first Document or Documents, you have to register them inside your ODM component schema. Such operation called "schema update" and does not require any operation rather then rutting console command "spiral update" or "spiral up". Schema update will automatically located all your Document and DocumentEntity models, analyze them, create automatic set of mutators and store information about compiled behaviours inside application memory.

The only database related operation performed while schema update - ensuring that requested indexes exists in associated collection.

> Spiral Framework in addion will generate virtual documentation for your models, so you IDE will highligt every aviable field and composition.

#### Modifying Documents schema
Since ODM schema does not stored in database it can be efficiently updated without any storage change, as result you are able to add new columns, compositions and etc into your Documents at any moment - simply declare them in your model schema and update ODM after.

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
Once your ODM schema has been updated we are ready to work with our models, to save some data into our users collection let's create our first entity in our controller method:

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

> If you working under PHPStorm, IDE will highlight all possible record fields for you.

![Tooltips](https://raw.githubusercontent.com/spiral/guide/master/resources/tooltips.png)

You can also create your entity using static method create which will respect fillable and secured fields.

```php
$u = User::create($request);
```

>Attention, create method will only return populated entity! You have to save it by it youself!

#### Validations
Document model fully follow principles of validations descibed in `DataEntity` model. If you wish to create more complex validation based on some external state you can redefine `validate` method:

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

Now our controller code will be giving us error related to email field:

```php
$user = new User();
$user->name = 'Anton';
$user->email = 'test@email.com';
$user->balance = 99;
if (!$user->save()) {
    dump($user->getErrors());
}
```

Such validation method will check if email is unique but only in cases when email field has some updates (got created or changed). Now, we will get an error message associated with model field.

Before we will jump to next step, lets try to create few more users. In this case we can use static method create which accepts set of initial model fields.

Since we have no fillable fields, this method should not work in our case (so we are going to use setFields method with disabled access policy - by setting second argument as true). In order to fill our demo entities let's connect faker package.

for ($i = 0; $i < 100; $i++) {
    $user = User::create()->setFields([
        'name'    => $faker->name,
        'email'   => $faker->email,
        'balance' => $faker->randomFloat(2, 0, 999)
    ], true);

    //Saving user to database
    $user->save();
}

> Attention, create method will only return populated Document entity, you have to save such entity to database manually (simply call save() method).

## Seletions
Once you have your collection populated with data, it's time to select some of the Users to do something. Spiral ODM exposes 3 ActiveRecord like methods which you can use for that purposes: `find`, `findOne` and `findByPK`.

There is 4th method which used inside every find method - `odmCollection`, you can use it to get access to `Collection` class associated with our document. You can also get associated collecion by calling method `odmCollection` of ODM component.

#### FindOne
To find one document based on some conditions and sorting we can use static method `findOne`, method will return null if it's unabled to locate required document. 
ODM component does not provide any query wrapper at top of MongoDB so you are able to use pure mongo queries for your requsstes:

```php
//Don't worry about second argument yet, it's about pre-loading
$user = User::findOne(['email' => 'test@email.com'];
dump($user);
```

> ODM provides only one feature of automatic convertion of `DateTime` classes in your queries into instance of `MongoDate`.

#### FindByPK
In cases where you would like to find your document based on it's primary key value (MongoId in "_id" field) we can use method `findByPK`. Method will behave same way as `findOne` if no entity can be found.

```php
$user = User::findByPK($mongoID);
dump($user);
```

> Method can accept mboth `MongoId` and string which will be automatically casted to valid id instance.

The most often option is to find model based on provided primary key or return client exception on failure, spiral does not embedds any application specific logic into ODM component so you have to raise an exception by your own, fortunatelly it's pretty easy to do:

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
To find one or multiple document instances from desired collection, first of all we have to get associated instance of `Collecion`, as stated before we can do that using ODM component method `odmCollection` or via static function `find` of our User model.

```php
$collection = User::find();
```

Class is built at top of MongoCollection and decorates some of it's functions, you are able to set or clarify selection query using methods `query` or `where`:

```php
$collection = User::find()->where([
    'email' => new \MongoRegex('/test@email.com/')
]);

dump($collection->count());
```

In addition to that you are able to paginate your result using same techniques described in [Pagination] (/components/pagination.md) component:

```php
$collection = User::find()->paginate(10);
```

> Check other Collection methods, such as limit, offset (skip) and sortBy.

##### Fetching Documents
Once you configured your Collection selection with desirec query you can fetch found document by simply iterating over collection, or explicitly calling method `getIterator` which will return an instance of `DocumentCursor`:

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

> You can also use method `getCursor` of your collection to make

##### DocumentCursor
Provided instance of `DocumentCursor` wraps at top of `MongoCursor` and provides support for lazy creation of collection documents (no entity cache used), as result you can iterate over huge amount of Documents without any significant memory usage. In addition, you can access to `MongoCursor` functionality:

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
If you wish to fetch only specified set of fields from your collection simply specify their names in a form of array into `fetchFields` method:

```php
foreach (User::find()->fetchFields(['name']) as $data) {
    dump($data);
}
```

Such methods will read all desired fields and return them in a form of array, you can also read them in a streaming mode by utilizing `fields` method of your `DocumentCursor`:

```php
$cursor = User::find()->getCursor()->fields([
    'email' => true
]);

while ($data = $cursor->getNext()) {
    dumP($data);
}

dumP($cursor->info());
```

> Attention, format in this case follows [offical documentation] (http://php.net/manual/ru/mongocursor.fields.php)!

## Modify Document
Once you selected needed document (for example using `findByPK`) you can modify it's fields using same way as with any other `DataEntity`. To save your updates into database you have to run method save(), method returns false if model validations failed.

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
If you will dump User models before saving it into database you will see property "atomics" which demonstrates what query will be sent to MongoDB to perform requested updated. 

```
atomics:dynamic = array(1)
(
·    ['$set'] = array(1)
·    (
·    ·    ['name'] = string(8) New Name
·    )
)
```

As you might notice, Document sends only chaged values of name field using atomic operation set. Such approach can be useful in many cases when data is updated from many places, however if you wish to save your entity data into database as solid dataset you can force entity "solid state". When model (Document) are in solid state every small update will cause full dataset update.

```php
$user = User::findByPK($id);
$user->name = 'New Name';

$user->solidState(true);
dump($user);
```

Now our atomic operation will look like:

```
atomics:dynamic = array(1)
(
·    ['$set'] = array(4)
·    (
·    ·    ['balance'] = double(5) 27.920000000000002
·    ·    ['id'] = null(0) 
·    ·    ['name'] = string(8) New Name
·    ·    ['email'] = string(22) klangworth@hotmail.com
·    )
)
```

> If you want to keep our model in solid state by default, you can call such method in overwritten model constructor. SolidState automatically applied to every newly created model.

#### Filters and Accessors
As in case with ORM or regular DataEntity class you are able to assign setters, getters and accessors to your fields. ODM component has two notable accessor we might need to check: `MongoTimestamp` and `ScalarArrar`.

##### MongoTimestamp Accessor
Based on provided configuration, you might notice that ODM will assign MongoTimestamp accessor to every timestamp or "MongoDate" field. Let's try to add such fields into our model:

```php
protected $schema = [
    'id'             => 'MongoId',
    'name'           => 'string',
    'email'          => 'string',
    'balance'        => 'float',
    'timeRegistered' => 'MongoDate'
];
```

After schema have been updated, we are able to use that field as Carbon instance.

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

You can also simply assign time to "timeRegistered" field, as accessor must automatically handle it using strtotime function:

```php
$user->timeRegistered = 'next friday 10am';
```
