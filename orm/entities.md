# Record and RecordEntity
ORM engine include 2 base classes to carry your database data across application: `Record` and `RecordEnity`. When `Record` class can be described as classic AR implementation, `RecordEntity` requires external transaction or UoW to persist it's data.

> Implement `RecordInterface` if you want to use custom data carrier.

Record and RecordEntity are cross-compatible.

> If you wish to use pure DataMapper implementation take a look at Doctrine, or implement your hydration using `fetchData` method of RecordSelector.

You can read about base entity class [here](/components/data-entity.md).

## ActiveRecord
To create AR model extent base class `Record`.

Your model must include a constant **SCHEMA** which will describe the table columns and record relationships. Since the ORM component uses DBAL as it's backbone, you can declare a set of desired columns the same way as [schema declaration](/database/declaration.md).

Put your model into `app/classes/Database/User.php`:

```php
class User extends Record 
{
    const SCHEMA = [
        'id'     => 'primary',
        'name'   => 'string(64)',
        'email'  => 'string',
        'status' => 'enum(active,blocked)'
    ];
}
```

> ORM requires to have PK for every entity.

You can read how to scaffold your database models and create migrations [here](/orm/scaffolding.md).

## RecordEntity
Declaration of RecordEntity is identical to `Record`, though `save` method would not exist in your model:

```php
$user = new User();
$user->save();
```

Versus:

```php
$user = new UserEntity();
$transaction->store($user);
```

> All future descriptions will be given for AR records.

#### Schema
The most important part of any Record/RecordEntity model is it's `SCHEMA`. Schema declares what columns you would like to see in the associated table as well as defines a set of record [relations](/orm/relations.md). In this tutorial, we will stick to columns only.

Since we are creating a model from scratch (non existing table), we need to declare the table primary key. We can do this the same way as in DBAL Schema Writers by assigning the type "primary" or "bigPrimary".

```php
'id'     => 'primary',
```

>  Spiral ORM will detect your primary key name automatically so you don't have to use "id".

To provide an additional type arguments, such as length, scale or enum values, you can simply use braces as you would call `AbstractColumn` [methods](/database/declaration.md).

```php
'name'   => 'string(64)',
'status' => 'enum(active,blocked)'
```

Note that all declared columns are *NOT NULL* by default, to mark column as nullable use `nullable` or `null` flag:

```php
'column' => 'string, nullable'
```

#### Default Values
To declare custom default value use constant `DEFAULTS`:

```php
const DEFAULES = [
    'status' => 'active'    
];
```

> Using `null` as default value for column will mark it as `nullable`.

#### Associated Table and Database
By default, Spiral will associate your Record model with the default DBAL database and table whose names are generated based on the class name (in our case, User => users). Y

ou can alter both of these values by declaring constants `DATABASE` and `TABLE`.

> It's recommended to force DATABASE and TABLE constants in all of your entities.

#### Indexes
To create indexes in associated table, do this in your Record via constant `INDEXES`. Each piece of index information must be located in a sub array and include index type (self::INDEX or self::UNIQUE) and column(s) index it's based on. Let's try to add a unique index to our `email` column.

```php
const INDEXES = [
    [self::UNIQUE, 'email'],
    [self::INDEX, 'email', 'status'] //Compound index 
];
```

## Create Record
To persist data associated with your Record or RecordEntity model:

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

> Make sure to [scaffold database structure and update ORM schema first](/orm/scaffolding.md) using `spiral orm:schema -m -vv`. 

You can use every `DataEntity` method to populate your model such as setFields, setField or even the set of magic methods - setName, setEmail, etc.

> Note, method `setFields` will only populate values allowed by `FILLABLE`/`SECURED` constants.

## Modify Record
You can save modified entity calling `save` method as you would do that for creation:

```php
protected function updateAction(string $id, UsersRepository $users)
{
    if (empty($user = $users->findByPK((int)$id))) {
        throw new NotFoundException('No such user');
    }

    $user->name = 'New Name';
    $user->save();
}
```

For RecordEntity:

```php
$transaction->store($user);
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
$user->name = 'New Name';
$user->solidState(true)->save();
```

This time our SQL will look like this:

```sql
UPDATE "primary_users"
SET "name" = 'New Name',  "email" = 'test@email.com',  "status" = 'active ',  "balance" = '0.00'
WHERE "id" = 1
```

> If you want to keep our model in solid state by default, you can call this method in an overwritten model constructor.

## Delete Records
To delete AR record and all related data (FK based) call method `delete()`, or if you use entity implementation:

```php
$transaction->delete($entity);
```

## Timestamps Trait
Spiral Framework bundle include trait `TimestampsTrait` which will automatically add `time_created` and `time_updated` fields to your model and update them automatically when model is being saved in database:

```php
class User extends Record
{
    use TimestampsTrait;

    const SCHEMA = [
        'id'              => 'primary',
        'time_registered' => 'datetime',
        'name'            => 'string(64)',
        'email'           => 'string',
        'status'          => 'enum(active,blocked)',
        'balance'         => 'decimal(10,2)'
    ];

    const INDEXES = [
        [self::UNIQUE, 'email']
    ];

    const ACCESSORS = [
        'balance' => AtomicNumber::class
    ];
}
```

## Static Methods
If you wish to access record repository and selector using static functions use `Spiral\ORM\SourceTrait`.

```php
class User extends Record
{
    use SourceTrait;

    const SCHEMA = [
        'id'    => 'primary',
        'name'  => 'string',
        'email' => 'string'
    ];

    const DEFAULTS = [];
    const INDEXES = [];
}
```

```php
//Repository
dump(User::source());

//Shortcut
dump(User::findOne(['name' => $name]));
```

> Please note, this method will only work in global container scope (inside your application).