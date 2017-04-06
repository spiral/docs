# Repositories and Selectors
In order to select or create specific entity you must create repository class.

## Example
To create and automatically link repository class to your `Document` create class extending `DocumentSource` and define `DOCUMENT` constant pointing to your model:

```php
class UserRepository extends DocumentSource
{
    const DOCUMENT = User::class;
}
```

You can access your repository right after running `odm:schema` command:

```php
protected function indexAction(UserRepository $users)
{
    //Alternative approach 
    $users = $this->odm->source(User::class);
}
```

### Create entity
To create record entity call method `create` of your repository:

```php
$user = $users->create([
    'name' => 'Antony'
]);

$user->save();
```

Note, that entity values will be populated using `setFields` method, make sure that you have `FILLABLE` and `SECURED` constants property set. 

> Attention, create would only initiate blank model entity, you have to save it into database manually.

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
Method `find` of your Repository will return entity specific `DocumentSelector` with included query builder and paginator:

```php
$users = $users->find()->where([
    'name' => 'Antony'
])->paginate(10)->fetchAll();
```

### Custom find methods 
In order to simplify your domain layer, it is recommended to define custom find commands specific to your application into repository class:

```php
public function findActive(): DocumentSelector
{
    return $this->find(['status' => 'active']);
}
```

You can also chain this methods inside your repository:

```php
public function findAuthors(): DocumentSelector
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
If you wish to access record repository and selector using static functions use `Spiral\ODM\SourceTrait`.

```php
class User extends Record
{
    use SourceTrait;

    const SCHEMA = [
        '_id'   => ObjectId::class,
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

## Get Projection
To fetch only specific fields from your database use DocumentSelector method `getProjection`:

```php
$cursor = $users->find(['active' => true])->getProjection(['email']);
dump($cursor); // MongoDB\Driver\Cursor
```