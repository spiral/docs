# Columns
You can move column type definition to external class, or even combine it with accessor `RecordAccessorInterface`. In order to do that, implement `ColumnInterface` and write column type definition method.

## ColumnInterface
```php
interface ColumnInterface
{
    /**
     * Configure column.
     *
     * @param AbstractColumn $column
     */
    public static function describeColumn(AbstractColumn $column);
}
```

## Enum Example
ORM include base type for enum columns with associated accessor - `EnumColumn`. Let's create column definition:

```php
class UserStatus extends EnumColumn
{
    const VALUES = ['active', 'blocked'];  
    const DEFAULT = 'active';
}
```

Use this column type in your model schema as normal type:

```php
class User extends RecordEntity
{
    const SCHEMA = [
        'id'     => 'primary',
        'name'   => 'string',
        'status' => UserStatus::class
    ];
}
```

You can define your own method in such columns to be used as accessors later:

```php
class UserStatus extends EnumColumn
{
    const VALUES = ['active', 'blocked'];  
    const DEFAULT = 'active';
    
    public function isBlocked(): bool
    {
        return $this->packValue() == 'blocked';
    }
    
    //...
}
```

```php
if($user->status->isBlocked()) {
    $user->status->setActive();
}
```