# Exporting ODM schema to UML
Because Spiral's ODM is based on the principles of composition and aggreation, it's easy to export it's schema into UML, which simplifies the project documentation and analyzing the database.

Let's build a schema for a given set of models:

```php
class User extends Document
{
    use TimestampsTrait;

    protected $schema = [
        '_id'          => 'MongoId',
        'name'         => 'string',
        'email'        => 'string',
        'departmentID' => 'MongoId',
        'profile'      => Profile::class,
        'sessions'     => [Session::class],
        'department'   => [self::ONE => Department::class, ['_id' => 'key::departmentID']]
    ];
}
```

```php
class Moderator extends User
{
    protected $schema = [
        'forumIDs' => ['MongoId']
    ];
}
```

```php
//To be embedded
class Session extends DocumentEntity
{
    protected $schema = [
        'timeCreated'  => 'MongoDate',
        'ipAddress'    => 'string',
        'sessionToken' => 'string'
    ];
}
```

```php
//To be embedded
class Profile extends DocumentEntity
{
    protected $schema = [
        'biography' => 'string',
        'someValue' => 'string'
    ];
}
```

```php
class Department extends Document
{
    protected $schema = [
        '_id'   => 'MongoId',
        'name'  => 'string',
        'users' => [self::MANY => User::class, ['departmentID' => 'key::_id']]
    ];
}
```

To export the UML diagram to a file, we have to run the 'odm:uml filename' command. This will create a PlantUML compatible file which we can render later. 

```uml
@startuml
class "Database\\Department" { 
  + MongoId _id
  + string name
}

"Database\\Department" ..o "Database\\User":users

class "Database\\Moderator" { 
  + MongoId _id
  + string name
  + string email
  + MongoId departmentID
  + Database\\Profile profile
  + Database\\Session[] sessions
  + timestamp timeCreated
  + timestamp timeUpdated
  + MongoId[] forumIDs
}

"Database\\Moderator" --|> "Database\\User"

class "Database\\Profile" { 
  + string biography
  + string someValue
}

class "Database\\Session" { 
  + MongoDate timeCreated
  + string ipAddress
  + string sessionToken
}

class "Database\\User" { 
  + MongoId _id
  + string name
  + string email
  + MongoId departmentID
  + Database\\Profile profile
  + Database\\Session[] sessions
  + timestamp timeCreated
  + timestamp timeUpdated
  + $this touch()
  {static} # void initTimestampsTrait(analysis)
  {static} - \Closure __timestampsSave()
  {static} - callable __timestampsSchema()
}

"Database\\User" --* "Database\\Profile":profile

"Database\\User" ..*"Database\\Session":sessions

"Database\\User" --o "Database\\Department":department

abstract class "Spiral\\ORM\\Accessors\\JsonDocument" { 
  + void defaultValue(Driver driver)
  + void serializeData()
  + void compileUpdates(field)
  + void setData(data)
}

@enduml
```

The resulting image will look like this:

![UML](https://raw.githubusercontent.com/spiral/guide/master/resources/uml.png)
