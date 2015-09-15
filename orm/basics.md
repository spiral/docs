# Spiral ORM, Basics
Spiral Framework provide simple but yet powerful ORM engine you can use in everyday development. Spiral ORM provides ability to work with multiple databases, automatically scaffold tables and define set of realtions which can be requested on demand or pre-loaded with parent entity. 

Spiral ORM model **Record** are based on [DataEntity] (/components/entity.md) class, so it will be the best to read about it first. In addition check this [behaviour schema] (/schemas.md) concept which used as base for ORM component.

## Record
Spiral ORM component does not require any configuration rather than making sure that [DBAL] (/database/overview.md) has valid database connections. The only thing you have to do to that using ORM is to create or generate desired **Record** model.

Such model must include property **schema** which is going to describe needed table colums and record relationships. Since ORM component uses DBAL as it's backbone, you can declare set of desired columns in a similar format as [schema writers] (/database/syncing.md).

Record generation command accept pre-defined set of fields using key `-f`, let's try to generate our first model: 'create:record user -d -f id:primary -f name:string(64) -f email:string -f status:enum(active,blocked)', we are using key `-d` to generate "defaults" property (see next). Resulted entity will be generated in `application/classes/Database/User.php`:

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
     * @var array
     */
    protected $defaults = [
        
    ];
}
```

As you can see most of Record properties are really similar to `DataEntity` as can be configured same way, however we have few additional things we have to remember.

#### Schema
The most important part of any Record models is it's schema. Schema declares what columns you would like to see in associated table and, in addition to that, contains set of record [relations] (relations.md). In this tutorial we are going to stick to columns only.

Since we are creating model for new (non existed table) we would need to declare primary key, we can do it same way as in DBAL Schema Writers by assigning type "primary" or "bigPrimary".

```php
'id'     => 'primary',
```

>  Spiral ORM will detect your primary key name automatically so you don't nesessary need to use "id".

To provide additional type arguments, such as length, scale or enum values you can simply use braces as you would call `AbstractColumn` method.

```php
'name'   => 'string(64)',
'status' => 'enum(active,blocked)'
```

The only important difference between way how columns decribed in DBAL and ORM that Record model will force **NOT NULL** flag and default values for each column (to solve problems when schema changes modifies existed non empty table). If you wish to declare your column as nullable, simply add "nullable" to it's type definition.

```php
'column' => 'string, nullable'
```

#### Default Values
As you already know ORM will force set of default values for every declared column, usually such default values will be just an empty type casted value ("", 0 etc). You can declare your own default value using record property "defaults", in our case we have "enum" column, it's the best to force default value for it:

```php
protected $defaults = [
    'status' => 'active'    
];
```

#### Associated Table and Database
By default spiral will associte your Record model with default DBAL database and table which name are generated based on class name (in our case User => users). You can alter both of this values by declaring non empty properties `database` and `table`.

#### Indexes
In many cases you might want to declare indexes to be created in associated table, you can esily do it in your Record via propery `indexes`. Every index information must be located in a sub array of such property and include index type (self::INDEX or self::UNIQUE), let's try to add unique index on our `email` column.

```php
protected $indexes = [
    [self::UNIQUE, 'email']
];
```

> You can include as many columns into index as your DBMS allows you. Simply list columns as array values.

## Schema Update and Scaffolding
Once you done with declaring your `Record` model schema we can run ORM update. Such operation can be achieved by running command 'spiral up'. Such command should automatically generate related table "users" for your model using declared columns and indexes. We can view content of such table by executing 'db:describe users':

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
Since ORM based on DBAL syncronization mechanism, the only thing you need to do to add new columns into related record table - declare such column in model schema (same applied for indexes). Let's try add column "balance".

```php
protected $schema = [
    'id'     => 'primary',
    'name'   => 'string(64)',
    'email'  => 'string',
    'status' => 'enum(active,blocked)',
    'balance' => 'decimal(10,2)'
];
```

To add such column into our table we have to execute `spiral up` again (you cam set verbosity flag `-v` to see what is going on):

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

> You can use model schema only to add new columns, renaming and removing must be done via migrations, meaning removing column declaration from model will not remove such column from table.

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

#### Active vs Passive Schemas
In a given example, we created Record which will automatically ensure that table exists in database and have desired set of columns and indexes. ORM component does not require you to declare every table column, even more, you can associate your model with existed table without even creating schema propery. In this case model will fetch column list from table directly, our model may look like:

```php
class SomeRecord extends Record
{
    protected $table = 'some_table';
}
```

However, in a next guide section dedicated to relations, you'll find out that relations have ability to modify associated model schemas to declare needed columns and foreign keys. This is fine if we generating our database using Spiral Models, however in some cases you might want to connect Records to existed database. In this case, you can configure relations in a way to make them follow already existed structures. But, to be absolutelly sure that no relation or recor can modify our table we can delcare record constant ACTIVE_SCHEMA and set if value to false.

```php
class SomeRecord extends Record
{
    const ACTIVE_SCHEMA = false;
    
    protected $table = 'some_table';
}
```

Such constant will forbid any modifications for associated table and ensure that spiral can not touch existed table. The rest of ORM functionality such as validations, selections eager loading and realtions are still preserved.

> You can also use passive models if you prefer to generate your database using migrations.

## Create Record
Once your ORm schema updated we are ready to work with our models. Since we just created our table, it's time to push some data in. Let's do such operation in a controller:

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

You can apply any DataEntity method to your model.
> If you working under PHPStorm, IDE will highlight all possible record fields for you.

#### Validations
Given example will raise an exception when we will try to execute it again, the reason - non unique email. Since email validation requires us to send query to database, we might want to modify our validation method (same way as in DataEntity) to add more complex validation rule:

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
            'id'    => ['!=' => (int)$this->id]
        ]);

        if ($selection->count() != 0) {
            echo 1;
            $this->setError('email', self::translate("Email must be unique."));
        }
    }

    return false;
}
```

Such validation method will check if email is unique but only in cases when email field has some updates (got created or changed). Now, we will get an error message associated with model field rather than exception.

Before we will jump to next step, lets try to create few more users. In this case we can use static method `create` which accepts set of initial model fields. Since we have no fillable fields this method should not work in our case (so we are going to use `setFields` method with disabled access policy). In order to fill our demo entities let's connect [faker] (https://github.com/fzaninotto/Faker).

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

> Attention, `create` method will only return populated Record entity, you HAVE to save such entity to database manually (simply call `save()` method).

You you wish to check what is going on with queiries to database, you can to check logging tab of spiral profiler (this is Postgres connection):

```sql
INSERT INTO "primary_users" ("name", "email", "status", "balance")
VALUES ('Boris Kuhic', 'mrunte@yahoo.com', 'blocked', 386.600000) RETURNING "id"
```

## Seletions
Once you have your table populated with records, it's time to select some of them to do something. Spiral ORM exposes 3 ActiveRecord like methods which you can use for that purposes: `find`, `findOne` and `findByPK`.

#### FindOne
To find one record based on some conditions and sorting we can use static method `findOne`, method will return **null** if it's unabled to locate required record. Since ORM component based on DBAL you are able to specify needed conditions in a short array form (if you need normal where statements, check `find` method below).

```php
$user = User::findOne(['status' => 'active']);
dump($user);
```

#### FindByPK
In cases where you would like to find your record based on it's primary key value (usually ID) we can use method `findByPK`. Method will behave same way as `findOne` if no entity can be found.

```php
$user = User::findByPK(1);
dump($user);
```

#### Find
If you want to find multiple records, or run aggregation, you can use method `find` which will return instance of `Spiral\ORM\Entities\Selector`, such class has common parent with DBAL [SelectQuery] (/database/builders.md) which makes you able to use same methods and princiles:

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

Note that ORM Selector will automatically assigned alias to Record table which is singular model name ("user"). You can read more about aliases [here] (loading.md).

## Modify Record
Once you selected needed record (for example using `findByPK`) you can modify it's fields using same way as with any other DataEntity. To save your updates into database you have to run method `save()` which will return false if model validations failed.

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
If you will check SQL code generated for previous example, you might notice that it looks like:

```sql
UPDATE "primary_users"
SET "name" = 'New Name'
WHERE "id" = 1 
```

Record model included only modified fields into update query. Such approach can be useful in many cases, however if you wish to save your moveld into database with every declared value you can force, so called, "solid state". When model (Record) are in solid state every small update will force ORM to save whole record data.

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

> If you want to keep our model in solid State by default, simply declare protected property solidState with default value `true`.

#### Filters and Accessors
You can define setters, getters and accessors same way you would do that for DataEntity. However, spiral ORM will automatically assign some getters and accessors to your record based on column type, you check check what filters will be created by default by looking into ORM configuration file:

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

As you can see ORM will make sure that every column is validly type casted into scalar value while reading, this can be very useful since some DBMS returтs every table value as string.

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

After schema have been updated, we are able to use such model field as [Carbon] (https://github.com/briannesbitt/Carbon) instance.

```php
public function index()
{
    $user = User::findByPK(1);
    $user->name = 'New Name';

    $user->time_registered->now();
    dump($user->time_registered);

    if (!$user->solidState(true)->save()) {
        dump($user->getErrors());
    }
}
```

You can also simply assign time to `time_registered` field, accessor must automatically handle it using `strtotime` function:

```php
$user->time_registered = 'next friday 10am';
```

> DBAL will convert all dates into UTC timezone.

#### Atomic Number
Field accessors in ORM may not only provide mocking functionality but declare update behaviours. One of such accessors can be very useful to manage numeric values "atomic" way. Let's try to see such accessor in action, first of all we have to declare it in our Record model and update schema after:

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
You can delete any existed Record by executing it's method "delete".

## Events and Traits
As DataEntity model, Record declares few [events] (/components/events.md) which you can handle to track or handle model behaviour:

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

In most of cases you don't need to handle any of event as `save` and `delete` methods can be easily overwriten. However events can be very useful to create set of traits used to modify Record schema and save behaviour (see next).

#### Timestamps Trait
One of the most common Record trait you might want to use - TimestampsTrait. Such trait handle events "saving", "updating" and "describe" to alter Record columns and update two "magic" column "time_created" and "time_updated" (datetime). Our final demo Record User might look like:

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
                'id'    => ['!=' => (int)$this->id]
            ]);

            if ($selection->count() != 0) {
                echo 1;
                $this->setError('email', self::translate("Email must be unique."));
            }
        }

        return false;
    }
}
```

## Services and Controllers
While working with ORM models you are able to call `find`, `findByPK` and `save` methods inside your controllers directly. However spiral provides ability to pre-generate specific [Service] (/application/services.md) which can help you to abstract database specific operations from your controllers code. You can scaffold such class using console command "create:service user -e user", resulted code may look like:

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

Now you can use such service inside you controllers avoiding static methods of Record model (they can always be removed from service also):

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

## Inheritance and Abstract Records
Since Spiral ORM uses static analysis, there is no real limitation how you would like to create your models, as result you can declare **abstract** Record with it's schema, validations and etc and later extend this class in your application. This technique can be very useful while writing [modules] (/components/modules.md).

> While extending, ORM will merge schemas and other properties of Record and it's parent.

## Inspections
While running shema update (spiral up) command, you might notice line which contains list of inspected entities and resulted rating. Such information provided by spiral entity inspector which analyses ORM and ODM schema to find unprotected or blacklisted fields, we can get more details by running inspection on selected entity:

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
