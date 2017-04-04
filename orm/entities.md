# Record and RecordEntity


## ActiveRecord


## RecordEntity

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