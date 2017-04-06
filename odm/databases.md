# MongoDB Databases
Spiral ODM component includes DatabaseManager component which simplifies access to MongoDB and provides support for contextual injections.

## Configuration
Locate your MongoDB databases configuration in `app/config/mongo`:
 
```php
return [
    /*
    * Here you can specify name/alias for database to be treated as default in your application.
    * Such database will be returned from ODM->database(null) call and also can be
    * available using $this->db shared binding.
    */
    'default'   => 'default',

    /*
     * Set of database aliases.
     */
    'aliases'   => [
        'database' => 'default',
        'db'       => 'default',
        'mongo'    => 'default'
    ],

    /*
     * Mongo database configured with connection options.
     */
    'databases' => [
        'default' => [
            'server'   => 'mongodb://localhost:27017',
            'database' => env('MONGO_DB', 'spiral'),
            'options'  => [
                'connect' => true
            ]
        ],
        /*{{databases}}*/
    ]
];
```

Component works similar was with Database and makes you able to request database using shortcut or dependency in your code:

```php
public function indexAction(MongoDatabase $database)
{
    dump($database);
    dump($this->mongo);
}
```

ODM Databases are built at top of [mongodb library](https://github.com/mongodb/mongo-php-library):

```php
public function indexAction(MongoDatabase $database): string
{
    dump($database->selectCollection('users')->find());
}
```