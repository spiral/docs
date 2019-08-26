# Pre-Compiled Relations, Relate to Interfaces
Due to the nature of the components that make up the ORM and the way it pre-compiles your schema, you can relate your entities
to external models using interface instead of class name.

> Attention, do not confuse this with [Morphed relations](/v1.0.0/orm/orm/morphed-relations.md)!

## Example
We can demonstrate such functionality by relating our `Post` model to an `User` using `UserInterface`:

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

Now, while updating your schema ORM will locate proper implementation and link it to our relation.

> Note that you can relate only one model type to reation, when morphed relations can point to multiple model classes.

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
