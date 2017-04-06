# Accessors and Filters
To automatically typecast your value into specific format use DataEntity getters, setters and accessors.

## Filters and Accessors
ORM engine can automatically associate setter, getter or accessor to your entity fields based on database type, check `schemas/records` config to locate such mapping:

```php
'mutators' => [
    'timestamp'  => ['accessor' => Accessors\SqlTimestamp::class],
    'datetime'   => ['accessor' => Accessors\SqlTimestamp::class],
    'php:int'    => ['setter' => 'intval', 'getter' => 'intval'],
    'php:float'  => ['setter' => 'floatval', 'getter' => 'floatval'],
    'php:string' => ['setter' => 'strval'],
    'php:bool'   => ['setter' => 'boolval', 'getter' => 'boolval'],
    /*{{mutators}}*/
],
```

## Custom Filters
To associate filter with entity value use constants `SETTERS` and `GETTERS`:

```php
const SETTERS = [
    'name' => 'ucfirst'
];
```

## Timestamps Accessor
ORM includes pre-defined accessor to help you manage `datetime` fields with respect of database timezone: 

```php
const SCHEMA = [
    'id'              => 'primary',
    'time_registered' => 'datetime',
    'name'            => 'string(64)',
    'email'           => 'string',
    'status'          => 'enum(active,blocked)',
    'balance'         => 'decimal(10,2)'
];
```

```php
protected function indexAction()
{
    $user = new User();
    $user->name = 'New Name';

    $user->time_registered->setDateTime(2015, 1, 1, 12, 0, 0);
    dump($user->time_registered);

    $user->save();
}
```

You can also simply assign the time to a `time_registered` field. An accessor must automatically handle it using `strtotime` function:

```php
$user->time_registered = 'next friday 10am';
```

> DateTime will be automatically converted into DB specific timezone (UTC by default).