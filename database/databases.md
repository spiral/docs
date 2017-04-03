

## Access the Database
After we make sure that Spiral can talk to our databases, we can start working with our component. The first step will be to get an instance of Database. We can do this first by getting an instance of `DatabasesInterface` or `DatabaseManager` (provider). Database component can also be retrieved using the short binding "dbal" (avaiable in application services including controllers and commands).

```php
protected function indexAction(DatabaseManager $dbal)
{
    //Default database
    dump($dbal->db());

    //Using alias default which points to primary database
    dump($dbal->db('default'));

    //Secondary
    dump($dbal->db('slave'));

    //Short binding + database name
    dump($this->dbal->db('secondary'));
}
```

### Controllable Injections
You might remember the one specific feature of [Spiral IoC container] (/framework/container.md) - controllable injections. This feature lets us resolve the dependency in constructor and method injections based on a context parameter. The Database component supports this feature and uses the parameter name to resolve the database. This simplifies our code and makes it possible to write controller actions (or init methods, or constructors) such as:

```php
protected function indexAction(Database $database, Database $primary, Database $slave)
{
    //Database is an alias for "primary"
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

> As result we can add a touch of magic to our code but make it much more readable. As you can see, aliases play a big role here because you can use them to create different variable names based on context.
