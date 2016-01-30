# Spiral ORM, Basics
The Spiral Framework provides a simple yet powerful ORM engine that you can use in your everyday development. Spiral ORM works with multiple databases, automatically scaffolds tables and columns, it defines set of relations and pre-loads them on demand for later use. 

The ORM model **Record** is based on the [**DataEntity**] (/components/entity.md) class. It's best to read about this component first. In addition, you can check out this [behaviour schema] (/schemas.md) concept which used as the base for the ORM component.

## Record
The Spiral ORM component does not require any configuration other than making sure that the [DBAL] (/database/overview.md) has valid database connections. To begin using ORM, just create or generate your desired **Record** model.

Your model must include a property **schema** which will describe the table columns and record relationships. Since the ORM component uses DBAL as it's backbone, you can declare a set of desired columns the same way as [schema writers] (/database/syncing.md).

The easiest way to create new ORM models is to use CLI toolkit. Record generation command accepts a pre-defined set of fields using the key `-f`. We can now generate our first database entity: 'create:record user -d -f id:primary -f name:string(64) -f email:string -f status:enum(active,blocked)'. Note that we are using the key `-d` to create "defaults" property (see next).

The ensuing entity will be located in `application/classes/Database/User.php`:

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

Most of the Record properties are really similar to `DataEntity` and they can be configured in the same way. However we have a few additional things we have to remember.

#### Schema
The most important part of any Record model is it's schema property. Schema declares what columns you would like to see in the associated table as well as defines a set of record [relations] (relations.md). In this tutorial, we will stick to columns only.

Since we are creating a model from scratch (non existing table), we need to declare the table primary key. We can do this the same way as in DBAL Schema Writers by assigning the type "primary" or "bigPrimary".

```php
'id'     => 'primary',
```

>  Spiral ORM will detect your primary key name automatically so you don't have to use "id".

To provide an additional type arguments, such as length, scale or enum values, you can simply use braces as you would call `AbstractColumn` methods.

```php
'name'   => 'string(64)',
'status' => 'enum(active,blocked)'
```

An important difference to note between the way columns are described in DBAL and ORM is that the Record model will force **NOT NULL** flag and use a default value for each column (to solve any problems when the schema changes and modifies existing table that aren't empty). To declare your column as nullable, simply add "nullable" to it's type definition.

```php
'column' => 'string, nullable'
```

#### Default Values
As you already know, ORM will force a set of default values for every declared column. Usually these default values will be just an empty type casted value ("", 0 etc). To declare your own default value using the record property "defaults", in our case we have "enum" column, it's best to force a specific value for it:

```php
protected $defaults = [
    'status' => 'active'    
];
```

#### Associated Table and Database
By default, Spiral will associate your Record model with the default DBAL database and table whose names are generated based on the class name (in our case, User => users). You can alter both of these values by declaring them non empty properties `database` and `table`.

> If you're unsure about what table will be generated based on the model name - simply force the table for every Record. It's won't harm anything and will make your code more understandable.

#### Indexes
To create indexes in associated table, you can easily do this in your Record via propery `indexes`. Each piece of index information must be located in a sub array of this property to include index type (self::INDEX or self::UNIQUE) and column(s) index it's based on. Let's try to add a unique index to our `email` column.

```php
protected $indexes = [
    [self::UNIQUE, 'email']
];
```

> You can include as many columns into the index as your DBMS lets you. Simply list the columns as array values.

## Schema Update and Scaffolding
Once you're done declaring your `Record` model schema, you can update your ORM schema. This operation can be performed by executing the command 'spiral up'. The update will automatically generate a related table "users" for your model using the declared columns and indexes. We can view the content of this table by running CLI 'db:describe users':

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

Our table now looks exactly like it should (your output could look different if you using a different DBMS)!

> You have to execute `spiral up` each time you change any entity or ORM property for your Record to pass updates into your cached behaviour schema. There is no need to run the schema update if you're modifying any custom properties or methods.

#### Adding new Columns
Since ORM is based on DBAL syncronization mechanism, you only have to declare new columns into your model schema (same applied for indexes) to add new columns into related Record table. Let's try to add a column "balance".

```php
protected $schema = [
    'id'     => 'primary',
    'name'   => 'string(64)',
    'email'  => 'string',
    'status' => 'enum(active,blocked)',
    'balance' => 'decimal(10,2)'
];
```

> Modifications which might change application flow on existed projects is more reliable to duplicate/move to migration scripts.

To add this, we have to execute `spiral up` again (you can set the verbosity flag `-v` to see what is happening here):

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

> **Attention, dropping column from ORM schema will also drop it from realted table!** If you want to rename column which already have associated data execute classical migrations/commands before updating your schema (see [schema declaration](/database/declaration.md).

Again we can check if our table was successfully modified using the command "db:describe users". This time we switched our connection to PostgresSQL:

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

> You might notice that there is no enum type in Postgres. It will be emulated using the constrained string.

#### Active vs Passive Schemas
In the given example, we created Record which automatically ensures that table exists in the database and has the desired structure. ORM component does not require you to declare every table column. Even more so, you can associate your model with an existing table without declaring the schema property. In this case, the model will fetch the columns list from the table directly. Our model may look like this:

```php
class SomeRecord extends Record
{
    protected $table = 'some_table';
}
```

However, in a next section that is dedicated to relations, you'll find out that relations can modify the associated model schemas to declare needed columns and foreign keys. This is fine if we are generating our database using Spiral, but in some cases you might want to connect Records to an existing database without letting Spiral modify it. In this case, we can configure relations in a way to make them follow previously defined structures. However, to be absolutely sure that no relation or record will modify our table, we can declare Record constant **ACTIVE_SCHEMA** and set it's value to `false`.

```php
class SomeRecord extends Record
{
    const ACTIVE_SCHEMA = false;
    
    protected $table = 'some_table';
}
```

This constant will forbid *any modifications* to an associated table and ensure that Spiral can not touch the existing structure. The rest of the ORM functionality such as validations and relations are still preserved.

> You can use passive models if you prefer to generate your database using migrations. ORM will create a schema exception if some column is required for relation but missing from the schema. You can also use passive models to validate auto generated structure before doing a real update.

## External migration mechanisms
At this moment spiral ORM will run table migrations in house, however since SchemaBuilder, it's tables and their comparators are easily accessible it's possible to combine them with classical migrations like Phinx:

```php
$schemaBuilder = $this->orm->schemaBuilder();

//See Spiral\Support\DFSSorter to sort tables in needed order
foreach($schemaBuilder->getTables() as $table) {
    if (!$table->exists()) {
        //Table creation migration
        $this->createMigration(
            $table->getName(), 
            $table->getColumns(), 
            $table->getIndexes(), 
            $table->getReferences()
        );
    } else {
        $comparator = $table->comparator();
        
        //$comparator->addedColumns(); and etc
    }
}
```

Following technique will provide you ability to only generate migrations based on ORM schema changes and run them manually (to disable automatic sync for ORM schema check options set of `orm:schema` in console config so your `update` command wont alter schemas).

> Any help of creating such module will be appreciated. Once such module/code will be created it will be treaded as default behaviour for ORM schema syncronization.

## Create Record
Once your ORM schema has been updated, we are ready to work with our model. Since we just created our table, it's time to push some data to it. Let's do this operation in a controller:

```php
protected function indexAction()
{
    $u = new User();
    $u->name = 'Anton';
    $u->email = 'test@email.com';
    $u->status = 'active';
    $u->save();

    dump($u);
}
```

> If you working under PHPStorm, your IDE will highlight all the possible record fields for you.

You can use every `DataEntity` method to populate your model such as setFields, setField or even the set of magic methods - setName, setEmail, etc.

You can also create your entity using the static method `create`, which will respect fillable and secured fields.

```php
$u = User::create($request);
```

> Attention, the `create` method will only return a populated entity! You have to save it by it yourself!

#### Validations
That example will raise an exception, when we try to execute it again. This is because of the non unique email. Since email validation requires us to send a query to the database, we might want to modify our validation method (same way as in `DataEntity` examples) to add a more complex validation rule:

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

This validation method will check if the email is unique but only when the email field has some updates (created or changed). Now, we will get an error message associated with the model field rather than the database exception.

Before we jump to the next step, lets create a few more users. We can use the static method `create` which accepts  set of initial model fields.

Since we don't have any fillable fields, this method won't work in our case (so we are going to use the `setFields` method with a disabled access policy. We do this by setting the second argument to `true`). In order to fill our demo entities, let's connect a [faker] (https://github.com/fzaninotto/Faker) package.

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

> Pay attention, the `create` method will only return populated Record entity. You **have** to save this entity to your database manually (simply call `save()` method).

> In addition, every DataEntity, including ORM Records and ODM Documents, must be populated from the outside. This means there is no database related code in entity constuctor. If you want to create the entity based on an existing array of data (skipping setters), just pass this array as a constructor argument `new User([...])`.

To check whats happening with your queries to the database, lets check the logging tab of Spiral [Profiler] (/modules/profiler.md) (this is Postgres connection):

```sql
INSERT INTO "primary_users" ("name", "email", "status", "balance")
VALUES ('Boris Kuhic', 'mrunte@yahoo.com', 'blocked', 386.600000) RETURNING "id"
```

> All people that appear in this section are fictitious. Any resemblance to a real person, living or dead, is purely coincidencel.

## Seletions
Once you have your table populated with data, it's time to select some of them to perform something. The Spiral ORM exposes 3 ActiveRecord *like* methods, which you can use for these purposes: `find`, `findOne` and `findByPK`.

> There is a 4th method, which is used inside every find method - `ormSelector`. You can use it to get access to the Selector class associated with our record. You can also get the associated selector by calling the method `selector` of the ORM component. 

#### FindOne
To find a record based on some conditions and sorting, we can use the static method `findOne`. The method will return **null** if it's unable to locate the required record. Since the ORM component is based on DBAL, you can specify the necessary conditions in a short array form (if you want normal where statements, check the `find` method below).

```php
//Don't worry about second argument yet, it's about pre-loading
$user = User::findOne(['status' => 'active'], [], ['id' => 'DESC']);
dump($user);
```

#### FindByPK
If you would like to find your record based on it's primary key value (usually "id"), we can use the method `findByPK`. This Method will behave the same way as `findOne` if no entity is found.

```php
$user = User::findByPK(1);
dump($user);
```

The most common option is to find the model based on the provided primary key or return client exception on failure. Spiral does not embed any application specific logic into the ORM component, so you have to raise an exception on your own. This is pretty easy to do:

```php
protected function indexAction($id)
{
    if (empty($user = User::findByPK((int)$id))) {
        throw new NotFoundException('No such user');
    }

    dump($user);
}
```

Or using the entity service:

```php
protected function indexAction(UserService $users, $id)
{
    if (empty($user = $users->findByPK((int)$id))) {
        throw new NotFoundException('No such user');
    }

    dump($user);
}
```

#### Find
To find multiple Records or run aggregation, let's use the method `find`, which will return an instance of `Spiral\ORM\Entities\Selector`. This class has a common parent with DBAL [SelectQuery] (/database/builders.md) and lets you use the same methods and principles:

```php
protected function indexAction()
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

Let's check the SQL statement generated for the last selection:

```sql
SELECT
*
FROM "primary_users" AS "user"
WHERE "user"."balance" > 500
ORDER BY "user"."balance" DESC
```

Note that the ORM Selector will automatically assign the **alias for associated table**, which is a singular model name ("user"). You can learn more about aliases [here] (loading.md).

> To find one record using normal where conditions, try the variant `User::find()->where(...)->findOne()`

## Modify Record
Once you've selected the necessary record (for example using `findByPK`), you can modify it's fields using the same way as any other DataEntity. To save your updates into the database, you have to run the method `save()`. This method returns false if the model validations failed.

```php
protected function indexAction()
{
    $user = User::findByPK(1);
    $user->name = 'New Name';

    if (!$user->save()) {
        dump($user->getErrors());
    }
}
```

#### Dirty Fields and Solid State
If you check the SQL code generated for the previous example, you might notice that it looks like this:

```sql
UPDATE "primary_users"
SET "name" = 'New Name'
WHERE "id" = 1 
```

Record model added *only modified* fields into update query. This approach is useful in many cases when the data is updated from many places. However, if you want to save your entity data into the database as a solid dataset, you can force the entity "solid state". When the model (Record) is in a solid state every small update will cause a full dataset update.

```php
$user = User::findByPK(1);
$user->name = 'New Name';

if (!$user->solidState(true)->save()) {
    dump($user->getErrors());
}
```

This time our SQL will look like this:

```sql
UPDATE "primary_users"
SET "name" = 'New Name',  "email" = 'test@email.com',  "status" = 'active ',  "balance" = '0.00'
WHERE "id" = 1
```

> If you want to keep our model in solid state by default, you can call this method in an overwritten model constructor.

#### Filters and Accessors
You can define setters, getters and accessors the same way you would do for `DataEntity`. However, the spiral ORM can automatically assign some getters and accessors to your record based on column type. You can check what filters will be created by default by looking into the ORM configuration file:

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

As you can see, ORM will make sure that every column is a valid type casted into a scalar value while reading (this can be very useful since some DBMS will return the column values as strings only).

#### Timestamps Accessor
Based on the provided configuration, you may notice that the ORM will assign the `ORMTimestamp` accessor to every timestamp or datetime field. Let's try to add this field into our table (again, lets avoid using timestamps):

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

After the schema updates, we can use that field as a [Carbon] (https://github.com/briannesbitt/Carbon) instance.

```php
protected function indexAction()
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

You can also simply assign the time to a `time_registered` field. An accessor must automatically handle it using `strtotime` function:

```php
$user->time_registered = 'next friday 10am';
```

> Attentio, the DBAL will convert and fetch all dates using UTC timezone!

#### Atomic Number
Field accessors in ORM not only provide mocking functionality but declare the update behaviours. One of these accessors can be very useful to manage the numeric values in an "atomic" way. Let's see this accessor in action. First, we have to declare it in our Record model and then update the schema after:

```php
protected $accessors = [
    'balance' => AtomicNumber::class
];
```

Now we can manipulate the field values using deltas:

```php
$user = User::findByPK(1);
$user->balance->inc(1.6);

if (!$user->save()) {
    dump($user->getErrors());
}
```

> You can use the accessor method `setData` to specify the new balance value without deltas. Use `serializeData()` to get the mocked accessor value.

The whole point of solid state and such accessor is described using the resulting update statement:

```sql
UPDATE "primary_users"
SET "balance" = "balance" + 1.6
WHERE "id" = 1
```

> You can create your own accessors by implementing `RecordAccessorInterface`. Also check out the [JsonDocument] (/odm/standalone.md) accessor.

## Delete Records
You can delete any existing Record by executing it's method "delete". 

## Events and Traits
As the DataEntity model, Record declares a few [events] (/components/events.md), which you can use to track or change the model behaviour:

Event                   | Context                                   | Return | Description
---                     | ---                                       | ---    | ---
setFields               | $fields                                   | *      | Called inside `setFields` method.
publicFields            | $fields                                   | *      | Called before return in `publicFields` method.
jsonSerialize           | $publicFields                             | *      | Called while packing model into json.
validation              | -                                         | -      | Before validation.
validated               | $errors                                   | *      | After validation.
**describe** (static)   | [$property, $value, EntitySchema $schema] | value  | Called while model analysis. Such event can be used to redefine or alter schema, validations, mutators etc.
created                 | -                                         | -      | Called inside static method "create" after assigning model fields.
selector                | Selector $selector                        | *      | Called by "find" and other selection methods.
saving                  | -                                         | -      | Before model data is saved into database (model is only created). 
saved                   | -                                         | -      | After model data is saved into database.
updating                | -                                         | -      | Before model data is updated in the  database (model already exist).
updated                 | -                                         | -      | After model data is updated in the database.
deleting                | -                                         | -      | Before model is deleted from the database.
deleted                 | -                                         | -      | After model is deleted from the database.

In most cases, you don't need to handle any event as `save` and `delete` and methods can be easily overwritten. However, events can be very useful when you want to create a set of traits that are used to modify the Record schema and update/save behaviour (see next).

#### Timestamps Trait
One of the most common Record trait you might want to use is `TimestampsTrait`. This trait handles event "saving", "updating" and "describing" to alter the Record columns and update two "magic" fields "time_created" and "time_updated" (both type datetime). In our final demo, the Record User might look like this:

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
While working with the ORM models you are able to directly call `find`, `findByPK` and `save` methods inside your controllers. However, Spiral provides the ability to pre-generate specific [Service] (/application/services.md), which can help you to abstract any database specific operations from your controllers code. 

You can scaffold this class using the console command "create:service user -e user". The resulting code may look like this:

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

You can use this service inside your controllers. This avoids the static methods of the Record model (they can always be removed from service as well):

```php
protected function indexAction(UserService $users)
{
    $user = $users->findByPK(1);
    $user->balance->inc(1.6);

    if (!$users->save($user)) {
        dump($user->getErrors());
    }
}
```

Services like this are the best place to locate custom selection methods (like `findActive`) or even alter the delete method to implement "soft deletes".

> You can also generate a CRUD controller to work with the existing entity service using the command 'create:controller name -s service'.

## Inheritance and Abstract Records
Since the Spiral ORM uses static analysis, there are no real limitations for how you can create your models. As a result, you can declare the **abstract** Record with it's schema, validations etc and then later extend this class in your application models. This technique can be very useful when writing [modules] (/components/modules.md).

> When extending, the ORM will merge the schemas and other properties of the Record and it's parent. ORM does not provide table inheritance.

## Post Development Stage
Write about switching to migrations mechanism.

## Inspections
While running the shema update (spiral up) command, you may notice text that contains a list of the inspected entities and resulting rating. This information provided by the Spiral **Entity Inspector** analyses the ORM and ODM schemas to find any unprotected or blacklisted fields. We can get more details by running the inspection on selected entity:

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

> You can also check other inspector commands to receive a list of every fillable or non hidden field in your models.
