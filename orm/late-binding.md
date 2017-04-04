# Pre-Compiled Relations, Relate to Interfaces
Due nature of ORM component and the way how it pre-compiles your schema, you can relate your entities
to external `RecordInterface` compatible modules using interfaces and roles instead of concrete implementation.

Such approach will work for any of your relation types and provides the most use in your modules.

> Attention, do not confuse with [Morphed relations](/orm/morphed.md)!

## Example
We can demonstrate such functionality by relating our `Post` model to an User using UserInterface:

```php
class User extends Record implements UserInterface
{
    //...
}
```

Where `UserInterface` can be empty or declare needed methods:

```php
interface UserInterface
{
    public function getName(): string;
}
```

```php
class Post extends Record 
{
    const SCHEMA = [
        'id'     => 'primary',
        'title'  => 'string',
        
        'author' => [
            self::BELONGS_TO   => UserInterface::class,
            self::LATE_BINDING => true
        ] 
    ];
}
```

You must always declare `LATE_BINGING` option in your relation to indicate interface or role reference.

> Other relation options are still allowed.

## Relate to roles
In some cases you might want to relate to an role instead of interface, this can be achieved using same practice:

```php
class Post extends Record 
{
    const SCHEMA = [
        'id'     => 'primary',
        'title'  => 'string',
        
        'author' => [
            self::BELONGS_TO   => 'user',
            self::LATE_BINDING => true
        ] 
    ];
}
```

## Verification
Note, that ORM will not allow you to create interface relation when such interface is implemented by
multiple classes!