# Testing — Database Seeder

When you build apps that use databases, it's really important to make sure your database works right. This means
checking if it stores, changes, and gives back data the way it's supposed to. But sometimes, testing databases can be
tricky and a bit boring. You might have to write a lot of complicated commands and be very careful about how data is
added or removed.

This is where Spiral's database testing tools come in handy. They make this job much easier and faster. Spiral has a
special set of tools in a package
called [spiral-packages/database-seeder](https://github.com/spiral-packages/database-seeder). It's designed to help you
test databases without too much hassle.

## What Database Seeder tools offer

1. **Easy Testing:** With Spiral, you don't need to deal with complex commands. The tools are simple to use, which means
   your tests are easier to write and understand.

2. **Different Ways to Reset Your Database:** After you test something, you need to make your database clean again for
   the next test. Spiral has different ways to do this, like the Transaction, Migration, Refresh, and SqlFile methods.
   Each one has its own way of working, so you can choose what fits best for your test.

3. **Seeders and Factories:** These are like shortcuts to fill your database with test data. This data looks like the
   real data you would use in your app. You can quickly set up the data you need for testing with these tools.

4. **Checking Your Database:** After you do something in your database, you want to make sure it worked right. Spiral's
   tools let you check if the data is there or not, and if your database structure is correct.

Spiral's database testing tools are great for any developer, no matter how much experience you have. They help make sure
your database is doing what it should, which is really important for your app to work well.

## Installation

To install the package, run the following command:

```terminal
composer require spiral-packages/database-seeder --dev
```

> **Note**
> It's important to note that the `--dev` flag is used because the package is intended for use in development and
> testing environments.

After package install you need to register bootloader from the package.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader::class,
];
```

We also recommend you to add the following to your `composer.json` file. This allows you to separate your factories,
seeders from the rest of your application code.

```json
{
  // ...
  "autoload": {
    "psr-4": {
      "Database\\": "app/database/"
    }
  }
}
```

After these steps, you have installed the package and registered required bootloader, so you can now use the package in
your application.

## Setting up database testing environment

### Environment variables

When running unit tests in a Spiral environment, a crucial step to improve efficiency is the proper configuration of
environment variables. Specifically, setting `CYCLE_SCHEMA_CACHE` and `TOKENIZER_CACHE_TARGETS` to `true` in
your `phpunit.xml` file can significantly enhance the speed of your tests. These settings prevent unnecessary repeated
operations, such as directory scanning and rebuilding of the Cycle ORM cache for each test run.

To apply these optimizations, modify your phpunit.xml file accordingly. Here’s an example of how you might set it up:

```xml phpunit.xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit>
    <php>
        <env name="CYCLE_SCHEMA_CACHE" value="true"/>
        <env name="TOKENIZER_CACHE_TARGETS" value="true"/>
        <!-- ... other configurations ... -->
    </php>
</phpunit>
```

When you're testing your database, one key step is setting up your environment correctly. This means getting your
database ready for each test and then putting things back the way they were afterward. There are several strategies to
help you do this in Spiral. Let's look at different ways to set up and reset
your database for testing, using simple test cases as examples.

### 1. Transaction Strategy

It sets up your database for testing when you run your first test. Then, for each test, it does everything inside a
transaction – a kind of protected space. After the test is done, it undoes everything (rolls back) that happened in the
transaction. Imagine you're testing adding a new user to your database. The Transaction Strategy would add this user
inside a transaction and then, after the test, remove any trace of this new user, as if they were never added.

This is the fastest way and is great for most tests. It's especially good when you have lots of tests that don't change
the database structure but just add, change, or remove data.

**Here is an example of a test case that uses it:**

```php
declare(strict_types=1);

namespace Tests;

use Spiral\DatabaseSeeder\Database\Traits\{
    DatabaseAsserts, Helper, ShowQueries, Transactions
};

abstract class DatabaseTestCase extends TestCase
{
    use Transactions,
        Helper,
        DatabaseAsserts,
        ShowQueries;
}
```

Let's examine each trait used:

- `Transactions` trait: It manages the wrapping of each test case in a database transaction. When a test begins, it
  starts a transaction. Once the test is completed, whether it passes or fails, the transaction is rolled back. This
  ensures that each test is isolated and the database is left unchanged after the test, making the tests independent of
  each other. It also handles running database migrations before the tests, setting up the necessary database schema.
- `Helper` trait: It provides a set of helper methods for interacting with the database. It includes utility functions
  that simplify various database operations, making it easier to perform common tasks in your tests without writing
  repetitive code.
- `DatabaseAsserts` trait: It offers a range of assertions you can use to validate the state of your database,
  such as checking if a record exists, verifying the count of records, etc. It's crucial for ensuring that your database
  operations produce the expected results.
- `ShowQueries` trait: This is particularly useful for debugging. When you enable this trait, it logs all the SQL
  queries executed during the test to the terminal. This can help you understand what's happening in your database
  during tests and troubleshoot any issues that arise.

### 2. Migration Strategy

This method sets up your database using migrations (steps to prepare your database) before each test. After the test, it
rollbacks these migrations. t's slower than the Transaction Strategy but useful if you need to test changes in the
database structure itself.

**Example:** If you're testing creating a new table in your database, the Migration Strategy would create this table for
the test and then remove it afterward.

**Here is an example of a test case that uses it:**

```php
declare(strict_types=1);

namespace Tests;

use Spiral\DatabaseSeeder\Database\Traits\{
    DatabaseAsserts, DatabaseMigrations, Helper, ShowQueries
};

abstract class DatabaseTestCase extends TestCase
{
    use DatabaseMigrations,
        Helper,
        DatabaseAsserts,
        ShowQueries;
}
```

- `DatabaseMigrations` trait: It manages the running of database migrations before each test and rolling them back
  afterward. This ensures that each test is isolated and the database is left unchanged after the test, making the tests
  independent of each other. It also handles running database migrations before the tests, setting up the necessary
  database schema.

### 3. Refresh Strategy

This strategy cleans your database after each test using `Spiral\DatabaseSeeder\Database\Cleaner`. It's useful if your
database already has the structure you need, and you're just testing data changes. It's slower than the Transaction
Strategy.

**Here is an example of a test case that uses it:**

```php
declare(strict_types=1);

namespace Tests;

use Spiral\DatabaseSeeder\Database\Traits\{
    DatabaseAsserts, Helper, RefreshDatabase, ShowQueries
};

abstract class DatabaseTestCase extends TestCase
{
    use RefreshDatabase,
        Helper,
        DatabaseAsserts,
        ShowQueries;
}
```

- `RefreshDatabase` trait: It manages the cleaning of the database tables (except `migrations` table) after each test.

### 4. SqlFile Strategy

This approach uses an SQL file to set up your database with the necessary structure and data for testing. It's handy
when you have a complex setup that can be easily loaded from a file, but it's slower than the Transaction Strategy.

**Example:** If you need a specific setup with many tables and data, the SqlFile Strategy lets you load all this from a
pre-prepared SQL file.

**Here is an example of a test case that uses it:**

```php
declare(strict_types=1);

namespace Tests;

use Spiral\DatabaseSeeder\Database\Traits\{
    DatabaseAsserts, Helper, DatabaseFromSQL, ShowQueries
};

abstract class DatabaseTestCase extends TestCase
{
    use DatabaseFromSQL,
        Helper,
        DatabaseAsserts,
        ShowQueries;
   
   protected function getPrepareSQLFilePath(): string
   {
       return __DIR__ . '/database/prepare.sql';
   }

    protected function getDropSQLFilePath(): string
    {
        return __DIR__ . '/database/prepare.sql';
    }
}
```

As you can see, there are lots of ways to set up and reset your database for testing. You can choose the one that fits
your needs best. After you set up your database, you can start defining your factories and seeders.

## Defining entity factories

To define a seed factory, you should create a class that extends the `Spiral\DatabaseSeeder\Factory\AbstractFactory`
class.

Here is an example of a seed factory that defines a `User` entity:

```php
namespace Database\Factory;

use App\Entity\User;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;
use Spiral\DatabaseSeeder\Factory\FactoryInterface;

/**
 * @implements FactoryInterface<User>
 */
final class UserFactory extends AbstractFactory
{
    public function entity(): string
    {
        return User::class;
    }
    
    public function makeEntity(array $definition): User
    {
        return new User(
            username: $definition['username']
        );
    }

    public function definition(): array
    {
        return [
            'firstName' => $this->faker->firstName(),
            'lastName' => $this->faker->lastName(),
            'birthday' => \DateTimeImmutable::createFromMutable($this->faker->dateTime()),
            'comments' => CommentFactory::new()->times(3)->make(), // Can use other factories.
            // Be careful, circular dependencies are not allowed!
        ];
    }
}
```

> **Note**
> Since **v2.4.0** you can define the return type for a factory `@implements FactoryInterface<...>` annotation as in
> the example above. With this annotation in place, your IDE will now provide suggestive autocomplete options, making
> the code interaction more intuitive and error-prone.
> ![image](https://github.com/spiral-packages/database-seeder/assets/773481/2da1fcf3-9214-4188-8993-1b4eaa65ac98)

The `entity` method should return the fully qualified class name of the target entity that the factory is responsible
for creating. In some cases, the factory may use the class name returned by the `entity` method to create a new instance
of the entity without calling its constructor. Instead, it may use reflection to directly set the properties of the
entity using data from the `definition` method.

The `makeEntity` method allows you to control the process of creating an entity through its constructor. The method
takes an array of definitions as an argument, which is generated by the `definition` method.

The `definition` method is where you can define all of the properties of the entity. The method should return an array
where the keys are the property names and the values are the values that should be set for those properties. These
values can be hard-coded or generated using the [Faker](https://fakerphp.github.io/) library, which provides a wide
range of fake data generation methods such as names, addresses, phone numbers, etc. This method is responsible for
generating the definition array that will be passed to the makeEntity method to construct the entity object or used to
set properties directly.

## Using entity factory

A factory can be created by calling the `new` method on the factory class:

```php
use Database\Factory\UserFactory;

$factory = UserFactory::new();
```

This will create a new instance of the factory. It provides several useful methods for generating entities.

You can also pass an array of definition to the new method of the factory class.

```php
use Database\Factory\UserFactory;

$factory = UserFactory::new(['admin' => true]);
```

This is a useful feature when you have a common set of definitions that you want to use across multiple factories or
when you want to set a default value for a property that will be overridden in specific cases.

### Create entities

The `create` method creates an array of entities, stores them in the database and returns them for further use in the
code.

```php
$users = $factory->times(10)->create();
```

The `createOne` method creates a single entity, stores it in the database and returns it for further use in the code.

```php
$user = $factory->createOne();
```

### Make entities

The `make` method creates an array of entities and returns them for further use in code, but does not store them in
the database.

```php
$users = $factory->times(10)->make();
```

The `makeOne` method creates a single entity and returns it for further use in code, but does not store it in the
database.

```php
$user = $factory->makeOne();
```

### Entity states

The `state` method allows developers to easily define specific states for entities. It can be used to set a specific set
of properties on an entity when it is created. The returned values from the closure will replace the corresponding
values in the definition array, allowing developers to easily change the state of an entity in a specific way.

> **Note**
> It is non-destructive, it will only update the properties passed in the returned array and will not remove
> any properties from the definition array.

```php
$factory->state(fn(\Faker\Generator $faker, array $definition) => [
    'admin' => $faker->boolean(),
])->times(10)->create();
```

In addition to the `state` method, there also the `entityState` method. This method allows developers to change the
state of an entity object using the available methods of that entity. It takes a closure as an argument, which should
accept the entity as an argument and should return the modified entity. This allows developers to take full advantage of
the object-oriented nature of their entities and use the methods that are already defined on the entity to change its
state.

```php
$factory->entityState(static function(User $user) {
    return $user->markAsDeleted();
})->times(10)->create();
```

The `state` and `entityState` methods can be used inside a factory class to create additional methods for creating
entities with specific states. By creating these additional methods, developers can create a more expressive and
readable code, and make it easier to understand the intent of the code.

```php
final class UserFactory extends AbstractFactory
{
    // ....

    public function admin(): self
    {
        return $this->state(fn(\Faker\Generator $faker, array $definition) => [
            'admin' => true,
        ]);
    }
    
    public function fromCity(string $city): self
    {
        return $this->state(fn(\Faker\Generator $faker, array $definition) => [
            'city' => $city,
        ]);
    }
    
    public function deleted(): self
    {
        return $this->entityState(static function (User $user) {
            return $user->markAsDeleted();
        });
    }
    
    public function withBirthday(\DateTimeImmutable $date): self
    {
        return $this->entityState(static function (User $user) use ($date) {
            $user->birthday = $date;
            return $user;
        });
    }
}
```

And example of using these methods:

```php
$factory->admin()
    ->fromCity('New York')
    ->deleted()
    ->withBirthday(new \DateTimeImmutable('2010-01-01 00:00:00'))
    ->times(10)
    ->create();
```

Factories can be used in your feature test cases to create entities in the database without seeding. This can be
useful in situations where you want to create a specific set of test data for a feature test, or when you want to test
the behavior of your application with a specific set of data.

Here is an example of using a factory in a feature test case:

```php
final class UserServiceTest extends DatabaseTestCase
{
    // ...

    public function testDeleteUser(): void
    {
        $user = UserFactory::new()
            ->fromCity('New York')
            ->createOne();

        $this->userService->delete(
            uuid: $user->uuid,
        );

        $this->assertEntity(User::class)->where([
            'uuid' => (string)$user->uuid,
            'deleted_at' => ['!=' => null],
        ])->assertExists();
    }
}
```

## Seeding

The package provides the ability to seed the database with test data. To do this, developers can create a Seeder class
and extend it from the `Spiral\DatabaseSeeder\Seeder\AbstractSeeder` class. The Seeder class should implement the `run`
method which returns a generator with entities to store in the database.

```php
namespace Database\Seeder;

use Spiral\DatabaseSeeder\Seeder\AbstractSeeder;
use Spiral\DatabaseSeeder\Attribute\Seeder;

#[Seeder]
final class UserTableSeeder extends AbstractSeeder
{
    public function run(): \Generator
    {
        foreach (UserFactory::new()->times(100)->make() as $user) {
            yield $user;
        }
        
        foreach (UserFactory::new()->admin()->times(10)->make() as $user) {
            yield $user;
        }
        
        foreach (UserFactory::new()->admin()->deleted()->times(1)->make() as $user) {
            yield $user;
        }
    }
}
```

> **Note**
> Don't forget to add the `#[Seeder]` attribute to your seeder class. This attribute is used to register the seeder
> class with the package. If you don't add this attribute, the seeder class will not be registered and the seeder will
> not be executed.

Seeders are primarily used to fill the database with test data for your stage server, providing a consistent set of data
for the developers and stakeholders to test and use.

They are especially useful when testing complex applications that have many different entities and relationships
between them. By using seeders to populate the database with test data, you can ensure that your tests are run against a
consistent and well-defined set of data, which can help to make your tests more reliable and less prone to flakiness.

### Running seeders

Use the following command to run the seeders:

```terminal
php app.php db:seed
```

This will execute all of the seeder classes that are registered and insert the test data into the database.

## Assertions

The package provides several additional assertion methods with `Spiral\DatabaseSeeder\Database\Traits\DatabaseAsserts`
trait. These methods can be used to test the state of the database after performing some operation.

### Entity assertions

The `assertEntity` method can be used to assert the state of an entity in the database. It takes the entity class name
and provides methods for testing database entities:

#### `assertExists()`

Checks if the entity exists in the database.

```php
$this->assertEntity(User::class)->where([
    'uuid' => (string)$user->uuid,
    'deleted_at' => ['!=' => null],
])->assertExists();
```

#### `assertMissing()`

Checks that the entity is not in the database.

```php
$this->assertEntity(User::class)->where([
    'uuid' => (string)$user->uuid,
])->assertMissing();
```

#### `assertCount(int $total)`

Checks that there is a specified number of entity items in the database.

```php
$this->assertEntity(User::class)->where([
    'uuid' => (string)$user->uuid,
])->assertCount(1);
```

#### `assertEmpty()`

Checks that there are no entity items in the database.

```php
$this->assertEntity(User::class)->assertEmpty();
```

#### `where(array $where)`

Allows you to specify the conditions for the entity in the database.

```php
$this->assertEntity(User::class)->where([
    'uuid' => (string)$user->uuid,
])->...;
```

#### `count()`

Returns the number of entity items in the database.

```php
$total = $this->assertEntity(User::class)->where([
    'deleted_at' => ['!=' => null],
])->count();
```

#### `select(\Closure $closure)`

Allows you to make complex queries to the database.

```php
$this->assertEntity(User::class)->select(function(\Cycle\ORM\Select $select) {
    $select->where('deleted_at', '!=', null);
})->...;
```

#### `withoutScope`

Allows you to disable the global scope for the entity.

```php
$this->assertEntity(User::class)->withoutScope()->...;
```

#### `withScope(ScopeInterface $scope)`

Allows you to enable the global scope for the entity.

```php
$this->withScope(new SoftDeletedScope())->...;
```

### Table assertions

#### `select(\Closure $closure)`

Allows you to make complex queries to the database.

```php
$this->assertTable('users')->select(function(\Cycle\Database\Query\SelectQuery $query) {
    $query->where('deleted_at', '!=', null);
})->...;
```

#### `where(array $where)`

Allows you to specify the conditions for the table in the database.

```php
$this->assertTable('users')->where([
    'id' => 5,
])->assertCount(1);
```

#### `assertRecordExists()`

Allows you to check if record exists in the database.

```php
$this->assertTable('users')->where([
    'id' => 5,
])->assertRecordExists();
```

#### `assertCountRecords(int $total)`

Allows you to check if there is a specified number of records in the database.

```php
$this->assertTable('users')->where([
    'id' => 5,
])->assertCountRecords(2);
```

#### `countRecords()`

Returns the number of records in the database.

```php
$total = $this->assertTable('users')->where([
    'deleted_at' => ['!=' => null],
])->countRecords();
```

#### `assertEmpty()`

Allows you to check if there are no records in the database.

```php
$this->assertTable('users')->where([
    'id' => 5,
])->assertEmpty();
```

#### `assertExists()`

Allows you to check if table exists in the database.

```php
$this->assertTable('users')->assertExists();
```

#### `assertColumnExists(string $column)`

Allows you to check if column exists in the table.

```php
$this->assertTable('users')->assertColumnExists('id');
```

#### `assertColumnMissing(string $column)`

Allows you to check if column missing in the table.

```php
$this->assertTable('users')->assertColumnMissing('id');
```

#### `assertColumnSame(...)`

Asserts that a column in the table matches specified characteristics, including type, size, precision, scale, nullability, and default value. This method is particularly useful for validating the detailed schema of a table.

```php
$assertion = $this->assertTable('users');

$assertion->assertColumnSame(
     column: 'id', 
     type: 'uuid', 
     size: 64,
     ...
);
```

#### `assertIndexPresent(array $columns)`

Asserts the presence of an index on specified columns in the table. It's vital for ensuring that necessary indexes, which can affect performance and uniqueness constraints, are in place.

```php
$this->assertTable('users')->assertIndexPresent(['email']);
```

#### `assertForeignKeyPresent(array $columns)`

Checks for the existence of a foreign key related to specified table columns. It's important for validating relational integrity constraints in the database schema.

```php
$this->assertTable('users')->assertForeignKeyPresent(['org_id']);
```

#### `assertPrimaryKeyExists(string ...$columns)`

Asserts that specified columns are part of the primary key for the table. It's crucial for ensuring the correct configuration of primary keys, which are fundamental to the identification and integrity of records in a table.

```php
$this->assertTable('users')->assertPrimaryKeyExists('id');
```

## Console commands

The package provides several console commands to quickly create a factory, seeder. These commands make it easy to
quickly create the necessary classes for seeding and testing your application, without having to manually create the
files or ensure that the necessary boilerplate code is included.

### Create a factory

The `Spiral\DatabaseSeeder\Console\Command\FactoryCommand` console command is used to create a factory. The name of the
factory is passed as an argument.

For example, to create a factory named `UserFactory`, the command would be:

```terminal
php app.php create:factory UserFactory
```

### Create a seeder

The `Spiral\DatabaseSeeder\Console\Command\SeederCommand` console command is used to create a seeder. The name of the
seeder is passed as an argument.

For example, to create a seeder named `UserSeeder`, the command would be:

```terminal
php app.php create:seeder UserSeeder
```