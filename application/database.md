# Database Models
Default spiral application suggest placing your DAO models into separate namespace `Database`. Both entity and repository class can be located in it.

Let's review such approach by example:

```php
namespace Database;

use Spiral\ORM\Record;

class User extends Record
{
    const SCHEMA = [
        'id'   => 'primary',
        'name' => 'string'
    ];
}
```

```php
class UserRepository extends RecordSource
{
    const RECORD = User::class;
    
    public function findByName(string $name): RecordSelector
    {
        return $this->find(compact('name'));
    }
}
```

> Consider "Source" to be equal to "Repository".

You can use repositories in your controllers and services using DI:

```php
public function countUsersAction(string $name, UserRepository $users): array
{
    return [
        'status'     => 200, 
        'countUsers' => $users->findByName($name)->count()
    ];
}
```