
# Inheritance
In addition to aggregation and composition, the ODM engine provides the ability to store Document and it's children implementations in the same location. When class is created, ODM will automatically resolve what instance has to be used to represent such data. We can demonsrate this functionality using the following example:

```php
class Moderator extends User
{
    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'moderates' => ['string']
    ];
}
```

Now we can create our Moderator and save it in the same collection as User (you can also re-define the collection property):

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

if (!$user->save()) {
    dump($user->getErrors());
}
```

Iteration through our users and entity will be represented as `Moderator`:

```php
foreach (User::find() as $user) {
    dump($user);

    if ($user instanceof Moderator) {
        echo 'Found it!';
    }
}
```

> One of the side effects of using multiple types stored in one collection is that you have to verify the class type supplied by `find`, `findOne` and `findByPK` methods. Calling this method in Moderator can and will return an instance of User in some cases (This could be checked automatically in `findOne` and `findByPK` methods, but not in `find`).

You can also assign Document or DocumentEtity children to compositions, for example:

```php
class SuperSession extends Session
{
    protected $schema = [
        'superKey' => 'string'
    ];
}
```

And, in your code:

```php
$user = User::findOne();
dump($user->sessions->all());

$user->sessions[] = new SuperSession([
    'timeCreated' => new \MongoDate(),
    'accessToken' => 'random',
    'superKey'    => 'some value'
]);

$user->save();
```

#### How Inheritance Work
Every time Compositor or Collection tries to create `Document` or `DocumentEntity`, entity data passed to ODM component method `document`. This method will analyze the provided data and based on the shown fields, select the specific class to be used as a data representer. This method automatically adds constraints to your code. No entity can be extended without adding unique fields into it's schema.

#### Logical Class Definition
In some cases you might want to manually control which class is selected for the data array. To do this, you have to switch inheritance logic for your entity to `DEFINITION_LOGICAL` and declare constant `DEFINITION`.

```php
class User extends Document
{
    use TimestampsTrait;

    /**
     * Indication to ODM component of method to resolve Document class using it's fieldset. This
     * constant is required since Document can inherit another Document.
     */
    const DEFINITION = self::DEFINITION_LOGICAL;

    ...
```

Now, every time ODM will constucts an instance of User, it will try to find the appropriate class by calling the static method `defineClass` or the User model. Let's try to write this method in a way to emulate the default inheritance logic:

```php
public static function defineClass(array $fields, ODM $odm)
{
    if (array_key_exists('moderates', $fields)) {
        return Moderator::class;
    }

    return self::class;
}
```

> You can obviosuly use more complex logic.
