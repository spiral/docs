# Repositories and Selectors
In order to select or create specific entity you must create repository class.

## Example
To create and automatically link a repository class to your `Record` or `RecordEntity` create class extending `RecordSource` and define `RECORD` constant pointing to your record class:

```php
class UserRepository extends RecordSource
{
    const RECORD = User::class;
}
```

You can access your repository right after running `orm:schema` command:

```php
protected function indexAction(UserRepository $users)
{
    //Alternative approach 
    $users = $this->orm->source(User::class);
}
```

### Create entity
To create record entity call method `create` of your repository:

```php
$users = $users->create([
    'name' => 'Antony'
]);
```

Note, that entity values will be populated using `setFields` method, make sure that you have `FILLABLE` and `SECURED` constants property set. 

> Attention, create would only initiate blank model entity, you have to manually persist your changes to the database by calling `save()`.

### Find Entities
Use methods `findByPK`, `findOne` and `find` to load entities from database:

```php
$user = $users->findOne(['name' => 'Antony']);
```

### FindByPK
Use method `findByPK` to select entity using id:

```php
protected function indexAction(string $id, UsersRepository $users)
{
    if (empty($user = $users->findByPK((int)$id))) {
        throw new NotFoundException('No such user');
    }

    dump($user);
}
```

## RecordSelector
Method `find` of your Repository will return entity specific `RecordSelector` which can be used as QueryBuilder:

```php
$users = $users->find()->where('name', 'Antony')->paginate(10)->fetchAll();
```

### Custom find methods 
In order to simplify your domain layer, it is recommended to define custom find commands specific to your application into repository class:

```php
public function findActive(): RecordSelector
{
    return $this->find(['status' => 'active']);
}
```

You can also chain this methods inside your repository:

```php
public function findAuthors(): RecordSelector
{
    return $this->findActive()->where(['type' => 'author']);
}
```

### Overwrite repository selector
In some cases you might want to set base query for all of your find method. You can do it in your repository constructor or use public method `withSelector`:

```php
$deletedUsers = $users->withSelector(
    $users->find(['status' => 'deleted'])
);
```

> All repositories are treated as immutable.

## Static Access
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
dump(User::findOne());
```

> Please note, this method will only work in global container scope (inside your application).