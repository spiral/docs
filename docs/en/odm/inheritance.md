# Inheritance
In addition to aggregation and composition, the ODM engine provides the ability to store Document and it's children implementations in the same location. When class is created, ODM will automatically resolve what instance based on specific rule:

```php
class Moderator extends User
{
    /**
     * Entity schema.
     *
     * @var array
     */
    const SCHEMA = [
        'moderates' => ['string']
    ];
}
```

Declare Moderator class and assign it to the same collection as User (you can also re-define the collection property):

```php
$user = new Moderator();
$user->name = 'Moderator';
$user->email = 'moderator@email.com';
$user->balance = 99;
$user->moderates = ['a', 'b', 'c'];

//Required
$user->profile = new Profile([
    'biography'   => 'Some biography.',
    'facebookUID' => 4532467890
]);

$user->save();
```

Iteration through our users and entity will be represented as `Moderator`:

```php
foreach ($this->odm->source(User::class)->find() as $user) {
    dump($user);

    if ($user instanceof Moderator) {
        echo 'Found it!';
    }
}
```

Note, that due both types stored in one collection following situation is possible:
 
 
```php
foreach ($this->odm->source(Moderator::class)->find() as $user) {
    dump($user);

    if ($user instanceof User) {
        echo 'Found User!';
    }
}
```

> Use custom queries or sources to handle it.

You can also assign Document or DocumentEntity children to compositions, for example:

```php
class SuperSession extends Session
{
    const SCHEMA = [
        'superKey' => 'string'
    ];
}
```

And, in your code:

```php
$user = new User();

$user->sessions->add(new SuperSession([
    'timeCreated' => new \MongoDate(),
    'accessToken' => 'random',
    'superKey'    => 'some value'
]));

$user->save();
```

## How it works?
By default, inheritance is handled by `DocumentInstantiator` class assigned as constructor to your entities. It will look to find unique field which identify specific model implementation.
 
> Define your own instantiator implementation using `INSTANTIATOR` constant of your models.