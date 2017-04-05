
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
