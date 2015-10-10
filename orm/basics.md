# Spiral ORM, Basics
Spiral Framework provide simple but yet powerful ORM engine you can use in everyday development. Spiral ORM is able work multiple databases, automatically scaffold tables and columns, define set of realtions and pre-load them on demand later. 

ORM model **Record** are based on the [**DataEntity**] (/components/entity.md) class, it will be the best to read about such component first. In addition, check this [behaviour schema] (/schemas.md) concept which used as base for ORM component.

> Technically Spiral ORM is a hybrid between DataMapper and ActiveRecord, in a short plans i want to keep same functionality but make it more universal and give ability to create your own entity models based on interface rather that parent RecordEntity class (i already have RecordInteface for such purposes).

## Record
Spiral ORM component does not require any configuration rather than making sure that [DBAL] (/database/overview.md) has valid database connections. The only thing you have to do to start using ORM is to create or generate desired **Record** model.

Such model must includes property **schema** which is going to describe needed table colums and record relationships. Since ORM component uses DBAL as it's backbone, you can declare set of desired columns in a similar format as [schema writers] (/database/syncing.md).

The easies way to create new ORM models is to use CLI toolkit, record generation command accept pre-defined set of fields using key `-f`, we can try to generate our first database entity: 'create:record user -d -f id:primary -f name:string(64) -f email:string -f status:enum(active,blocked)', note that we are using key `-d` to create "defaults" property (see next).

Resulted entity will be located in `application/classes/Database/User.php`:

```php
class User extends Record 
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
        'id'     => 'primary',
        'name'   => 'string(64)',
        'email'  => 'string',
        'status' => 'enum(active,blocked)'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'   => ['notEmpty'],
        'email'  => ['notEmpty'],
        'status' => ['notEmpty']
    ];

    /**
     * @var array
     */
    protected $defaults = [
        
    ];
}
```

As you can see most of the Record properties are really similar to `DataEntity` as can be configured same way, however we have few additional things we have to remember.

#### Schema
The most important part of any Record model is it's schema property. Schema declares what columns you would like to see in associated table and, in addition to that, defines set of record [relations] (relations.md). In this tutorial we are going to stick to columns only.

Since we are creating model for new (non existed table) we would need to declare table primary key, we can do it same way as in DBAL Schema Writers by assigning type "primary" or "bigPrimary".

```php
'id'     => 'primary',
```

>  Spiral ORM will detect your primary key name automatically so you don't nesessary need to use "id".

To provide additional type arguments, such as length, scale or enum values you can simply use braces as you would call `AbstractColumn` methods.

```php
'name'   => 'string(64)',
'status' => 'enum(active,blocked)'
```

The only important difference between way how columns decribed in DBAL and ORM that Record model will force **NOT NULL** flag and default value for each column (to solve problems when schema changes modifies existed non empty table). If you wish to declare your column as nullable, simply add "nullable" to it's type definition.

```php
'column' => 'string, nullable'
```

#### Default Values
As you already know ORM will force set of default values for every declared column, usually such default values will be just an empty type casted value ("", 0 etc). To declare your own default value using record property "defaults", in our case we have "enum" column, it's the best to force specific value for it:

```php
protected $defaults = [
    'status' => 'active'    
];
```

#### Associated Table and Database
By default spiral will associate your Record model with the default DBAL database and table which name are generated based on class name (in our case User => users). You can alter both of this values by declaring non empty properties `database` and `table`.

> If you unsure what table will be generated based on model name - simply force table for every Record, it's not going to harm anyone but will make your code more readable.

#### Indexes
To declare indexes to be created in associated table, you can easily do it in your Record via propery `indexes`. Every index information must be located in a sub array of such property to include index type (self::INDEX or self::UNIQUE) and column(s) index based on, let's try to add unique index on our `email` column.

```php
protected $indexes = [
    [self::UNIQUE, 'email']
];
```

> You can include as many columns into index as your DBMS allows you. Simply list columns as array values.

## Schema Update and Scaffolding
Once you done with declaring your `Record` model schema we can run ORM schema update. Such operation can be performed by executing command 'spiral up'. The update will automatically generate related table "users" for your model using declared columns and indexes. We can view content of such table by running CLI 'db:describe users':

```
Columns of primary.users:
+---------+----------------+----------------+-----------+----------------+
| Column: | Database Type: | Abstract Type: | PHP Type: | Default Value: |
+---------+----------------+----------------+-----------+----------------+
| id      | int (11)       | primary        | int       | ---            |
| name    | varchar (64)   | string         | string    | ---            |
| email   | varchar (255)  | string         | string    | ---            |
| status  | enum           | enum           | string    | active         |
+---------+----------------+----------------+-----------+----------------+

Indexes of primary.users:
+-------------------------------------------+--------------+----------+
| Name:                                     | Type:        | Columns: |
+-------------------------------------------+--------------+----------+
| newprefix_users_index_email_55f84337a039a | UNIQUE INDEX | email    |
+-------------------------------------------+--------------+----------+
```

As you can see our table looks exacly as it should (output may differ if you using different DBMS)!

> You have to execute `spiral up` every time you change any entity or orm property of your Record to pass updates into cached behaviour schema. There is no need to run schema update if you modifying custom properties or methods.

#### Adding new Columns
Since ORM based on DBAL syncronization mechanism, the only thing you need to do to add new columns into related Record table - declare such column in model schema (same applied for indexes). Let's try add column "balance".

```php
protected $schema = [
    'id'     => 'primary',
    'name'   => 'string(64)',
    'email'  => 'string',
    'status' => 'enum(active,blocked)',
    'balance' => 'decimal(10,2)'
];
```

To add such column into our table we have to execute `spiral up` again (set verbosity flag `-v` to see what is going on):

```
> spiral up -v
Migrator does not configured, skipping.

[primary_users] Adding column [`balance` decimal (10, 2) NOT NULL DEFAULT 0.000000] into table `primary_users`.
ORM Schema has been updated: 0.494 s, found records: 1
ODM Schema has been updated: 0.163 s, found documents: 1

Generating tooltips and hints for PHPStorm...
ODM virtual documentation were created: 0 classes
ORM virtual documentation were created: 3 classes

Inspecting available DataEntities...
+---------------+------+--------+----------+-----------+
| Entity        | Rank | Fields | Fillable | Validated |
+---------------+------+--------+----------+-----------+
| Database\User | Good | 5      | 0        | 3         |
+---------------+------+--------+----------+-----------+

Inspected entities 1, average rank Good (0.95).
```

> You can use model schema only to add new columns, renaming and removing must be done via migrations - meaning removing column declaration from model will not remove such column from table.
> Due migrations are mounted right before schema update, you are able to use them as part of ORM flow.

And again we can check if our table was successfully modified using command "db:describe users", this time i switched my connection to PostgresSQL:

```
Columns of primary.users:
+---------+-------------------------+----------------+-----------+-------------------------------------------+
| Column: | Database Type:          | Abstract Type: | PHP Type: | Default Value:                            |
+---------+-------------------------+----------------+-----------+-------------------------------------------+
| id      | serial                  | primary        | int       | nextval('primary_users_id_seq'::regclass) |
| name    | character varying (64)  | string         | string    | ---                                       |
| email   | character varying (255) | string         | string    | ---                                       |
| status  | character (7)           | enum           | string    | active                                    |
| balance | numeric (10, 2)         | decimal        | float     | ---                                       |
+---------+-------------------------+----------------+-----------+-------------------------------------------+

Indexes of primary.users:
+-----------------------------------------+--------------+----------+
| Name:                                   | Type:        | Columns: |
+-----------------------------------------+--------------+----------+
| primary_users_index_email_55f8450374ca9 | UNIQUE INDEX | email    |
+-----------------------------------------+--------------+----------+
```

> You might notice that there is no enum type in Postgres, it will be emulated using constrained string.

#### Active vs Passive Schemas
In a given example, we created Record which automatically ensures that table exists in database and have desired structure. ORM component does not require you to declare every table column, even more, you can associate your model with existed table without declaring schema propery. In this case model will fetch columns list from table directly, our model may look like:

```php
class SomeRecord extends Record
{
    protected $table = 'some_table';
}
```

However, in a next guide section dedicated to relations, you'll find out that relations have ability to modify associated model schemas to declare needeed columns and foreign keys. This is fine if we generating our database using Spiral, but in some cases you might want to connect Records to existed database without letting spiral do modify it. In this case, we can configure relations in a way to make them follow already existed structures, but, to be absolutelly sure that no relation or record can modify our table we can declare Record constant **ACTIVE_SCHEMA** and set if value to `false`.

```php
class SomeRecord extends Record
{
    const ACTIVE_SCHEMA = false;
    
    protected $table = 'some_table';
}
```

Such constant will forbid *any modifications* in associated table and ensure that spiral can not touch existed structure. The rest of ORM functionality such as validations, selections eager loading and realtions are still preserved.

> You can use passive models if you prefer to generate your database using migrations. ORM in this case will thrown an schema exception if some column is requied for relation but missed from schema. You can also use passive models to validate auto generated structure before doing real update.

## Create Record
Once your ORM schema has been updated we are ready to work with our model. Since we just created our table, it's time to push some data in. Let's do such operation in a controller:

```php
public function index()
{
    $u = new User();
    $u->name = 'Anton';
    $u->email = 'test@email.com';
    $u->status = 'active';
    $u->save();

    dump($u);
}
```

> If you working under PHPStorm, IDE will highlight all possible record fields for you.

You can use every `DataEntity` method to populate your model, such as setFields, setField or even set of magic methods - setName, setEmail, etc.

You can also create your entity using static method `create` which will respect fillable and secured fields.

```php
$u = User::create($request);
```

> Attention, `create` method will only return populated entity! You have to save it by it youself!

#### Validations
Given example will raise an exception when we will try to execute it again, the reason - non unique email. Since email validation requires us to send query to database, we might want to modify our validation method (same way as in `DataEntity` examples) to add more complex validation rule:

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
        $selection = $this->sourceTable()->where([
            'email' => $this->email,
            'id'    => ['!=' => $this->id]
        ]);

        if ($selection->count() != 0) {
            $this->setError('email', self::translate("Email must be unique."));
        }
    }

    return false;
}
```

Such validation method will check if email is unique but only in cases when email field has some updates (got created or changed). Now, we will get an error message associated with model field rather than database exception.

Before we will jump to next step, lets try to create few more users. In this case we can use static method `create` which accepts set of initial model fields.

Since we have no fillable fields this method should not work in our case (so we are going to use `setFields` method with disabled access policy -  by setting second argument as `true`). In order to fill our demo entities let's connect [faker] (https://github.com/fzaninotto/Faker) package.

```php
for ($i = 0; $i < 100; $i++) {
    $user = User::create()->setFields([
        'name'    => $faker->name,
        'email'   => $faker->email,
        'balance' => $faker->randomFloat(2, 0, 999),
        'status'  => $faker->randomElement(['active', 'blocked'])
    ], true);

    //Saving user to database
    $user->save();
}
```

> Attention, `create` method will only return populated Record entity, you **have** to save such entity to database manually (simply call `save()` method).

> Even more, every DataEntity including ORM Records and ODM Documents must be populated from outside, meaning there is no database related code in entity constuctor, if you wish to create entity based on existed array of data (skipping setters) simply pass such array as constructor argument `new User([...])`.

To check what is going on with queries to database, let to check logging tab of spiral [Profiler] (/modules/profiler.md) (this is Postgres connection):

```sql
INSERT INTO "primary_users" ("name", "email", "status", "balance")
VALUES ('Boris Kuhic', 'mrunte@yahoo.com', 'blocked', 386.600000) RETURNING "id"
```

> All characters appearing in this work are fictitious. Any resemblance to real persons, living or dead, is purely coincidental.

## Seletions
Once you have your table populated with data, it's time to select some of them to do something. Spiral ORM exposes 3 ActiveRecord *like* methods which you can use for that purposes: `find`, `findOne` and `findByPK`.

> There is 4th method which used inside every find method - `ormSelector`, you can use it to get access to Selector class associated with our record. You can also get associated selector by calling method `selector` of ORM component. 

#### FindOne
To find one record based on some conditions and sorting we can use static method `findOne`, method will return **null** if it's unabled to locate required record. Since ORM component based on DBAL you are able to specify needed conditions in a short array form (if you want normal where statements, check `find` method below).

```php
//Don't worry about second argument yet, it's about pre-loading
$user = User::findOne(['status' => 'active'], [], ['id' => 'DESC']);
dump($user);
```

#### FindByPK
In cases where you would like to find your record based on it's primary key value (usually "id") we can use method `findByPK`. Method will behave same way as `findOne` if no entity can be found.

```php
$user = User::findByPK(1);
dump($user);
```

The most often option is to find model based on provided primary key or return client exception on failure, spiral does not embedds any application specific logic into ORM component so you have to raise an exception by your own, fortunatelly it's pretty easy to do:

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

```php
public function index(UserService $users, $id)
{
    if (empty($user = $users->findByPK((int)$id))) {
        throw new NotFoundException('No such user');
    }

    dump($user);
}
```

#### Find
To find multiple Records or run aggregation, let's use method `find` which will return instance of `Spiral\ORM\Entities\Selector`, such class has common parent with DBAL [SelectQuery] (/database/builders.md) and makes you able to use same methods and princiles:

```php
public function index()
{
    dump(User::find()->count());
    dump(User::find()->sum('user.balance'));

    foreach (User::find(['status' => 'active']) as $user) {
        dump($user->name);
    }

    $selection = User::find()->where('user.balance', '>', 500)->orderBy(
        'user.balance',
        'DESC'
    );

    foreach ($selection as $user) {
        dump($user);
    }
}
```

Let's try to check SQL statement generated for last selection:

```sql
SELECT
*
FROM "primary_users" AS "user"
WHERE "user"."balance" > 500
ORDER BY "user"."balance" DESC
```

Note that ORM Selector will automatically assign **alias for associated table** which is singular model name ("user"). You can read more about aliases [here] (loading.md).

> To find one record using normal where conditions, try variant `User::find()->where(...)->findOne()`

## Modify Record
Once you selected needed record (for example using `findByPK`) you can modify it's fields using same way as with any other DataEntity. To save your updates into database you have to run method `save()`, method returns false if model validations failed.

```php
public function index()
{
    $user = User::findByPK(1);
    $user->name = 'New Name';

    if (!$user->save()) {
        dump($user->getErrors());
    }
}
```

#### Dirty Fields and Solid State
If you will check SQL code generated for previous example, you might notice that it looks like that:

```sql
UPDATE "primary_users"
SET "name" = 'New Name'
WHERE "id" = 1 
```

Record model included *only modified* fields into update query. Such approach can be useful in many cases when data is updated from many places, however if you wish to save your entity data into database as solid dataset you can force entity "solid state". When model (Record) are in solid state every small update will cause full dataset update.

```php
$user = User::findByPK(1);
$user->name = 'New Name';

if (!$user->solidState(true)->save()) {
    dump($user->getErrors());
}
```

This time our SQL will look like:

```sql
UPDATE "primary_users"
SET "name" = 'New Name',  "email" = 'test@email.com',  "status" = 'active ',  "balance" = '0.00'
WHERE "id" = 1
```

> If you want to keep our model in solid state by default, you can call such method in overwritten model constructor.

#### Filters and Accessors
You can define setters, getters and accessors same way you would do that for `DataEntity`. However, spiral ORM can automatically assign some getters and accessors to your record based on column type, you check check what filters will be created by default by looking into ORM configuration file:

```php
'mutators'       => [
    'timestamp'  => ['accessor' => Accessors\ORMTimestamp::class],
    'datetime'   => ['accessor' => Accessors\ORMTimestamp::class],
    'php:int'    => ['setter' => 'intval', 'getter' => 'intval'],
    'php:float'  => ['setter' => 'floatval', 'getter' => 'floatval'],
    'php:string' => ['setter' => 'strval'],
    'php:bool'   => ['setter' => 'boolval', 'getter' => 'boolval']
],
```

As you can see ORM will make sure that every column is validly type casted into scalar value while reading (this can be very useful since some DBMS returns column values as strings only).

#### Timestamps Accessor
Based on provided configuration, you might notice that ORM will assign `ORMTimestamp` accessor to every timestamp or datetime field. Let's try to add such field into our table (again, avoid using timestamps):

```php
protected $schema = [
    'id'              => 'primary',
    'time_registered' => 'datetime',
    'name'            => 'string(64)',
    'email'           => 'string',
    'status'          => 'enum(active,blocked)',
    'balance'         => 'decimal(10,2)'
];
```

After schema have been updated, we are able to use that field as [Carbon] (https://github.com/briannesbitt/Carbon) instance.

```php
public function index()
{
    $user = User::findByPK(1);
    $user->name = 'New Name';

    $user->time_registered->setDateTime(2015, 1, 1, 12, 0, 0);
    dump($user->time_registered);

    if (!$user->solidState(true)->save()) {
        dump($user->getErrors());
    }
}
```

You can also simply assign time to `time_registered` field, as accessor must automatically handle it using `strtotime` function:

```php
$user->time_registered = 'next friday 10am';
```

> Attentio, DBAL will convert and fetch all dates using UTC timezone!

#### Atomic Number
Field accessors in ORM may not only provide mocking functionality but declare update behaviours. One of such accessors can be very useful to manage numeric values in "atomic" way. Let's try to see such accessor in action, first of all we have to declare it in our Record model and update schema after:

```php
protected $accessors = [
    'balance' => AtomicNumber::class
];
```

Now we are able to manipulate with field values using deltas:

```php
$user = User::findByPK(1);
$user->balance->inc(1.6);

if (!$user->save()) {
    dump($user->getErrors());
}
```

> You can use accessor method `setData` to specify new balance value without deltas. Use `serializeData()` to get mocked accessor value.

The whole point of solid state and such accessor can be decribed using resulted update statement:

```sql
UPDATE "primary_users"
SET "balance" = "balance" + 1.6
WHERE "id" = 1
```

> You can create your own accessors by implementing `RecordAccessorInterface`, also check [JsonDocument] (/odm/standalone.md) accessor.

## Delete Records
You can delete any existed Record by executing it's method "delete". There is not much to talk about it.

## Events and Traits
As DataEntity model, Record declares few [events] (/components/events.md) which you can handle to track or change model behaviour:

Event                   | Context                                   | Return | Description
---                     | ---                                       | ---    | ---
setFields               | $fields                                   | *      | Called inside `setFields` method.
publicFields            | $fields                                   | *      | Called before return in `publicFields` method.
jsonSerialize           | $publicFields                             | *      | Called while packing model into json.
validation              | -                                         | -      | Before validation.
validated               | $errors                                   | *      | After validation.
**describe** (static)   | [$property, $value, EntitySchema $schema] | value  | Called while model analysis. Such evet can be used to redefine or alter schema, validations, mutators etc.
created                 | -                                         | -      | Called inside static method "create" after assigning model fields.
selector                | Selector $selector                        | *      | Called by "find" and other selection methods.
saving                  | -                                         | -      | Before model data saved into database (model is only created). 
saved                   | -                                         | -      | After model data saved into database.
updating                | -                                         | -      | Before model data updated in database (model already exist).
updated                 | -                                         | -      | After model data updated in database.
deleting                | -                                         | -      | Before model deleted from database.
deleted                 | -                                         | -      | After model deleted from database.

In most cases you don't need to handle any of event as `save` and `delete` methods can be easily overwriten. However events can be very useful to create set of traits used to modify Record schema and update/save behaviour (see next).

#### Timestamps Trait
One of the most common Record trait you might want to use - `TimestampsTrait`. Such trait handle events "saving", "updating" and "describe" to alter Record columns and update two "magic" fields "time_created" and "time_updated" (both type datetime). Our final demo Record User might look like:

```php
class User extends Record
{
    use TimestampsTrait;

    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'id'              => 'primary',
        'time_registered' => 'datetime',
        'name'            => 'string(64)',
        'email'           => 'string',
        'status'          => 'enum(active,blocked)',
        'balance'         => 'decimal(10,2)'
    ];

    /**
     * @var array
     */
    protected $defaults = [
        'status' => 'active'
    ];
    
    /**
     * @var array
     */
    protected $indexes = [
        [self::UNIQUE, 'email']
    ];

    /**
     * @var array
     */
    protected $accessors = [
        'balance' => AtomicNumber::class
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'   => [
            'notEmpty'
        ],
        'email'  => [
            'notEmpty'
        ],
        'status' => [
            'notEmpty'
        ]
    ];

    /**
     * @param bool|false $reset
     * @return bool
     */
    protected function validate($reset = false)
    {
        parent::validate($reset);

        if ($this->hasUpdates('email') && !$this->hasError('email')) {

            //Let's try to check if email is unique
            $selection = $this->sourceTable()->where([
                'email' => $this->email,
                'id'    => ['!=' => $this->id]
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
While working with ORM models you are able to call `find`, `findByPK` and `save` methods inside your controllers directly. However spiral provides ability to pre-generate specific [Service] (/application/services.md) which can help you to abstract database specific operations from your controllers code. 

You can scaffold such class using console command "create:service user -e user", resulted code may look like:

```php
class UserService extends Service  implements SingletonInterface
{
    /**
     * Declares to IoC container that class must be treated as Singleton.
     */
    const SINGLETON = self::class;

    /**
     * Create new blank User. You must save entity using save method.
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
     * @param array $errors Will be populated if save fails.
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

As result you can use such service inside you controllers avoiding static methods of Record model (they can always be removed from service also):

```php
public function index(UserService $users)
{
    $user = $users->findByPK(1);
    $user->balance->inc(1.6);

    if (!$users->save($user)) {
        dump($user->getErrors());
    }
}
```

Services like that is the best place to locate custom selection methods (like `findActive`) or even alter delete method to implement "soft deletes".

> You can also generate CRUD controller to work with existed entity service using command 'create:controller name -s service'.

## Inheritance and Abstract Records
Since Spiral ORM uses static analysis, there is no real limitation how you would like to create your models, as result you can declare **abstract** Record with it's schema, validations and etc and later extend this class in your application models. This technique can be very useful while writing [modules] (/components/modules.md).

> While extending, ORM will merge schemas and other properties of Record and it's parent. Attention, ORM does not provide table inheritance.

## Inspections
While running shema update (spiral up) command, you might notice text which contains list of inspected entities and resulted rating. Such information provided by spiral **Entity Inspector** which analyses ORM and ODM schemas to find unprotected or blacklisted fields, we can get more details by running inspection on selected entity:

```
> spiral inspect:entity Database/User
+-----------------+-----------+----------+----------+-----------+----------------+
| Field           | Rank      | Fillable | Filtered | Validated | Hidden         |
+-----------------+-----------+----------+----------+-----------+----------------+
| id              | Very Good | no       | yes      | no        | no             |
| name            | Very Good | no       | yes      | yes       | no             |
| email           | Good      | no       | yes      | yes       | no (blacklist) |
| status          | Very Good | no       | yes      | yes       | no             |
| balance         | Very Good | no       | yes      | no        | no             |
| time_registered | Very Good | no       | yes      | no        | no             |
| time_created    | Very Good | no       | yes      | no        | no             |
| time_updated    | Very Good | no       | yes      | no        | no             |
+-----------------+-----------+----------+----------+-----------+----------------+
```

> You can also check other inspector commands to receive list of every fillable or non hidded field in your models.
